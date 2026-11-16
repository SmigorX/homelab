# OnlyOffice Docs

Argo CD deploys the ONLYOFFICE Document Server through
`argocd/apps/workloads/onlyoffice-docs.yaml`.

This runs the standalone `onlyoffice/documentserver` image. It has no
database or cache of its own to wire up — it is a stateless conversion and
editing API that Nextcloud calls into.

## Secret

JWT validation is enabled (`JWT_ENABLED=true`), so every request to the
Document Server must carry a token signed with the same secret.

| Vault path | Secret | Keys |
| --- | --- | --- |
| `secret/onlyoffice-docs/jwt` | `onlyoffice-docs-secret` | `JWT_SECRET` |

    vault kv put secret/onlyoffice-docs/jwt JWT_SECRET=…

Nextcloud syncs the same Vault path into its own namespace as
`nextcloud-onlyoffice`, so rotating it in Vault updates both; restart both
pods afterwards.

## Storage

`onlyoffice-data` is a single PVC, mounted with `subPath` for three of the
directories the image declares as volumes:

- `/var/www/onlyoffice/Data` — instance certs/keys used to encrypt the cache;
  must persist across restarts.
- `/var/log/onlyoffice` — logs.
- `/var/lib/onlyoffice/documentserver/App_Data/cache/files` — conversion
  cache.

Two of the image's volumes are intentionally **not** mounted here:

- `/usr/share/fonts` — a PVC mount starts empty and hides whatever the image
  ships at that path, which would strip the built-in fonts and break text
  rendering in converted documents. For custom fonts, add an `initContainer`
  that seeds a volume from the image before the main container starts, rather
  than mounting straight over `/usr/share/fonts`.
- `/var/www/onlyoffice/documentserver-example/public/files` — backs the
  built-in `/welcome` demo page only; not needed for real usage.

## Access

Served at `https://onlyoffice.k8s.internal.smigorx.eu`, TLS terminated at
Traefik, backend plain HTTP (same pattern as the other apps in this repo).
There's no login page — access control is the JWT secret above plus whatever
network boundary `*.k8s.internal.smigorx.eu` already sits behind.
