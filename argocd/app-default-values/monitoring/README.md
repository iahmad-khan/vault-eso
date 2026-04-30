# Monitoring Stack (kube-prometheus-stack)

Deploys the full Prometheus monitoring stack via the `prometheus-community/kube-prometheus-stack` Helm chart into the `monitoring` namespace. Synced at **wave 0** so Prometheus Operator CRDs are available before the wave-3 monitoring manifests for Vault and ESO.

## Components

| Component | Description |
|-----------|-------------|
| Prometheus Operator | Manages Prometheus/AlertManager instances, watches CRDs |
| Prometheus | Metrics scraper, 2h local retention (Thanos handles long-term) |
| Thanos Sidecar | Uploads TSDB blocks to S3 and exposes gRPC endpoint for Thanos Query |
| Grafana | Dashboards UI, auto-imports dashboards from labeled ConfigMaps |
| AlertManager | Alert routing and grouping |
| kube-state-metrics | Exposes Kubernetes object metrics |
| node-exporter | Per-node OS/hardware metrics (DaemonSet) |

## Prometheus Retention Strategy

Local retention is intentionally short (2h). The Thanos sidecar continuously ships 2-hour TSDB blocks to S3. Thanos Store Gateway serves historical queries directly from S3, so no data is lost when Prometheus storage rolls over.

```
Prometheus (2h window) ──sidecar──► S3 bucket (thanos-<env>)
                                          │
Thanos Query ◄──────────────────────────┘  (via Store Gateway)
     │
Grafana (Thanos datasource)
```

## Grafana Dashboards

Grafana runs with a sidecar that watches for ConfigMaps labeled `grafana_dashboard: "1"` across all namespaces. The existing Vault and ESO dashboards are auto-imported — no manual provisioning needed.

To add a new dashboard, create a ConfigMap with that label in any namespace:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-dashboard
  namespace: my-app
  labels:
    grafana_dashboard: "1"
data:
  my-app.json: |
    { ... Grafana dashboard JSON ... }
```

### Built-in Dashboards

Two dashboards are deployed with the monitoring apps at wave 3:

- **HashiCorp Vault** (`app-default-values/vault/monitoring/grafana-dashboard.yaml`) — node health, Raft replication, leases, audit
- **External Secrets Operator** (`app-default-values/eso/monitoring/grafana-dashboard.yaml`) — reconcile rates, error rates, work queue

Grafana is also pre-configured with kube-prometheus-stack's bundled dashboards (node overview, Kubernetes cluster, Prometheus self-monitoring, etc.).

## Grafana Datasources

| Name | URL | Usage |
|------|-----|-------|
| Prometheus (default) | `http://kube-prometheus-stack-prometheus:9090` | Real-time metrics (last 2h) |
| Thanos | `http://thanos-query.monitoring:9090` | Full history from S3 |

## AlertManager — Slack & PagerDuty Integration

Each environment routes alerts to its own Slack channel. Production critical alerts are additionally paged via PagerDuty.

| Environment | Slack channel | PagerDuty |
|-------------|--------------|-----------|
| dev | `#dev-alerts` | no |
| uat | `#staging-alerts` | no |
| prod — `warning` | `#prod-alerts` | no |
| prod — `high` / `critical` / `emergency` | `#prod-alerts` | yes — on-call paged |

### How credentials are managed

```
Vault (secret/monitoring/alertmanager)
  └── ESO ExternalSecret (app-default-values/k8s-monitoring/alertmanager-externalsecret.yaml)
        └── K8s Secret  alertmanager-credentials  (monitoring namespace)
              └── AlertManager pod  /etc/alertmanager/secrets/alertmanager-credentials/
                    ├── slack_webhook_url
                    └── pagerduty_routing_key
```

The `ExternalSecret` is deployed by the `k8s-monitoring` ArgoCD app (wave 3). AlertManager's base values mount the secret via `alertmanagerSpec.secrets`, making the files available at the path above. The `alertmanager.config` in each environment's values then references those files with `api_url_file` / `routing_key_file` — no credentials ever appear in Git.

**Secret keys:**

| Key | Used by | Required in |
|-----|---------|-------------|
| `slack_webhook_url` | All environments | dev, uat, prod |
| `pagerduty_routing_key` | Prod only (routing config ignores it in dev/uat) | All envs (use a non-empty dummy in dev/uat) |

**Vault path (same path in every environment's Vault instance):** `secret/monitoring/alertmanager`

```bash
vault kv put secret/monitoring/alertmanager \
  slack_webhook_url="https://hooks.slack.com/services/T.../B.../..." \
  pagerduty_routing_key="<ROUTING_KEY_OR_DUMMY>"
```

**PagerDuty severity mapping:** the `pagerduty_configs.severity` field is set to `{{ .CommonLabels.severity }}`, so the alert's actual severity label (`high`, `critical`, or `emergency`) is forwarded to PagerDuty as-is. Map these to PagerDuty escalation policies as needed in the PagerDuty console.

### Routing logic

```
All alerts
  └── slack-<env>                            (all severities → Slack only)
       └── severity =~ high|critical|emergency  (prod only)
             └── pagerduty-and-slack-prod    (Slack + PagerDuty)
```

An inhibition rule suppresses `warning` alerts when a `high`, `critical`, or `emergency` alert with the same `alertname` and `namespace` is already firing, preventing duplicate noise.

### Repeat intervals

| Environment | repeat_interval |
|-------------|----------------|
| dev | 1h |
| uat | 2h |
| prod | 4h |

## Configuration Files

```
app-default-values/monitoring/values/base.yaml      ← base Helm values (all envs)
environments/dev/monitoring/values.yaml             ← dev overrides
environments/uat/monitoring/values.yaml             ← uat overrides
environments/prod/monitoring/values.yaml            ← prod overrides
```

## Environment Differences

| Setting | dev | uat | prod |
|---------|-----|-----|------|
| Prometheus replicas | 1 | 1 | 2 |
| Prometheus PVC | 20Gi | 30Gi | 50Gi |
| AlertManager replicas | 1 | 1 | 3 (HA) |
| AlertManager PVC | 2Gi | 2Gi | 5Gi |

## Pre-requisites

Before syncing this application:

1. **S3 bucket** — create `thanos-<env>` in the target AWS account
2. **IAM role** — `thanos-<env>` with the following policy and an IRSA trust policy for the `monitoring` namespace ServiceAccount:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:DeleteObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::thanos-<env>",
    "arn:aws:s3:::thanos-<env>/*"
  ]
}
```

3. Fill in environment values placeholders: `<AWS_REGION>`, `<AWS_ACCOUNT_ID>`, `<GRAFANA_ADMIN_PASSWORD>`, `<YOUR_DOMAIN>`
4. Populate Vault secrets for AlertManager at `secret/monitoring/alertmanager` — see the AlertManager section above
