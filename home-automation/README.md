# Home Automation

Argo CD deploys Mosquitto, Zigbee2MQTT and Home Assistant through
`argocd/apps/workloads/home-automation.yaml`, driving a SONOFF Zigbee/Thread
USB Dongle Plus MG24 plugged into `fedora-server`.

## One namespace, three components

Every other workload in this repo is one directory, one Application, one
namespace. This one is three components under a single `home-automation`
namespace and a single Application, with a subdirectory each. They share MQTT
credentials, they are never upgraded independently, and Zigbee2MQTT is useless
without the broker — splitting them would buy nothing but ordering problems.
`directory.recurse: true` picks up the subdirectories, exactly as it does for
joplin.

Sync waves keep the start-up readable rather than correct: Mosquitto and
Postgres are wave `0`, Home Assistant and Zigbee2MQTT wave `1`. Both clients
recover on their own if they start first, they just log failures while they
wait.

## The dongle

The coordinator is addressed by its `by-id` path:

    /dev/serial/by-id/usb-SONOFF_SONOFF_Dongle_Plus_MG24_2ca7e4dca4a2ef11a1aa906661ce3355-if00-port0

That path is derived from the stick's serial number and survives a replug.
`/dev/ttyUSB0` is whatever number the kernel hands out next and would silently
point at a different device the moment a second USB-serial adapter is attached.
The `hostPath` uses `type: CharDevice` so the pod fails loudly if the dongle is
missing rather than starting with an empty directory in its place.

The MG24 is an EFR32MG24 running EmberZNet, so the Zigbee2MQTT driver is
`ember` and the stock SONOFF firmware speaks `115200`. `rtscts` is disabled up
front: hardware flow control is what puts this dongle into its
`ASH_ERROR_TIMEOUT` loop. If it still loops, the firmware itself is the
suspect, and `ember-zli` reflashes it from the host.

### Why no `privileged: true`

`/dev/ttyUSB0` is `root:dialout 0660`, and the host runs SELinux in enforcing
mode with the device labelled `usbtty_device_t`. Neither blocks the pod:

- k3s here runs **without** `--selinux`, so containerd applies no SELinux label
  to containers and the device's type is never checked. Should k3s ever be
  started with `--selinux`, the fix is `setsebool -P container_use_devices 1`
  on the host — not raising the pod's privileges.
- The Zigbee2MQTT image runs as root today, which bypasses the file mode.
  `supplementalGroups: [18]` (`dialout`) is set anyway, so the dongle stays
  reachable if upstream ever switches to a non-root user.

The repo has no other pod holding a device, and this one stays unprivileged.

### Node pinning

`nodeSelector: home-automation=true` on the Zigbee2MQTT Deployment is the only
explicit scheduling constraint in this repo. It has no effect on a single-node
cluster — everything else here is already pinned by its `local-path` volume —
but the coordinator is a physical stick in one machine, and that is worth
stating rather than inheriting by accident.

## Storage

| Claim | Size | Holds |
| --- | --- | --- |
| `zigbee2mqtt-data` | 1Gi | Network key, PAN ID, device database |
| `home-assistant-config` | 10Gi | `/config`, including `.storage` |
| `mosquitto-data` | 1Gi | Retained messages, persistent sessions |

`zigbee2mqtt-data` **is** the Zigbee network. Lose it and every device has to be
factory-reset and re-paired by hand; no amount of Home Assistant configuration
survives it. `local-path-core` sets `reclaimPolicy: Retain`, which protects
against an accidental Argo CD prune, but nothing in this repo backs any volume
up and that is not the same thing.

All three Deployments use `strategy: Recreate`: ReadWriteOnce volumes, and in
Zigbee2MQTT's case a serial device exactly one process may hold open.

## Configuration

The two applications are configured in opposite ways, on purpose.

**Zigbee2MQTT** is driven entirely by `ZIGBEE2MQTT_CONFIG_*` environment
variables and has no ConfigMap. It rewrites its own `configuration.yaml` at
runtime — network key, PAN ID, anything changed in the frontend — so a mounted
file would either be read-only and break it, or drift from git. Environment
variables override the file, so the settings that matter stay authoritative in
the Deployment while Zigbee2MQTT keeps ownership of the rest.

**Home Assistant** gets its `configuration.yaml` from a ConfigMap, mounted
read-only through `subPath`. That is safe because Home Assistant never writes
this file; everything added through the UI goes to `/config/.storage`. Two
consequences worth knowing:

- Editing it means a commit and a pod restart. `subPath` mounts do not
  hot-update, so Argo CD syncing the ConfigMap alone changes nothing.
- Because we supply `configuration.yaml`, Home Assistant skips the bootstrap
  that would otherwise create `automations.yaml`, `scripts.yaml` and
  `scenes.yaml`, while the `!include` lines still demand they exist. The
  `seed-config` init container creates them once so the UI editors work from
  first boot.

`db_url` uses `!env_var`, which is absent from the Home Assistant secrets
documentation but registered by `annotatedyaml`, the loader Home Assistant
delegates to. It is what keeps the Postgres password out of the ConfigMap.

## Database

`home-assistant-postgres` is a CloudNativePG `Cluster` with **no**
`bootstrap.initdb.secret`, unlike paperless and nextcloud. The operator
therefore generates `home-assistant-postgres-app` itself and owns the password,
so there is nothing to keep in sync with Vault. The trade-off is that the
password lives only in the cluster: recoverable from the Secret, not from Vault.

The Deployment builds the connection URL from that Secret's `username` and
`password` keys via `$(VAR)` interpolation rather than using the operator's
convenience `uri` key, so it does not depend on which extra keys a given CNPG
version adds.

## Secrets

| Vault path | Secret | Keys |
| --- | --- | --- |
| `secret/home-automation/mqtt` | `mqtt-credentials` | `homeassistant-password`, `zigbee2mqtt-password`, `lan-password` |
| `secret/home-automation/zigbee2mqtt` | `zigbee2mqtt-secrets` | `frontend-auth-token` |

`home-assistant-postgres-app` is deliberately not in that table — see above.

Mosquitto only reads hashed password files, so an init container hashes the
plaintext from Vault into an `emptyDir` at every start with `mosquitto_passwd`.
The hashes are never persisted and never committed; rotating a password is a
`vault kv put` followed by a pod restart.

Write both paths **before** the first sync, and remember Vault re-seals itself
on every pod restart and needs three manual unseals before anything syncs at
all — see [vault/README.md](../vault/README.md).

    vault kv put secret/home-automation/mqtt \
      homeassistant-password=… zigbee2mqtt-password=… lan-password=…
    vault kv put secret/home-automation/zigbee2mqtt frontend-auth-token=…

## Network

Home Assistant and the Zigbee2MQTT frontend go through Traefik at
`home-assistant.k8s.internal.smigorx.eu` and
`zigbee2mqtt.k8s.internal.smigorx.eu`. `*.k8s.internal.smigorx.eu` is a
wildcard record, so neither host needed a DNS change.

MQTT is raw TCP and bypasses Traefik entirely. The broker is a `LoadBalancer` on
**192.168.1.13** from the `ingress` MetalLB pool, requested explicitly through
`metallb.universe.tf/loadBalancerIPs` because that pool sets
`autoAssign: false`. `.11` is Traefik and `.12` is samba.

Exposing the broker on the LAN is what makes Wi-Fi devices (ESPHome, Tasmota,
Shelly) possible later without touching this manifest, and is also why
`allow_anonymous false` and the ACL file are not optional. The `lan` account
exists for exactly that, scoped to `lan/#` so a compromised sensor cannot
publish Home Assistant discovery messages.

### hostNetwork

Home Assistant runs on ordinary cluster networking, so mDNS, SSDP and DHCP
broadcasts do not reach it and its auto-discovery of LAN devices — Chromecast,
Sonos, ESPHome, HomeKit — will find nothing. Zigbee needs none of that: the
coordinator is a USB device and Zigbee2MQTT reaches Home Assistant over MQTT.

Setting `hostNetwork: true` on the Home Assistant Deployment restores
discovery. It costs a fixed host port on `fedora-server`, drops the pod out of
the cluster network namespace, and would be the first such pod in this repo. It
is a deliberate "not yet", not an oversight — revisit it when there is a
non-Zigbee device worth discovering.

## Authentication

Home Assistant has no OIDC support, so unlike almost everything else here it is
**not** behind Authentik. It uses its own accounts.

Putting Traefik forward-auth in front of it is possible but a bad trade: the
companion app, webhooks and the REST API all authenticate with bearer tokens
and would break, and carving out `/api/`, `/auth/` and webhook paths reopens
most of what the middleware was meant to close. The reasoning is the same as
[onlyoffice-docs/README.md](../onlyoffice-docs/README.md) — the boundary is
that `*.k8s.internal.smigorx.eu` is not on the public internet.

The Zigbee2MQTT frontend has no login of its own whatsoever, which is why
`frontend.auth_token` is mandatory rather than optional here.

## First run

1. `https://zigbee2mqtt.k8s.internal.smigorx.eu`, log in with the frontend
   token, then *Permit join* and pair a device.
2. `https://home-assistant.k8s.internal.smigorx.eu`, complete onboarding, then
   Settings → Devices & Services → Add Integration → **MQTT**, broker
   `mosquitto`, port `1883`, username `homeassistant`. Home Assistant has been
   config-entry-only for MQTT for several releases, so this step cannot be
   expressed in YAML. Devices paired in step 1 appear through discovery within
   seconds.
