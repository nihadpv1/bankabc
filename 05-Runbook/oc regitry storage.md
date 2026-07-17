



Here is the refined runbook tailored specifically for your \*\*NFS storage class\*\*.



\---



\## Step 1: Verify Current Storage Configuration



First, log in to your cluster and check what the registry is currently using for storage:



```bash

\# Extract only the storage definition to see the current backend

oc get configs.imageregistry.operator.openshift.io cluster -o jsonpath='{.spec.storage}'



```



\---



\## Step 2: Create the NFS-Backed PVC



Switch your context to the image registry namespace:



```bash

oc project openshift-image-registry



```



Now, create the PVC. \*\*Replace `<YOUR\_NFS\_STORAGE\_CLASS>\*\*` with the exact name of your NFS storage class (you can find your available classes using `oc get sc`).



```bash

cat <<EOF | oc apply -f -

apiVersion: v1

kind: PersistentVolumeClaim

metadata:

&#x20; name: image-registry-storage-pvc

&#x20; namespace: openshift-image-registry

spec:

&#x20; accessModes:

&#x20;   - ReadWriteMany

&#x20; resources:

&#x20;   requests:

&#x20;     storage: 100Gi

&#x20; storageClassName: <YOUR\_NFS\_STORAGE\_CLASS>

EOF



```



Verify that the PVC successfully binds to your NFS export:



```bash

oc get pvc image-registry-storage-pvc



```



\*Wait until the `STATUS` column shows \*\*Bound\*\* before running the next step.\*



\---



\## Step 3: Attach the NFS PVC to the Registry



Because NFS handles concurrent connections beautifully, we don't need to tweak replica counts or force rigid recreation strategies. Run this single patch command to clear out old storage blocks, link your new PVC, and ensure the operator is set to actively manage the lifecycle:



```bash

oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed","storage":{"pvc":{"claim":"image-registry-storage-pvc"}}}}'



```



\---



\## Step 4: Watch the Rollout



The operator will instantly trigger a rolling update. Because NFS allows `RWX`, the new registry pods will spin up and connect to the storage \*before\* the old ones terminate, meaning you won't drop image push/pull traffic.



Monitor the pods as they cycle:



```bash

oc get pods -n openshift-image-registry -w



```



Confirm that the core cluster operator flags look clean and healthy:



```bash

oc get clusteroperator image-registry



```



Once `AVAILABLE` registers as `True`, your OpenShift registry is fully backed by your NFS persistent volume.

