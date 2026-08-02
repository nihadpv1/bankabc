
You can verify that the image registry successfully picked up and bound to your NFS storage backend using a few quick `oc` health checks.

Run these steps in order to confirm everything is configured correctly:

### Step 1: Check the Operator Status

Verify that the `configs.imageregistry` cluster configuration is currently in a `Managed` state and that OpenShift registers the PVC storage type:

Bash

```
oc get configs.imageregistry.operator.openshift.io cluster -o jsonpath='{.spec.storage}'
```

**Expected Output:**

JSON

```
{"pvc":{"claim":"image-registry-storage-pvc"}}
```

_(If this outputs `{"emptyDir":{}}`, the registry is still utilizing temporary local node storage instead of your target file share)._

### Step 2: Confirm the Registry Pod is running and Healthy

When you change the configuration, the operator terminates the old registry pod and brings up a new one using the network mount. Check that the pod rolled out successfully:

Bash

```
oc get pods -n openshift-image-registry
```

**Expected Output:**

Plaintext

```
NAME                                               READY   STATUS    RESTARTS   AGE
cluster-image-registry-operator-xxxxxxxxx-xxxxx    2/2     Running   0          5d
image-registry-yyyyyyyyy-yyyyy                     1/1     Running   0          2m
```

Make sure `image-registry-...` reads `1/1 Running` and its `AGE` aligns with when you applied the YAML.

### Step 3: Inspect the Live Storage Mount Inside the Pod

To confirm the running registry container is actively writing to the NFS storage backend, inspect the internal storage paths of the active pod.

The registry internally maps its image layer backend to `/registry`. Run a `df -h` command directly inside that active pod:

Bash

```
oc exec -it deployment/image-registry -n openshift-image-registry -- df -h /registry
```

**Expected Output:**

Plaintext

```
Filesystem                                  Size  Used  Avail Use% Mounted on
10.x.x.x:/your-nfs-export-path/pvc-block    500G   15G   485G   3% /registry
```

If the `Filesystem` output shows your external NFS server IP or mount device name instead of a standard local root partition (`/dev/sda...`), **your verification is successful**. The master copies of all imported images are completely isolated on the network storage array and safe from individual bare-metal node crashes.


By default, when you run `oc import-image` without extra flags, OpenShift **resolves the metadata and sha256 digest**, but sets the **Reference Policy** to **`Source`**.

This means OpenShift only saves the _metadata/manifest_ locally, while forcing worker nodes to download the actual heavy image layers directly from `us-docker.pkg.dev` whenever a pod starts!

### How to Check Where It Currently Points

Run this command on your terminal:

Bash

```
oc get istag am:8.1.1 -o jsonpath='{.importPolicy.referencePolicy.type}'
```

- If it returns **`Source`**: The heavy layers are **NOT on NFS**. It is pulling from Google Cloud.
    
- If it returns **`Local`**: The layers **ARE stored on your NFS backend**.
    

### How to Fix It (Force Storage onto NFS)

To guarantee that the entire image payload is pulled into your cluster and stored on your **NFS storage backend**, update the reference policy to `local`:

#### Method 1: Re-import with `--reference-policy=local` (Recommended)

Run this command for each image:

Bash

```
oc import-image am:8.1.1 \
  --from=us-docker.pkg.dev/forgeops-public/images/am:8.1.1 \
  --confirm \
  --reference-policy=local
```

#### Method 2: Patch Existing ImageStream

If you already created the ImageStreams, you can patch them directly without re-importing:

Bash

```
oc patch is am -p '{"spec":{"tags":[{"name":"8.1.1","referencePolicy":{"type":"Local"}}]}}'
```

### How to Verify Storage on NFS

1. **Check Image Size:**
    
    Bash
    
    ```
    oc describe istag am:8.1.1
    ```
    
    Under the output, verify that **Image Size** displays the actual size (e.g., `~600MB`).
    
2. **Check Image Registry Pod Logs:**
    
    You will see active write/upload traffic on your image registry pod while it copies the layers to NFS:
    
    Bash
    
    ```
    oc logs -n openshift-image-registry -l docker-registry=default --tail=20
    ```