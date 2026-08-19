# Yamtrack

Argo CD deploys Yamtrack through `argocd/apps/workloads/yamtrack.yaml`.

## First migration

1. Stop the existing Docker Compose stack. SQLite must not be copied while
   Yamtrack is writing to it.
2. Confirm that `/home/smigorx/docker/yamtrack/db/db.sqlite3` exists on the
   Kubernetes node. The migration Job mounts this directory read-only.
3. Sync the `yamtrack` Argo CD application. The early sync migration Job copies
   the directory into the `yamtrack-data` PVC exactly once, then writes a
   `.yamtrack-migration-complete` marker.
4. Confirm the application works at
   `https://yamtrack.internal.smigorx.eu` before deleting the Docker data.

The source path is intentionally explicit in the migration Job. If the cluster
has more than one node, add a node selector to both the migration Job and the
Yamtrack Deployment so the host source and the local-path PVC stay on the same
node.

## Secret

The Yamtrack Pod's `initialize-secret` init container creates
`yamtrack-secrets` with a cryptographically random `secret` value only when it
does not already exist. It then passes the value to Yamtrack using the
upstream-supported `SECRET_FILE` setting. This avoids a separate initialization
Job and its ServiceAccount scheduling dependency. The ServiceAccount's
least-privilege RBAC is created in sync wave `-5`, before the Deployment at
wave `1`.

The init container never logs or replaces the value. Back up this Secret:
changing it invalidates signed sessions and other cryptographic state.
