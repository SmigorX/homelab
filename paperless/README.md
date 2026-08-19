Create a db secret:

```

    DB_SECRET="$(head -c 64 /dev/urandom | base64 | tr -d '\n')"
```

```
    kubectl -n paperless create secret generic paperless-postgres-app \
    --from-literal=username="paperless" \
    --from-literal=password="$DB_SECRET"
```

Create paperless secret

```
kubectl -n paperless create secret generic paperless-secret \
  --from-literal=PAPERLESS_SECRET_KEY="$(openssl rand -base64 64)"
```

Connect to oidc

```
PAPERLESS_OIDC_SECRET="$(openssl rand -base64 48)"

kubectl -n paperless create secret generic paperless-oidc \
  --from-literal=config="$(python -c 'import json, os; print(json.dumps({
    "openid_connect": {
      "APPS": [{
        "provider_id": "authentik",
        "name": "Authentik",
        "client_id": "paperless",
        "secret": os.environ["PAPERLESS_OIDC_SECRET"],
        "settings": {
          "server_url": "https://auth.k8s.internal.smigorx.eu/application/o/paperless/.well-known/openid-configuration"
        }
      }]
    }
  }))' PAPERLESS_OIDC_SECRET="$PAPERLESS_OIDC_SECRET")"
```
