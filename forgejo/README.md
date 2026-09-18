# Forgejo

Argo CD deploys the upstream Forgejo Helm chart through
`argocd/apps/workloads/forgejo.yaml`, published at
`forgejo.k8s.internal.smigorx.eu`. [values.yaml](values.yaml) holds overrides
only; the chart's own `values.yaml` is the reference.

## Storage

Two halves, and a restore needs both:

- `forgejo-data` (20Gi, `local-path-core`) — the Git repositories themselves,
  LFS objects, packages, and the `app.ini` the init container generates. That
  file holds `SECRET_KEY`, `INTERNAL_TOKEN`, `JWT_SECRET` and `LFS_JWT_SECRET`,
  which are generated once on first boot and are **not** in Git or Vault:
  losing this volume invalidates every existing session and LFS token even if
  the database survives.
- `Cluster/forgejo-postgres` (10Gi, `local-path-core`) — users, issues, pull
  requests, sessions and the authentication sources.

The claim is named `forgejo-data` rather than the chart's default
`gitea-shared-storage`. Worth knowing before changing it: the PVC is annotated
`helm.sh/resource-policy: keep`, so renaming it creates an empty volume and
leaves the repositories in the old claim.

`persistence.storageClass` is set explicitly and must stay that way. Without
it the claim falls through to k3s's built-in `local-path` StorageClass, which
is node-local and `reclaimPolicy: Delete` — not the NAS.

## Secrets

| Vault path | Secret | Keys |
| --- | --- | --- |
| `secret/forgejo/postgres` | `forgejo-postgres-app` | `username`, `password` |
| `secret/forgejo/admin` | `forgejo-admin` | `username`, `password` |
| `secret/forgejo/oidc` | `forgejo-oidc` | `key`, `secret` |

`forgejo-postgres-ca` is deliberately not in that table: CloudNativePG owns it
and regenerates it.

Write all of them **before** the first sync, and remember Vault re-seals itself
on every pod restart and needs three manual unseals before anything syncs at
all — see [vault/README.md](../vault/README.md).

    vault kv patch secret/authentik/oidc forgejo-client-secret=…

    vault kv put secret/forgejo/oidc key=forgejo secret=…   # same value
    vault kv put secret/forgejo/postgres username=forgejo password=…
    vault kv put secret/forgejo/admin username=… password=…

The Authentik half goes first, and with `patch`, not `put`:
`secret/authentik/oidc` already holds the client secrets for the other
applications, and `put` replaces a path wholesale. A `secretKeyRef` to a key
that does not exist yet leaves both Authentik pods in
`CreateContainerConfigError`, which takes SSO down for everything here.

### Why the admin password is not the chart's

Left to itself the chart generates one through bitnami's
`common.secrets.passwords.manage`, which preserves the existing password by
calling `lookup`. Argo CD renders with `helm template` and no cluster access,
so `lookup` returns nothing and **every render produces a different password**.
The Secret is then permanently OutOfSync, `selfHeal` rewrites it, and the
default `passwordMode: keepUpdated` makes the init container reset the admin
account to the new value on every pod start. `gitea.admin.existingSecret`
avoids all of that; `keepUpdated` is then useful, because rotating the password
in Vault propagates on the next restart.

## TLS, and why `ingress.tls` is not optional

The chart derives `ROOT_URL` from the ingress, and its `gitea.public_protocol`
helper returns `https` only when `ingress.tls` is non-empty. With an empty
list — even behind Traefik, which terminates TLS regardless — Forgejo would
advertise `http://` in clone URLs, webhook targets, e-mail links and OAuth
redirect URIs, and the Authentik login would fail on a redirect-URI mismatch.

## Git over SSH

Traefik only forwards HTTP(S), so the ssh Service is a `LoadBalancer` on
`192.168.1.14` from the MetalLB `ingress` pool (`.11` Traefik, `.12` samba,
`.13` mosquitto). The pool is `autoAssign: false`, so the address is requested
by annotation, as in `samba/manifests/svc.yaml`.

`gitea.config.server.SSH_DOMAIN` is that address rather than the web hostname.
`SSH_DOMAIN` otherwise follows `DOMAIN`, which resolves to Traefik, where
nothing listens on 22 — the UI would print a clone command that cannot work.
Point it at a DNS name instead if one is ever added for `.14`.

The container runs the rootless image and listens on 2222; the Service maps 22
onto it. That also explains the `containerSecurityContext` block in
`values.yaml`: it is safe only while `image.rootless` keeps its default.

## Authentik single sign-on

Both halves are declarative. The provider and application live in
`authentik/manifests/blueprints.yaml` under the `forgejo.yaml` key, and the
authentication source itself is in `gitea.oauth` in [values.yaml](values.yaml)
— the chart's init container runs `gitea admin auth add-oauth`, or
`update-oauth` if a source of that name exists, on every pod start.

Three values are easy to get subtly wrong:

- `autoDiscoverUrl` is the **full** well-known URL
  (`…/application/o/forgejo/.well-known/openid-configuration`). Miniflux wants
  the issuer only and appends the rest itself; Forgejo does not.
- The source name (`authentik`) determines the callback URL,
  `/user/oauth2/authentik/callback`, which the blueprint must allow verbatim.
- The application slug (`forgejo`) is what puts `forgejo` in the discovery URL
  above. Renaming it breaks discovery, not just the launch link.

Registration is handled with `ALLOW_ONLY_EXTERNAL_REGISTRATION`, not
`DISABLE_REGISTRATION`. The latter also blocks account creation through OAuth2,
which would defeat `oauth2_client.ENABLE_AUTO_REGISTRATION`. Local password
sign-in stays enabled so the Vault-backed admin remains the escape hatch; keep
it working until a browser login through Authentik is confirmed.
`ACCOUNT_LINKING: auto` matches an incoming Authentik identity to an existing
local account by verified e-mail, so `gitea.admin.email` should be the address
of the Authentik account that is meant to own the admin user.

## A new blueprint key does not apply itself

Adding `forgejo.yaml` to the ConfigMap is not enough and the failure is silent:
the OIDC discovery URL answers `404` and Forgejo's login button fails. Run the
discovery task by hand — see
[authentik/README.md](../authentik/README.md#a-new-blueprint-key-does-not-apply-itself).
