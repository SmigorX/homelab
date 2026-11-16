# Yamtrack

Argo CD deploys Yamtrack and its Redis through
`argocd/apps/workloads/yamtrack.yaml`, published at
`yamtrack.k8s.internal.smigorx.eu`.

## Secrets

| Vault path | Secret | Keys |
| --- | --- | --- |
| `secret/yamtrack/app` | `yamtrack-secrets` | `secret` |
| `secret/yamtrack/oidc` | `yamtrack-oidc-secrets` | `SOCIALACCOUNT_PROVIDERS` |

`secret` is Django's secret key. Replacing it invalidates sessions and other
signed state.

    vault kv put secret/yamtrack/app secret=…

## Authentik OIDC

`SOCIALACCOUNT_PROVIDERS` is the whole django-allauth provider list as JSON,
with the client secret inside it:

    {"openid_connect": {"OAUTH_PKCE_ENABLED": true, "APPS": [{
      "provider_id": "authentik",
      "name": "Authentik",
      "client_id": "yamtrack",
      "secret": "<client secret>",
      "settings": {"server_url": "https://auth.k8s.internal.smigorx.eu/application/o/yamtrack/.well-known/openid-configuration"}
    }]}}

The Authentik provider is a public client with PKCE, but it still carries
`yamtrack-client-secret` from `secret/authentik/oidc`; keep the two equal.
Write the Authentik half first, with `patch` — see
[authentik/README.md](../authentik/README.md#adding-an-oidc-application).

Local authentication stays enabled, so an existing user can link their
Authentik identity under **Settings → Accounts**.

## Storage

`yamtrack-data` (5Gi) holds the SQLite database with everything tracked;
`yamtrack-redis-data` (1Gi) only the task queue. Both are `local-path-core`.

The image is `ghcr.io/fuzzygrim/yamtrack:latest`, so a pod restart can pull a
newer release.
