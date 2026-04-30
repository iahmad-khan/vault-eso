# external-dns

Deploys external-dns into the `external-dns` namespace. Watches Ingress and Service resources and automatically creates/updates Route53 A/CNAME records so hostnames are resolvable without manual DNS management. Uses IRSA for credential-free AWS auth. Synced at **wave 0** alongside cert-manager.

## Architecture

```
Ingress (hostname: my-app.prod.example.com)
  └── external-dns watches → creates Route53 A record → my-app.prod.example.com
        └── TXT ownership record → prevents other clusters overwriting the record
```

## Key Configuration

| Setting | Value | Why |
|---------|-------|-----|
| `provider` | `aws` | Route53 |
| `sources` | `ingress`, `service` | Watch both resource types |
| `policy` | `sync` | Delete stale records when Ingress/Service is removed |
| `txtOwnerId` | `<env>-cluster` | Unique per cluster — prevents cross-cluster record conflicts |
| `domainFilters` | `[<env>.<YOUR_DOMAIN>]` | Only manage records in the env-specific zone |

## Files

```
app-default-values/external-dns/
└── values/
    └── base.yaml                     ← shared Helm values (all envs)

environments/<env>/external-dns/values.yaml   ← region, txtOwnerId, domainFilter, IRSA ARN
```

## Per-environment overrides

Each `environments/<env>/external-dns/values.yaml`:

```yaml
aws:
  region: <AWS_REGION>

txtOwnerId: <env>-cluster          # e.g. dev-cluster, uat-cluster, prod-cluster

domainFilters:
  - <env>.<YOUR_DOMAIN>            # prod uses the apex domain

serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/external-dns-<env>
```

### Domain filter per environment

| Environment | domainFilters |
|-------------|---------------|
| dev | `dev.<YOUR_DOMAIN>` |
| uat | `uat.<YOUR_DOMAIN>` |
| prod | `<YOUR_DOMAIN>` (apex, covers all subdomains) |

## IAM Pre-requisites

Create role `external-dns-<env>` trusted by the `external-dns` namespace `external-dns` ServiceAccount via IRSA:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["route53:ChangeResourceRecordSets"],
      "Resource": "arn:aws:route53:::hostedzone/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResources"
      ],
      "Resource": "*"
    }
  ]
}
```

## Usage

Simply set a hostname on an Ingress — external-dns picks it up automatically:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  rules:
    - host: my-app.prod.example.com
      ...
```

external-dns creates `my-app.prod.example.com → <LoadBalancer IP>` in Route53 within ~30 seconds.

To expose a `LoadBalancer` Service directly, annotate it:

```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: my-svc.prod.example.com
```

## Placeholders to Fill

| Placeholder | File | Description |
|-------------|------|-------------|
| `<AWS_REGION>` | All env values | AWS region |
| `<YOUR_DOMAIN>` | All env values | Base domain |
| `<DEV/UAT/PROD_AWS_ACCOUNT_ID>` | Respective env values | AWS account ID |
