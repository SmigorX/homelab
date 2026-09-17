# Authentik GitOps configuration

The `authentik-blueprints` ConfigMap is mounted by the Authentik Helm chart.
Authentik discovers and reconciles each `.yaml` key as a blueprint.

## Enrolled applications

- **Joplin** uses SAML. Its provider retains the `joplin` application slug,
  audience, ACS URL, POST binding, and Authentik's built-in signing key. The
  private key remains in Authentik's database and is deliberately not stored in
  Git.
- **Argo CD** uses native OIDC with a public PKCE client. The provider has no
  client secret to manage. Members of Authentik's built-in `authentik Admins`
  group are mapped to Argo CD's administrator role; other authenticated users
  receive read-only access.

- **Immich** uses OIDC, but only half of the configuration lives here: Immich
  keeps its own OAuth settings in its database, typed into its admin interface.
  Its provider carries three redirect URIs, one of which is the custom scheme
  `app.immich:///oauth-callback` that the mobile apps return through. See
  [immich/README.md](../immich/README.md).

Keep Argo CD's local administrator enabled until you have confirmed a browser
login through Authentik. The OIDC redirect URI is
`https://argocd.k8s.internal.smigorx.eu/pkce/verify`; changing the external
hostname requires changing it in both the Argo CD values and the blueprint.

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

That is exactly what happened when Immich was enrolled. The fix is to run
discovery once the file is actually in the pod:

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
