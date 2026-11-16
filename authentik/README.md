# Authentik

Argo CD deploys the Authentik chart through
`argocd/apps/platform/authentik.yaml`, with its bundled PostgreSQL. Everything
Authentik serves is declared as blueprints: the `authentik-blueprints`
ConfigMap in [manifests/blueprints.yaml](manifests/blueprints.yaml) is mounted
by the chart, and Authentik reconciles each `.yaml` key as one blueprint.

## Applications

| Blueprint key | Protocol | Who gets in |
| --- | --- | --- |
| `argocd.yaml` | OIDC, public PKCE client | everyone; `authentik Admins` → Argo CD admin, others read-only |
| `joplin.yaml` | SAML | everyone |
| `yamtrack.yaml` | OIDC, public client | everyone |
| `paperless.yaml` | OIDC | everyone |
| `vault.yaml` | OIDC | everyone; Vault policies decide the rest |
| `nextcloud.yaml` | OIDC | everyone; Authentik's `admin` group → Nextcloud admin |
| `miniflux.yaml` | OIDC | everyone |
| `immich.yaml` | OIDC | everyone |
| `forgejo.yaml` | OIDC | everyone |
| `grafana.yaml` | OIDC | `authentik Admins` only |
| `uptime-kuma.yaml` | forward auth, domain level | `guests` and `authentik Admins` |
| `arr-stack.yaml` | forward auth, one `forward_single` provider per app | `authentik Admins` only |
| `proxy-outpost.yaml` | — | the embedded outpost's provider list |

"Everyone" means every Authentik account: an application with no policy
binding admits any authenticated user, and the app's own roles take it from
there.

Some apps keep half of their setup outside this repo:

- **Joplin** signs with Authentik's built-in self-signed key; the private key
  stays in Authentik's database, not in Git.
- **Argo CD** has no client secret. Its redirect URI is
  `https://argocd.k8s.internal.smigorx.eu/pkce/verify`; changing the hostname
  means changing it in both `argocd/values.yaml` and the blueprint.
- **Immich** keeps its OAuth settings in its own database, typed into its
  admin UI. Its provider carries three redirect URIs, one of them the custom
  scheme `app.immich:///oauth-callback` the mobile apps return through. See
  [immich/README.md](../immich/README.md).
- **Forgejo** is fully declarative: the chart reconciles the authentication
  source from `forgejo/values.yaml` on every pod start. The source's name,
  `authentik`, is part of the redirect URI. See
  [forgejo/README.md](../forgejo/README.md).
- **The forward-auth apps** have no SSO of their own. Traefik asks Authentik
  about each request through a Middleware in the app's namespace. The arr
  apps' providers also need `/outpost.goauthentik.io/` routed on their own
  hosts, which [manifests/outpost-ingress.yaml](manifests/outpost-ingress.yaml)
  does; see [arr-stack/README.md](../arr-stack/README.md).

## Adding an OIDC application

1. Generate a client secret and add it to Vault **first**, with `patch` (that
   path holds every other client's secret too, and `put` would replace them):

       vault kv patch secret/authentik/oidc <app>-client-secret=…

   Plus the app's own copy, usually `secret/<app>/oidc`.
2. Wait for VSO to copy it (`refreshAfter: 1h`), then check:

       kubectl -n authentik get secret authentik-oidc-secrets \
         -o jsonpath='{.data.<app>-client-secret}' | wc -c     # non-zero

   A `secretKeyRef` to a key that is not there yet leaves both Authentik pods
   in `CreateContainerConfigError`, and SSO goes down for everything.
3. Add the `<APP>_OIDC_CLIENT_SECRET` variable to [values.yaml](values.yaml)
   and the blueprint key to `blueprints.yaml`, reading it with `!Env`. Create
   new providers with `grant_types` set explicitly (see the Nextcloud entry).
4. Check the blueprint applied, as below. The old worker may see the new file
   before it has the new variable, fail, and not retry until the file changes.

A forward-auth app also needs its provider added to `proxy-outpost.yaml`; a
missing entry does not error, the outpost just stops answering for that host.

## A new blueprint key does not apply itself

Adding a key to the ConfigMap is not enough, and the failure is silent: the
application simply does not exist, and its OIDC discovery URL answers `404`
while every other one answers `200`.

Two things have to happen and neither is immediate. The kubelet re-projects the
ConfigMap volume on its own sync period, so the file appears in the pod up to a
minute after Argo CD reports the sync as done. Authentik then has to run its
`blueprints_discovery` task, which is what creates a `BlueprintInstance` for a
newly seen file. Discovery runs on a timer roughly hourly — and a restart does
not reliably substitute for it, because a rollout triggered by the same commit
races the volume update: the task fires while the pod still has the old
projection, finds nothing new, and does not run again for an hour.

The fix is to run discovery once the file is actually in the pod:

    kubectl -n authentik exec deploy/authentik-worker -- \
      ls /blueprints/mounted/cm-authentik-blueprints/

    kubectl -n authentik exec deploy/authentik-worker -- ak shell -c \
      'from authentik.blueprints.v1.tasks import blueprints_discovery; blueprints_discovery.send()'

Then confirm the instance was created and applied:

    kubectl -n authentik exec deploy/authentik-worker -- ak shell -c \
      'from authentik.blueprints.models import BlueprintInstance
       print([(b.name, b.status) for b in BlueprintInstance.objects.all()])'

`successful` is the status to look for. A blueprint that parses but fails to
apply shows `error` here, which is the other thing worth checking before
blaming the application's own configuration.

The admin UI shows the same under *Customization → Blueprints*, where a failed
one can also be applied by hand.
