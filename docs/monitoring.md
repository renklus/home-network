No Human review yet.
# Monitoring

kube-prometheus-stack runs in the prod cluster (`k8s-prod/apps/monitoring.yaml`): Prometheus, Alertmanager, Grafana,
kube-state-metrics and node-exporter. The rancher cluster runs the same chart in agent mode
(`k8s-rancher/apps/monitoring-agent.yaml`), which scrapes that cluster and remote-writes to the prod Prometheus. Rules are
only evaluated in prod. Every series has a `cluster` label (`production` or `rancher`) set by a default scrape class, and the
default rules and dashboards aggregate by it.

## How apps are monitored

Each app ships its own monitors: the chart's ServiceMonitor/PodMonitor toggle, or a few lines in its subdirectory.
Prometheus picks up every monitor and rule in every namespace.

- **CRDs:** `prometheus-operator-crds.yaml` on each cluster, sync wave -1, so no app needs a wave for its monitors. They
  are not part of kube-prometheus-stack (`crds.enabled=false`), because then every app with a monitor would have to wait
  for the monitoring app's health. Its Ingresses only become healthy once MetalLB, itself an app with monitors, has given
  Traefik an IP, so a fresh cluster would deadlock. Deleting the CRD app removes all monitors and rules, no data.
- **Rules:** only the prod Prometheus evaluates rules. A PrometheusRule in the rancher cluster would be silently ignored.
  Rules live next to the prod app and cover both clusters through the `cluster` label (e.g. `k8s-prod/apps/cert-manager/rules.yaml`).
  Rules for apps that only run in rancher (Argo CD) are in `k8s-prod/apps/monitoring/rules.yaml`.
- **Dashboards:** from the app's chart where it has one (CNPG), so Renovate updates them with the chart. The Grafana sidecar
  loads ConfigMaps labelled `grafana_dashboard: "1"` from all namespaces; it doesn't read Secrets.
- **k3s-owned components** (Traefik): a PodMonitor, not a HelmChartConfig value. A missing CRD would fail k3s' whole Traefik
  install. On rancher it lives in `monitoring-agent/`, because app-of-apps dry-runs all its top-level files before wave -1.

| App | Monitoring data | Rules |
|---|---|---|
| cert-manager (both) | chart ServiceMonitor | `CertificateNotReady`, `CertificateExpiresSoon` |
| CNPG operator and the immich/paperless/mealie databases | chart PodMonitor + dashboard; one PodMonitor per Cluster in `<app>/postgres.yaml` | `CNPGInstanceDown`, `CNPGReplicaNotStreaming`, `CNPGReplicationLag` |
| immich | chart ServiceMonitors (`immich.metrics.enabled`, also turns on telemetry) | – |
| paperless | PodMonitor on Flower's Celery metrics | `PaperlessTaskFailed` |
| kured (both) | chart ServiceMonitor | `KuredRebootPending` |
| metallb (both) | chart ServiceMonitors (HTTPS with the service account token) | – |
| Traefik (both) | PodMonitor on the `metrics` port | – |
| argo-cd (rancher) | chart ServiceMonitors (controller, server, repo-server, applicationset) | `ArgoCDAppUnhealthy`, `ArgoCDAppStuckProgressing`, `ArgoCDAppOutOfSync` |
| haproxy (rancher) | built-in Prometheus exporter on port 8405, chart ServiceMonitor | – (backends have no health checks; a broken ACME path shows up as `CertificateNotReady`) |
| kyverno (rancher) | chart ServiceMonitors (admission, background controller) | – |
| mealie, csi-nfs, rancher | no metrics endpoint or chart option | – |
| everything | kubelet/cAdvisor, kube-state-metrics, node-exporter, CoreDNS | chart default rules: crash loops, pods not ready, failed Jobs (e.g. backups), PVCs filling up, nodes |

## URLs (LAN only, no authentication)

- Grafana: https://grafana.ingress.prod.k8s.renklus.ch
- Prometheus: https://prometheus.ingress.prod.k8s.renklus.ch (also the remote-write endpoint `/api/v1/write`)
- Alertmanager: https://alertmanager.ingress.prod.k8s.renklus.ch

Only HAProxy's ACME path is reachable from the internet, so the ingress hosts are LAN-only. Anyone in the LAN can read
metrics and write into Prometheus; a Traefik basicAuth middleware would be the next step if that changes.

## Grafana admin password

The repo is public, so the Secret is created manually, before the first sync of `monitoring`. Without it the Grafana pod
can't start, `monitoring` (wave 1) stays unhealthy and the later sync waves of `app-of-prod` (immich, paperless, mealie) wait for it.
Argo CD adopts the namespace created here.

```shell
kubectl create namespace monitoring
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 24)"
kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d
```

Grafana has no persistence: dashboards and datasources are provisioned, and UI changes to them are lost on restart.

## Alerts

Active alerts are shown in the Alertmanager UI and in Grafana (Alerting → Alert list, Alertmanager datasource). Nothing is
sent: the chart's default Alertmanager config routes every alert to the `null` receiver. `Watchdog` always fires; it shows
that the alerting pipeline works.

To get notified later, in `k8s-prod/apps/monitoring.yaml`:

1. Create the credential as a Secret by hand, e.g. `kubectl -n monitoring create secret generic alertmanager-receiver --from-literal=url=...`.
2. Mount it with `alertmanager.alertmanagerSpec.secrets: ['alertmanager-receiver']` (path `/etc/alertmanager/secrets/alertmanager-receiver/`).
3. Set `alertmanager.config` with a receiver that reads the credential through a `*_file` setting (`url_file`,
   `bot_token_file`, `auth_password_file`, ...), and route to it. `alertmanager.config` replaces the chart default, so copy
   its `route`, `inhibit_rules` and the `null` receiver first.
4. For a dead man's switch (e.g. healthchecks.io), route `Watchdog` to a webhook receiver with a short `repeat_interval`;
   the external service alerts when the pings stop, which covers a prod cluster that is down entirely.

`ClusterMetricsMissing` (`k8s-prod/apps/monitoring/rules.yaml`) fires when the rancher agent has sent nothing for 15 minutes.

## k3s specifics

k3s runs controller-manager, scheduler and kube-proxy inside its own process with metrics bound to localhost, and etcd is
not a pod. Their scraping and rules are disabled on both clusters. Scraping them would need `bind-address` arguments for
those components (`machineGlobalConfig` in `k8s-rancher/apps/rancher/cluster-production.yaml`) and endpoints with the node IPs.

## Storage

Prometheus keeps 15 days (at most 15GB) on a `truenas-nfs-temp` PVC. Prometheus doesn't officially support its TSDB on NFS;
the metrics are disposable, so the corruption risk is accepted. If the TSDB breaks, delete the PVC and the Prometheus pod.
The agent only buffers its write-ahead log in an emptyDir.

On NFS, a PVC reports the usage of the whole share, so `KubePersistentVolumeFillingUp` effectively alerts on the TrueNAS dataset.
