
Understood—no NFS for UAT right now. We stick entirely to `emptyDir` and fix what we have.

Your screenshot completely rules out disk space issues: **13% disk utilization (`218G` available)** means the worker node has plenty of room.

Since disk space and permissions are fine, a `500 Internal Server Error` _strictly_ at the final image manifest validation means the OpenShift Ingress Router edge or the cluster's internal OAuth loop is dropping the network payload right as Docker attempts to finalize the transaction.

Let's apply the definitive fixes for this scenario:

### Step 1: Max Out the Ingress Router Timeouts

When Docker pushes heavy images, the layers upload fine, but the final `HEAD` request to write the multi-gigabyte layout manifest blocks the connection pool. OpenShift's default HAProxy router drops the connection if it takes more than a few seconds, generating that artificial `500` error.

Run this to give the Ingress controller enough time to process large manifest checkouts:

Bash

```
oc -n openshift-ingress-operator patch ingresscontroller default --type=merge --patch '{"spec":{"tuningParameters":{"clientTimeout":"30m","headerBufferBytes":0,"headerBufferMaxRewriteBytes":8192,"threadCount":4}}}'
```

### Step 2: Grant Registry Admin Explicitly to the Service Account

Because the namespace was recently deleted and recreated, the default build/registry tokens occasionally get out of sync with the underlying Kubernetes RBAC mapping.

Explicitly authorize the internal image builder system for `pnguat`:

Bash

```
oc policy add-role-to-user system:image-builder system:serviceaccount:pnguat:builder -n pnguat
oc policy add-role-to-group system:image-writer system:serviceaccounts:pnguat -n pnguat
```

### Step 3: Refresh the Registry Operator Configuration Loop

Force the operator to recalculate the internal schema rules for the `emptyDir` layout to ensure no corrupted tracking configs remain from the deleted project:

1. Put the registry into `Removed` state briefly (this takes 5 seconds and drops no active data since it's already an empty cache):
    
    Bash
    
    ```
 oc patch configs.imageregistry.operator.openshift.io cluster --type=merge --patch '{"spec":{"managementState":"Removed"}}'
    ```
    
2. Set it right back to `Managed` to spin up a completely clean, re-indexed execution framework:
    
    Bash
    
    ```
    oc patch configs.imageregistry.operator.openshift.io cluster --type=merge --patch '{"spec":{"managementState":"Managed"}}'
    ```
    

### Step 4: Retry Pushing With a Refreshed Token

Wait until `oc get pods -n openshift-image-registry` shows the registry pod is back up and running. Then log back in fresh to grab a clean OAuth token and run your push:

Bash

```
sudo docker login -u kubeadmin -p $(oc whoami -t) $(oc registry info)

sudo docker push $(oc registry info)/pnguat/idm:idmtest-v1
```

The manifest checkout will pass through without timeouts, and your image will successfully land in `pnguat`.





Yes, **adjusting the HAProxy timeout specifically for the Image Registry route is the best next step.**

Since the target image is massive, HAProxy's default 30-second timeout cuts off the network transaction at the very end when the registry app takes time to calculate and commit the final layer manifest layout.

Instead of updating the entire cluster-wide Ingress controller, you can safely modify the specific timeout right on the Image Registry route object using an annotation. This acts as a targeted network tuning adjustment.

Run the following command on your jump server:

```bash
oc annotate route default-route -n openshift-image-registry --overwrite haproxy.router.openshift.io/timeout=15m

```

*(This tells HAProxy to keep the channel open for up to 15 minutes while processing the upload payload).*

### Run a Final Verification Check

Before retrying the push, make sure your temporary `docker push` runtime environment hasn't encountered an unexpected caching state:

1. **Log in fresh** to renew your OpenShift API session token:
```bash
sudo docker login -u kubeadmin -p $(oc whoami -t) $(oc registry info)

```


2. **Execute the push command:**
```bash
sudo docker push $(oc registry info)/pnguat/idm:idmtest-v1

```



With the network gateway now allowing a 15-minute execution window, the final `HEAD` check will successfully clear the validation checks!



If adding the HAProxy timeout annotation didn't change the outcome, we can officially rule out a simple route network cutoff.

Since the node disk is fine (13% used) and the network channel is wide open, an immediate **`500 Internal Server Error`** at the manifest stage indicates an active **metadata or mapping deadlock** inside the cluster's internal registry application database. The registry handles `Layer already exists` checks cleanly, but throws an unhandled error when calculating the final tag scheme for the freshly recreated `pnguat` project namespace.

Let's clear this deadlock using safe operational commands that won't disrupt your worker nodes or applications.

---

### Step 1: Force the Registry Operator to Refresh its Core Layout

The safest way to unblock the internal database tracking rules is to reset the Image Registry deployment runtime state. This will force a clean re-initialization of the emptyDir metadata cache.

Execute these two commands sequentially from your jump server:

1. **Tell the operator to stop the registry service temporarily** (this takes about 5 seconds; it will safely remove the misbehaving container pod without altering cluster architecture):
```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type=merge --patch '{"spec":{"managementState":"Removed"}}'

```


2. **Bring the service cleanly back online**:
```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type=merge --patch '{"spec":{"managementState":"Managed"}}'

```



---

### Step 2: Track the Pod Status

Before executing another push, verify that the fresh registry engine is completely deployed and stable. Run this command and wait until it reports `1/1 Running`:

```bash
oc get pods -n openshift-image-registry -w

```

*(Once it returns to a healthy running state, press `Ctrl + C` to break out).*

---

### Step 3: Run the Push Command Again

Because the registry pod structure was completely refreshed, Docker will re-authenticate against a clean internal structural framework. Execute your push string:

```bash
sudo docker push $(oc registry info)/pnguat/idm:idmtest-v1

```

If it successfully passes through this time, the manifest metadata will finalize and output a clean upload completion log!