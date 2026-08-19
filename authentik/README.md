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

Keep Argo CD's local administrator enabled until you have confirmed a browser
login through Authentik. The OIDC redirect URI is
`https://argocd.k8s.internal.smigorx.eu/pkce/verify`; changing the external
hostname requires changing it in both the Argo CD values and the blueprint.
