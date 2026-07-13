
## Cleanup Existing Integration
Remove old issuer and Vault secrets

```bash
kubectl delete clusterissuer vault-issuer
kubectl delete secret -n cert-manager vault-auth-token
kubectl delete secret -n cert-manager vault-ca
````

---

## Generate Kubernetes Auth Materials

Create service account token and cluster CA

```bash
kubectl -n cert-manager create token cert-manager > /tmp/cert-manager.jwt
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > /tmp/k8s-ca.crt
```

---

## Configure Vault Kubernetes Auth

Set API server, CA, and reviewer token

```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://api.kvm.lab:6443" \
  kubernetes_ca_cert=@/tmp/k8s-ca.crt \
  token_reviewer_jwt=@/tmp/cert-manager.jwt
```

---

## Create Vault Policy

Allow cert-manager to sign and issue certificates

```bash
vault policy write cert-manager /tmp/cert-manager-policy.hcl
```

---

## Create Vault Auth Role

Bind cert-manager service account to policy

```bash
vault write auth/kubernetes/role/cert-manager \
  bound_service_account_names="cert-manager" \
  bound_service_account_namespaces="cert-manager" \
  audience="vault" \
  policies="cert-manager" \
  ttl="1h"
```

---

## Create Vault Auth Secret in Kubernetes

Store JWT for cert-manager

```bash
kubectl -n cert-manager create token cert-manager --audience=vault > /tmp/cm.jwt

kubectl -n cert-manager create secret generic vault-auth-token \
  --from-file=token=/tmp/cm.jwt
```

---

## Configure Vault CA Trust

Trust Vault TLS certificate in cluster

```bash
kubectl -n cert-manager create secret generic vault-ca \
  --from-file=ca.crt=./kvm-root-ca.crt
```

---

## Update Reviewer Token (Fix Auth Issues)

Replace reviewer token used by Vault

```bash
kubectl -n cert-manager create token vault-reviewer > /tmp/reviewer.jwt

vault write auth/kubernetes/config \
  kubernetes_host="https://api.kvm.lab:6443" \
  kubernetes_ca_cert=@/tmp/k8s-ca.crt \
  token_reviewer_jwt=@/tmp/reviewer.jwt
```

---

## Generate Long-lived Tokens

Avoid frequent token expiration

```bash
kubectl -n cert-manager create token vault-reviewer \
  --duration=2160h > /tmp/reviewer.jwt

kubectl -n cert-manager create token cert-manager \
  --audience=vault \
  --duration=2160h > /tmp/cm.jwt
```

---

## Rotate Vault Auth Secret

Update JWT stored in cluster

```bash
kubectl -n cert-manager delete secret vault-auth-token

kubectl -n cert-manager create secret generic vault-auth-token \
  --from-file=token=/tmp/cm.jwt
```


---



