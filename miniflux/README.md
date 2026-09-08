# Miniflux

Argo CD deploys Miniflux through `argocd/apps/workloads/miniflux.yaml`,
published at `miniflux.k8s.internal.smigorx.eu`.

## Storage

There is no `PersistentVolumeClaim` here. Miniflux keeps feeds, entries, read
state, sessions and API keys in Postgres and nothing on local disk, so the
CloudNativePG `Cluster/miniflux-postgres` (10Gi, `local-path-core`) holds the
entire installation. Backing that up backs up everything.

`RUN_MIGRATIONS=1` stays set across upgrades rather than only on first run —
Miniflux ships no separate migration job and applies the schema itself on
boot.

## Secrets

| Vault path | Secret | Keys |
| --- | --- | --- |
| `secret/miniflux/postgres` | `miniflux-postgres-app` | `username`, `password` |
| `secret/miniflux/app` | `miniflux-secret` | `ADMIN_USERNAME`, `ADMIN_PASSWORD` |
| `secret/miniflux/oidc` | `miniflux-oidc` | `client-secret` |

`miniflux-postgres-ca` is deliberately not in that table: CloudNativePG owns
it and regenerates it.

Write all three **before** the first sync, and remember Vault re-seals itself
on every pod restart and needs three manual unseals before anything syncs at
all — see [vault/README.md](../vault/README.md).

    vault kv put secret/miniflux/postgres username=miniflux password=…
    vault kv put secret/miniflux/app ADMIN_USERNAME=… ADMIN_PASSWORD=…
    vault kv put secret/miniflux/oidc client-secret=…

Keep the Postgres password alphanumeric. Miniflux takes a single connection
string, so `DATABASE_URL` is assembled from those two keys through kubelet's
`$(VAR)` expansion. It is written in libpq keyword form precisely so that a
password is not parsed as URL syntax, but a space or a quote in one would
still break it.

## Authentik OIDC

Unlike Uptime Kuma, Miniflux speaks OIDC itself, so this is an ordinary
confidential client rather than forward auth. Two halves of one secret have to
stay equal:

| Half | Where |
| --- | --- |
| `client-secret` | `secret/miniflux/oidc` → `miniflux-oidc` in this namespace |
| `miniflux-client-secret` | `secret/authentik/oidc` → `authentik-oidc-secrets` in the `authentik` namespace |

**Write the Authentik half first.** `authentik/values.yaml` passes it to both
Authentik pods as a `secretKeyRef`, and a reference to a key that does not
exist yet leaves them in `CreateContainerConfigError` — which takes SSO down
for every other application here, not only Miniflux.

`patch`, not `put`: `secret/authentik/oidc` already holds the client secrets
for Paperless, Yamtrack, Vault and Nextcloud, and `vault kv put` replaces a
path wholesale rather than merging into it.

    vault kv patch secret/authentik/oidc miniflux-client-secret=…

Two values are easy to get subtly wrong:

- `OAUTH2_OIDC_DISCOVERY_ENDPOINT` is the **issuer URL only**
  (`…/application/o/miniflux/`). Miniflux's OIDC library appends
  `/.well-known/openid-configuration` itself, so including it here makes
  discovery ask for it twice over.
- `OAUTH2_REDIRECT_URL` ends in `/oauth2/oidc/callback`. That `oidc` comes
  from `OAUTH2_PROVIDER`, not from the application slug, so it stays the same
  no matter what the Authentik application is named.

The Authentik application slug is load-bearing beyond routing: the discovery
document is served at `/application/o/<slug>/`, so renaming the application
breaks login rather than just the launch link.

## First run

`CREATE_ADMIN=1` makes the local admin from `secret/miniflux/app` on first
boot; the step is skipped once the account exists, so it is safe on every
restart. Log in with it first.

Users arriving through Authentik are created as **regular** users —
`OAUTH2_USER_CREATION=1` grants no admin rights — so that local account stays
the only way to reach settings that matter. An existing user links their
Authentik identity from their own Settings page.

Local authentication is deliberately left enabled. `DISABLE_LOCAL_AUTH=1`
hides the login form entirely, which also hides the admin account and leaves
Authentik as the only way in.
