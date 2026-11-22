# Vault

Argo CD deploys Vault through `argocd/apps/platform/vault.yaml`, and the
[Vault Secrets Operator](../vault-secrets-operator/README.md) (VSO) through
`argocd/apps/platform/vault-secrets-operator.yaml`. Together they let a
`VaultStaticSecret` CR sync a secret stored in Vault into a native
Kubernetes `Secret`. Every app's secrets come through this, from its
`manifests/vault-secret.yaml`.

Vault itself runs in "standalone" mode: one replica, Shamir-sealed, file
storage backend on the `data-vault-0` PVC (`local-path-core`). A sidecar
unseals it after every restart from keys kept in a Kubernetes Secret — see
[Auto-unseal](#auto-unseal).

## One-time bootstrap

None of this can be automated safely: `vault operator init` prints the root
token and unseal keys exactly once, and you're the only one who should see
them. Do this after `vault-0` is `Running` (it starts sealed, which the
readiness probe deliberately tolerates — see the last section):

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

**Every Vault pod restart re-seals it.** The `auto-unseal` sidecar undoes
that within ten seconds once the `vault-unseal-keys` Secret exists; create it
now, as [Auto-unseal](#auto-unseal) describes. Without it, a node reboot or pod
eviction means Vault serves nothing — including to the Vault Secrets
Operator — until someone runs `vault operator unseal` three more times.
Existing Secrets stay as they are, but nothing new or rotated syncs, and VSO's
retries rewrite every `VaultStaticSecret`'s status in a loop that loads the API
server. The `VaultSecretsOperatorFailing` alert in monitoring/ posts to Discord
when this has gone on for 20 minutes.

Log in and enable the Kubernetes auth method:

```
kubectl -n vault exec -it vault-0 -- sh -c '
  vault login &&
  vault auth enable kubernetes &&
  vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
'
```

That last write works without any extra RBAC because `server.authDelegator.enabled: true`
in `values.yaml` already binds the Vault service account to
`system:auth-delegator`, which is what lets it call the TokenReview API.

### Do not set `token_reviewer_jwt`

It is tempting to add `token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token`
to that write, and this repo did until it broke. That file is a *projected*
token bound to the Vault pod, and Kubernetes invalidates it the moment that
pod is replaced — but Vault keeps the copy it was given forever. Leave the
field unset and Vault reads the token from its own filesystem on each request
instead, which kubelet keeps current across restarts.

The failure this causes is slow and misleading. Vault's TokenReview calls
start being rejected, which it reports to clients as nothing more specific
than:

    URL: PUT http://vault.vault.svc.cluster.local:8200/v1/auth/kubernetes/login
    Code: 403. Errors:

    * permission denied

Nothing breaks at the moment of the restart, because the Vault Secrets
Operator renews the tokens it already holds — so every existing
`VaultStaticSecret` keeps syncing while *new* ones fail, and the problem looks
like it belongs to whichever application was added last. It is not about that
application, its namespace, or the role's bindings.

To confirm before changing anything, review a token by hand the way Vault
would. This authenticates even while login is failing, which is the tell:

    REVIEWER=$(kubectl -n vault create token vault --duration=10m)
    TARGET=$(kubectl -n <namespace> create token default --duration=10m)

    cat > /tmp/tr.json <<EOF
    {"apiVersion":"authentication.k8s.io/v1","kind":"TokenReview","spec":{"token":"$TARGET"}}
    EOF

    kubectl --token="$REVIEWER" create --raw \
      /apis/authentication.k8s.io/v1/tokenreviews -f /tmp/tr.json

`"authenticated": true` there, against a login that answers 403, means the
credential Vault holds is the broken part rather than anything about the
namespace, the service account or the role.

The fix is to clear the stored copy. The empty string is required — omitting
the field leaves the old value in place:

    vault write auth/kubernetes/config \
      kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT" \
      kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
      token_reviewer_jwt=""

Enable a KV v2 engine and create the policy and role behind the `default`
`VaultAuth` that `vault-secrets-operator/values.yaml` declares:

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
    bound_service_account_namespaces="*" \
    policies=default-read \
    ttl=24h
'
```

Every `VaultStaticSecret` in this repo omits `vaultAuthRef`, so VSO logs in
through this one role as the `default` ServiceAccount **of the secret's own
namespace**: the role has to cover every app namespace, and every app can
read every path under `secret/`. `vault read auth/kubernetes/role/default`
shows what the live role is bound to. The tighter alternative, not done here,
is a policy per app scoped to `secret/data/<app>/*`, a role bound to that
app's namespace, and a `vaultAuthRef` to a matching `VaultAuth`.

## Auto-unseal

Vault cannot be stopped from sealing: the root key lives only in memory, so a
restarted process always starts sealed. What can be automated is the unseal.
The supported auto-unseal mechanisms all want something external to hold the
key — a cloud KMS, an HSM, or another Vault for Transit — and this cluster has
none of those, so `values.yaml` adds a sidecar instead. Every ten seconds it
asks `/v1/sys/seal-status`, and if Vault is sealed, posts each key under
`/unseal` to `/v1/sys/unseal`.

The keys come from a Secret that is created by hand and never committed. It
cannot come from Vault through VSO, since a sealed Vault serves nothing. Three
keys, the Shamir threshold, are enough; keep all five, and the root token,
wherever they were stored at bootstrap. In fish, so that the keys stay out of
shell history:

    read -s -P 'unseal key 1: ' k1
    read -s -P 'unseal key 2: ' k2
    read -s -P 'unseal key 3: ' k3
    kubectl -n vault create secret generic vault-unseal-keys \
      --from-literal=key1=$k1 --from-literal=key2=$k2 --from-literal=key3=$k3
    set -e k1 k2 k3

The sidecar mounts the Secret as optional, so Vault still starts without it;
the sidecar then logs that it has nothing to unseal with, and the manual
unseal above still works. Check what it did with:

    kubectl -n vault logs vault-0 -c auto-unseal

**This gives up what sealing is for.** Anyone who can read that Secret — or
the k3s datastore on `fedora-server`, where Secrets are stored unencrypted
unless k3s runs with `--secrets-encryption` — can unseal Vault. On this
cluster that is the same set of people who could read Vault's data off the
`data-vault-0` volume anyway, which is why it is acceptable here. Revert it by
deleting the Secret, the `extraContainers` and `volumes` blocks in
`values.yaml`, and the pod.

## Adding a secret

Write it to Vault, then copy any app's `manifests/vault-secret.yaml` — they all
use `type: kv-v2`, `mount: secret` and a `path` relative to it. VSO syncs it
within `refreshAfter` (an hour everywhere here).

## UI / external access

Served at `https://vault.k8s.internal.smigorx.eu`, same Traefik + cert-manager
+ homepage pattern as everything else in this repo, TLS terminated at the
ingress with a plain-HTTP backend (`global.tlsDisable: true`). The Vault UI is
reachable there too — it is switched on by `ui = true` inside the standalone
HCL config, not by the chart's `ui.enabled`, which stays `false` because it
governs a separate LoadBalancer Service this cluster does not need. None of
this is required for the bootstrap steps above: `kubectl exec` works before the
ingress exists.

## A sealed Vault still reports ready

`values.yaml` replaces the chart's default readiness probe. The default execs
`vault status`, which exits non-zero while Vault is sealed, so the pod goes
`0/1` and drops out of its Service. In a real HA cluster that is right: a
sealed node should stop taking traffic so its peers serve instead. Here there
are no peers, so the only effect is that the web UI — the one thing that can
unseal it — goes away exactly when it is needed.

The replacement asks Vault over HTTP instead, and remaps the two states that
are not faults:

    /v1/sys/health?standbyok=true&sealedcode=204&uninitcode=204

Vault answers `503` when sealed and `501` when uninitialized; the query
parameters turn both into `204`, which a kubelet probe counts as success. A
Vault that cannot answer at all still fails the probe, which is what a
readiness probe is actually for.

**This costs a signal.** `kubectl get pods` now shows `1/1` for a sealed Vault,
so it no longer tells you the seal state. Ask Vault directly:

    kubectl -n vault exec vault-0 -- vault status

Any change to `values.yaml` needs a manual pod delete to take effect:
`updateStrategyType: OnDelete` means the StatefulSet never rolls on its own,
because a replaced pod comes up sealed.

    kubectl -n vault delete pod vault-0

The new pod comes up `1/1` while sealed, and the web UI answers, so it can be
unsealed from the browser.

Worth knowing: `server.service.publishNotReadyAddresses` is `true` in this
chart by default, and the EndpointSlice controller forces an endpoint's `ready`
condition to `true` whenever that is set. So routing to a sealed Vault was
arguably supposed to work already. The probe change makes it not depend on
that, and makes `kubectl` agree with what the Service is doing either way.
