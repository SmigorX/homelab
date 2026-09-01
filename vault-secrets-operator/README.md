# Vault Secrets Operator

Argo CD deploys this through `argocd/apps/platform/vault-secrets-operator.yaml`.

It's the HashiCorp-maintained controller that watches `VaultConnection` /
`VaultAuth` / `VaultStaticSecret` / `VaultDynamicSecret` CRs and syncs the
referenced Vault data into native Kubernetes `Secret` objects, keeping them
refreshed on a schedule. `values.yaml` declares a default `VaultConnection`
and `VaultAuth` (both named `default`) pointed at the in-cluster
[Vault](../vault/README.md) deployment, so most `VaultStaticSecret` CRs
elsewhere in this repo can omit `vaultAuthRef` entirely.

This operator does nothing useful on its own — Vault has to be initialized,
unsealed, and have its Kubernetes auth method configured first. See
`vault/README.md` for that one-time bootstrap and a working
`VaultStaticSecret` example.
