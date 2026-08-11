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



Yes. **Let's do it.** We have enough information to stop debating the architecture.

We'll use:

* **Existing `ds-idrepo`** → remain untouched for `ou=am-config`, IDM, CTS/local data, etc.
* **New dedicated DS deployment** → `ou=identities` only.
* Reuse the exact DN **`ou=identities`**.
* Production and DR each get two identity DS replicas.
* The new identity DS topology will be the only cross-site replication topology.
* Existing DS local replication remains exactly as it is.

This is also consistent with PingDS's model of separate backends/data sets and with ForgeOps' Kustomize architecture. ([Ping Identity Documentation][1])

And one correction from earlier: **we don't need to create a new OU**. The new DS can own `ou=identities`; the fact that the old DS also has an `ou=identities` backend is not a conflict because they are separate DS deployments.

## Phase 1 — build the new DS from the actual ForgeOps base

Before creating anything, I want to see exactly what DS bases/profiles ForgeOps 2026.1 provides in your checkout. We will build from those rather than inventing a custom structure.

### READ-ONLY — Command 1

On the DR jump server:

```bash
find /home/pnguserapp/pngdr/kustomize/vendor/forgeops-2026.1/ds -maxdepth 2 -type f -name kustomization.yaml
```

**Nothing changes in OpenShift.**

Send me the output.

Then I'll give you the **next single command**. We will build this incrementally and won't touch the existing `ds-idrepo` until the new identity DS is proven healthy.

[1]: https://docs.pingidentity.com/pingds/8/install-guide/setup-profiles.html?utm_source=chatgpt.com "Setup profiles | PingDS"


That's actually useful: **no output is expected there**. I was looking in the wrong layer.

`amIdentityStore` is not defined in the Kustomize `idrepo` base. ForgeOps 2026.1 creates/configures DS backends through the **DS image setup process**, not directly in that Kustomization. Ping's ForgeOps documentation confirms that DS customization is done through the DS setup/runtime scripts rather than AM/IDM-style configuration profiles. ([Ping Identity Documentation][1])

So we now know where to look.

And this also means I don't want to start copying the `ds-idrepo` overlay yet. We need to know **which DS image your current deployment is actually running** and whether you already have a custom DS image.

### Next — READ-ONLY

Run this short command:

```bash
grep -nE 'image:|DS_|setup|config' /home/pnguserapp/pngdr/kustomize/overlay/pngdr/ds-idrepo/sts.yaml
```

This doesn't change anything.

I'm particularly looking for:

* the PingDS image
* `DS_BOOTSTRAP_REPLICATION_SERVERS`
* DS environment variables
* any mounted setup/configuration
* whether your current deployment already uses a customized DS image

Once we see that, **we can design the new identity-only DS properly instead of guessing how ForgeOps is creating the backends**.

And yes, we're still keeping the plan:

```text
Existing ds-idrepo
    untouched
    ├── ou=am-config
    ├── IDM
    └── local-only data

New ds-identity
    └── ou=identities
         ↕
       PROD ↔ DR
```

([Ping Identity Documentation][1])

[1]: https://docs.pingidentity.com/forgeops/2026.1/customize/overview.html?utm_source=chatgpt.com "Customization overview | ForgeOps Documentation"


Yes. **That changes the practical approach significantly.** Since both Production and DR are currently empty, we don't have to preserve or migrate existing identity data, and we can rebuild the DS layout cleanly.

Also, the command you just tried confirms I gave you the wrong `dsconfig` subcommand. `get-replication-server-prop` is for the replication-server object and does not accept `--baseDN`; we can stop pursuing that line. The actual PingDS 8 `dsrepl` tool confirms that `--baseDN`, `--fromServer`, and `--toServer` belong to `dsrepl initialize`. ([Ping Identity Documentation][1])

## What I recommend now

Let's **not create a second OU**.

Let's build the DS deployment so that the identity DS contains:

```text
amIdentityStore
    ↓
ou=identities
```

and nothing else application-related.

Then:

```text
PROD identity DS-0 ─────┐
                       ├── ou=identities replication
PROD identity DS-1 ─────┘
          │
          │ WAN
          │
DR identity DS-0 ───────┐
                       ├── ou=identities replication
DR identity DS-1 ───────┘
```

while the existing application repository remains:

```text
PROD ds-idrepo
 ├── ou=am-config
 ├── dc=openidm,dc=forgerock,dc=io
 └── other local domains

DR ds-idrepo
 ├── ou=am-config
 ├── dc=openidm,dc=forgerock,dc=io
 └── other local domains
```

with **no cross-site connection** between those two.

### Why I'm comfortable doing this now

ForgeOps 2026.1 explicitly supports customizing the DS image's setup behavior. The DS setup scripts determine which setup profiles/backends are created, and Ping's documentation describes creating a backend and then creating a matching replication domain for it. ([Ping Identity Documentation][2])

And importantly, Ping's current ForgeOps documentation says the DS customization model uses the `ds/ds-new` setup/runtime structure; the `ldif-ext` structure even has backend-specific areas for `identities`, `am-config`, `idm-repo`, etc. ([Ping Identity Documentation][3])

So the next task is **not replication yet**.

It's:

> Build an identity-only DS image/configuration that creates `amIdentityStore / ou=identities`.

Because you have no data, we can safely iterate.

---

# One thing I need to inspect

Your `vendor/forgeops-2026.1/ds/idrepo` directory is only the Kustomize deployment base. The backend creation logic isn't in that Kustomization, which is why our `grep amIdentityStore` returned nothing.

We need to find the actual DS setup script in your ForgeOps checkout.

### READ-ONLY — next command

Run this on the jump server:

```bash
find /home/pnguserapp/pngdr/kustomize/vendor/forgeops-2026.1 -type f \( -name 'ds-setup.sh' -o -path '*/runtime-scripts/setup' \)
```

This is just filesystem inspection.

### What we're expecting

Something along the lines of:

```text
.../docker/ds/ds-new/ds-setup.sh
.../docker/ds/ds-new/runtime-scripts/setup
```

or the equivalent location in your packaged/vendor tree.

Once we locate that, **we can make the first actual design change**: create the identity-only DS customization, then build/test the image.

And because the environments are empty, we don't need to worry about exporting users, preserving passwords, or migrating the existing `ou=identities`.

One important consequence: **we should not create `ou=identities` manually inside the running DS.** The clean approach is to have the DS image/setup create the correct backend and replication domain from the start. PingDS requires the backend base DN and replication-domain base DN to correspond. ([Ping Identity Documentation][4])

After that, we can deploy the new DS in DR first, verify:

```text
ou=identities
GOOD
2 local replicas
```

then build the identical Production side, establish the cross-site topology, and finally initialize Production → DR. `dsrepl initialize --baseDN ou=identities` will then copy **only that backend**, exactly as you wanted. ([Ping Identity Documentation][1])

**Run only that `find` command next.**

[1]: https://docs.pingidentity.com/pingds/8/tools-reference/dsrepl.html?utm_source=chatgpt.com "dsrepl | PingDS"
[2]: https://docs.pingidentity.com/forgeops/2026.1/consolidated.html?utm_source=chatgpt.com "Untitled | ForgeOps Documentation"
[3]: https://docs.pingidentity.com/forgeops/2026.1/reference/beyond-the-docs.html?utm_source=chatgpt.com "Beyond the docs | ForgeOps Documentation"
[4]: https://docs.pingidentity.com/pingds/8/install-guide/custom-replica.html?utm_source=chatgpt.com "Install DS for custom cases | PingDS"
