# Vault

Argo CD deploys Vault through `argocd/apps/platform/vault.yaml`, and the
[Vault Secrets Operator](../vault-secrets-operator/README.md) (VSO) through
`argocd/apps/platform/vault-secrets-operator.yaml`. Together they let a
`VaultStaticSecret` CR sync a secret stored in Vault into a native
Kubernetes `Secret` — replacing the manual `kubectl create secret` step
every other app in this repo currently documents in its own README.

Vault itself runs in "standalone" mode: one replica, Shamir-sealed, file
storage backend on the `data-vault-0` PVC (`local-path-core`). No
auto-unseal is configured, which matters operationally — see below.

## One-time bootstrap

None of this can be automated safely: `vault operator init` prints the root
token and unseal keys exactly once, and you're the only one who should see
them. Do this after `vault-0` is `Running` (it starts sealed and will fail
its readiness check until unsealed — that's expected):

```
kubectl -n vault exec -it vault-0 -- vault operator init
```

Store the 5 unseal keys and the root token somewhere durable and *not* in
this repo (a password manager, printed and locked away, etc.) — losing all
of the unseal keys means the data on that PVC is unrecoverable.

Unseal (repeat with 3 different keys — default Shamir threshold is 3-of-5):

```
kubectl -n vault exec -it vault-0 -- vault operator unseal
```

**Every Vault pod restart re-seals it.** Until auto-unseal (e.g. via a
cloud KMS or Vault Transit) is set up, a node reboot or pod eviction means
Vault serves nothing — including to the Vault Secrets Operator — until
someone runs `vault operator unseal` three more times. Worth knowing before
anything else in the cluster comes to depend on it.

Log in and enable the Kubernetes auth method:

```
kubectl -n vault exec -it vault-0 -- sh -c '
  vault login &&
  vault auth enable kubernetes &&
  vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
'
```

That last write works without any extra RBAC because `server.authDelegator.enabled: true`
in `values.yaml` already binds the Vault service account to
`system:auth-delegator`, which is what lets it call the TokenReview API.

Enable a KV v2 engine and create an example policy + role matching the
`default` `VaultAuth` that `vault-secrets-operator/values.yaml` declares:

```
kubectl -n vault exec -it vault-0 -- sh -c '
  vault secrets enable -path=secret kv-v2 &&
  vault policy write default-read - <<EOF
path "secret/data/*" {
  capabilities = ["read"]
}
EOF
  vault write auth/kubernetes/role/default \
    bound_service_account_names=default \
    bound_service_account_namespaces=default \
    policies=default-read \
    ttl=24h
'
```

**This `default` role is deliberately broad** (any pod using the `default`
service account in the `default` namespace) — good enough to prove the
pipeline works end-to-end, not something to leave in place once you start
migrating real app secrets. For each app, prefer a dedicated Vault policy
scoped to just its own `secret/data/<app>/*` path, plus a matching role
bound to that app's own ServiceAccount and namespace — then point its
`VaultStaticSecret` at a `vaultAuthRef` other than `default`.

## Trying it out

```
kubectl -n vault exec -it vault-0 -- vault kv put secret/demo username=foo password=bar
```

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: demo
  namespace: default
spec:
  path: secret/data/demo
  destination:
    name: demo-secret
    create: true
  refreshAfter: 1h
```

Apply that and `kubectl get secret demo-secret -o yaml` should show `username`/`password`
populated from Vault, kept in sync every `refreshAfter`.

## UI / external access

Served at `https://vault.k8s.internal.smigorx.eu`, same Traefik + cert-manager
+ homepage pattern as everything else in this repo, TLS terminated at the
ingress with a plain-HTTP backend (`global.tlsDisable: true`). The Vault UI
(`ui.enabled: true`) is reachable there too. None of this is needed for the
bootstrap steps above — `kubectl exec` works before the ingress exists.
