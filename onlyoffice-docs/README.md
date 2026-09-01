# OnlyOffice Docs

Argo CD deploys the ONLYOFFICE Document Server through
`argocd/apps/workloads/onlyoffice-docs.yaml`.

This runs the standalone `onlyoffice/documentserver` image (the same one the
original `docker-compose.yml` in this directory builds/pulls). It has no
database or cache of its own to wire up — it's a stateless conversion/editing
API that other apps (Nextcloud, a custom editor UI, etc.) call into.

## Required secret

JWT validation is enabled (`JWT_ENABLED=true`), so every request to the
Document Server must carry a token signed with the same secret. Create it
before syncing the application, and give the same value to whatever app(s)
will call this Document Server:

    kubectl create namespace onlyoffice-docs --dry-run=client -o yaml | kubectl apply -f -
    ONLYOFFICE_JWT_SECRET="$(head -c 48 /dev/urandom | base64 | tr -d '\n')"
    kubectl -n onlyoffice-docs create secret generic onlyoffice-docs-secret \
      --from-literal=JWT_SECRET="$ONLYOFFICE_JWT_SECRET"

Back up this secret's value — any integrated app needs it to talk to the
Document Server, and rotating it means updating every caller at the same time.

## Storage

`onlyoffice-data` is a single PVC, mounted with `subPath` for three of the
directories the upstream compose file declares as volumes:

- `/var/www/onlyoffice/Data` — instance certs/keys used to encrypt the cache;
  must persist across restarts.
- `/var/log/onlyoffice` — logs.
- `/var/lib/onlyoffice/documentserver/App_Data/cache/files` — conversion
  cache.

Two of the compose file's volumes are intentionally **not** mounted here:

- `/usr/share/fonts` — a Kubernetes PVC mount (unlike a Docker named/anonymous
  volume) starts empty and hides whatever the image ships at that path. An
  empty mount here would strip the container's built-in fonts and break text
  rendering in converted documents. If you need custom fonts, add an
  `initContainer` that seeds a volume from the image before the main
  container starts, rather than mounting straight over `/usr/share/fonts`.
- `/var/www/onlyoffice/documentserver-example/public/files` — backs the
  built-in `/welcome` demo page only; not needed for real usage.

## Access

Served at `https://onlyoffice.k8s.internal.smigorx.eu`, TLS terminated at
Traefik, backend plain HTTP (same pattern as the other apps in this repo).
There's no login page — access control is the JWT secret above plus whatever
network boundary `*.k8s.internal.smigorx.eu` already sits behind.
