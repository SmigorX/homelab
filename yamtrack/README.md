# Yamtrack

Argo CD deploys Yamtrack through `argocd/apps/workloads/yamtrack.yaml`.

## Required secret

Create the Secret before syncing the application:

    YAMTRACK_SECRET="$(head -c 64 /dev/urandom | base64 | tr -d '\n')"
    kubectl -n yamtrack create secret generic yamtrack-secrets \
      --from-literal=secret="$YAMTRACK_SECRET"

Back up this Secret. Replacing it invalidates sessions and other cryptographic
state.

## Authentik OIDC

The Authentik blueprint expects `authentik-oidc-secrets` in the `authentik`
namespace and `yamtrack-oidc-secrets` in the `yamtrack` namespace. Both use
the same confidential-client secret; create them before syncing Authentik or
Yamtrack.

First, generate the OIDC secret:

    YAMTRACK_OIDC_SECRET="$(head -c 64 /dev/urandom | base64 | tr -d '\n')"

Create the Authentik namespace secret:

    kubectl -n authentik create secret generic authentik-oidc-secrets \
      --from-literal=yamtrack-client-secret="$YAMTRACK_OIDC_SECRET"

Create the Yamtrack namespace secret using a temporary file to avoid shell escaping issues:

    cat > /tmp/SOCIALACCOUNT_PROVIDERS <<EOF
{"openid_connect":{"OAUTH_PKCE_ENABLED":true,"APPS":[{"provider_id":"authentik","name":"Authentik","client_id":"yamtrack","secret":"$YAMTRACK_OIDC_SECRET","settings":{"server_url":"https://auth.k8s.internal.smigorx.eu/application/o/yamtrack/.well-known/openid-configuration"}}]}}
EOF
    kubectl -n yamtrack create secret generic yamtrack-oidc-secrets \
      --from-file=SOCIALACCOUNT_PROVIDERS=/tmp/SOCIALACCOUNT_PROVIDERS
    rm /tmp/SOCIALACCOUNT_PROVIDERS

After both applications sync, Yamtrack's login page will offer Authentik.
Leave local authentication enabled initially so existing users can link their
Authentik identity under **Settings → Accounts**.

## Importing existing data

Export each user's tracked media from the existing Yamtrack UI as CSV. On the
new instance, create the matching user, open Import, select **Import from
Yamtrack**, and upload that CSV.

`yamtrack-data` and `yamtrack-redis-data` are persistent volumes. If either
PVC already contains data from a previous migration attempt, remove that PVC
before syncing when you want a completely fresh instance.
