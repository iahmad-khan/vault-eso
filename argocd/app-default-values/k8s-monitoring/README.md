# Kubernetes Monitoring

Deploys cluster-wide Kubernetes alerting rules and a Grafana dashboard into the `monitoring` namespace. Uses the same kustomize/directory pattern as `vault-monitoring` and `eso-monitoring`. Synced at **wave 3**, after kube-prometheus-stack (wave 0) has installed the Prometheus Operator CRDs.

Alert rules are sourced from [samber/awesome-prometheus-alerts](https://samber.github.io/awesome-prometheus-alerts/) — community-vetted PromQL expressions with accurate `for` durations and severity levels.

## Files

| File | Description |
|------|-------------|
| `prometheusrule.yaml` | 37 alerts — Kubernetes workloads, nodes, HPA, storage, jobs, API server |
| `prometheusrule-self-monitoring.yaml` | 28 alerts — Prometheus, Alertmanager, TSDB, scraping, cardinality |
| `grafana-dashboard.yaml` | ConfigMap (label `grafana_dashboard: "1"`) auto-imported by the Grafana sidecar |
| `alertmanager-externalsecret.yaml` | ESO ExternalSecret — pulls AlertManager credentials from Vault into `alertmanager-credentials` K8s Secret |

---

## Alert Groups

### `kubernetes.nodes` (6 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `KubernetesNodeNotReady` | Node Ready condition false | 10m | critical |
| `KubernetesNodeSchedulingDisabled` | Node tainted `unschedulable` | 30m | warning |
| `KubernetesNodeMemoryPressure` | Node MemoryPressure=true | 2m | critical |
| `KubernetesNodeDiskPressure` | Node DiskPressure=true | 2m | critical |
| `KubernetesNodeNetworkUnavailable` | Node NetworkUnavailable=true | 2m | critical |
| `KubernetesNodeOutOfPodCapacity` | >90% of allocatable pods used | 2m | warning |

### `kubernetes.pods` (3 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `KubernetesPodCrashLooping` | >3 restarts in 1 minute | 2m | warning |
| `KubernetesPodNotHealthy` | Pod in Pending/Unknown/Failed for 15m | 0m | critical |
| `KubernetesContainerOomKiller` | Restart + last terminated reason OOMKilled | 0m | warning |

### `kubernetes.workloads` (9 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `KubernetesDeploymentReplicasMismatch` | Available replicas ≠ spec replicas | 10m | warning |
| `KubernetesDeploymentGenerationMismatch` | Observed generation ≠ metadata generation | 10m | critical |
| `KubernetesReplicaSetReplicasMismatch` | Ready replicas ≠ spec replicas | 10m | warning |
| `KubernetesStatefulSetReplicasMismatch` | Ready replicas ≠ total replicas | 10m | warning |
| `KubernetesStatefulSetDown` | Ready replicas / current replicas < 1 | 1m | critical |
| `KubernetesStatefulSetGenerationMismatch` | Observed generation ≠ metadata generation | 10m | critical |
| `KubernetesStatefulSetUpdateNotRolledOut` | Current revision ≠ update revision | 10m | warning |
| `KubernetesDaemonSetRolloutStuck` | Ready < desired or scheduling gaps | 10m | warning |
| `KubernetesDaemonSetMisscheduled` | Pods on nodes with wrong selectors | 1m | critical |

### `kubernetes.hpa` (4 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `KubernetesHPAScaleInability` | At max replicas + ScalingLimited=true | 2m | warning |
| `KubernetesHPAMetricsUnavailability` | ScalingActive=false | 0m | warning |
| `KubernetesHPAScaleMaximum` | Desired replicas ≥ max replicas | 2m | info |
| `KubernetesHPAUnderutilized` | Never exceeded 50% of max over 1 day | 0m | info |

### `kubernetes.storage` (4 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `KubernetesPersistentVolumeClaimPending` | PVC in Pending phase | 2m | warning |
| `KubernetesVolumeOutOfDiskSpace` | Volume <10% free | 2m | warning |
| `KubernetesVolumeFullInFourDays` | `predict_linear` projects full in <4 days | 0m | critical |
| `KubernetesPersistentVolumeError` | PV in Failed or Pending phase | 0m | critical |

### `kubernetes.jobs` (6 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `KubernetesJobFailed` | Job has failed pods | 0m | warning |
| `KubernetesJobNotStarting` | Job >10m old with no active/succeeded/failed pods | 0m | warning |
| `KubernetesCronJobFailing` | Last schedule > last success, no active pods, not suspended | 0m | critical |
| `KubernetesCronJobSuspended` | CronJob spec.suspend=true | 0m | warning |
| `KubernetesCronJobTooLong` | Active CronJob job running >1h | 0m | warning |
| `KubernetesJobSlowCompletion` | Completions still pending after 12h | 12h | critical |

### `kubernetes.apiserver` (5 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `KubernetesAPIServerErrors` | 5xx error rate >3% | 2m | critical |
| `KubernetesAPIClientErrors` | Client 4xx/5xx error rate >1% | 2m | critical |
| `KubernetesClientCertificateExpiresNextWeek` | Client cert expiry <7 days | 0m | warning |
| `KubernetesClientCertificateExpiresSoon` | Client cert expiry <24h | 0m | critical |
| `KubernetesAPIServerLatency` | API server p99 latency >1s | 2m | warning |

---

## Prometheus Self-Monitoring Alert Groups (`prometheusrule-self-monitoring.yaml`)

Source: [awesome-prometheus-alerts — Prometheus self-monitoring](https://samber.github.io/awesome-prometheus-alerts/rules/basic-resource-monitoring/prometheus-self-monitoring/)

### `prometheus.availability` (6 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `PrometheusJobMissing` | `absent(up{job="prometheus"})` | 0m | warning |
| `PrometheusTargetMissing` | Target down while others in same job are up | 1m | critical |
| `PrometheusAllTargetsMissing` | Every target in a job is down | 1m | critical |
| `PrometheusTargetMissingWithWarmupTime` | Target down >1m on a node booted >10m | 1m | critical |
| `PrometheusNotConnectedToAlertmanager` | Discovered Alertmanagers < 1 | 0m | critical |
| `PrometheusAlertManagerJobMissing` | `absent(up{job="alertmanager"})` | 0m | warning |

### `prometheus.configuration` (3 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `PrometheusConfigurationReloadFailure` | `prometheus_config_last_reload_successful != 1` | 0m | warning |
| `PrometheusAlertManagerConfigurationReloadFailure` | `alertmanager_config_last_reload_successful != 1` | 0m | warning |
| `PrometheusAlertManagerConfigNotSynced` | >1 distinct config hash across Alertmanager cluster | 0m | warning |

### `prometheus.rules` (4 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `PrometheusRuleEvaluationFailures` | Rule evaluation failures in last 3m | 0m | critical |
| `PrometheusTemplateTextExpansionFailures` | Template expansion failures in last 3m | 0m | critical |
| `PrometheusRuleEvaluationSlow` | Group eval duration > group interval | 5m | warning |
| `PrometheusAlertManagerE2EDeadManSwitch` | Always-firing dead man switch (`vector(1)`) | 0m | critical |

### `prometheus.notifications` (2 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `PrometheusNotificationsBacklog` | Notification queue non-empty for 10m | 0m | warning |
| `PrometheusAlertManagerNotificationFailing` | Failed notification rate >0.05/s | 0m | critical |

### `prometheus.scraping` (5 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `PrometheusTooManyRestarts` | >2 process restarts in 15m | 0m | warning |
| `PrometheusTargetEmpty` | Service discovery has 0 targets | 0m | critical |
| `PrometheusTargetScrapingSlow` | p90 scrape duration >5% above p50 | 5m | warning |
| `PrometheusLargeScrape` | >10 scrapes exceeded sample limit in 10m | 5m | warning |
| `PrometheusTargetScrapeDuplicate` | >3 duplicate-timestamp samples in 5m | 0m | warning |

### `prometheus.tsdb` (7 alerts)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `PrometheusTSDBCheckpointCreationFailures` | Checkpoint creation failures | 0m | critical |
| `PrometheusTSDBCheckpointDeletionFailures` | Checkpoint deletion failures | 0m | critical |
| `PrometheusTSDBCompactionsFailed` | TSDB compaction failures | 0m | critical |
| `PrometheusTSDBHeadTruncationsFailed` | Head truncation failures | 0m | critical |
| `PrometheusTSDBReloadFailures` | TSDB reload failures | 0m | critical |
| `PrometheusTSDBWALCorruptions` | WAL corruption detected | 0m | critical |
| `PrometheusTSDBWALTruncationsFailed` | WAL truncation failures | 0m | critical |

### `prometheus.cardinality` (1 alert)

| Alert | Trigger | For | Severity |
|-------|---------|-----|----------|
| `PrometheusTimeseriesCardinality` | Any metric has >10,000 active series | 0m | warning |

---

## Grafana Dashboard

The **Kubernetes Overview** dashboard (UID: `k8s-monitoring`) includes a **Namespace** template variable (defaults to All) and:

**Stat row** — cluster health at a glance:
Running pods · CrashLoopBackOff pods · OOMKilled containers · Nodes Ready · HPAs at max · Pending PVCs

**Time-series panels:**
- Container restart rate (top 10 by namespace/pod/container)
- Pod phase distribution over time (Running / Pending / Failed / Succeeded)
- Node memory usage % per node
- Node CPU usage % per node

**Table panels:**
- CrashLoopBackOff and OOMKilled pods (live)
- Top restarting containers in the last hour (with threshold colouring)
- HPA status — min / max / desired / current replicas
- PVC usage — available bytes, capacity, used % (with threshold colouring)
- Node status — Ready, MemoryPressure, DiskPressure, PIDPressure per node

---

## AlertManager Credentials (ExternalSecret)

`alertmanager-externalsecret.yaml` is deployed into the `monitoring` namespace alongside the PrometheusRules and dashboard. It uses the cluster-wide `ClusterSecretStore` (`vault-backend`) — already deployed by the `eso-secretstore` app at wave 2 — to pull two keys from Vault and materialise them as a Kubernetes Secret named `alertmanager-credentials`.

### Vault path

The same path is used in every environment's Vault instance:

```
secret/monitoring/alertmanager
  ├── slack_webhook_url       ← Slack incoming webhook URL
  └── pagerduty_routing_key  ← PagerDuty Events API v2 routing key
```

Populate before syncing:

```bash
vault kv put secret/monitoring/alertmanager \
  slack_webhook_url="https://hooks.slack.com/services/T.../B.../..." \
  pagerduty_routing_key="<KEY>"   # non-empty dummy is fine for dev/uat
```

### How the secret is consumed

`app-default-values/monitoring/values/base.yaml` declares:

```yaml
alertmanager:
  alertmanagerSpec:
    secrets:
      - alertmanager-credentials
```

This mounts the secret into every AlertManager pod at:

```
/etc/alertmanager/secrets/alertmanager-credentials/
  ├── slack_webhook_url
  └── pagerduty_routing_key
```

Each environment's `alertmanager.config` then references the files directly — credentials are never written to Git:

```yaml
slack_configs:
  - api_url_file: /etc/alertmanager/secrets/alertmanager-credentials/slack_webhook_url

pagerduty_configs:                          # prod only
  - routing_key_file: /etc/alertmanager/secrets/alertmanager-credentials/pagerduty_routing_key
```

### Routing summary

| Environment | Slack channel | PagerDuty triggered by |
|-------------|--------------|------------------------|
| dev | `#dev-alerts` | never |
| uat | `#staging-alerts` | never |
| prod | `#prod-alerts` | `severity =~ "high\|critical\|emergency"` |

---

## Metrics Sources

All queries use metrics from components already deployed by kube-prometheus-stack:

| Metric prefix | Source |
|---------------|--------|
| `kube_*` | kube-state-metrics |
| `node_*` | node-exporter |
| `kubelet_volume_stats_*` | kubelet (scraped by Prometheus Operator) |
| `apiserver_*`, `rest_client_*` | kube-apiserver (scraped by Prometheus Operator) |
| `prometheus_*` | Prometheus itself (scraped by Prometheus Operator) |
| `alertmanager_*` | Alertmanager (scraped by Prometheus Operator) |
| `process_start_time_seconds` | All monitored processes via Prometheus Operator |
