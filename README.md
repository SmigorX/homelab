# Homelab

A k3s cluster on `fedora-server` (Fedora 43, k3s 1.36, the server) and
`fedora-server-2` (Fedora 45, an agent), deployed entirely by Argo CD from this
repository. Each directory is one Argo CD
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

**Storage** is one of three classes from `local-path-provisioner`, all
`reclaimPolicy: Retain` and `WaitForFirstConsumer`:

| Class | Host path | Disk | For |
| --- | --- | --- | --- |
| `local-path-core` | `/mnt/nas/core/k3s` | `md0`, the HDD mirror | databases, app state |
| `local-path-bulk` | `/mnt/nas/bulk/k3s` | the 4TB HDD, not mirrored | large, regenerable or media data |
| `local-path-ssd` | `/var/lib/local-path-ssd` | the OS SSD, not mirrored | busy data that can be lost with the node: metrics, logs, Grafana |

`/var/lib/local-path-ssd` needs `container_file_t` (SELinux is enforcing, and
nothing relabels a hostPath): `sudo semanage fcontext -a -t container_file_t
'/var/lib/local-path-ssd(/.*)?'` and `sudo restorecon -R` on it.

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

**Pinned pods share a node.** `ReadWriteOnce` volumes are shared between pods
freely, since RWO binds to a node, not a pod. That also means RWO does not stop a rolling update
from running two copies against the same data, so apps with state on a volume
use `strategy: Recreate`. Paperless is the one exception that still rolls.

**Monitoring and logs** are in [monitoring/README.md](monitoring/README.md):
metrics, dashboards, alerts to Discord, and 30 days of pod and host logs.

## Nodes and the firewall

| Node | Address | Role | firewalld zone the other node lands in |
| --- | --- | --- | --- |
| `fedora-server` | `192.168.1.10` | k3s server, all storage | `lan` (source `192.168.1.0/24`) |
| `fedora-server-2` | `192.168.1.90` | k3s agent, Jellyfin's iGPU | `FedoraServer` (default, on `eno1`) |

firewalld on both hosts rejects by default, and nothing here is opened by the
k3s installer. Every port the nodes need from each other is a rich rule scoped
to the other node's address alone:

| Port | Open on | Used by | When it is missing |
| --- | --- | --- | --- |
| `6443/tcp` | server | agent → API server | the agent never joins (already open to the whole LAN in `lan`) |
| `8472/udp` | both | flannel VXLAN, all pod-to-pod traffic across nodes | cross-node pod IPs, Service IPs and cluster DNS time out; Traefik cannot reach pods on the agent |
| `7946/tcp` + `/udp` | both | MetalLB speakers' memberlist | split brain: both speakers announce every `externalTrafficPolicy: Cluster` address, LAN clients flip between the two MACs, and the speakers rewrite each other's `ServiceL2Status` in a loop fast enough to stall the API server |
| `10250/tcp` | both | kubelet, scraped by vmagent | the other node's kubelet metrics go missing |
| `9100/tcp` | both | node-exporter, scraped by vmagent from whichever node it runs on | that node's host metrics go missing |
| `2049/tcp` | server | NFS, Jellyfin's media ([arr-stack/README.md](arr-stack/README.md#jellyfin-runs-on-fedora-server-2)) | the Jellyfin pod waits in `ContainerCreating` |

The pod and Service ranges, `10.42.0.0/16` and `10.43.0.0/16`, are in each
host's `trusted` zone. On `fedora-server`, each rule looks like this, with the
port and protocol swapped in:

    sudo firewall-cmd --permanent --zone=lan --add-rich-rule='rule family="ipv4" source address="192.168.1.90" port port="8472" protocol="udp" accept'

and on `fedora-server-2`:

    sudo firewall-cmd --permanent --zone=FedoraServer --add-rich-rule='rule family="ipv4" source address="192.168.1.10" port port="8472" protocol="udp" accept'

followed by `sudo firewall-cmd --reload` on each. `sudo firewall-cmd --zone=<zone> --list-rich-rules` shows what is open. A third node needs the same set
in both directions, against every existing node.
