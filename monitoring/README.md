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

That frees 768Mi, enough for this stack's ~850Mi with what was left over. The
margin is thin: anything new that requests memory will need room made for it
the same way, and this stack's own dashboards are now the place to find it —
*Kubernetes / Compute Resources / Namespace (Pods)* puts requests and usage
side by side.

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

## Alerting

Alertmanager posts `warning` and `critical` alerts to Discord, resolved ones
included. Everything else — the default rules have plenty of `info`-level
alerts such as CPU throttling — goes to a `blackhole` receiver and is only
visible in Grafana and the Alertmanager UI. A `critical` alert suppresses the
`warning`s of the same name in the same namespace.

The webhook URL is read from a file mounted from the `alertmanager-discord`
Secret (`webhook_url_file`), so it never appears in the Alertmanager config
the operator renders from Git.

The UIs, when needed:

    kubectl -n monitoring port-forward svc/vmalertmanager-vmks 9093
    kubectl -n monitoring port-forward svc/vmalert-vmks 8080

## Storage

`vmsingle-vmks` (10Gi, `local-path-core`) holds the metrics, and
`vmks-grafana` (1Gi, `local-path-core`) holds Grafana's SQLite database —
users, preferences, anything built by hand in the UI. Dashboards from the chart
are re-created on every sync and need no backup. Neither claim may fall back
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
- Default dashboards and rules are fetched by a sync Job that runs as a
  PostSync hook. What it creates is owned by its ServiceAccount, which Argo CD
  tracks, so they are cleaned up with the app rather than orphaned.
