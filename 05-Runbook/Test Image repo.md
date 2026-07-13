## Part 1: The Public Pull Test

First, let's verify that your namespace can pull directly from the public internet without any special image secrets.

### 1. Create a Test Pod YAML

Save this as `public-test-pod.yaml`:

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: public-busybox-test
  namespace: forgerock
spec:
  serviceAccountName: default
  containers:
  - name: busybox
    image: docker.io/library/busybox:musl
    command: ["sleep", "3600"]
```

### 2. Deploy and Check Status

Run these commands in your terminal:

Bash

```
oc apply -f public-test-pod.yaml
oc get pod public-busybox-test -n forgerock
```

- **What to look for:** The status should change to `Running`. If you run `oc describe pod public-busybox-test`, you will see a success event in the log: _Successfully pulled image "docker.io/library/busybox:musl"_.
    

## Part 2: The Internal/Private Registry Test

Now, let's mirror that image into an internal OpenShift ImageStream (simulating your private repo setup) and prove that your `forgerock` namespace can pull it securely using Method 1.

### 1. Create your Protected Namespace & Import the Image

As a cluster admin, run these commands to set up the secure internal repository zone:

Bash

```
# Create the registry project
oc new-project forgerock-protected

# Import busybox from public registry into your internal OpenShift image repository
oc import-image busybox:musl --from=docker.io/library/busybox:musl --confirm -n forgerock-protected
```

### 2. Grant Cross-Namespace Pull Permissions

Give the service account in your app namespace permission to read from the protected repository namespace:

Bash

```
oc policy add-role-to-user system:image-puller system:serviceaccount:forgerock:default --namespace=forgerock-protected
```

### 3. Deploy an Internal Test Pod

Get the internal registry path for your newly imported image. OpenShift automatically creates an internal loopback address for this.

Save this file as `private-test-pod.yaml`:

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: private-busybox-test
  namespace: forgerock
spec:
  serviceAccountName: default
  containers:
  - name: busybox
    # This points straight to OpenShift's internal registry path for your protected namespace
    image: image-registry.openshift-image-registry.svc:5000/forgerock-protected/busybox:musl
    command: ["sleep", "3600"]
```

### 4. Deploy and Verify

Run the deployment:

Bash

```
oc apply -f private-test-pod.yaml
oc get pod private-busybox-test -n forgerock
```

Run a describe to confirm where it pulled from:

Bash

```
oc describe pod private-busybox-test -n forgerock
```

Look at the events at the bottom of the description. You should see:

> _Successfully pulled image "image-registry.openshift-image-registry.svc:5000/forgerock-protected/busybox:musl"_

### Clean Up

Once you see both pods running successfully, you can clean up the test resources:

Bash

```
oc delete pod public-busybox-test private-busybox-test -n forgerock
```

Let me know what you see in the terminal or if you hit any roadblocks while running these steps!