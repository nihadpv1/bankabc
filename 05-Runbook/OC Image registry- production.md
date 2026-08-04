

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


# PNG-PROD-IMAGES (ForgeOps Images & Local Storage Sync)

This namespace holds all official ForgeOps container images and dependencies, physically saved on the cluster's internal NFS image registry storage.

## 1. Project Creation & RBAC Setup

Bash

```shell
# 1. Create the dedicated image repository project
oc new-project png-prod-images

# 2. Grant pull permissions to application service accounts
oc policy add-role-to-group system:image-puller system:serviceaccounts:pngprd -n png-prod-images

# (Optional) Grant pull permission to a specific service account if needed:
# oc policy add-role-to-user system:image-puller system:serviceaccount:pngprd:pngprd-sa -n png-prod-images
```

## 2. Expose External Registry Route & Authenticate

Bash

```shell
# 1. Expose the cluster's default registry route (if not already exposed)
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"defaultRoute":true}}'

# 2. Capture external route host and log in using Podman
HOST=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
podman login -u pnguserapp -p $(oc whoami -t) ${HOST} --tls-verify=false
```

## 3. Mirror Full Images Directly to NFS Storage

> **Note:** `oc image mirror` automatically creates the `ImageStream`, assigns the tag, and streams all physical binary layers into your NFS storage backend (`/registry`).
> 
> - `--force`: Forces physical layer writes to disk.
>     
> - `--max-per-registry=1`: Prevents HAProxy `504 Gateway Timeout` errors on large binary layers (such as AM/DS).
>     

Bash

```shell
HOST=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')

# ==========================================
# ForgeOps Core Stack
# ==========================================
oc image mirror us-docker.pkg.dev/forgeops-public/images/am:8.1.1 ${HOST}/png-prod-images/am:8.1.1 --force --max-per-registry=1 --insecure=true
oc image mirror us-docker.pkg.dev/forgeops-public/images/amster:8.0.2 ${HOST}/png-prod-images/amster:8.0.2 --force --max-per-registry=1 --insecure=true
oc image mirror us-docker.pkg.dev/forgeops-public/images/ds:8.0.2 ${HOST}/png-prod-images/ds:8.0.2 --force --max-per-registry=1 --insecure=true
oc image mirror us-docker.pkg.dev/forgeops-public/images/idm:8.1.1 ${HOST}/png-prod-images/idm:8.1.1 --force --max-per-registry=1 --insecure=true
oc image mirror us-docker.pkg.dev/forgeops-public/images/ig:8.0.2 ${HOST}/png-prod-images/ig:8.0.2 --force --max-per-registry=1 --insecure=true

# ==========================================
# ForgeOps UI Components
# ==========================================
oc image mirror us-docker.pkg.dev/forgeops-public/images/admin-ui:8.1.1 ${HOST}/png-prod-images/admin-ui:8.1.1 --force --max-per-registry=1 --insecure=true
oc image mirror us-docker.pkg.dev/forgeops-public/images/end-user-ui:8.0.1 ${HOST}/png-prod-images/end-user-ui:8.0.1 --force --max-per-registry=1 --insecure=true
oc image mirror us-docker.pkg.dev/forgeops-public/images/login-ui:8.0.1 ${HOST}/png-prod-images/login-ui:8.0.1 --force --max-per-registry=1 --insecure=true

# ==========================================
# Init Containers (Busybox)
# ==========================================
oc image mirror quay.io/prometheus/busybox:latest ${HOST}/png-prod-images/busybox:musl --force --max-per-registry=1 --insecure=true
oc image mirror quay.io/prometheus/busybox:latest ${HOST}/png-prod-images/busybox:latest --force --max-per-registry=1 --insecure=true
```

## 4. Storage & Image Verification

Bash

```shell
# 1. Verify that NFS disk usage increased from ~384K to >2GB+
oc exec -it deployment/image-registry -n openshift-image-registry -- df -h /registry

# 2. Verify all ImageStreams were created automatically in the namespace
oc get is -n png-prod-images
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