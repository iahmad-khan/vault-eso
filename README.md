# vault-eso

Deploys HashiCorp Vault and External Secrets Operator (ESO) across `dev`, `uat`, and `prod` environments using Helm charts managed by ArgoCD. 

---

## Architecture

```
vault-eso/
├── vault/
│   ├── values/
│   │   ├── base.yaml          # Shared Helm values (image, HA, storage, resources)
│   │   ├── dev.yaml           # Dev overrides (KMS key, domain, IRSA role)
│   │   ├── uat.yaml
│   │   └── prod.yaml
│   ├── manifests/
│   │   ├── base/              # Kustomize base: RBAC + PostSync config job
│   │   ├── dev/               # Kustomize overlay: includes base + dev UI ingress
│   │   ├── uat/
│   │   └── prod/
│   └── monitoring/
│       ├── podmonitor.yaml    # Prometheus PodMonitor for all vault server pods
│       ├── prometheusrule.yaml# Alerting rules (sealed, no leader, audit failures, …)
│       └── grafana-dashboard.yaml  # ConfigMap with Vault Grafana dashboard
├── eso/
│   ├── values/
│   │   ├── base.yaml          # Shared ESO Helm values
│   │   ├── dev.yaml           # Dev IRSA role annotation
│   │   ├── uat.yaml
│   │   └── prod.yaml
│   ├── secretstore/
│   │   ├── dev.yaml           # ClusterSecretStore → Vault (Kubernetes auth)
│   │   ├── uat.yaml
│   │   └── prod.yaml
│   └── monitoring/
│       ├── servicemonitor.yaml # Prometheus ServiceMonitor for ESO metrics
│       ├── prometheusrule.yaml # Alerting rules (sync errors, queue depth, …)
│       └── grafana-dashboard.yaml  # ConfigMap with ESO Grafana dashboard
└── argocd/
    ├── dev/                   # 6 ArgoCD Applications for dev
    ├── uat/                   # 6 ArgoCD Applications for uat
    └── prod/                  # 6 ArgoCD Applications for prod
```

### ArgoCD Applications per environment

| Application | What it deploys | Sync wave |
|---|---|---|
| `vault-<env>` | Vault Helm chart (HA, Raft, KMS unseal) | 0 |
| `vault-config-<env>` | RBAC, UI ingress, PostSync config job | 1 |
| `eso-<env>` | External Secrets Operator Helm chart | 0 |
| `eso-secretstore-<env>` | `ClusterSecretStore` pointing to Vault | 2 |
| `vault-monitoring-<env>` | PodMonitor, PrometheusRule, Grafana dashboard (Vault) | 3 |
| `eso-monitoring-<env>` | ServiceMonitor, PrometheusRule, Grafana dashboard (ESO) | 3 |

---

## Key Settings

| Setting | Value |
|---|---|
| Vault image | `hashicorp/vault:1.20.1` |
| Vault Helm chart | `hashicorp/vault 0.30.1` |
| HA mode | 3-node Raft cluster |
| Auto-unseal | AWS KMS (per-environment key) |
| TLS | Disabled — HTTP on 8200, TLS terminated at NGINX ingress |
| Vault agent injector | Disabled |
| Data storage | 20Gi gp3 PVC per pod |
| Audit storage | 30Gi gp3 PVC per pod |
| ESO chart | `external-secrets/external-secrets 0.9.13` |
| ESO auth to Vault | Kubernetes auth method, `external-secrets-role`, 24h TTL |

---

## Prerequisites

- ArgoCD ≥ 2.6 (multi-source Application support)
- NGINX Ingress Controller installed on each cluster
- `gp3` StorageClass available on each cluster (standard on EKS with the EBS CSI driver)
- Each target cluster registered in ArgoCD with names `dev`, `uat`, `prod`
- An AWS KMS key per environment with a policy allowing the Vault pod's IAM role to use it
- An IAM role per environment for Vault with the following KMS permissions:
  ```json
  {
    "Effect": "Allow",
    "Action": ["kms:Encrypt","kms:Decrypt","kms:DescribeKey"],
    "Resource": "<KMS_KEY_ARN>"
  }
  ```
- IRSA (IAM Roles for Service Accounts) configured on each EKS cluster

---

## Configuration — fill in placeholders

Before applying, replace all placeholder values in the env-specific files:

### `vault/values/<env>.yaml`

| Placeholder | Description |
|---|---|
| `<DEV/UAT/PROD_AWS_ACCOUNT_ID>` | AWS account ID for that environment |
| `<DEV/UAT/PROD_KMS_KEY_ARN>` | Full ARN of the KMS key, e.g. `arn:aws:kms:us-east-1:123456789012:key/xxxx-xxxx` |
| `vault-server.<env>.<YOUR_DOMAIN>` | Hostname for the Vault API ingress |

### `eso/values/<env>.yaml`

| Placeholder | Description |
|---|---|
| `<DEV/UAT/PROD_AWS_ACCOUNT_ID>` | AWS account ID for that environment |

### `eso/secretstore/<env>.yaml`

No placeholders — the `ClusterSecretStore` connects to Vault via the in-cluster DNS name `vault-active.vault.svc.cluster.local`, which works once Vault is running.

### `argocd/<env>/*.yaml`

The `destination.name` field in each Application must match the cluster name registered in ArgoCD:
```yaml
destination:
  name: dev   # must match: argocd cluster list
```

---

## Deployment

### 1. Fill in all placeholder values

Edit the four files for each environment you are deploying:
- `vault/values/<env>.yaml`
- `eso/values/<env>.yaml`
- Each ArgoCD Application's `destination.name`

### 2. First-time Vault initialisation (once per cluster)

Vault must be manually initialised on first boot. ArgoCD deploys the pods, but the init step generates the root token and recovery keys and must be done by an operator.

```bash
# Wait for vault pods to be running
kubectl get pods -n vault -w

# Initialise (AWS KMS auto-unseal uses recovery shares, not unseal keys)
kubectl exec -it vault-0 -n vault -- vault operator init \
  -recovery-shares=5 \
  -recovery-threshold=3
```

**Save the output securely** (recovery keys + root token). Then store the root token so the config job can use it:

```bash
kubectl create secret generic vault-bootstrap-token -n vault \
  --from-literal=root-token=<ROOT_TOKEN_FROM_INIT_OUTPUT>
```

> After this one-time step, AWS KMS handles unsealing automatically on every pod restart.

### 3. Apply the ArgoCD Applications

Apply all four Application manifests for the target environment:

```bash
kubectl apply -f argocd/dev/
# or
kubectl apply -f argocd/uat/
kubectl apply -f argocd/prod/
```

ArgoCD will:
1. Deploy the Vault Helm release (wave 0)
2. Apply RBAC + UI ingress + trigger the PostSync config job (wave 1)
3. Deploy ESO (wave 0, parallel with Vault)
4. Apply the `ClusterSecretStore` after ESO is ready (wave 2)

### 4. Verify

```bash
# Vault status
kubectl exec -it vault-0 -n vault -- vault status

# Config job logs
kubectl logs -n vault -l app=vault-configure

# ESO SecretStore health
kubectl get clustersecretstore vault-backend
```

---

## What the PostSync config job does

The `vault-config` Application triggers a Kubernetes `Job` after every sync. It is idempotent and safe to re-run.

1. Waits for `vault-active` to be reachable
2. Reads the root token from the `vault-bootstrap-token` secret
3. Enables KV v2 secrets engine at path `secret/`
4. Enables file audit logging to `/vault/logs/vault_audit.log` with `log_raw=true`
5. Enables and configures the Kubernetes auth method
6. Creates the `external-secrets-policy` (read access to `secret/data/*`)
7. Creates the `external-secrets-role` bound to the `external-secrets` service account with a 24h token TTL

---

## Using secrets in your applications

Once deployed, create an `ExternalSecret` in any namespace to pull a secret from Vault:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secret
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: my-app-secret
  data:
    - secretKey: db-password
      remoteRef:
        key: secret/my-app
        property: db-password
```

Secrets must first be written to Vault at the `secret/` path:

```bash
kubectl exec -it vault-0 -n vault -- vault kv put secret/my-app db-password=supersecret
```

---

## Ingress hostnames

| Environment | API (server) | UI |
|---|---|---|
| dev | `vault-server.dev.<YOUR_DOMAIN>` | `vault.dev.<YOUR_DOMAIN>` |
| uat | `vault-server.uat.<YOUR_DOMAIN>` | `vault.uat.<YOUR_DOMAIN>` |
| prod | `vault-server.prod.<YOUR_DOMAIN>` | `vault.prod.<YOUR_DOMAIN>` |

Both use `ingressClassName: nginx` and terminate TLS at the ingress layer (Vault itself runs HTTP).

---

## Hello World demo app (`hello-world/`)

A Helm chart for a Python app that demonstrates end-to-end ESO + Vault secret injection. It is intended as a reference pattern for onboarding real applications.

### Chart structure

```
hello-world/
├── Chart.yaml
├── values.yaml           # Base values (image, resources, vault config)
├── values-dev.yaml       # Dev overrides (image tag, ingress host)
├── values-uat.yaml
├── values-prod.yaml
└── templates/
    ├── _helpers.tpl
    ├── serviceaccount.yaml
    ├── externalsecret.yaml   # ← pulls secrets from Vault via ESO
    ├── deployment.yaml       # ← consumes the k8s Secret as env vars + volume
    ├── service.yaml
    ├── ingress.yaml
    └── NOTES.txt
```

### Secret flow

```
Vault KV v2 (secret/hello-world)
    │  db_password, api_key, secret_message
    │
    │  ESO reads via Kubernetes auth
    │  ClusterSecretStore: vault-backend
    │  Refresh interval: configurable per env
    ▼
Kubernetes Secret (hello-world-secrets)  ← created and owned by ESO
    │
    │  Injected by Kubernetes as:
    │    • env vars  (DB_PASSWORD, API_KEY, SECRET_MESSAGE)
    │    • files     (/vault-secrets/<key>)
    ▼
hello-world pod
```

### One-off Vault setup

Before the first deploy, populate the secrets in Vault:

```bash
kubectl exec -it vault-0 -n vault -- vault kv put secret/hello-world \
  db_password="supersecret" \
  api_key="my-api-key-123" \
  secret_message="Hello from HashiCorp Vault!"
```

### ArgoCD Application

The `hello-world-<env>` Application (sync wave 4 — after ESO and the SecretStore are ready) deploys the chart from this repo using the matching env values file:

```bash
kubectl apply -f argocd/dev/hello-world.yaml
```

### Key design points in the chart

| Feature | Where |
|---|---|
| ExternalSecret with explicit key mapping | [externalsecret.yaml](hello-world/templates/externalsecret.yaml) |
| `dataFrom` alternative (commented out) | [externalsecret.yaml](hello-world/templates/externalsecret.yaml) |
| Init container waits for ESO sync before app starts | [deployment.yaml](hello-world/templates/deployment.yaml) |
| Secrets as env vars AND as `/vault-secrets/` volume | [deployment.yaml](hello-world/templates/deployment.yaml) |
| `creationPolicy: Owner` — ESO owns the Secret lifecycle | [externalsecret.yaml](hello-world/templates/externalsecret.yaml) |

---

## Monitoring

Monitoring resources are deployed by the `vault-monitoring-<env>` and `eso-monitoring-<env>` ArgoCD Applications (sync wave 3 — after everything else is up).

### How it integrates with your kube-prometheus-stack

Your existing Prometheus Operator is configured with empty (`{}`) selectors, meaning it auto-discovers resources from **all namespaces** without requiring any special labels. The monitoring resources in this folder are placed in the same namespaces as the workloads (`vault` and `external-secrets`) and will be picked up automatically.

Grafana's sidecar is configured with `searchNamespace: ALL` and label `grafana_dashboard: "1"`, so both dashboard ConfigMaps are auto-provisioned without any Grafana restart.

### Vault monitoring (`vault/monitoring/`)

**PodMonitor** — scrapes every vault server pod individually at `http://<pod>:8200/v1/sys/metrics?format=prometheus`. Scrapes all 3 HA pods so per-pod metrics (sealed status, Raft role, memory, etc.) are visible separately.

Vault must have `unauthenticated_metrics_access = "true"` in the telemetry block — this is already set in all env values files under `ha.raft.config`.

**PrometheusRule — alerts:**

| Alert | Severity | Condition |
|---|---|---|
| `VaultDown` | critical | Prometheus cannot scrape a pod for 5m |
| `VaultSealed` | critical | Any pod reports `vault_core_unsealed == 0` for 1m |
| `VaultNoActivePod` | critical | No pod is active for 2m |
| `VaultRaftPeerCountLow` | warning | Raft peers < 3 for 5m |
| `VaultRaftHighCommitLatency` | warning | p99 commit time > 500ms for 5m |
| `VaultAuditLogFailure` | critical | Audit write failures detected (immediate) |
| `VaultHighLeaseCount` | warning | Active leases > 10,000 for 15m |
| `VaultLeadershipLoss` | warning | Leader changed > 2 times in 10m |

**Grafana dashboard** (`uid: vault-monitoring`) — 12 panels:
- Stat row: Active nodes, Sealed nodes, Raft peers, Active leases, Token count, Audit failures
- Time series: Heap memory, Goroutines, Raft commit latency (p50/p99), Barrier ops/s, Token count by type, Lease count over time

### ESO monitoring (`eso/monitoring/`)

**ServiceMonitor** — scrapes the `external-secrets` service on port `metrics` (8080) at `/metrics`.

**PrometheusRule — alerts:**

| Alert | Severity | Condition |
|---|---|---|
| `ESODown` | critical | Prometheus cannot scrape ESO for 5m |
| `ESOSyncErrors` | warning | ExternalSecret reconcile errors for 15m |
| `ESOClusterSecretStoreReconcileErrors` | critical | ClusterSecretStore reconcile errors for 5m |
| `ESOHighReconcileErrorRate` | warning | Error rate > 10% for 10m |
| `ESOWorkQueueGrowing` | warning | Work queue depth > 50 for 10m |

**Grafana dashboard** (`uid: eso-monitoring`) — 8 panels:
- Stat row: Reconcile success rate, Reconcile errors, Active workers, Work queue depth
- Time series: Reconcile rate by controller, Reconcile duration (p50/p95/p99), Queue depth over time, Queue retries/s

### Notes on metric names

Vault uses **go-metrics** with a Prometheus sink. Summary metrics like `vault_raft_commitTime` are exported with `{quantile="0.5/0.99"}` labels. Counter-like metrics (e.g. `vault_audit_log_response_failure`) are monotonically increasing values — `increase()` works correctly.

If any dashboard panel shows "No data", check the exact metric names in your environment:
```bash
kubectl port-forward -n vault vault-0 8200
curl -s http://localhost:8200/v1/sys/metrics?format=prometheus | grep vault_raft
```
