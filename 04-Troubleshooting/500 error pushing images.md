
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