# Homelab

A single-node k3s cluster on `fedora-server` (Fedora 43, k3s 1.36), deployed
entirely by Argo CD from this repository. Each directory is one Argo CD
Application; its README covers what is specific to it. This file covers what
they share.

## Layout

[argocd/app-of-apps.yaml](argocd/app-of-apps.yaml) syncs every Application
under [argocd/apps/](argocd/apps/), grouped into four projects:

| Project | Applications |
| --- | --- |
| `system` | [traefik](traefik/manifests/traefik-config.yaml) (k3s's own, configured), [metallb](metallb/) |
| `storage` | [local-path-provisioner](local-path-provisioner/values.yaml), [postgres-operator](postgres-operator/values.yaml) (CloudNativePG) |
| `platform` | [argocd](argocd/values.yaml), [cert-manager](cert-manager/), [vault](vault/README.md), [vault-secrets-operator](vault-secrets-operator/README.md), [authentik](authentik/README.md), [monitoring](monitoring/README.md) |
| `workloads` | [arr-stack](arr-stack/README.md), [forgejo](forgejo/README.md), [home-automation](home-automation/README.md), [homepage](homepage/values.yaml), [immich](immich/README.md), [joplin](joplin/README.md), [miniflux](miniflux/README.md), [nextcloud](nextcloud/README.md), [onlyoffice-docs](onlyoffice-docs/README.md), [paperless](paperless/README.md), [samba](samba/README.md), [uptime-kuma](uptime-kuma/README.md), [yamtrack](yamtrack/README.md) |

An app installed from a chart has `values.yaml` (overrides only; the chart's
own values file is the reference) and, for whatever the chart does not
provide, `manifests/`. A hand-written app has only `manifests/`.

## Conventions

**Secrets live in Vault, never in Git.** Each app's
`manifests/vault-secret.yaml` has the Vault Secrets Operator copy them into
Kubernetes Secrets. Vault re-seals on every restart and needs three manual
unseals; see [vault/README.md](vault/README.md). OIDC clients for Authentik
follow [authentik/README.md](authentik/README.md#adding-an-oidc-application).

**Storage** is one of two classes from `local-path-provisioner`, both
`reclaimPolicy: Retain` and `WaitForFirstConsumer`:

| Class | Host path | For |
| --- | --- | --- |
| `local-path-core` | `/mnt/nas/core/k3s` | databases, app state |
| `local-path-bulk` | `/mnt/nas/bulk/k3s` | large, regenerable or media data |

Always name the class: left out, a claim falls through to k3s's built-in
`local-path`, which is `reclaimPolicy: Delete` and not on the NAS. Directories
that predate the cluster (the arr media tree, the Immich library, the samba
share) are static PersistentVolumes or `hostPath` mounts with
`type: Directory`, so a missing mount fails the pod instead of serving an empty
directory. Do not put Argo CD sync waves between a PVC and its consumer:
`WaitForFirstConsumer` deadlocks the sync (see
[home-automation/README.md](home-automation/README.md)).

**Pinned to `fedora-server`, for now.** Every pod that mounts a
`local-path-*` claim, and every chart value or CloudNativePG `Cluster` that
creates one, sets `nodeSelector: kubernetes.io/hostname: fedora-server`
(`spec.affinity.nodeSelector` on a `Cluster`). A provisioned PV already carries
`nodeAffinity` to the node it was made on, so existing pods could not move
anyway; the selector is for new claims, which `WaitForFirstConsumer` would
otherwise provision on whichever node the first pod happens to land on. A new
consumer of either class needs the same selector. Forgetting it does not go
unnoticed: the provisioner's `nodePathMap` lists only `fedora-server`, so a
claim whose pod lands anywhere else stays `Pending` with a provisioning error.
The static `hostPath` PVs (arr-stack, Immich's library) and samba's `hostPath`
mount have no `nodeAffinity` at all and rely on the same selector alone. The one
exception is Jellyfin, on `fedora-server-2` for its iGPU, with media over NFS
from this node; see [arr-stack/README.md](arr-stack/README.md#jellyfin-runs-on-fedora-server-2).

**HTTP** goes through Traefik at `<app>.k8s.internal.smigorx.eu` (a wildcard
DNS record; Joplin alone is on `joplin.internal.smigorx.eu`), with a
`letsencrypt-cloudflare` certificate from cert-manager and `gethomepage.dev`
annotations for the Homepage dashboard. TLS ends at Traefik; backends are plain
HTTP. Nothing is published to the internet.

**Other protocols** get a MetalLB address from the `ingress` pool,
`192.168.1.11`–`.20`, which has `autoAssign: false`, so each Service requests
its address by annotation:

| Address | Service |
| --- | --- |
| `.11` | Traefik |
| `.12` | Samba |
| `.13` | Mosquitto (MQTT) |
| `.14` | Forgejo SSH |

**Resources.** Every container declares requests and a memory limit, and none
has a CPU limit: on one node a CPU limit only throttles a burst that nothing
else is waiting for. Scheduling is by requests, and free requests must stay
above the largest RollingUpdate Deployment's request or its next rollout
hangs; see [monitoring/README.md](monitoring/README.md#the-resource-budget).

**One node.** `ReadWriteOnce` volumes are shared between pods freely, since RWO
binds to a node, not a pod. That also means RWO does not stop a rolling update
from running two copies against the same data, so apps with state on a volume
use `strategy: Recreate`. Two exceptions still roll: Paperless and Grafana
(whose SQLite database is on `vmks-grafana`).

**Monitoring and logs** are in [monitoring/README.md](monitoring/README.md):
metrics, dashboards, alerts to Discord, and 30 days of pod and host logs.
