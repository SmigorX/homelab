# Joplin Server

Argo CD deploys Joplin Server and its PostgreSQL through
`argocd/apps/workloads/joplin.yaml`, published at `joplin.internal.smigorx.eu`
— not under `k8s.internal` like everything else, and the SAML URLs on both
sides are tied to that hostname.

PostgreSQL here is a plain `StatefulSet` running `postgres:16`, not a
CloudNativePG `Cluster` like the other databases in this repo.

## Login

SAML through Authentik only: `LOCAL_AUTH_ENABLED` is `false`, so there is no
password form. The provider is the `joplin.yaml` blueprint in
`authentik/manifests/blueprints.yaml`.

Joplin reads the IdP's metadata from a file, so
[manifests/joplin/saml-config.yaml](manifests/joplin/saml-config.yaml) carries
a copy of it, including Authentik's self-signed signing certificate. **That
certificate expires on 7 August 2027.** When Authentik's certificate is
renewed, download the provider's metadata again (Authentik → Applications →
Providers → joplin → *Download* metadata) and replace `joplin-idp.xml`, or
logins fail signature validation.

## Secrets

| Vault path | Secret | Keys |
| --- | --- | --- |
| `secret/joplin/postgres` | `joplin-secrets` | `postgres-password` |

    vault kv put secret/joplin/postgres postgres-password=…

Both the database and Joplin read it. Changing it later does not change the
password already set inside PostgreSQL, which only reads
`POSTGRES_PASSWORD` when it initialises an empty volume.

## Storage

`data-joplin-postgres-0` (10Gi, `local-path-bulk`) holds every note. It is the
one volume to back up.
