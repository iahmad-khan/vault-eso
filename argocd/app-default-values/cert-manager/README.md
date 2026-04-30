# cert-manager

Deploys cert-manager into the `cert-manager` namespace for automated TLS certificate provisioning via Let's Encrypt. Uses Route53 DNS-01 ACME challenges with IRSA for credential-free AWS auth. Synced at **wave 0** alongside vault/eso/monitoring, so CRDs and the webhook are ready before the wave-1 `cert-manager-issuers` app applies ClusterIssuers.

## Architecture

```
Ingress (cert-manager.io/cluster-issuer annotation)
  └── cert-manager controller
        └── ClusterIssuer (letsencrypt-staging or letsencrypt-prod)
              └── ACME DNS-01 challenge → Route53 TXT record
                    └── Let's Encrypt validates → issues cert
                          └── Stored as K8s Secret → TLS at nginx ingress
```

## Components

| Pod | Description |
|-----|-------------|
| cert-manager (controller) | Watches Certificate/Ingress resources, drives ACME flow |
| cert-manager-webhook | Validates/mutates cert-manager CRDs |
| cert-manager-cainjector | Injects CA bundles into webhook configurations |

## ClusterIssuers

Two ClusterIssuers are deployed per environment by the `cert-manager-issuers` ArgoCD app (wave 1):

| Name | ACME server | Use |
|------|-------------|-----|
| `letsencrypt-staging` | `acme-staging-v02.api.letsencrypt.org` | Testing — issues untrusted certs, no rate limits |
| `letsencrypt-prod` | `acme-v02.api.letsencrypt.org` | Production — browser-trusted certs |

Use `letsencrypt-staging` on dev/uat Ingresses during initial setup to avoid Let's Encrypt rate limits. Switch to `letsencrypt-prod` only when TLS flow is confirmed working.

## Files

```
app-default-values/cert-manager/
├── values/
│   └── base.yaml                     ← shared Helm values (all envs)
└── issuers/
    ├── dev/
    │   ├── clusterissuer-staging.yaml
    │   └── clusterissuer-prod.yaml
    ├── uat/
    │   ├── clusterissuer-staging.yaml
    │   └── clusterissuer-prod.yaml
    └── prod/
        ├── clusterissuer-staging.yaml
        └── clusterissuer-prod.yaml

environments/<env>/cert-manager/values.yaml   ← IRSA role ARN per env
```

## Configuration

### Base values (`values/base.yaml`)

- `installCRDs: true` — CRDs installed with the chart, no separate CRD job needed
- Resource requests/limits set for controller, webhook, and cainjector
- `global.leaderElection.namespace: cert-manager` — scoped leader election

### Per-environment values

Each `environments/<env>/cert-manager/values.yaml` sets the IRSA role ARN:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/cert-manager-<env>
```

### ClusterIssuer placeholders

Fill these in each `issuers/<env>/` directory:

| Placeholder | Description |
|-------------|-------------|
| `<ACME_EMAIL>` | Email for Let's Encrypt expiry notifications |
| `<AWS_REGION>` | AWS region where the Route53 hosted zone lives |
| `<ROUTE53_HOSTED_ZONE_ID>` | Route53 hosted zone ID (e.g. `Z1234ABCD`) |

## IAM Pre-requisites

Create role `cert-manager-<env>` with the following policy, trusted by the `cert-manager` namespace `cert-manager` ServiceAccount via IRSA:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:GetChange",
        "route53:ChangeResourceRecordSets",
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResource"
      ],
      "Resource": "*"
    }
  ]
}
```

## Usage

### Annotate an Ingress for automatic TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # or letsencrypt-staging
spec:
  tls:
    - hosts:
        - my-app.prod.example.com
      secretName: my-app-tls
  rules:
    - host: my-app.prod.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

cert-manager detects the annotation, creates a Certificate resource, runs the DNS-01 challenge, and stores the issued cert in the `my-app-tls` Secret.

### Verify a certificate

```bash
kubectl get certificate -n <namespace>
kubectl describe certificate my-app-tls -n <namespace>
```

`READY=True` means the cert is issued and stored.
