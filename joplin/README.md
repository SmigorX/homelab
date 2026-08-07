# Joplin Server

This directory contains a native Kubernetes deployment for Joplin Server and
PostgreSQL. Argo CD deploys it through
`argocd/apps/workloads/joplin.yaml`.

## Required secret

The database password is intentionally not stored in Git. Create the Secret
before syncing the application:

    kubectl create namespace joplin --dry-run=client -o yaml | kubectl apply -f -
    JOPLIN_DB_PASSWORD="$(head -c 32 /dev/urandom | base64 -w 0)"
    kubectl -n joplin create secret generic joplin-secrets \
      --from-literal=postgres-password="$JOPLIN_DB_PASSWORD"

An Infisical-managed Secret named `joplin-secrets` with a
`postgres-password` key can be used instead.

PostgreSQL data is stored in the `data-joplin-postgres-0` PVC. Back up this
volume before upgrades or destructive maintenance.

After the first successful deployment, sign in at
`https://joplin.k8s.internal.smigorx.eu` with the upstream default
administrator credentials and change them immediately.
