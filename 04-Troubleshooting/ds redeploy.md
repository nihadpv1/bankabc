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

That tells us the PVC is mounted, but the data is not under `/mnt/opendj`. For this DS image, the mounted persistence may be linked into the container’s own data path, so we need to inspect the actual directory layout before deleting anything. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/images/60085377/009b31cf-ca0e-4a1f-b1cc-7ceea00fc614/image.jpg)

## Next read-only command

Please run this inside the cleanup pod:

```sh
find /mnt -maxdepth 2 -type d -o -type f | sed -n '1,80p'
```

If `find` is not available, use:

```sh
ls -la /mnt
ls -la /mnt/*
```

We’re looking for the directory that maps to the PingDS database files so we can delete only the backend data and not unrelated files.

## Why I’m being careful

Because the pod mount is empty at the root, I do not want to assume the delete path. On this DS image, the actual persistence structure may differ from what a standard `/opt/opendj/data` mount would show, and deleting the wrong path would either do nothing or remove the wrong files.

Send the screenshot of that directory listing, and I’ll give you the exact delete command after verifying the path.


You’re right — you already pasted `sts.yaml` in the earlier screenshots. I should not have asked you to repeat it.

The important line is already confirmed:

```yaml
- name: DS_BOOTSTRAP_REPLICATION_SERVERS
  value: "prbhvspngprw1.arabbanking.local:30989,prbhvspngprw1.arabbanking.local:30989"
```

That is the old DR-to-Production bootstrap relationship. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/images/60085377/109b8643-4b0a-477b-bae1-42e5e19ecd7c/image.jpg)

## Do not edit that line yet

Simply removing the variable and starting this existing two-replica deployment would not create the required final architecture. It could leave the replicas standalone or fail to establish the intended local replication relationship.

Your current `ds-idrepo` contains multiple datasets, while the target requires:

- cross-site replication only for `ou=identities`.
- local-only replication for the other datasets.

Therefore, we should first inspect what reusable DS base overlays are available in your ForgeOps vendor directory and then create separate Kustomize roles rather than modifying this deployment blindly.

## Next read-only command

From the current directory, run:

```bash
find ../../../vendor/forgeops-2026.1/ds -maxdepth 3 -type f | sort
```

This is **READ-ONLY**. It will show whether the vendor package already provides separate DS bases or only the combined `idrepo` base. Then we can determine whether to:

- reuse separate vendor roles.
- create separate Kustomize overlays around the DS base.
- or use one local topology plus a dedicated identity-only DS topology.

Do not run `oc apply -k` or scale `ds-idrepo` up yet.

You are correct: the local bootstrap and the cross-site identity replication should be treated as **two separate stages**.

## Recommended method

### Stage 1 — StatefulSet bootstrap

The StatefulSet should define only the **local DR topology**:

- `DS_GROUP_ID=dr`.
- Bootstrap from DR DS endpoints only.
- No Production hostname.
- No cross-site identity endpoint.
- No `ou=identities`-specific cross-site configuration in the StatefulSet.

Do not use the StatefulSet environment variable to encode the final cross-site identity topology. That variable is a bootstrap seed, not a safe mechanism for selecting one backend for cross-site replication.

### Stage 2 — Manual identity replication

After both DR DS servers are running and their local topology is healthy, configure cross-site replication from inside a DS pod using the PingDS replication tools:

1. Enable replication between the selected Production and DR servers for only:

   ```text
   ou=identities
   ```

2. Initialize only `ou=identities` from the chosen source.

3. Verify that `ou=am-config`, `dc=openidm,dc=forgerock,dc=io`, CTS, and monitoring data remain local.

PingDS replication domains are base-DN scoped, and the initialization command supports selecting the specific base DN. [docs.pingidentity](https://docs.pingidentity.com/pingdirectory/11.1/pingdirectory_server_administration_guide/pd_ds_replica_initialization.html)

## About adding both DR endpoints

For a normal existing topology, multiple bootstrap endpoints improve availability. However, your two DR PVCs are currently empty. Adding both empty pod addresses creates a bootstrap dependency between two servers that do not yet contain a valid replication topology.

For this clean rebuild, use a staged startup:

1. Start `ds-idrepo-0` as the initial DR seed.
2. Verify that it initializes successfully and creates the local topology.
3. Start `ds-idrepo-1` using `ds-idrepo-0` as its bootstrap server.
4. Verify local replication for the local domains.
5. Configure cross-site replication for `ou=identities` only.

Therefore, I do **not** recommend immediately placing both empty pod addresses in `DS_BOOTSTRAP_REPLICATION_SERVERS`.

## Important distinction

The same secrets can be reused where appropriate for:

- DS admin authentication.
- Replication administration.
- TLS trust/key material.

But shared secrets do not make the topology correct. Each DS instance still needs its own identity and must be registered in the correct replication topology. Cross-site replication also requires routable replication/admin endpoints and TLS trust, not only matching credentials. [docs.pingidentity](https://docs.pingidentity.com/pingdirectory/11.0/pingdirectory_server_administration_guide/pd_ds_replication_arch.html)

## Next safe step

Before changing the Kustomize file, we need to confirm how the ForgeOps startup script behaves when `DS_BOOTSTRAP_REPLICATION_SERVERS` is absent or empty.

Run this **READ-ONLY** command from the OpenShift terminal:

```bash
oc run ds-script-check -n pngdr --rm -i --restart=Never --image=image-registry.openshift-image-registry.svc:5000/png-dr-images/ds:8.0.2 --command -- sh -c 'grep -n -E "DS_BOOTSTRAP_REPLICATION_SERVERS|DS_GROUP_ID|replication" /opt/opendj/runtime-scripts/init-and-restore.sh /opt/opendj/default-scripts/init-and-restore.sh'
```

This does not mount a PVC and does not alter DS data. It only checks the image’s startup-script logic so we do not guess whether an empty bootstrap variable creates a valid first server.