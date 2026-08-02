
Using `oc import-image` is even better! It allows OpenShift to directly pull the image from Docker Hub into your namespace without needing `podman` or local registry logins.

Here is how to import an official image directly into your `png-prod-images` namespace:

### Step 1: Import the Official Image

Run this command to import `alpine:latest` directly from Docker Hub into an ImageStream named `test-app`:

Bash

```
oc import-image test-app:v1 --from=docker.io/library/alpine:latest --confirm -n png-prod-images
```

### Step 2: Verify the ImageStream

Check that OpenShift successfully created and populated the `test-app` ImageStream:

Bash

```
oc get imagestream test-app -n png-prod-images
```

_You should see `test-app` with the tag `v1` and a valid repository path._

### Step 3: Test Pull Access from `pngprd` Namespace

Now test that the `system:image-puller` RBAC rule we configured earlier works! Deploy a quick test pod in the `pngprd` project referencing the imported image:

Bash

```
oc run test-pod --image=image-registry.openshift-image-registry.svc:5000/png-prod-images/test-app:v1 -n pngprd -- sleep 3600
```

Check the status of the pod:

Bash

```
oc get pod test-pod -n pngprd
```

Once it shows **`Running`**, it confirms:

1. The OpenShift Image Registry stored the image layers on your 50GB NFS storage.
    
2. The `pngprd` project successfully authenticated and pulled the image from `png-prod-images`.
    

### Step 4: Cleanup (Optional)

When you are done testing, clean up the test resources:

Bash

```
oc delete pod test-pod -n pngprd
oc delete imagestream test-app -n png-prod-images
```