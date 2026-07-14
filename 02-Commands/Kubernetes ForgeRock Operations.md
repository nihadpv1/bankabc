
## Apply ForgeRock base configuration
Deploy base platform config

```bash
kubectl apply -k kustomize/overlay/lab/base
````

---

## Apply secrets and certificates

Deploy TLS and secret-agent configs

```bash
kubectl apply -k kustomize/overlay/lab/secrets
```

---

## Deploy DS components

Apply idrepo and cts StatefulSets

```bash
kubectl apply -k kustomize/overlay/lab/ds-idrepo
kubectl apply -k kustomize/overlay/lab/ds-cts
```

---

## Deploy AM

Apply AM overlay

```bash
kubectl apply -k kustomize/overlay/lab/am
```

---

## Restart AM

Restart deployment after changes

```bash
kubectl rollout restart deployment/am -n forgerock
```

---

## Apply password sync job

Sync DS passwords with secrets

```bash
kubectl apply -k kustomize/overlay/lab/ds-set-passwords
```

## See pod image in a namespace


```shell
kubectl get pods -n forgerock -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[*].image}{"\n"}{end}'
```


## See Secret agent configuration

See what secret agent ran in a namespace

```shell
kubectl get secretagentconfiguration -n forgerock
```

## See configuration of secret agent job

To see more details of a secret agent configuration job

```
kubectl describe secretagentconfiguration forgerock-sac -n forgerock
```

## See Secret passoword

To see password in secret normal text


```shell
kubectl get secret am-env-secrets -n forgerock \
  -o jsonpath='{.data.AM_PASSWORDS_AMADMIN_CLEAR}' | base64 -d
```


## Kubernetes current config

```shell
kubectl config current-context

# Get current config
kubectl config get-contexts

# Use a config
kubectl config use-context kubernetes-admin@kubernetes

```

## Apply config map

```shell
oc create configmap fr-proxy-config \  
--from-file=nginx.conf=nginx-proxy.conf \  
-n forgerock \  
--dry-run=client -o yaml | oc apply -f -

```


## Safe restart / roll out

```shell
oc rollout restart deploy/fr-proxy -n forgerock
```


## Secret extraction to yaml file

```shell
for secret in $(oc get secrets -n forgerock -o jsonpath='{.items[*].metadata.name}'); do
  oc get secret $secret -n forgerock -o json | yq 'del(.metadata.uid, .metadata.resourceVersion, .metadata.creationTimestamp, .metadata.generation, .metadata.managedFields, .metadata.namespace, .status)' > "${secret}.yaml"
done
```

## Argocd admin password extration

```yaml
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath="{.data.admin\.password}" | base64 -d ; echo
```