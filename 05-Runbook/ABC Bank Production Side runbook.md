

## Label Nodes Worker node 1 and 2

```shell
oc label node <worker1> node-role.kubernetes.io/forgerock=""  
oc label node <worker2> node-role.kubernetes.io/forgerock=""
```

## Taint nodes (Optional)

```shell
oc adm taint node worker1 dedicated=forgerock:NoSchedule
oc adm taint node worker2 dedicated=forgerock:NoSchedule
```

## Create New project or namespace

```shell
oc new-project pngprod
```

## Create new sa and apply role and custom SCC

```shell
# 1. Create SA
oc apply -f bootstrap/cluster/forgerock-sa.yaml

# 2. Create SCC
oc apply -f bootstrap/cluster/forgerock-scc.yaml

# 3. Bind SCC to SA (THIS WAS MISSING!)
oc adm policy add-scc-to-user forgerock-scc -z forgerock-sa -n forgerock

# 4. Grant namespace admin (optional, or use custom role)
oc adm policy add-role-to-user admin -z forgerock-sa -n forgerock
```


## Import all images


### 1. Create new pvc for oc registry

Create and apply 'registry-pvc.yaml'
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage-pvc
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteMany # ◄ Uses RWX because your NFS supports it
  resources:
    requests:
      storage: 50Gi # Adjust size based on your requirements
  storageClassName: fast # ◄ Targets your NFS StorageClass
```

Apply the file
```bash
oc apply -f registry-pvc.yaml
```

### Patch registry storage 
```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed","storage":{"pvc":{"claim":"image-registry-storage-pvc"}}}}'
```

### Import Forgerock components 

```bash
# Core Platform Engines
oc import-image am:8.0.2 --from=us-docker.pkg.dev/forgeops-public/images/am:8.0.2 --confirm
oc import-image amster:8.0.2 --from=us-docker.pkg.dev/forgeops-public/images/amster:8.0.2 --confirm
oc import-image ds:8.0.2 --from=us-docker.pkg.dev/forgeops-public/images/ds:8.0.2 --confirm
oc import-image idm:8.0.1 --from=us-docker.pkg.dev/forgeops-public/images/idm:8.0.1 --confirm
oc import-image ig:8.0.2 --from=us-docker.pkg.dev/forgeops-public/images/ig:8.0.2 --confirm

# End User & Admin Interfaces
oc import-image admin-ui:8.0.1 --from=us-docker.pkg.dev/forgeops-public/images/admin-ui:8.0.1 --confirm
oc import-image end-user-ui:8.0.1 --from=us-docker.pkg.dev/forgeops-public/images/end-user-ui:8.0.1 --confirm
oc import-image login-ui:8.0.1 --from=us-docker.pkg.dev/forgeops-public/images/login-ui:8.0.1 --confirm

# Optional but Recommended: Public Utilities
oc import-image busybox:musl --from=docker.io/library/busybox:musl --confirm

```

Or apply below file
Create a file 'forgerock-Images.yaml'

```yaml
# =====================================================================
# FORGEROCK IMAGES - AUTOMATIC IMPORT DEFINITION
# =====================================================================

apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: am
  namespace: forgerock
spec:
  lookupPolicy:
    local: true # Allows local deployments to resolve this short-name image
  tags:
    - name: "8.0.2"
      from:
        kind: DockerImage
        name: us-docker.pkg.dev/forgeops-public/images/am:8.0.2
      referencePolicy:
        type: Source

---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: amster
  namespace: forgerock
spec:
  lookupPolicy:
    local: true
  tags:
    - name: "8.0.2"
      from:
        kind: DockerImage
        name: us-docker.pkg.dev/forgeops-public/images/amster:8.0.2
      referencePolicy:
        type: Source

---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: ds
  namespace: forgerock
spec:
  lookupPolicy:
    local: true
  tags:
    - name: "8.0.2"
      from:
        kind: DockerImage
        name: us-docker.pkg.dev/forgeops-public/images/ds:8.0.2
      referencePolicy:
        type: Source

---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: idm
  namespace: forgerock
spec:
  lookupPolicy:
    local: true
  tags:
    - name: "8.0.1"
      from:
        kind: DockerImage
        name: us-docker.pkg.dev/forgeops-public/images/idm:8.0.1
      referencePolicy:
        type: Source

---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: ig
  namespace: forgerock
spec:
  lookupPolicy:
    local: true
  tags:
    - name: "8.0.2"
      from:
        kind: DockerImage
        name: us-docker.pkg.dev/forgeops-public/images/ig:8.0.2
      referencePolicy:
        type: Source

---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: admin-ui
  namespace: forgerock
spec:
  lookupPolicy:
    local: true
  tags:
    - name: "8.0.1"
      from:
        kind: DockerImage
        name: us-docker.pkg.dev/forgeops-public/images/admin-ui:8.0.1
      referencePolicy:
        type: Source

---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: end-user-ui
  namespace: forgerock
spec:
  lookupPolicy:
    local: true
  tags:
    - name: "8.0.1"
      from:
        kind: DockerImage
        name: us-docker.pkg.dev/forgeops-public/images/end-user-ui:8.0.1
      referencePolicy:
        type: Source

---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: login-ui
  namespace: forgerock
spec:
  lookupPolicy:
    local: true
  tags:
    - name: "8.0.1"
      from:
        kind: DockerImage
        name: us-docker.pkg.dev/forgeops-public/images/login-ui:8.0.1
      referencePolicy:
        type: Source

---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: busybox
  namespace: forgerock
spec:
  lookupPolicy:
    local: true
  tags:
    - name: "musl"
      from:
        kind: DockerImage
        name: docker.io/library/busybox:musl
      referencePolicy:
        type: Source
        
```

### Apply Image pull permission to service account 

```bash
oc policy add-role-to-user system:image-puller system:serviceaccount:forgerock:default --namespace=pngprod
```

## Use the new image defaulter 

```bash
kind: Component
apiVersion: kustomize.config.k8s.io/v1alpha1
images:
- name: am
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/am
  newTag: 8.0.2
- name: am-custom
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/busybox
  newTag: musl
- name: amster
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/amster
  newTag: 8.0.2
- name: ds
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/ds
  newTag: 8.0.2
- name: idm
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/idm
  newTag: 8.0.1
- name: idm-custom
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/busybox
  newTag: musl
- name: ig
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/ig
  newTag: 8.0.2
- name: admin-ui
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/admin-ui
  newTag: 8.0.1
- name: end-user-ui
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/end-user-ui
  newTag: 8.0.1
- name: login-ui
  newName: image-registry.openshift-image-registry.svc:5000/forgerock-protected/login-ui
  newTag: 8.0.1
labels:
- includeSelectors: true
  pairs:
    app.kubernetes.io/part-of: identity-platform
namespace: forgerock

```

