### Recommended Way (using kubectl -k)

Go to your overlay directory and apply in this strict order:

```shell
cd kustomize/overlay/lab

# 1. Foundation
kubectl apply -k base
kubectl apply -k image-defaulter
kubectl apply -k secrets
kubectl apply -k keystore-create

# 2. Directory Services
kubectl apply -k ds-idrepo
kubectl apply -k ds-cts
kubectl apply -k ds-set-passwords

# 3. Core
kubectl apply -k am
kubectl apply -k amster
kubectl apply -k idm

# 4. UIs
kubectl apply -k admin-ui
kubectl apply -k end-user-ui
kubectl apply -k login-ui

```

**Tip**: You can also apply multiple at once if the order is correct inside your main kustomization.yaml.

Would you like me to give you the **best resources: order** to put inside your kustomize/overlay/lab/kustomization.yaml so that a single kubectl apply -k . works reliably?