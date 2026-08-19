# Yamtrack

Argo CD deploys Yamtrack through `argocd/apps/workloads/yamtrack.yaml`.

## Required secret

Create the Secret before syncing the application:

    YAMTRACK_SECRET="$(head -c 64 /dev/urandom | base64 | tr -d '\n')"
    kubectl -n yamtrack create secret generic yamtrack-secrets \
      --from-literal=secret="$YAMTRACK_SECRET"

Back up this Secret. Replacing it invalidates sessions and other cryptographic
state.

## Importing existing data

Export each user's tracked media from the existing Yamtrack UI as CSV. On the
new instance, create the matching user, open Import, select **Import from
Yamtrack**, and upload that CSV.

`yamtrack-data` and `yamtrack-redis-data` are persistent volumes. If either
PVC already contains data from a previous migration attempt, remove that PVC
before syncing when you want a completely fresh instance.
