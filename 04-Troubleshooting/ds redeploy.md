Understood. We can perform the cleanup entirely from OpenShift by creating a temporary pod that mounts the existing idrepo PVC. We will clean one PVC at a time and verify the mount before deleting anything.

The first command below **does not delete data**. It creates a temporary pod using the same PingDS 8.0.2 image, mounts only `data-ds-idrepo-0`, and keeps the pod alive for inspection.

## Create cleanup pod

**CHANGE — Create temporary cleanup pod for idrepo replica 0**

Type this as one command:

```bash
oc run idrepo-cleaner-0 -n pngdr --image=image-registry.openshift-image-registry.svc:5000/png-dr-images/ds:8.0.2 --restart=Never --command -- /bin/sh -c "sleep 3600" --overrides='{"spec":{"containers":[{"name":"idrepo-cleaner-0","image":"image-registry.openshift-image-registry.svc:5000/png-dr-images/ds:8.0.2","command":["/bin/sh","-c","sleep 3600"],"volumeMounts":[{"name":"data","mountPath":"/mnt"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"data-ds-idrepo-0"}}]}}'
```

### Impact

- Creates pod `idrepo-cleaner-0`.
- Mounts `data-ds-idrepo-0` at `/mnt`.
- Does not delete or modify any files.
- Does not start PingDS.
- Does not touch `data-ds-idrepo-1` or CTS.
- Does not modify the StatefulSet.

### Rollback

If the pod starts successfully, rollback is simply deleting this temporary pod later:

```bash
oc delete pod idrepo-cleaner-0 -n pngdr
```

Do not run the delete command yet. First run only the pod-creation command and send a screenshot of the result. We will verify that the pod is running and that `/mnt` is the expected mounted filesystem before issuing the wipe command.