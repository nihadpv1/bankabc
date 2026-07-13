
## Create oauth2 proxy secret
Store client credentials

```bash
kubectl create secret generic oauth2-proxy-secret \
  --from-literal=client-id=labapp \
  --from-literal=client-secret='XlPbU92RYJwKZi1N5ttuRQ==' \
  --from-literal=cookie-secret='XlPbU92RYJwKZi1N5ttuRQ==' \
  -n forgerock \
  --dry-run=client -o yaml | kubectl apply -f -
````

---

## Deploy oauth2 proxy app

Apply test application

```bash
kubectl apply -f kustomize/overlay/lab/oauth2-test
```

---

## Test token endpoint

Validate OAuth2 flow

```bash
curl -v -X POST "https://iam.apps.kvm.lab/am/oauth2/access_token" \
-d "grant_type=authorization_code" \
-d "client_id=labapp" \
-d "client_secret=labapp" \
-d "redirect_uri=https://app.apps.kvm.lab/oauth2/callback"
```

---