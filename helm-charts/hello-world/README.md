# hello-world

A Helm chart for a Python app that demonstrates end-to-end ESO + Vault secret injection. Intended as a reference pattern for onboarding real applications to the `vault-eso` setup.

---

## How it works

```
Vault KV v2 (secret/hello-world)
    │  db_password, api_key, secret_message
    │
    │  ESO reads via Kubernetes auth
    │  ClusterSecretStore: vault-backend
    │  Refresh interval: configurable per env (default 1h)
    ▼
Kubernetes Secret (<release>-hello-world-secrets)  ← created and owned by ESO
    │
    │  Injected by Kubernetes as:
    │    • env vars   DB_PASSWORD, API_KEY, SECRET_MESSAGE
    │    • files      /vault-secrets/<key>
    ▼
hello-world pod
```

The app never talks to Vault directly. It reads standard environment variables. The entire secret pipeline is declared in `templates/externalsecret.yaml`.

---

## Prerequisites

The following must already be deployed before this chart is installed:

| Dependency | ArgoCD Application | What it provides |
|---|---|---|
| Vault (HA, Raft) | `vault-<env>` | KV v2 secrets engine at `secret/` |
| ESO | `eso-<env>` | `ExternalSecret` CRD controller |
| ClusterSecretStore | `eso-secretstore-<env>` | `vault-backend` store that authenticates to Vault |

This chart is deployed at sync wave `4` in ArgoCD — after all of the above (waves 0–3).

---

## Chart structure

```
hello-world/
├── Chart.yaml
├── values.yaml             # Base values — image, resources, vault config
├── values-dev.yaml         # Dev overrides — image tag, ingress host
├── values-uat.yaml
├── values-prod.yaml
└── templates/
    ├── _helpers.tpl         # Name/label helpers
    ├── serviceaccount.yaml
    ├── externalsecret.yaml  # Pulls secrets from Vault via ESO
    ├── deployment.yaml      # Consumes the k8s Secret as env vars + volume
    ├── service.yaml
    ├── ingress.yaml
    └── NOTES.txt            # Post-install instructions printed by helm
```

---

## Step 1 — Populate secrets in Vault (one-off per environment)

Before deploying the chart, the secrets must exist in Vault. Run this once per cluster:

```bash
kubectl exec -it vault-0 -n vault -- vault kv put secret/hello-world \
  db_password="supersecret" \
  api_key="my-api-key-123" \
  secret_message="Hello from HashiCorp Vault!"
```

To update a secret later:

```bash
kubectl exec -it vault-0 -n vault -- vault kv patch secret/hello-world \
  secret_message="Updated message!"
```

ESO will pick up the new value within the configured `refreshInterval` (default `1h`, `5m` in dev).

---

## Step 2 — Deploy via ArgoCD

Apply the ArgoCD Application for the target environment:

```bash
# Dev
kubectl apply -f argocd/dev/hello-world.yaml

# UAT
kubectl apply -f argocd/uat/hello-world.yaml

# Prod
kubectl apply -f argocd/prod/hello-world.yaml
```

ArgoCD will render the chart using `values.yaml` + the matching `values-<env>.yaml` and sync it to the `hello-world` namespace on the target cluster.

---

## Step 3 — Verify the secret pipeline

```bash
# 1. Check ExternalSecret is Ready
kubectl get externalsecret -n hello-world
# Expected: READY=True, STATUS=SecretSynced

# 2. Inspect the Kubernetes Secret created by ESO
kubectl get secret -n hello-world
kubectl describe secret hello-world-hello-world-secrets -n hello-world

# 3. Decode a value to confirm Vault content reached the pod
kubectl get secret hello-world-hello-world-secrets -n hello-world \
  -o jsonpath='{.data.secret-message}' | base64 -d && echo

# 4. Check the pod is running
kubectl get pods -n hello-world

# 5. Confirm env vars are injected in the running pod
kubectl exec -it deploy/hello-world-hello-world -n hello-world \
  -- env | grep -E 'DB_PASSWORD|API_KEY|SECRET_MESSAGE'
```

---

## Step 4 — Access the app

**With ingress enabled** (UAT / prod / dev with `ingress.enabled: true`):

```
https://hello-world.<env>.<YOUR_DOMAIN>
```

**Without ingress** (local testing):

```bash
kubectl port-forward svc/hello-world-hello-world 8080:80 -n hello-world
# Open http://localhost:8080
```

---

## Configuration reference

### Core values (`values.yaml`)

| Key | Default | Description |
|---|---|---|
| `replicaCount` | `1` | Number of pod replicas |
| `image.repository` | `your-registry/hello-world` | Container image (replace before deploying) |
| `image.tag` | `latest` | Image tag |
| `service.port` | `80` | Service port |
| `service.targetPort` | `8080` | Container port |
| `ingress.enabled` | `false` | Enable NGINX ingress |
| `ingress.host` | `hello-world.example.com` | Ingress hostname |

### Vault / ESO values

| Key | Default | Description |
|---|---|---|
| `vault.secretPath` | `hello-world` | KV v2 path relative to `secret/` mount |
| `vault.secretStoreRef.name` | `vault-backend` | ClusterSecretStore name |
| `vault.secretStoreRef.kind` | `ClusterSecretStore` | Store kind |
| `vault.refreshInterval` | `1h` | How often ESO re-reads Vault |
| `vault.secrets[].vaultKey` | — | Key name inside the Vault KV secret |
| `vault.secrets[].secretKey` | — | Key name in the resulting Kubernetes Secret |
| `vault.secrets[].envVar` | — | Environment variable injected into the pod |

Adding a new secret from Vault requires only a values change — no template edits:

```yaml
# values-dev.yaml
vault:
  secrets:
    - vaultKey: db_password
      secretKey: db-password
      envVar: DB_PASSWORD
    - vaultKey: api_key
      secretKey: api-key
      envVar: API_KEY
    - vaultKey: secret_message
      secretKey: secret-message
      envVar: SECRET_MESSAGE
    # add new secrets here
    - vaultKey: stripe_key
      secretKey: stripe-key
      envVar: STRIPE_KEY
```

---

## Key design decisions

### ExternalSecret — explicit `data` vs `dataFrom`

The chart defaults to explicit `data` mappings, which are more transparent — you can see exactly which Vault keys the app needs:

```yaml
# templates/externalsecret.yaml
data:
  - secretKey: db-password
    remoteRef:
      key: hello-world
      property: db_password
```

`dataFrom` (commented out in the template) pulls **all keys** from a Vault path at once, which is simpler but less explicit:

```yaml
# dataFrom alternative — uncomment in externalsecret.yaml
dataFrom:
  - extract:
      key: hello-world
```

### Init container — blocks startup until ESO has synced

The Deployment has an init container that waits for the Kubernetes Secret to be populated before the app container starts. Without this, the pod crash-loops on first deploy if ESO hasn't synced yet:

```yaml
initContainers:
  - name: wait-for-secret
    image: busybox:1.36
    command: ["/bin/sh", "-c", "until [ $(wc -c < /vault-secrets/secret-message) -gt 0 ]; do sleep 3; done"]
```

### Secrets as env vars AND volume

Both injection patterns are wired up simultaneously so you can choose whichever the app prefers:

- **Env vars** — `DB_PASSWORD`, `API_KEY`, `SECRET_MESSAGE` (12-factor style)
- **Volume** — `/vault-secrets/db-password`, `/vault-secrets/api-key`, etc. (useful for TLS certs or config files)

### `creationPolicy: Owner`

The ExternalSecret uses `creationPolicy: Owner`, meaning ESO creates and owns the Kubernetes Secret. Deleting the ExternalSecret (or the Helm release) also deletes the Secret. Use `creationPolicy: Merge` if other controllers also write keys to the same Secret.

---

## Troubleshooting

**ExternalSecret stuck in `SecretSyncedError`**

```bash
kubectl describe externalsecret hello-world-hello-world -n hello-world
```

Common causes:
- `vault-backend` ClusterSecretStore is not Ready — check `kubectl get clustersecretstore vault-backend`
- The Vault path does not exist — run the `vault kv put` command in Step 1
- ESO's Kubernetes auth role does not have read permission on the path — check `external-secrets-policy` in Vault

**Pod stuck in `Init:0/1`**

The init container is waiting for the Secret to be populated by ESO. Check the ExternalSecret status above.

**Secrets not refreshing after a Vault update**

ESO refreshes on the `refreshInterval`. Force an immediate refresh:

```bash
kubectl annotate externalsecret hello-world-hello-world \
  force-sync=$(date +%s) -n hello-world --overwrite
```
