# ArgoCD App-of-Apps

GitOps configuration for a multi-environment Kubernetes platform using ArgoCD's App-of-Apps pattern. Manages Vault, External Secrets Operator, monitoring (kube-prometheus-stack + Thanos), cert-manager, external-dns, Langfuse, and a sample workload across `dev`, `uat`, and `prod` clusters.

## Architecture

```
root-apps/{dev,uat,prod}.yaml          ← manually applied once per cluster
        │
        └── apps/ (Helm chart)         ← renders all child ArgoCD Applications
                │
                ├── vault
                ├── eso (External Secrets Operator)
                ├── vault-config
                ├── eso-secretstore
                ├── monitoring (kube-prometheus-stack)
                ├── thanos
                ├── vault-monitoring
                ├── eso-monitoring
                ├── k8s-monitoring
                ├── cert-manager
                ├── cert-manager-issuers
                ├── external-dns
                ├── langfuse
                └── hello-world
```

Value resolution per child app:

```
app-default-values/<app>/values/base.yaml   ← shared defaults
environments/<env>/<app>/values.yaml        ← environment overrides
app-versions-<env>.yaml                     ← image tags
```

## Sync Wave Order

Sync waves enforce deployment dependencies:

| Wave | Applications |
|------|-------------|
| 0 | `vault`, `eso`, `monitoring`, `cert-manager`, `external-dns` |
| 1 | `vault-config`, `thanos`, `cert-manager-issuers` |
| 2 | `eso-secretstore` |
| 3 | `vault-monitoring`, `eso-monitoring`, `k8s-monitoring`, `langfuse` |
| 4 | `hello-world` |

`monitoring` must be at wave 0 so the Prometheus Operator CRDs (PrometheusRule, ServiceMonitor, PodMonitor) exist before the wave-3 monitoring apps apply them. `cert-manager` must be at wave 0 so its CRDs and webhook are ready before `cert-manager-issuers` (wave 1) applies ClusterIssuer resources.

## Applications

| App | Chart | Namespace | Description |
|-----|-------|-----------|-------------|
| vault | hashicorp/vault 0.30.1 | vault | HA Vault with Raft, AWS KMS auto-unseal, IRSA |
| eso | external-secrets 0.9.13 | external-secrets | Syncs Vault secrets to Kubernetes Secrets |
| vault-config | kustomize (local) | vault | Post-sync Job: init, auth, policies |
| eso-secretstore | kustomize (local) | external-secrets | ClusterSecretStore pointing to Vault |
| monitoring | kube-prometheus-stack 67.5.0 | monitoring | Prometheus + Grafana + AlertManager |
| thanos | bitnami/thanos 15.7.0 | monitoring | Long-term metrics storage via S3 |
| vault-monitoring | kustomize (local) | vault | PodMonitor + PrometheusRules + Grafana dashboard |
| eso-monitoring | kustomize (local) | external-secrets | ServiceMonitor + PrometheusRules + Grafana dashboard |
| k8s-monitoring | kustomize (local) | monitoring | Kubernetes + Prometheus self-monitoring rules, Grafana dashboard, AlertManager ExternalSecret |
| cert-manager | jetstack/cert-manager v1.14.5 | cert-manager | Automated TLS via Let's Encrypt DNS-01 + Route53 IRSA |
| cert-manager-issuers | kustomize (local) | cert-manager | Per-env ClusterIssuers (letsencrypt-staging, letsencrypt-prod) |
| external-dns | kubernetes-sigs/external-dns 1.14.4 | external-dns | Automatic Route53 A/CNAME records from Ingress/Service hostnames |
| langfuse | langfuse 1.3.1 | langfuse | LLM observability platform |
| hello-world | local Helm chart | hello-world | Sample workload with Vault secret injection |

## Repository Layout

```
argocd/
├── root-apps/              # One Application per environment (apply these once)
│   ├── dev.yaml
│   ├── uat.yaml
│   └── prod.yaml
├── apps/                   # Helm chart that renders child Applications
│   ├── Chart.yaml
│   ├── values.yaml         # App registry: enabled flag, sync wave, chart version, namespace
│   └── templates/          # One Application template per child app
├── app-default-values/     # Base Helm values for each app
│   ├── vault/
│   ├── eso/
│   ├── monitoring/         # kube-prometheus-stack base values
│   ├── thanos/             # Thanos base values
│   ├── cert-manager/       # cert-manager base values + per-env ClusterIssuers
│   ├── external-dns/       # external-dns base values
│   └── langfuse/
├── environments/           # Per-environment overrides
│   ├── dev/
│   ├── uat/
│   └── prod/
└── app-versions-{dev,uat,prod}.yaml   # Image tags per environment
```

## Bootstrap

### Prerequisites

- ArgoCD running in the target cluster
- The cluster registered in ArgoCD with a name matching the environment (`dev`, `uat`, or `prod`)
- For Vault: AWS KMS key created and IAM roles for auto-unseal and IRSA
- For monitoring + Thanos: S3 buckets and IAM roles (see [Monitoring README](app-default-values/monitoring/README.md))
- For AlertManager: Slack incoming webhook URL and (prod only) PagerDuty Events API v2 routing key
- For cert-manager: IAM roles `cert-manager-<env>` with Route53 permissions (see [cert-manager README](app-default-values/cert-manager/README.md))
- For external-dns: IAM roles `external-dns-<env>` with Route53 permissions (see [external-dns README](app-default-values/external-dns/README.md))

### 1. Fill in placeholders

Replace all `<PLACEHOLDER>` values in environment files before committing:

| Placeholder | Where | Description |
|-------------|-------|-------------|
| `<YOUR_DOMAIN>` | All `environments/*/` values files | Base domain for ingress hosts |
| `<DEV/UAT/PROD_AWS_ACCOUNT_ID>` | All env values files | AWS account ID per environment |
| `<AWS_REGION>` | monitoring + thanos + external-dns env values | AWS region |
| `<DEV/UAT/PROD_KMS_KEY_ARN>` | vault env values | KMS key for Vault auto-unseal |
| `<GRAFANA_ADMIN_PASSWORD>` | monitoring env values | Grafana admin password |
| `<ACME_EMAIL>` | cert-manager ClusterIssuer files | Email for Let's Encrypt notifications |
| `<ROUTE53_HOSTED_ZONE_ID>` | cert-manager ClusterIssuer files | Route53 hosted zone ID (e.g. `Z1234ABCD`) |

### 2. Populate AlertManager secrets in Vault

The `k8s-monitoring` app deploys an ESO `ExternalSecret` that reads AlertManager credentials from Vault and creates the Kubernetes Secret `alertmanager-credentials` in the `monitoring` namespace. Populate this path in each environment's Vault **before** syncing:

```bash
# Run against each environment's Vault instance
vault kv put secret/monitoring/alertmanager \
  slack_webhook_url="https://hooks.slack.com/services/T.../B.../..." \
  pagerduty_routing_key="<ROUTING_KEY>"   # use any non-empty dummy for dev/uat
```

| Environment | Slack channel | PagerDuty routing key |
|-------------|--------------|----------------------|
| dev | `#dev-alerts` | dummy value |
| uat | `#staging-alerts` | dummy value |
| prod | `#prod-alerts` | real Events API v2 key |

### 3. Apply the root Application

```bash
# dev cluster
kubectl apply -f root-apps/dev.yaml

# uat cluster
kubectl apply -f root-apps/uat.yaml

# prod cluster
kubectl apply -f root-apps/prod.yaml
```

ArgoCD will create all child Applications and sync them in wave order.

### 4. Post-sync: Vault bootstrap

After wave-0 sync, create the bootstrap secret so the vault-config Job can run:

```bash
kubectl create secret generic vault-bootstrap-token \
  --from-literal=root-token=<VAULT_ROOT_TOKEN> \
  -n vault
```

## Enabling / Disabling Apps

Each app has an `enabled` flag in `apps/values.yaml`. To disable an app for all environments:

```yaml
apps:
  helloWorld:
    enabled: false
```

To disable only in one environment, override in `environments/<env>/values.yaml`:

```yaml
apps:
  helloWorld:
    enabled: false
```

## Upgrading a Chart

Update the `chartVersion` in `apps/values.yaml` (applies to all environments) or override it in the environment's `values.yaml`.

```yaml
apps:
  monitoring:
    chartVersion: "68.0.0"
```
