# Paperless-ngx

Argo CD deploys Paperless-ngx through `argocd/apps/workloads/paperless.yaml`,
published at `paperless.k8s.internal.smigorx.eu`: the app, a Valkey task
broker, and a CloudNativePG `Cluster` for PostgreSQL.

## Storage

| Claim | Size, class | Holds |
| --- | --- | --- |
| `paperless-data` | 50Gi, `local-path-core` | `media/` (the documents), `data/` (search index, classifier), `consume/`, `export/` |
| `Cluster/paperless-postgres` | 20Gi, `local-path-core` | tags, correspondents, metadata, users |

A restore needs both: the database without `media/` points at missing files,
and `media/` without the database is a pile of unnamed PDFs.

## Secrets

| Vault path | Secret | Keys |
| --- | --- | --- |
| `secret/paperless/postgres` | `paperless-postgres-app` | `username`, `password` |
| `secret/paperless/app` | `paperless-secret` | `PAPERLESS_SECRET_KEY` |
| `secret/paperless/oidc` | `paperless-oidc` | `config`, `client-secret` |

`paperless-postgres-ca` is deliberately not in that table: CloudNativePG owns
it and regenerates it.

    vault kv put secret/paperless/postgres username=paperless password=…
    vault kv put secret/paperless/app PAPERLESS_SECRET_KEY=…

## Authentik OIDC

Paperless reads its OIDC client from `PAPERLESS_SOCIALACCOUNT_PROVIDERS`, a
JSON blob for django-allauth, so `config` holds the whole thing with the client
secret inside it:

    {"openid_connect": {"APPS": [{
      "provider_id": "authentik",
      "name": "Authentik",
      "client_id": "paperless",
      "secret": "<client secret>",
      "settings": {"server_url": "https://auth.k8s.internal.smigorx.eu/application/o/paperless/.well-known/openid-configuration"}
    }]}}

The secret inside it must stay equal to `paperless-client-secret` in
`secret/authentik/oidc`. Write the Authentik half first, with `patch` — see
[authentik/README.md](../authentik/README.md#adding-an-oidc-application):

    vault kv patch secret/authentik/oidc paperless-client-secret=…
    vault kv put secret/paperless/oidc client-secret=… config=@config.json

`PAPERLESS_SOCIAL_AUTO_SIGNUP` creates a Paperless account on first login
through Authentik.
