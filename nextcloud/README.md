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
| `secret/onlyoffice-docs/jwt` | `nextcloud-onlyoffice`  | `JWT_SECRET`            |

`secret/nextcloud/oidc`'s `client-secret` **must stay equal to**
`nextcloud-client-secret` in `secret/authentik/oidc`. Authentik reads its copy
as the `NEXTCLOUD_OIDC_CLIENT_SECRET` env var (see `authentik/values.yaml`)
and stamps it into the provider through the blueprint. Rotate both together or
logins break.

`admin-user`/`admin-password` only matter on first install; changing them
later has no effect on the existing account.

The last row is not a Nextcloud secret at all — it is the Document Server's
own JWT secret, read straight from the path `onlyoffice-docs` uses. Because
Vault is the source of truth, there is no copied Secret to drift; rotating
`secret/onlyoffice-docs/jwt` updates both namespaces.

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

Document editing is backed by the `onlyoffice-docs` deployment in this repo.
`20-onlyoffice.sh` installs the `onlyoffice` connector app and writes four
values through `occ config:app:set`:

| Key                         | Value                                                   | Who calls it |
| --------------------------- | ------------------------------------------------------- | ------------ |
| `DocumentServerUrl`         | `https://onlyoffice.k8s.internal.smigorx.eu/`            | the browser  |
| `DocumentServerInternalUrl` | `http://onlyoffice-docs.onlyoffice-docs.svc.cluster.local/` | Nextcloud |
| `StorageUrl`                | `http://nextcloud.nextcloud.svc.cluster.local/`          | Document Server |
| `jwt_secret`                | from `nextcloud-onlyoffice`                              | both         |

Only the first is browser-facing, so it has to be the public hostname; the
other two stay inside the cluster instead of hairpinning back through
Traefik.

**Trailing slashes are required.** The connector's PHP setters normalise them,
but `occ config:app:set` writes the raw value straight past that code.

`jwt_header` is pinned to `Authorization` to match `JWT_HEADER` on the
Document Server. If the two disagree every editor session fails with a token
error rather than anything more descriptive.

Because the Document Server fetches files from Nextcloud over `StorageUrl`,
`nextcloud.nextcloud.svc.cluster.local` must be a trusted domain — otherwise
Nextcloud rejects those callbacks. That is why
`NEXTCLOUD_TRUSTED_DOMAINS` lists it, and why `00-trusted-domains.sh` exists:
the entrypoint only applies that variable during the initial install, so
editing it later would otherwise have no effect on an existing instance.

Check the handshake at any time:

    kubectl -n nextcloud exec deploy/nextcloud -- \
      su -s /bin/sh www-data -c "php occ onlyoffice:documentserver --check"

The hook runs the same check on every start and logs the result. It is
deliberately non-fatal, so a Document Server that is down cannot stop
Nextcloud from starting.
