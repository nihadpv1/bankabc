

# Image Registry
## 1. Patch Storage class to allow expansion in future 

>if done already skip this

```bash
oc patch storageclass png-nfs-static -p '{"allowVolumeExpansion": true}'
```

## 2. Create a PVC for Image registry

```bash
vi image-registry-pvc
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage-pvc
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: png-nfs-static
```

## 3. Create PV for PVC

```bash
vi image-registry-pv
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-image-registry-nfs
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: png-nfs-static
  nfs:
    path: /pngprdir   # Update with your actual NFS path
    server: 10.150.161.189      # Update with your actual NFS server IP/host
  claimRef:
    namespace: openshift-image-registry
    name: image-registry-storage-pvc
```

1. Run `oc apply -f pv-image-registry.yaml`.
    
2. Verify the status changes to `Bound`:

```bash
  oc get pvc image-registry-storage-pvc -n openshift-image-registry
```

## 4. Patch storage backend to  Mount pvc as image registry storage

```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed","storage":{"pvc":{"claim":"image-registry-storage-pvc"}}}}'
```

## Final Step: Verify the Registry Pod Health

To confirm that the OpenShift Image Registry is actually mounted and running happily on your new NFS storage, run these two quick verification commands:

#### 1. Check the Image Registry Pods

Bash

```
oc get pods -n openshift-image-registry
```

_You should see `image-registry-xxxxx` pods transitioning to **`1/1 Running`**._

#### 2. Check the Operator Status

Bash

```
oc get configs.imageregistry.operator.openshift.io cluster
```

_The status should show **`MANAGED`** and **`Available: True`**._

# PNG-PROD-IMAGES
This namespace is specifically for forgerock image

## 1. Create new namespace

```bash
oc new-project png-prod-images
```


## 2. Give permission to pull

```bash
oc policy add-role-to-group system:image-puller system:serviceaccounts:pngprd -n png-prod-images

#Or single account

oc policy add-role-to-user system:image-puller system:serviceaccount:pngprd:pngprd-sa --n png-prod-images
```

# 3. Pull image

```shell
#change to png-prod-images project
oc project png-prod-images
#create image streams 
oc create is idm
#tag images
oc tag us-docker.pkg.dev/forgeops-public/images/idm:8.1.1 idm:8.1.1
#Import images
oc import-image idm:8.0.1 --from=us-docker.pkg.dev/forgeops-public/images/idm:8.1.1 --confirm
```

## Full images pull

```bash
# 1. Switch to the image repository project
oc project png-prod-images


# ==========================================
# 2. CREATE IMAGESTREAMS
# ==========================================
oc create is am
oc create is amster
oc create is ds
oc create is idm
oc create is ig
oc create is admin-ui
oc create is end-user-ui
oc create is login-ui
oc create is busybox


# ==========================================
# 3. TAG & IMPORT FORGEROCK STACK IMAGES
# ==========================================

# AM (Access Management)
oc tag us-docker.pkg.dev/forgeops-public/images/am:8.1.1 am:8.1.1
oc import-image am:8.1.1 --from=us-docker.pkg.dev/forgeops-public/images/am:8.1.1 --confirm

# AMSTER
oc tag us-docker.pkg.dev/forgeops-public/images/amster:8.0.2 amster:8.0.2
oc import-image amster:8.0.2 --from=us-docker.pkg.dev/forgeops-public/images/amster:8.0.2 --confirm

# DS (Directory Services / CTS / IDRepo)
oc tag us-docker.pkg.dev/forgeops-public/images/ds:8.0.2 ds:8.0.2
oc import-image ds:8.0.2 --from=us-docker.pkg.dev/forgeops-public/images/ds:8.0.2 --confirm

# IDM (Identity Management)
oc tag us-docker.pkg.dev/forgeops-public/images/idm:8.1.1 idm:8.1.1
oc import-image idm:8.1.1 --from=us-docker.pkg.dev/forgeops-public/images/idm:8.1.1 --confirm

# IG (Identity Gateway)
oc tag us-docker.pkg.dev/forgeops-public/images/ig:8.0.2 ig:8.0.2
oc import-image ig:8.0.2 --from=us-docker.pkg.dev/forgeops-public/images/ig:8.0.2 --confirm

# ADMIN UI
oc tag us-docker.pkg.dev/forgeops-public/images/admin-ui:8.1.1 admin-ui:8.1.1
oc import-image admin-ui:8.1.1 --from=us-docker.pkg.dev/forgeops-public/images/admin-ui:8.1.1 --confirm

# END USER UI
oc tag us-docker.pkg.dev/forgeops-public/images/end-user-ui:8.0.1 end-user-ui:8.0.1
oc import-image end-user-ui:8.0.1 --from=us-docker.pkg.dev/forgeops-public/images/end-user-ui:8.0.1 --confirm

# LOGIN UI
oc tag us-docker.pkg.dev/forgeops-public/images/login-ui:8.0.1 login-ui:8.0.1
oc import-image login-ui:8.0.1 --from=us-docker.pkg.dev/forgeops-public/images/login-ui:8.0.1 --confirm


# ==========================================
# 4. TAG & IMPORT BUSYBOX (for am-custom & idm-custom init containers)
# ==========================================
oc tag docker.io/library/busybox:musl busybox:musl
oc import-image busybox:musl --from=docker.io/library/busybox:musl --confirm
```

# 4. Example image defaulter file

```yaml
kind: Component
apiVersion: kustomize.config.k8s.io/v1alpha1
images:
- name: am
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/am
  newTag: 8.1.1
- name: am-custom
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/busybox
  newTag: musl
- name: amster
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/amster
  newTag: 8.0.2
- name: ds
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/ds
  newTag: 8.0.2
- name: idm
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/idm
  newTag: 8.1.1
- name: idm-custom
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/busybox
  newTag: musl
- name: ig
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/ig
  newTag: 8.0.2
- name: admin-ui
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/admin-ui
  newTag: 8.1.1
- name: end-user-ui
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/end-user-ui
  newTag: 8.0.1
- name: login-ui
  newName: image-registry.openshift-image-registry.svc:5000/png-prod-images/login-ui
  newTag: 8.0.1
labels:
- includeSelectors: true
  pairs:
    app.kubernetes.io/part-of: identity-platform
namespace: pngprd
```


# Image managment

## OC Build

Here is the clean, corrected, and well-structured command guide for building your custom IDM image in OpenShift.

Since you previously used `oc set env`, those exact steps are streamlined below with the proper namespace flags (`-n png-prod-images` and `-n pngprd`).

## 1. Initialize the Build Config (Run Once)

This creates the BuildConfig (`bc`) and sets up the target ImageStream tag (`idm:custom-test`).

Bash

```
oc new-build --binary=true --strategy=docker --name=idm-binary-build --to=idm:1.0.1 -n png-prod-images
```

## 2. Set the Build Argument Environment Variable

Since you are using the `oc set env` approach, run this to permanently inject `CONFIG_PROFILE=oc-profile` into your BuildConfig:

Bash

```
oc set env bc/idm-binary-build CONFIG_PROFILE=oc-profile -n png-prod-images
```

> **Note:** If OpenShift demands explicit Docker `buildArgs` rather than standard environment variables, use `oc patch` instead: 

```bash
oc patch bc/idm-binary-build -n png-prod-images -p '{"spec":{"strategy":{"dockerStrategy":{"buildArgs":[{"name":"CONFIG_PROFILE","value":"oc-profile"}]}}}}'
```

## 3. Execute and Follow the Build (Run for every config change)

This uploads your local `./idm-oc` folder to OpenShift, executes the Docker build using your preset environment variable, and streams the logs live:

Bash

```bash
# Example directory structure:
# /home/pngprd/docker
# └── idm/               <-- Contains Dockerfile, config-profiles, etc.

# Run from /home/pnguser:
oc start-build idm-binary-build --from-dir=./idm --to=idm:1.0.3 --follow -n png-prod-images
```

## 4. Useful Management Commands

### Check All Build Configurations

Bash

```bash
oc get bc -n png-prod-images
```

### View Ongoing/Past Builds

Bash

```bash
oc get builds -n png-prod-images
```

### Cancel a Running Build

Bash

```bash
# Correct syntax requires targeting the specific running build instance (e.g., idm-binary-build-1)
oc cancel-build build/idm-binary-build-1 -n png-prod-images

# Or cancel all currently running builds for this BC:
oc cancel-build bc/idm-binary-build --state=running -n png-prod-images
```

### Verify the Updated Image Tag

To inspect the newly created image tag using debug mode:

Bash

```
oc debug istag/idm:custom-test -n png-prod-images
```


## Update image tag
#### Method 1: The Quick Way (Update Existing BuildConfig)

If you just want to push a new tag (e.g., `1.0.2`) using your existing `idm-binary-build` config, you **don't** need to run `oc new-build` or `oc set env` again.

You can just override the destination tag directly when starting the build:

```bash
oc start-build idm-binary-build --from-dir=./idm --to=idm:1.0.2 --follow -n png-prod-images
```

- **Why this works:** The `idm-binary-build` configuration already exists and already retains your `CONFIG_PROFILE=oc-profile` environment variable. The `--to=idm:1.0.2` flag simply redirects the output of this specific run to tag `1.0.2`.

### If new env variable is there

1. Update the environment variable permanently on the BuildConfig oc set env bc/idm-binary-build CONFIG_PROFILE=stage-profile -n png-prod-images

```bash
oc set env bc/idm-binary-build CONFIG_PROFILE=stage-profile -n png-prod-images
#start the build
oc start-build idm-binary-build --from-dir=./idm --to=idm:1.0.2 --follow -n png-prod-images
```

### Method 2: The Independent Way (Create a New BuildConfig)

If you want a dedicated, standalone BuildConfig specifically for that tag (for example, if you want `idm-binary-build-1.0.2` as a separate pipeline):

Then **yes**, you would run all three steps with a unique name:

Bash

```
# 1. Initialize with a new name and target tag
oc new-build --binary=true --strategy=docker --name=idm-binary-build-v102 --to=idm:1.0.2 -n png-prod-images

# 2. Set the environment variable for this new BuildConfig
oc set env bc/idm-binary-build-v102 CONFIG_PROFILE=oc-profile -n png-prod-images

# 3. Start the build
oc start-build idm-binary-build-v102 --from-dir=./idm-oc --follow -n png-prod-images
```

## Delete the images

To delete an unwanted image or tag after importing it into OpenShift, you have two options depending on whether you want to delete just a **specific tag** or the **entire ImageStream**.

### Option 1: Delete a Specific Tag (Recommended)

If you imported `ds:8.0.1` by mistake and only want to remove that tag while keeping the `ds` ImageStream:

Bash

```
oc delete istag ds:8.0.1 -n png-prod-images
```

_(Replace `ds:8.0.1` with your `<imagestream-name>:<tag-name>`)_

### Option 2: Delete the Entire ImageStream

If you want to remove the entire ImageStream and all of its tags completely:

Bash

```
oc delete is ds -n png-prod-images
```

_(Replace `ds` with your `<imagestream-name>`)_

### Step 3: (Optional) Clean up unused storage on NFS

Deleting the ImageStream or tag removes the metadata from OpenShift immediately. If you want the OpenShift Image Registry to clean up the actual image layers from your **50GB NFS volume** right away, run the image pruner:

Bash

```bash
oc adm prune images
#confirm
oc adm prune images --confirm
```