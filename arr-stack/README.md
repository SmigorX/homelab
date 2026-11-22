# Arr stack

Argo CD deploys six components through `argocd/apps/workloads/arr-stack.yaml`:
qBittorrent, Prowlarr, Sonarr, Radarr, FlareSolverr and Jellyfin, all in the
`arr-stack` namespace.

| Component | Host | Auth |
| --- | --- | --- |
| qBittorrent | `qbittorrent.k8s.internal.smigorx.eu` | Authentik forward auth |
| Prowlarr | `prowlarr.k8s.internal.smigorx.eu` | Authentik forward auth |
| Sonarr | `sonarr.k8s.internal.smigorx.eu` | Authentik forward auth |
| Radarr | `radarr.k8s.internal.smigorx.eu` | its own OIDC login, against Authentik |
| Jellyfin | `jellyfin.k8s.internal.smigorx.eu` | its own accounts |
| FlareSolverr | none | none, ClusterIP only |

## Only qBittorrent is behind the VPN

The stack used to run under Docker Compose with every service inside Gluetun's
network namespace, so a lapsed VPN subscription took out the whole stack, not
just downloading. Containers in a pod already share a network namespace, so
here the tunnel is a sidecar in qBittorrent's pod alone, and every other
component reaches it at `http://qbittorrent.arr-stack.svc.cluster.local:8080`
like any other backend. Nothing but qBittorrent's outbound traffic stops when
the VPN does.

Prowlarr and FlareSolverr stay off the tunnel on purpose. They fetch indexer
pages, not torrents, and indexers are markedly more likely to challenge or ban
a shared VPN exit address than a residential one.

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

One static `PersistentVolume` over `/mnt/nas/bulk/arr` for the whole tree.
Consumers take the subdirectory they need through `subPath`:

| Pod | subPath | Mount |
| --- | --- | --- |
| qBittorrent | `torrents` | `/data/torrents` |
| Sonarr, Radarr | *(none)* | `/data` |

Jellyfin is not on this volume: it runs on the other node and reads `media/`
over NFS, mounted at the same `/data/media` (see below).

Sonarr and Radarr get the whole tree on purpose. They import by hardlinking out
of qBittorrent's download directory into the library, and a hardlink cannot
cross filesystems — split into separate volumes, every import silently
downgrades to a full copy. The mount paths are recorded in the apps' databases
and in qBittorrent's `.fastresume` files, so they must not change.

`ReadWriteOnce` while three pods mount it is not a contradiction: RWO restricts
a volume to one *node*, and every pod here except Jellyfin and FlareSolverr carries
`nodeSelector: kubernetes.io/hostname: fedora-server`, the node holding
`/mnt/nas` (see the root README, "Storage"). The PVs themselves have no
`nodeAffinity`, so that selector is the only thing keeping a pod off a node
without the tree. Lift it and RWO becomes a real constraint.

Each app's `/config` is its own static PV under `/mnt/nas/bulk/arr/config/`.
Every PV uses `hostPath.type: Directory`, so a missing path leaves the pod in
`ContainerCreating` instead of being created empty; on an empty `/config`
Sonarr and Radarr see no library and would re-grab everything already on disk.

### Jellyfin runs on fedora-server-2

That node's UHD 630 (i5-9500T, Coffee Lake) decodes 10-bit HEVC and VP9 in
hardware; the HD 530 (i5-6500, Skylake) in `fedora-server` cannot, which left
every 4K HDR transcode on the CPU. Moving Jellyfin moved its two volumes:

- **`/config` is on `fedora-server-2`'s own disk**, `/var/lib/jellyfin`, as the
  static PV `jellyfin-config-local` with `nodeAffinity` to that node. It holds a
  SQLite database, which must not go over NFS. It is **not on the RAID**: the
  old copy at `/mnt/nas/bulk/arr/config/jellyfin` was left in place, but it is a
  snapshot as of the move. SELinux is enforcing and hostPath volumes are not
  relabelled, so the directory carries `container_file_t` through a rule:

      sudo semanage fcontext -a -t container_file_t '/var/lib/jellyfin(/.*)?'
      sudo restorecon -R /var/lib/jellyfin

- **Media is NFS, read-only**, PV `jellyfin-media`. `fedora-server` exports
  `media/` alone, to `fedora-server-2` alone, from
  `/etc/exports.d/jellyfin.exports`:

      /mnt/nas/bulk/arr/media 192.168.1.90(ro,root_squash,no_subtree_check,mp=/mnt/nas/bulk)

  `mp=` withholds the export while the bulk pool is unmounted, so the pod waits
  in `ContainerCreating` instead of scanning an empty library. Read-only means
  Jellyfin cannot save artwork or `.nfo` files next to the media; leave *Save
  artwork into media folders* and the NFO saver off in each library's settings.

NFS also does not forward inotify events, so Jellyfin's real-time monitoring
never sees an import Sonarr or Radarr writes on the other node. New episodes
appear at the next scheduled library scan, or at once if Sonarr and Radarr are
given a Jellyfin connection (*Settings → Connect → Emby / Jellyfin*), which
asks Jellyfin to refresh after each import.

## Authentik

qBittorrent, Prowlarr and Sonarr have no SSO of their own. Unlike Uptime Kuma,
each gets its **own** forward auth provider in `forward_single` mode, bound to `authentik Admins` only. They can start
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
path on each of the three hosts. Traefik ranks routers by rule specificity, so it
wins over the application's `/` and everything else still reaches the app.

The embedded outpost's provider list lives in its own blueprint key,
`proxy-outpost.yaml`. The assignment *replaces* the list rather than adding to
it, so exactly one blueprint may declare it — and `!KeyOf` cannot reach an id
defined in another blueprint, hence `!Find` by name. **Every future forward auth
provider belongs in that list.** A missing entry does not error; the outpost
just stops answering for that host.

### Radarr: OIDC

Since v7, Radarr has an OIDC login of its own, so it is an ordinary OAuth2 client
like Grafana: the `radarr.yaml` key in `authentik/manifests/blueprints.yaml`,
with no Middleware and no outpost path. All of its settings are environment
variables in [manifests/radarr/deployment.yaml](manifests/radarr/deployment.yaml),
which take precedence over `config.xml`, so the UI cannot change them.

- **Exactly one user.** Radarr admits a single verified email or `sub`. The
  Deployment names akadmin's `sub`, and the Authentik binding also limits the
  app to `authentik Admins`. Someone added to that group still gets Radarr's
  "not authorized" page until `Radarr__Auth__OidcUserIdentifier` names them.
  An email address would only work if Authentik reported it as verified.
- **`TrustedNetworks` is what keeps the login working.** Without
  `10.42.0.0/16`, Radarr ignores Traefik's `X-Forwarded-Proto`, sends
  `http://…/signin-oidc` as the redirect URI, and Authentik's strict match
  rejects it.
- **Locked out?** Remove the `Radarr__Auth__*` variables (or set
  `Radarr__Auth__Method` to `Forms`) and Radarr falls back to the
  username and password in its `config.xml`.

The client secret has two copies that must stay equal:

| Vault path | Read by |
| --- | --- |
| `secret/authentik/oidc`, key `radarr-client-secret` | Authentik, through `RADARR_OIDC_CLIENT_SECRET` |
| `secret/arr-stack/radarr-oidc`, key `client-secret` | Radarr, through the `radarr-oidc` Secret |

Set them up as described in `authentik/README.md`, "Adding an OIDC application".
Use `patch` for the first one.

### The forward auth apps have no login of their own

Behind forward auth, each app's own login would only be a second password
prompt, so qBittorrent, Sonarr and Prowlarr skip it for in-cluster clients.
Traefik's requests come from its pod IP, so whoever got past Authentik lands
straight in the UI. These settings live in each app's `/config`, not in Git:

| App | Setting |
| --- | --- |
| qBittorrent | Options → Web UI → *Bypass authentication for clients in whitelisted IP subnets*: `10.42.0.0/16` |
| Sonarr, Prowlarr | Settings → General → *Authentication Required*: Disabled for Local Addresses |

"Local addresses" means every private range, so every pod qualifies.

On their own, these bypasses would give every pod in the cluster the apps
unauthenticated. The *arr UIs hand the API key to any client they let in, and
qBittorrent's API can add torrents and delete files. A NetworkPolicy beside each
app narrows who can reach it. Radarr has one too, as defence in depth:

| App | Reachable from |
| --- | --- |
| qBittorrent | Traefik, Sonarr, Radarr, Prowlarr |
| Sonarr, Radarr | Traefik, Prowlarr (which pushes indexers into them) |
| Prowlarr | Traefik, Sonarr, Radarr (which search through it) |

**Never enable a bypass without its policy in place.** Anything else that needs
one of these APIs, such as a Homepage widget or an Uptime Kuma monitor on the
internal address, needs an entry in that app's policy first.

Sonarr and Radarr can keep the qBittorrent credentials they have. qBittorrent
ignores them for whitelisted clients.

To check that the policies are enforced, run this from a pod they do not admit.
Every line must time out:

    kubectl -n default run np-test --rm -i --restart=Never --image=curlimages/curl -- sh -c '
      for u in qbittorrent:8080 sonarr:8989 radarr:7878 prowlarr:9696; do
        printf "%s " $u; curl -sS -o /dev/null -w "%{http_code}\n" --max-time 5 http://$u.arr-stack.svc.cluster.local/ 2>&1 | tail -1
      done'

### This fails open

Same trap as Uptime Kuma. `traefik.ingress.kubernetes.io/router.middlewares`
takes a `<namespace>-<name>@kubernetescrd` string that nothing validates — drop
it, rename the namespace, rename the Middleware, and Traefik serves the app with
no authentication at all. Worth an actual check after any rename:

    for h in qbittorrent prowlarr sonarr; do
      printf '%s ' $h
      curl -sk -o /dev/null -w '%{http_code}\n' https://$h.k8s.internal.smigorx.eu/
    done

Anything but a redirect to Authentik means it is open.

## Privileged containers

Two, both hitting the same wall as Zigbee2MQTT, written up in
`home-automation/manifests/zigbee2mqtt/deployment.yaml`:

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

## How the apps reach each other

Set in each app's UI and stored in its `/config`, not in Git:

- Sonarr and Radarr → Settings → Download Clients → qBittorrent: host
  `qbittorrent`, port `8080`.
- Prowlarr → Settings → Apps: Sonarr at `http://sonarr:8989`, Radarr at
  `http://radarr:7878`.
- Prowlarr → Settings → Indexers → FlareSolverr: `http://flaresolverr:8191`.

If qBittorrent's WebUI answers `401 Unauthorized` through Traefik while working
through `kubectl port-forward`, it is host header validation: Options → Web UI
→ *Server domains* must list `qbittorrent.k8s.internal.smigorx.eu`, or `*`.

## Replacing the WireGuard config

The tunnel's whole configuration is one Vault key. To move to another Mullvad
server or a new key, generate a WireGuard config in the Mullvad account portal
and reshape it as:

    [Interface]
    PrivateKey = <from the portal>
    Address = <from the portal>
    MTU = 1320
    FwMark = 51820

    [Peer]
    PublicKey = <server key>
    Endpoint = <server address>:51820
    AllowedIPs = 0.0.0.0/0
    PersistentKeepalive = 25

Then:

    vault kv put secret/arr-stack/qbittorrent-wireguard wg0.conf=@wg0.conf

and, once VSO has synced it (`refreshAfter: 1h`, or sooner after a restart of
the operator), `kubectl -n arr-stack rollout restart deploy/qbittorrent`.

Three things differ from what the portal hands over, and all matter:

- **No `DNS =`.** wg-quick would try to rewrite `/etc/resolv.conf`, which
  kubelet owns; the pod's `dnsConfig` does that job instead. If the portal's
  resolver is not `10.64.0.1`, change it in
  `manifests/qbittorrent/deployment.yaml` to match.
- **`FwMark = 51820`** is what the kill switch matches on. Changing it here
  means changing the iptables rule too.
- **`Endpoint` must be an address, not a hostname.** DNS fails closed before
  the tunnel is up, so a hostname there cannot be resolved and the tunnel could
  never start.

## If the host is rebuilt

The manifests assume these things about the nodes. On `fedora-server-2`, for
Jellyfin:

    getent group render      # gid 105, as in jellyfin/deployment.yaml
    ls -l /dev/dri/          # renderD128
    ls -Zd /var/lib/jellyfin # container_file_t, see "Jellyfin runs on fedora-server-2"

On `fedora-server`:

    modinfo wireguard        # the sidecar uses the kernel module
    sudo exportfs -v         # /mnt/nas/bulk/arr/media to 192.168.1.90

## Verifying

**The tunnel carries qBittorrent's traffic:**

    kubectl -n arr-stack exec deploy/qbittorrent -c wireguard -- wg show
    kubectl -n arr-stack exec deploy/qbittorrent -c qbittorrent -- curl -s https://am.i.mullvad.net/json

The second should report `"mullvad_exit_ip":true`, in whichever city the server
in `wg0.conf` is.

**The kill switch holds.** Drop the tunnel and confirm the pod goes deaf
outbound but stays reachable inbound:

    kubectl -n arr-stack exec deploy/qbittorrent -c wireguard -- wg-quick down /run/wireguard/wg0.conf
    kubectl -n arr-stack exec deploy/qbittorrent -c wireguard -- iptables -L OUTPUT -n -v | head -1
    kubectl -n arr-stack exec deploy/qbittorrent -c qbittorrent -- curl -sS --max-time 10 https://1.1.1.1/cdn-cgi/trace   # must time out
    kubectl -n arr-stack exec deploy/qbittorrent -c wireguard -- iptables -L OUTPUT -n -v | head -1
    curl -sk -o /dev/null -w '%{http_code}\n' https://qbittorrent.k8s.internal.smigorx.eu/                               # must still answer

The full path is needed: the sidecar keeps its config in `/run/wireguard`, not
the `/etc/wireguard` that `wg-quick down wg0` would look in.

The probe goes to an address, not a hostname, on purpose. With the tunnel down
DNS fails first (Mullvad's resolver is only reachable through it), so a
hostname dies at `Could not resolve host` without a single packet testing the
OUTPUT rules — reassuring, and proving nothing about qBittorrent, which reaches
peers by IP. Expect curl's exit code 28 and the `policy DROP` counter to climb
between the two readings; qBittorrent's own peer attempts make it climb by
thousands, not by curl's handful.

Any reply to the `curl` — a `ip=` line in the trace — is a leak: with the tunnel
down it could only have left over the home address, and that is the whole point
of this layout failing. Restore with a rollout restart.

**Imports hardlink rather than copy.** After one import, compare inode numbers
on the node — equal means hardlinked, and a second copy of the file is not
sitting in the downloads directory:

    stat -c '%i %n' /mnt/nas/bulk/arr/torrents/<release>/<file> \
                    /mnt/nas/bulk/arr/media/<...>/<file>

**Jellyfin transcodes in hardware.** The device access above only makes the GPU
reachable; Jellyfin uses it only once *Dashboard → Playback → Transcoding* says
so. That setting lives in `/config/encoding.xml` on the config volume, not in
Git; with it at `none`, every transcode runs on the CPU. Set:

- **Hardware acceleration:** VAAPI, device `/dev/dri/renderD128`.
- **Hardware decoding:** H264, HEVC, MPEG2, VC1, VP8, VP9, **HEVC 10bit and
  VP9 10bit**. The last three are new with the move to Coffee Lake: `vainfo`
  lists `HEVCMain10`, `VP9Profile0` and `VP9Profile2`, none of which the old
  Skylake iGPU had. Not AV1, which no Gen9 iGPU decodes, and not *HEVC RExt*;
  ticking one the hardware cannot do asks it for something it cannot do. AV1
  is decoded on the CPU. Check with
  `kubectl -n arr-stack exec deploy/jellyfin -- /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128`.
- **Hardware encoding:** on.
- **Tone mapping:** on, or 4K HDR transcodes come out washed out. It runs on
  OpenCL, which the image does not include; the `DOCKER_MODS` variable in the
  Deployment installs Intel's runtime at container start. Without it, every
  transcode with tone mapping fails outright with *Failed to get number of
  OpenCL platforms: -1001* in the FFmpeg log, and the client shows a playback
  error. Check with `kubectl -n arr-stack exec deploy/jellyfin -- clinfo -l`.

To check, play something that forces a transcode. The newest
`/config/log/FFmpeg.Transcode-*.log` should show `h264_vaapi` rather than
`libx264` as the video encoder.
