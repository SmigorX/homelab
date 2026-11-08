# Arr stack

Argo CD deploys six components through `argocd/apps/workloads/arr-stack.yaml`,
replacing the `~/docker/arr/compose.yaml` stack: qBittorrent, Prowlarr, Sonarr,
Radarr, FlareSolverr and Jellyfin, all in the `arr-stack` namespace.

| Component | Host | Auth |
| --- | --- | --- |
| qBittorrent | `qbittorrent.k8s.internal.smigorx.eu` | Authentik forward auth |
| Prowlarr | `prowlarr.k8s.internal.smigorx.eu` | Authentik forward auth |
| Sonarr | `sonarr.k8s.internal.smigorx.eu` | Authentik forward auth |
| Radarr | `radarr.k8s.internal.smigorx.eu` | Authentik forward auth |
| Jellyfin | `jellyfin.k8s.internal.smigorx.eu` | its own accounts |
| FlareSolverr | none | none, ClusterIP only |

## Why this is not a translation of the compose file

Under Docker, Gluetun owned a network namespace and every other service joined
it with `network_mode: service:gluetun`. That had two consequences worth being
rid of:

- Reaching qBittorrent meant being *inside* the VPN's network, so the whole
  stack had to live there whether or not it needed a VPN.
- When Gluetun stopped it stopped publishing that namespace, and Sonarr,
  Radarr, Prowlarr and FlareSolverr went down with it. A lapsed subscription
  took out the entire stack, not just downloading.

Kubernetes does not need the trick: containers in a pod already share a network
namespace. So the tunnel is scoped to qBittorrent's pod alone, and every other
component is an ordinary Deployment reaching it at
`http://qbittorrent.arr-stack.svc.cluster.local:8080` like any other backend.
Nothing but qBittorrent notices the VPN exists, and nothing but qBittorrent's
outbound traffic stops when the VPN does.

The stack also stopped routing Prowlarr and FlareSolverr through the tunnel.
They fetch indexer pages, not torrents, and indexers are markedly more likely
to challenge or ban a shared VPN exit address than a residential one.

## The kill switch

`manifests/qbittorrent/deployment.yaml` has to hold up two claims at once:
qBittorrent never reaches the internet except through `wg0`, and qBittorrent's
WebUI and API answer regardless of the tunnel. Both come from only ever
filtering **OUTPUT**:

    iptables -P OUTPUT DROP
    iptables -A OUTPUT -o lo -j ACCEPT
    iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
    iptables -A OUTPUT -o wg0 -j ACCEPT
    iptables -A OUTPUT -m mark --mark 51820 -j ACCEPT

Inbound connections are never touched, and their replies match the
`ESTABLISHED,RELATED` line, so Traefik, Sonarr and Radarr keep getting answers
out of a pod that cannot otherwise send a packet.

Four details carry weight:

- **It is an initContainer.** Sibling containers have no defined start order,
  so rules installed by the sidecar would leave a window with qBittorrent up
  and unfiltered. iptables state lives in the network namespace, which belongs
  to the pod sandbox rather than any container, so it outlives the
  initContainer that wrote it.
- **`-o wg0` is installed before `wg0` exists.** iptables accepts a rule naming
  an absent interface; it simply never matches. That *is* the fail-closed
  behaviour — until wg-quick creates the interface there is no egress at all.
- **The fwmark rule is not optional.** WireGuard's own encrypted packets leave
  over `eth0`, not `wg0`, so without it the tunnel could never establish.
  Matching `FwMark` rather than the endpoint address keeps the choice of
  Mullvad server out of Git.
- **Same image for the initContainer and the sidecar.** iptables has a legacy
  and an nft backend writing to two different rule sets; two images could
  disagree about which one they mean and the policy would silently not compose.

DNS is set at pod level (`dnsPolicy: None`) rather than through wg0.conf's
`DNS=` line. Containers in a pod share a network namespace but not a
filesystem, so wg-quick rewriting `/etc/resolv.conf` would only rewrite the
sidecar's copy. Cluster DNS is deliberately unused: every tracker hostname
looked up through CoreDNS is a plaintext query leaving over the home address.
With Mullvad's resolver behind the tunnel, DNS fails closed along with
everything else.

### What a dead tunnel looks like

Downloads stall. The WebUI opens, Sonarr and Radarr keep talking to the API,
Jellyfin plays, and nothing leaks. The sidecar does **not** restart itself on a
failed handshake — that would crash-loop for as long as a subscription is
lapsed, which is exactly the situation this layout exists to survive quietly.
Recover with `kubectl -n arr-stack rollout restart deploy/qbittorrent`.

## Storage

One `PersistentVolume` over `/mnt/nas/bulk/arr` for the whole tree, mounted in
place — the very directory the compose stack used, not a copy. Consumers take
the subdirectory they need through `subPath`:

| Pod | subPath | Mount |
| --- | --- | --- |
| qBittorrent | `torrents` | `/data/torrents` |
| Sonarr, Radarr | *(none)* | `/data` |
| Jellyfin | `media` | `/data/media` |

Sonarr and Radarr get the whole tree on purpose. They import by hardlinking out
of qBittorrent's download directory into the library, and a hardlink cannot
cross filesystems — split into separate volumes, every import silently
downgrades to a full copy. The paths match the compose file exactly so the
existing databases' file references still resolve, and qBittorrent's
`.fastresume` files still point at files that exist.

`ReadWriteOnce` while four pods mount it is not a contradiction: RWO restricts a
volume to one *node*, and everything schedules onto the single node holding
`/mnt/nas`. It would become one on a second node.

Each app's `/config` is its own static PV under `/mnt/nas/bulk/arr/config/`.
These did **not** live on the NAS under Docker — they were `./config/<app>`
beside the compose file, on the server's own disk — so they have to be rsynced
across before the volumes are applied. Every PV uses `hostPath.type: Directory`,
which means a missing path leaves the pod in `ContainerCreating` instead of
being created empty; on an empty `/config` Sonarr and Radarr see no library and
would re-grab everything already on disk.

## Authentik

Unlike Uptime Kuma, the four gated apps each get their **own** provider in
`forward_single` mode, bound to `authentik Admins` only. They can start
downloads, rewrite the library and read every indexer credential in Prowlarr; a
guest-readable status page they are not.

Domain level, which `uptime-kuma/README.md` recommends copying, does not
stretch to this. A `forward_domain` provider claims an entire cookie domain,
and a second one claiming `k8s.internal.smigorx.eu` leaves the outpost no way
to tell which application's policies a request for `sonarr` belongs to —
per-application authorization needs per-application providers.

What made `forward_single` look impossible was that
`/outpost.goauthentik.io/*` has to answer on each application's own host, and
an Ingress cannot name a Service in another namespace without an `ExternalName`
that this Traefik refuses to route. The way around it is to stop routing that
path from the application's namespace: `Host` is a routing rule, not a property
of a namespace, so `authentik/manifests/outpost-ingress.yaml` sits beside
`authentik-server`, names it as an ordinary local Service, and claims that one
path on each of the four hosts. Traefik ranks routers by rule specificity, so it
wins over the application's `/` and everything else still reaches the app.

The embedded outpost's provider list moved to its own blueprint key,
`proxy-outpost.yaml`. The assignment *replaces* the list rather than adding to
it, so exactly one blueprint may declare it — and `!KeyOf` cannot reach an id
defined in another blueprint, hence `!Find` by name. **Every future forward auth
provider belongs in that list.** A missing entry does not error; the outpost
just stops answering for that host.

### This fails open

Same trap as Uptime Kuma. `traefik.ingress.kubernetes.io/router.middlewares`
takes a `<namespace>-<name>@kubernetescrd` string that nothing validates — drop
it, rename the namespace, rename the Middleware, and Traefik serves the app with
no authentication at all. Worth an actual check after any rename:

    for h in qbittorrent prowlarr sonarr radarr; do
      printf '%s ' $h
      curl -sk -o /dev/null -w '%{http_code}\n' https://$h.k8s.internal.smigorx.eu/
    done

Anything but a redirect to Authentik means it is open.

## Privileged containers

Two, where the repo previously had one. Both are the same wall Zigbee2MQTT hit,
written up in `home-automation/manifests/zigbee2mqtt/deployment.yaml`:

- **Jellyfin**, for `/dev/dri`. runc gives every unprivileged container a cgroup
  device filter allowing only a default set, and a DRI render node is not in it,
  so opening the device fails with `EPERM` whatever the file mode says. The
  failure is quiet — Jellyfin falls back to software transcoding and only the
  CPU graph says so. `supplementalGroups: [105]` is still needed alongside it:
  privileged settles the kernel, the group settles file permissions, and the
  ffmpeg doing the work runs as uid 1000.
- **The WireGuard sidecar**, for `net.ipv4.conf.all.src_valid_mark`. wg-quick
  writes that sysctl whenever a peer's `AllowedIPs` carry a default route, and
  `/proc/sys` is read-only in an unprivileged container, so wg-quick aborts. The
  narrower fix is a node-level flag this cluster does not set:
  `--kubelet-arg=allowed-unsafe-sysctls=net.ipv4.conf.all.src_valid_mark`, after
  which `NET_ADMIN` plus a pod-level `sysctls` entry would do.

## Migrating from the compose stack

Before Argo CD syncs anything. Every step is on the server.

1. **Stop the old stack**, so nothing is writing the config directories while
   they are copied and no second qBittorrent holds the torrents open:

       cd ~/docker/arr && docker compose down

2. **Copy the config directories onto the NAS.** They are small; the media tree
   is already in place and must not be touched.

       mkdir -p /mnt/nas/bulk/arr/config
       for app in qbittorrent prowlarr sonarr radarr jellyfin; do
         rsync -a ~/docker/arr/config/$app/ /mnt/nas/bulk/arr/config/$app/
       done
       chown -R 1000:1000 /mnt/nas/bulk/arr/config

   Keep the originals until the stack is verified — they are the rollback.

3. **Check the node** for the two values the manifests assume:

       getent group render      # expect gid 105, as in the compose group_add
       ls -l /dev/dri/          # expect renderD128
       modinfo wireguard        # the sidecar uses the kernel module

4. **Generate a WireGuard config** in the Mullvad account portal, picking a
   Düsseldorf server to match the old `SERVER_CITIES=Dusseldorf`. Gluetun chose
   a server per city on its own; plain wg-quick has no equivalent, so the server
   is now a fixed choice in the config.

5. **Compose `wg0.conf`** from it and push it into Vault as one key:

       [Interface]
       PrivateKey = <from the portal>
       Address = <from the portal>
       MTU = 1320
       FwMark = 51820

       [Peer]
       PublicKey = <Düsseldorf server key>
       Endpoint = <Düsseldorf server address>:51820
       AllowedIPs = 0.0.0.0/0
       PersistentKeepalive = 25

       vault kv put secret/arr-stack/qbittorrent-wireguard wg0.conf=@wg0.conf

   Two lines differ from what the portal hands over, and both matter:

   - **No `DNS =`.** wg-quick would try to rewrite `/etc/resolv.conf`, which
     kubelet owns; the pod's `dnsConfig` does that job instead. If the portal's
     resolver is not `10.64.0.1`, change it in
     `manifests/qbittorrent/deployment.yaml` to match.
   - **`FwMark = 51820`** is what the kill switch matches on. Changing it here
     means changing the iptables rule too.

   `Endpoint` must be an **address, not a hostname**. DNS fails closed before
   the tunnel is up, so a hostname there cannot be resolved and the tunnel could
   never start.

6. **Let Argo CD sync**, then point the apps at each other. The hostnames
   changed from `<app>.internal.smigorx.eu` to `<app>.k8s.internal.smigorx.eu`,
   and qBittorrent moved from `gluetun:8080` to `qbittorrent:8080`:

   - Sonarr and Radarr → Settings → Download Clients → qBittorrent: host
     `qbittorrent`, port `8080`.
   - Prowlarr → Settings → Apps: Sonarr at `http://sonarr:8989`, Radarr at
     `http://radarr:7878`.
   - Prowlarr → Settings → Indexers → FlareSolverr: `http://flaresolverr:8191`.

   The migrated qBittorrent config carries the old hostname with it. If the
   WebUI answers `401 Unauthorized` through Traefik while working through
   `kubectl port-forward`, it is host header validation: Options → Web UI →
   *Server domains* still lists `torrent.internal.smigorx.eu`. Set it to the new
   host, or to `*`.

## Verifying

**The tunnel carries qBittorrent's traffic:**

    kubectl -n arr-stack exec deploy/qbittorrent -c wireguard -- wg show
    kubectl -n arr-stack exec deploy/qbittorrent -c qbittorrent -- curl -s https://am.i.mullvad.net/json

The second should report a Mullvad exit in Düsseldorf.

**The kill switch holds.** Drop the tunnel and confirm the pod goes deaf
outbound but stays reachable inbound:

    kubectl -n arr-stack exec deploy/qbittorrent -c wireguard -- wg-quick down wg0
    kubectl -n arr-stack exec deploy/qbittorrent -c qbittorrent -- curl -sS --max-time 10 https://am.i.mullvad.net/json   # must fail
    curl -sk -o /dev/null -w '%{http_code}\n' https://qbittorrent.k8s.internal.smigorx.eu/                               # must still answer

A reply from the first command containing the home address is a leak and the
whole point of this layout failing. Restore with a rollout restart.

**Imports hardlink rather than copy.** After one import, compare inode numbers
on the node — equal means hardlinked, and a second copy of the file is not
sitting in the downloads directory:

    stat -c '%i %n' /mnt/nas/bulk/arr/torrents/<release>/<file> \
                    /mnt/nas/bulk/arr/media/<...>/<file>

**Jellyfin transcodes in hardware.** Play something that forces a transcode and
check the playback info reports a hardware pipeline rather than `sw`.
