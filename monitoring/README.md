# Monitoring

Argo CD deploys the `victoria-metrics-k8s-stack` chart through
`argocd/apps/platform/monitoring.yaml` into the `monitoring` namespace.
[values.yaml](values.yaml) holds overrides only; the chart's own `values.yaml`
is the reference.

| Component | What it does | Reachable at |
| --- | --- | --- |
| VMSingle | Stores metrics, 30 days | in-cluster only |
| VMAgent | Scrapes every target, writes to VMSingle | in-cluster only |
| VMAlert | Evaluates the alerting rules | `kubectl port-forward` |
| Alertmanager | Groups alerts, posts to Discord | `kubectl port-forward` |
| Grafana | Dashboards | `grafana.k8s.internal.smigorx.eu`, Authentik |
| kube-state-metrics, node-exporter | Kubernetes object and host metrics | — |
| VLSingle | Stores logs, 30 days | Grafana, *Explore → VictoriaLogs* |
| VLAgent | Ships pod logs and the host's syslog to VLSingle | — |
| VMAuth (`vmks-internal`) | Routes VMAlert's queries to VMSingle or VLSingle | in-cluster only |

The VictoriaMetrics operator turns the VM* custom resources into those
Deployments, and also converts any Prometheus-operator `ServiceMonitor` it
finds, so charts that only know how to ship a `ServiceMonitor` work unchanged.

## Why VictoriaMetrics, not kube-prometheus-stack

Memory. On a single node whose requests were already at 98% of allocatable,
VMSingle does the job of Prometheus in a fraction of the RAM, and the chart
brings the same default rules and dashboards kube-prometheus-stack does.
Grafana, Alertmanager, kube-state-metrics and node-exporter are the same
components either way.

## The resource budget

Scheduling is by requests, not usage, and before this stack the node had
15.0Gi of its 15.3Gi allocatable requested while using about 7.6Gi. Room was
made by trimming the requests (never the limits) of three workloads that idled
far below them:

| Workload | Requests before | after | Idle usage at the time |
| --- | --- | --- | --- |
| qBittorrent | 100m / 512Mi | 50m / 256Mi | ~30m / ~60Mi |
| Forgejo | 100m / 512Mi | 50m / 256Mi | ~3m / ~100Mi |
| Nextcloud | 100m / 512Mi | 50m / 256Mi | ~7m / ~70Mi |

That freed 768Mi, enough for this stack's ~850Mi with what was left over, and
the stack came in at exactly that. It was still not enough: the budget had
forgotten rolling updates. A Deployment with the default RollingUpdate strategy
starts its new pod *before* stopping the old one, so for a moment it needs its
request twice. With 165Mi free, the first Authentik change after this stack
landed left the new `authentik-server` (768Mi) Pending, Unschedulable,
indefinitely. A second round of trims made that room:

| Workload | Requests before | after | Idle usage at the time |
| --- | --- | --- | --- |
| immich-machine-learning | 100m / 1Gi | 100m / 512Mi | ~2m / ~230Mi |
| FlareSolverr | 100m / 512Mi | 100m / 256Mi | ~0m / ~145Mi |
| Jellyfin | 200m / 1Gi | 200m / 512Mi | ~0m / ~400Mi; limit stays 4Gi for transcodes |

The rule since then: **free requestable memory must stay above the largest
memory request of any RollingUpdate Deployment** (768Mi, Authentik and
Paperless). Recreate Deployments and StatefulSets replace in place and do not
count. The *Resources* dashboard computes exactly this as *Memory rolling-update
headroom*. When it goes red, its *Memory by namespace* chart and container table
show where requests are far above use.

**The same holds for CPU**, which ran out next: after the memory trims, 140m
of CPU requests was free against authentik-server's 200m, so its next rollout
would have stuck the same way. CPU requests had been set by habit (100m) for
containers idling at 1–20m. A third round, CPU only:

| Workload | CPU request before | after | Average use at the time |
| --- | --- | --- | --- |
| immich-server | 150m | 50m | ~4m, bursts past 150m on uploads |
| immich-machine-learning | 100m | 50m | ~3m |
| OnlyOffice | 150m | 50m | ~1m |
| Radarr, Sonarr | 100m each | 50m each | ~2–4m |
| Home Assistant | 100m | 50m | ~1m |

No container here has a CPU limit, so a lower request never caps a burst. What
it changes is the share a container gets while the CPU is contended, which on
this node means during a Jellyfin transcode. *CPU rolling-update headroom* on the
*Resources* dashboard tracks the CPU side of the rule. Both stats count init
containers the way the scheduler does; without that they came out 50m short,
because of Forgejo's.

## k3s specifics

k3s runs every control-plane component, the kubelet and kine (its SQLite
stand-in for etcd) as one process with one metrics registry, so every endpoint
serves the metrics of all of them. Two consequences:

- **The scheduler, controller manager, kube-proxy and kine are not missing.**
  Their metrics (`scheduler_*`, `workqueue_*`, `kubeproxy_*`, `kine_*`) arrive
  with the API server scrape, under `job="apiserver"`; query them in Explore.
  The chart's dedicated scrapes for them are disabled. They would only collect
  the same series again under another job label, and their "down" alerts mean
  nothing when one process runs all of them or none. Disabling them also drops
  their dashboards, which filter on those job labels.
- **The kubelet's `/metrics` is a copy of the API server's.** About 51k
  series, of which about 3k belong to the kubelet. values.yaml drops the other
  components' families from the kubelet scrape, so they are stored once rather
  than twice.

CoreDNS scrapes normally.

## Extra scrape targets

[manifests/scrapes.yaml](manifests/scrapes.yaml), for things the chart does not
know about:

- **Traefik**, whose k3s chart exposes metrics on container port 9100 without a
  Service, so it is a `VMPodScrape`.
- **Vault Secrets Operator**, behind kube-rbac-proxy, scraped with vmagent's
  ServiceAccount token. Its reconcile errors are what would have caught Vault
  sitting sealed for five days in November 2026.

- **Argo CD's application controller**, on the `metrics` port it always opens
  (the chart's metrics Services are off). It provides `argocd_app_info`, every
  Application's sync and health status. The controller's NetworkPolicy already
  admits that port from any namespace.

The `metrics` switches already present in `authentik/values.yaml`,
`forgejo/values.yaml` and `cert-manager/values.yaml` can now be turned on;
each needs a scrape object to go with it.

## Secrets

| Vault path | Secret | Keys |
| --- | --- | --- |
| `secret/monitoring/grafana-admin` | `grafana-admin` | `admin-user`, `admin-password` |
| `secret/monitoring/grafana-oidc` | `grafana-oidc` | `client-secret` |
| `secret/monitoring/alertmanager-discord` | `alertmanager-discord` | `webhook-url` |
| `secret/authentik/oidc` | `authentik-oidc-secrets` | `grafana-client-secret` |

Write all of them **before** the first sync:

    vault kv patch secret/authentik/oidc grafana-client-secret=…

    vault kv put secret/monitoring/grafana-oidc client-secret=…   # same value
    vault kv put secret/monitoring/grafana-admin admin-user=… admin-password=…
    vault kv put secret/monitoring/alertmanager-discord webhook-url=https://discord.com/api/webhooks/…

The Authentik half goes first, with `patch`, not `put`: that path holds every
other application's client secret too, and `put` replaces it wholesale. Then
wait until VSO has copied it — `refreshAfter` is an hour — before letting the
Authentik change sync, because a `secretKeyRef` to a key that is not there yet
leaves both Authentik pods in `CreateContainerConfigError` and takes SSO down
for everything:

    kubectl -n authentik get secret authentik-oidc-secrets \
      -o jsonpath='{.data.grafana-client-secret}' | wc -c     # non-zero

Pick a non-obvious `admin-user` as well as a strong password: it is the one
account that bypasses SSO.

## Grafana and Authentik

The provider, application and an `authentik Admins`-only binding are in
`authentik/manifests/blueprints.yaml` under the `grafana.yaml` key. Grafana's
side is `grafana.ini` in values.yaml.

Roles come from the `groups` claim of Authentik's `profile` scope:
`role_attribute_path` maps `authentik Admins` to Grafana's Admin, and
`role_attribute_strict` refuses anyone it maps to nothing. That is the second
lock behind the binding, not the first — without the binding, a guest would
reach Grafana's login and only then be refused.

`oauth_auto_login` sends every visit straight to Authentik. When Authentik
itself is what is broken, the local admin account still works at
`https://grafana.k8s.internal.smigorx.eu/login?disableAutoLogin=true`.

The admin password is taken from Vault rather than generated by the chart: the
chart's `randAlphaNum` would produce a new one on every Argo CD render — the
same trap `forgejo/README.md` describes.

## Dashboards

The chart's ~40 stock dashboards are off (`defaultDashboards.enabled: false`).
Most covered things this cluster does not have (Windows, AIX and Darwin nodes,
multiple clusters, the components k3s does not split out), and the rest
answered general questions rather than this cluster's. Four hand-written ones
replace them, in Grafana's *Homelab* folder:

| Dashboard | Answers |
| --- | --- |
| **Health** (the home page) | What is broken: firing alerts, pods not ready and why, restarts and their exit reasons, OOM kills, Argo CD apps not Synced/Healthy, Vault errors from VSO, k3s certificate expiry |
| **Resources** | Will the next pod fit: requests against allocatable, rolling-update headroom, requested-vs-used per namespace, and every container's usage as a share of its request and limit |
| **Node** | The host: CPU, memory, swap, PSI stalls, temperatures, RAID, filesystems, disk and network |
| **Ingress** | Traefik: traffic, status codes, response times and bandwidth per service, and every TLS certificate's expiry |

They are plain JSON in [manifests/dashboards/](manifests/dashboards/), which
[manifests/kustomization.yaml](manifests/kustomization.yaml) turns into the
`homelab-dashboards` ConfigMap. The Grafana sidecar loads any ConfigMap in this
namespace labelled `grafana_dashboard: "1"`. A new dashboard is a new file plus
a line in the kustomization.

Provisioned dashboards cannot be saved from the UI. To change one, edit it in
Grafana, then *Export → Export as JSON* and overwrite its file here. Keep the
`uid`, because the Health dashboard is the home page by its file name and the
dashboards link to each other by tag.

All queries target the datasource UID `VictoriaMetrics` directly. That is the
UID the chart provisions, so the dashboards need no datasource variable.

**One-off cleanup** after the sync that turns the stock dashboards off. The
ConfigMaps the sync job created stay behind, because the job loses its
ConfigMap permissions together with the dashboards:

    kubectl -n monitoring delete configmap -l app.kubernetes.io/managed-by=sync-job,grafana_dashboard=1

## Logs

VLSingle keeps 30 days of logs, and at most 8GiB of them, on a 10Gi
`local-path-core` claim. VLAgent runs on the node and sends it two things:

- **Every container's output**, read from `/var/log/pods`, with `namespace`,
  `pod` and `container` attached. A pod's logs outlive the pod.
- **The host's syslog**: `/var/log/messages` (k3s itself, systemd, kernel,
  dnf) and `/var/log/secure` (sshd, sudo), tagged with `file`. These come from
  the systemd journal, which vlagent cannot read, through the rsyslog the
  Fedora host also runs. If rsyslog is ever removed from the host, these two
  stop and the journal is the only copy left.

Search them in Grafana under *Explore*, datasource *VictoriaLogs*, in
[LogsQL](https://docs.victoriametrics.com/victorialogs/logsql/):

    {kubernetes.pod_namespace="authentik"} error
    {file="/host/var/log/messages"} k3s

The chart also routes VMAlert through a small VMAuth (`vmks-internal`), so a
VMRule can alert on log queries as well as metrics. VMAlert now depends on it:
if it is down, no rule is evaluated.

## Alerting

Alertmanager posts `warning` and `critical` alerts to Discord, resolved ones
included. Everything else — the default rules have plenty of `info`-level
alerts such as CPU throttling — goes to a `blackhole` receiver and is only
visible in Grafana and the Alertmanager UI. A `critical` alert suppresses the
`warning`s of the same name in the same namespace.

Rules of this cluster's own, beyond the chart's defaults, are in
[manifests/rules.yaml](manifests/rules.yaml):

- **VaultSecretsOperatorFailing** (critical): VSO has failed Vault calls in
  every 10-minute window for 20 minutes. In practice that means Vault is
  sealed, which nothing else reports, since the pod stays `1/1` by design (see
  `vault/README.md`). Every `vault-0` restart triggers it until Vault is
  unsealed.

The webhook URL is read from a file mounted from the `alertmanager-discord`
Secret (`webhook_url_file`), so it never appears in the Alertmanager config
the operator renders from Git.

The UIs, when needed:

    kubectl -n monitoring port-forward svc/vmalertmanager-vmks 9093
    kubectl -n monitoring port-forward svc/vmalert-vmks 8080

## Storage

`vmsingle-vmks` (10Gi, `local-path-core`) holds the metrics,
`vlsingle-vmks` (10Gi, `local-path-core`) the logs, and
`vmks-grafana` (1Gi, `local-path-core`) holds Grafana's SQLite database —
users, preferences, anything built by hand in the UI. The dashboards in
`manifests/dashboards/` are re-created from Git and need no backup. None of the claims may fall back
to k3s's default `local-path` class, which is node-local and
`reclaimPolicy: Delete`.

## Argo CD notes

- `ServerSideApply=true`: the operator's CRDs exceed the 256KiB annotation
  client-side apply writes.
- `SkipDryRunOnMissingResource=true`: `manifests/` holds VM* objects whose CRDs
  arrive in the same sync.
- The operator's admission webhook is disabled. Its certificate is generated
  at render time, so it would differ on every Argo CD refresh and the app would
  never settle Synced.
- Default rules are fetched by a sync Job that runs as a PostSync hook. What
  it creates is owned by its ServiceAccount, which Argo CD tracks, so they are
  cleaned up with the app rather than orphaned.
- Grafana installs the `victoriametrics-logs-datasource` plugin from
  grafana.com at start-up, so a Grafana restart needs internet access.
- `manifests/` is a kustomization, which Argo CD detects on its own, only so
  the dashboards can stay `.json` files. Its other files are listed under
  `resources:`, so a new manifest there needs a line too.
