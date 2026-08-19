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
