# Nextcloud

Argo CD deploys Nextcloud through `argocd/apps/workloads/nextcloud.yaml`:
the app itself, a CloudNativePG `Cluster` for PostgreSQL, a Valkey cache, and
a `CronJob` for background jobs.

## Secrets

All three come from Vault via the Vault Secrets Operator — there is no
`kubectl create secret` step. See [vault/README.md](../vault/README.md).

| Vault path            | Secret                   | Keys                          |
| --------------------- | ------------------------ | ----------------------------- |
| `secret/nextcloud/postgres` | `nextcloud-postgres-app` | `username`, `password`  |
| `secret/nextcloud/app`      | `nextcloud-secrets`      | `admin-user`, `admin-password` |
| `secret/nextcloud/oidc`     | `nextcloud-oidc`         | `client-secret`         |

`secret/nextcloud/oidc`'s `client-secret` **must stay equal to**
`nextcloud-client-secret` in `secret/authentik/oidc`. Authentik reads its copy
as the `NEXTCLOUD_OIDC_CLIENT_SECRET` env var (see `authentik/values.yaml`)
and stamps it into the provider through the blueprint. Rotate both together or
logins break.

`admin-user`/`admin-password` only matter on first install; changing them
later has no effect on the existing account.

## Single sign-on

Authentik is registered through the `nextcloud.yaml` blueprint in
`authentik/manifests/blueprints.yaml`. The redirect URI is
`https://nextcloud.k8s.internal.smigorx.eu/apps/user_oidc/code` — that path is
the `login#code` route of the `user_oidc` app, so it changes only if the
external hostname changes.

Nextcloud has no environment variables for OIDC, so `oidc-hook.yaml` mounts a
script into `/docker-entrypoint-hooks.d/before-starting`. The upstream image
runs every *executable* `*.sh` there on each container start, as `www-data`
with `cwd=/var/www/html` — hence `defaultMode: 0755` on the ConfigMap volume;
without the executable bit the entrypoint silently skips the script. The hook
installs and enables `user_oidc`, switches background jobs to cron mode, and
converges the provider. `occ user_oidc:provider <identifier>` creates the
provider if absent and updates it in place otherwise, so re-running is safe.

The client secret reaches `occ` via `--clientsecret-env`, which takes the
*name* of an environment variable rather than the value, keeping it out of the
process list.

Local login stays enabled. Keep it that way until you have confirmed a browser
login through Authentik, otherwise a bad provider config locks you out of your
own instance.

## Reverse proxy

TLS terminates at Traefik and the backend is plain HTTP, like everything else
here. Nextcloud needs telling, or it builds `http://` URLs and the OIDC
redirect fails: `OVERWRITEPROTOCOL`, `OVERWRITEHOST` and `TRUSTED_PROXIES`
(the `10.42.0.0/16` cluster CIDR) are set in `nextcloud.yaml` for exactly that
reason. `NEXTCLOUD_TRUSTED_DOMAINS` must also list the external hostname or
Nextcloud refuses the request outright.

## Storage

- `nextcloud-data` (100Gi, `local-path-bulk` → `/mnt/nas/bulk/k3s`) is mounted
  at `/var/www/html`. User files live under `data/` inside it. This is the one
  volume worth backing up.
- PostgreSQL uses `local-path-core`, 20Gi, managed by CloudNativePG.

The PVC is ReadWriteOnce, so the Deployment uses `strategy: Recreate` — a
rolling update would deadlock waiting for a second pod to attach the same
volume. The cron `CronJob` mounts it too and is safe only because this is a
single-node cluster.

## Access

Served at `https://nextcloud.k8s.internal.smigorx.eu`.

## OnlyOffice

`onlyoffice-docs` is already running in this cluster and can back Nextcloud's
document editing. It is not wired up here — doing so needs the ONLYOFFICE
connector app pointed at `https://onlyoffice.k8s.internal.smigorx.eu` with the
JWT secret from `onlyoffice-docs/README.md`.
