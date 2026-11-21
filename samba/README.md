# Samba

Argo CD deploys an SMB file server through `argocd/apps/workloads/samba.yaml`.

## Image

`servercontainers/samba`, not the once-popular `dperson/samba`, which was last
published in **July 2021** — years of unpatched Samba for something serving
the whole LAN. This one tracks upstream (Samba 4.23.8) and is 25MB.

The `smbd-only` variant drops avahi and wsdd2. Neither is useful here: their
discovery broadcasts do not survive a MetalLB LoadBalancer, and dropping
wsdd2 also drops its `CAP_NET_ADMIN` requirement. Connect by address or DNS
name rather than by browsing the network.

## Storage

`/mnt/nas/core/SMB` is mounted as a **hostPath**, deliberately not a PVC. The
directory already holds ~133G of existing data; a PVC would provision an
empty volume under `/mnt/nas/core/k3s` and serve nothing, with no error to
say so. A `hostPath` does nothing to keep the pod on the node holding that
disk, so `nodeSelector: kubernetes.io/hostname: fedora-server` does that (see
the root README, "Storage").

`strategy: Recreate` for the same reason — two smbd instances writing one
directory is not something to allow during a rollout.

## Share and permissions

One share, `Data`: browseable, writable, no guests, `valid users` and
`admin users` both `smigorx`.

`admin users` is load-bearing. `Data/Andrzej` and
`Data/Backups` are owned `0:101` with mode `drwxrwxr-x`, so uid 1000 matches
only the "other" bits and cannot write them. `admin users = smigorx` runs
file operations as root, which is how it reaches those directories. Dropping
it silently turns those two subtrees read-only.

`UID_smigorx=1000` matches everything else under the share, so newly written
files land under the same owner as the existing ones.

Note this makes the account root-equivalent *within the share*.

## Network

SMB is raw TCP, so this does not go through Traefik. The Service is a
`LoadBalancer` on **192.168.1.12**, taken from the `ingress` MetalLB pool.
That pool sets `autoAssign: false`, so the address is requested explicitly
through the `metallb.universe.tf/loadBalancerIPs` annotation.

The pool is `192.168.1.11`–`192.168.1.20`: `.11` Traefik, `.12` samba, `.13`
Mosquitto, `.14` Forgejo's SSH. Keep that range out of the router's DHCP pool.

`externalTrafficPolicy: Local` keeps the real client address visible in
Samba's logs instead of a rewritten node IP.

Port 139 is published alongside 445 for older clients. Anything current uses
445 only.

## Secret

| Vault path             | Secret          | Keys       |
| ---------------------- | --------------- | ---------- |
| `secret/samba/smigorx` | `samba-secrets` | `password` |

Changing it means reconfiguring every client that has it saved. See
[vault/README.md](../vault/README.md).

## Mounting

    smbclient -L //192.168.1.12 -U smigorx
    mount -t cifs //192.168.1.12/Data /mnt/point -o username=smigorx,vers=3.1.1
