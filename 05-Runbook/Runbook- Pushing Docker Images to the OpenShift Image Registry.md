
## 1. Overview & Prerequisites

This runbook covers logging into the internal OpenShift image registry, preparing local Docker images, and successfully pushing them to a target project namespace.

### Prerequisites

- Access to the OpenShift cluster via the `oc` CLI tool.
    
- An active session authenticated as a user with write permissions (e.g., `kubeadmin` or an equivalent cluster administrator/developer).
    
- Docker or Podman installed on your jump server.
    

## 2. Standard Step-by-Step Execution Procedure

### Step 1: Authenticate with the OpenShift Cluster

Log into the OpenShift cluster via the CLI to refresh your session:

Bash

```
oc login <API_URL> -u <username> -p <password>
```

### Step 2: Log into the OpenShift Container Registry

Use your active OpenShift OAuth token to authenticate the local Docker daemon against the cluster's internal registry gateway:

Bash

```
sudo docker login -u $(oc whoami) -p $(oc whoami -t) $(oc registry info)
```

_Expected Output: `Login Succeeded`_

### Step 3: Target the Destination Project Namespace

Switch to the target project namespace where the images will reside:

Bash

```
oc project pnguat
```

### Step 4: Tag Your Local Image

Tag your local development or third-party image with the absolute URL of the OpenShift registry, including the target namespace, image name, and version tag:

Bash

```
# Syntax: sudo docker tag <local_image>:<tag> $(oc registry info)/<namespace>/<image_name>:<tag>
sudo docker tag idm:idm-uat-v2.11 $(oc registry info)/pnguat/idm:idm-uat-v2.11
```

### Step 5: Push the Image

Stream the image layers up to the OpenShift registry:

Bash

```
sudo docker push $(oc registry info)/pnguat/idm:idm-uat-v2.11
```

## 3. Post-Verification Check

Verify that the image layers successfully landed and that OpenShift is actively tracking the new tags:

Bash

```
oc get imagestream -n pnguat
```

_Expected Output: A table mapping the image repository name with your pushed tags explicitly listed._

## 4. Troubleshooting: The Namespace Recreation Deadlock (500 Error)

### The Issue Faced

During execution inside a namespace that was recently deleted and recreated (e.g., `pnguat`), the final manifest validation step failed at the very end of the layer uploads with an explicit error:

> `unknown: unexpected status from HEAD request to https://...: 500 Internal Server Error`

### Root Cause

When a namespace is completely dropped and rebuilt with the exact same name, the cluster's internal API engine occasionally suffers an asynchronous cache or UUID mismatch mismatch. When Docker tries to push an image directly, the registry attempts to dynamically create a matching tracking endpoint on the fly, triggering an unhandled internal system deadlock (`500`).

### How We Overcame It (The Safe Fix)

Instead of forcing a heavy operational restart of the registry pod (which risks clearing out user data caches on an `emptyDir` storage backend), **we manually provisioned the tracking schema in the Kubernetes API beforehand.**

By explicitly creating the empty placeholder first, the registry's automated generation logic is bypassed, allowing the Docker push manifest to commit seamlessly.

**The Fix Action:**

Before attempting the push, manually pre-create the base structural ImageStream object for your components:

Bash

```
# 1. Create the structural tracking slots
oc create imagestream am -n pnguat
oc create imagestream idm -n pnguat

# 2. Re-run your push safely
sudo docker push $(oc registry info)/pnguat/idm:idm-uat-v2.11
```

> **Note:** You only need to run the `oc create imagestream` command **once per base image component**. Subsequent tags or versions (e.g., `idm:idm-uat-v2.12`) will automatically map into this existing slot without requiring any extra commands.