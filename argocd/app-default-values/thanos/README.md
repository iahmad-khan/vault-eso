# Thanos

Deploys Thanos via the `bitnami/thanos` Helm chart into the `monitoring` namespace. Synced at **wave 1**, after kube-prometheus-stack (wave 0) has started Prometheus with the Thanos sidecar.

## Components

| Component | Description |
|-----------|-------------|
| Thanos Query | Unified PromQL query layer across Prometheus + S3. Used as Grafana datasource. |
| Thanos Query Frontend | Caches and splits large queries; sits in front of Thanos Query |
| Thanos Store Gateway | Serves historical metrics from S3 object storage |
| Thanos Compactor | Downsamples and compacts S3 blocks to reduce storage and query cost |

Thanos Ruler is disabled — alert rules are managed as PrometheusRule CRDs picked up by Prometheus directly.

## Data Flow

```
Prometheus pod
  └── Thanos Sidecar
        ├── uploads 2h TSDB blocks ──► S3 bucket (thanos-<env>)
        └── exposes gRPC :10901 ──────► Thanos Query (live data)

Thanos Store Gateway
  └── reads blocks from S3 ──────────► Thanos Query (historical data)

Thanos Compactor
  └── reads/writes S3 ─── compaction, downsampling (5m, 1h resolutions)

Thanos Query Frontend
  └── wraps Thanos Query ─── caching, query splitting

Grafana ──► Thanos Query Frontend :9090
```

## Connecting Thanos Query to Prometheus

The Thanos Query `stores` list points to the Prometheus sidecar via a headless Kubernetes service created by kube-prometheus-stack:

```
dnssrv+_grpc._tcp.kube-prometheus-stack-thanos-discovery.monitoring.svc.cluster.local
```

This DNS SRV record resolves to all Prometheus pod gRPC endpoints, enabling automatic discovery when Prometheus scales to multiple replicas (prod uses 2 replicas).

## S3 Object Storage

Each environment uses a dedicated S3 bucket. The `objstoreConfig` value contains the Thanos object store configuration:

```yaml
# environments/<env>/thanos/values.yaml
objstoreConfig: |
  type: S3
  config:
    bucket: thanos-<env>
    endpoint: s3.amazonaws.com
    region: <AWS_REGION>
```

Authentication uses IRSA — no static credentials. The Thanos ServiceAccount is annotated with the IAM role ARN in each environment's values file.

## Configuration Files

```
app-default-values/thanos/values/base.yaml      ← base Helm values (all envs)
environments/dev/thanos/values.yaml             ← dev: S3 bucket, IRSA role
environments/uat/thanos/values.yaml             ← uat: S3 bucket, IRSA role
environments/prod/thanos/values.yaml            ← prod: S3 bucket, IRSA role
```

## Pre-requisites

1. **S3 bucket** per environment: `thanos-dev`, `thanos-uat`, `thanos-prod`

2. **IAM role** per environment with this policy:

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

Trust policy must allow the `thanos` ServiceAccount in the `monitoring` namespace via IRSA:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<EKS_OIDC_PROVIDER>"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "<EKS_OIDC_PROVIDER>:sub": "system:serviceaccount:monitoring:thanos"
    }
  }
}
```

3. The kube-prometheus-stack Thanos sidecar uses the **same IAM role** (configured in `environments/<env>/monitoring/values.yaml` under `prometheus.prometheusSpec.serviceAccount.annotations`). Both the sidecar and Thanos components must share access to the same S3 bucket.

## Querying Data

Access Thanos Query UI via port-forward:

```bash
kubectl port-forward svc/thanos-query 9090:9090 -n monitoring
```

Then open `http://localhost:9090` — this gives full history beyond Prometheus's 2h window.

Grafana uses the Thanos datasource (`http://thanos-query.monitoring.svc.cluster.local:9090`) by default for historical panels.

## Storage Growth

The Compactor runs downsampling after blocks age:
- **Raw** (5s resolution): kept for the Prometheus retention window and then in S3
- **5-minute downsampled**: created after 40h
- **1-hour downsampled**: created after 10 days

This keeps long-term query performance fast even as the bucket grows.
