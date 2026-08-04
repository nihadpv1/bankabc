
`oc image mirror` uses `--registry-config` for credentials instead of `--dest-creds`.

To pass your token directly into `oc image mirror` without needing a physical file, create a temporary auth file on the fly using standard CLI flags or a Podman login:

### Option 1: Podman Login (Easiest)

Log in via Podman first so `oc` automatically uses the active credentials from your session:

Bash

```
podman login -u pnguserapp -p $(oc whoami -t) $(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}') --tls-verify=false
```

Then run the mirror command:

Bash

```
oc image mirror us-docker.pkg.dev/forgeops-public/images/ds:8.0.2 $(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')/png-prod-images/ds:8.0.2 --insecure=true
```

### Option 2: Single-line Command (Without Podman)

If you want a single command that generates the temporary Docker auth JSON inline:

Bash

```
oc image mirror us-docker.pkg.dev/forgeops-public/images/ds:8.0.2 $(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')/png-prod-images/ds:8.0.2 --registry-config=<(echo "{\"auths\":{\"$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')\":{\"auth\":\"$(echo -n pnguserapp:$(oc whoami -t) | base64 -w0)\"}}}") --insecure=true
```


Looking at your terminal screenshots, **two key things are happening here:**

### 1. Why `am:8.1.1` Threw `504 Gateway Timeout`

Notice that `AM` is by far the largest image in the ForgeOps stack (usually ~1.5 GB to 2 GB+, whereas UI and busybox images are tiny, often < 50MB).

When pushing `am:8.1.1`:

- The OpenShift Ingress/HAProxy router gave up waiting for the massive blob stream after 30 seconds and returned **`504 Gateway Timeout`**.
    
- **Good news:** The OpenShift Image Registry has an internal service IP that bypasses the external router timeout completely!
    

#### The Fix for AM Mirroring:

Bypass the router by mirroring directly through the internal registry service URL:

Bash

```
oc image mirror us-docker.pkg.dev/forgeops-public/images/am:8.1.1 image-registry.openshift-image-registry.svc:5000/png-prod-images/am:8.1.1 --insecure=true
```

### 2. Why `df -h /registry` Shows Only `37M`

If you look closely at your mirror logs for `login-ui`, `end-user-ui`, `busybox`, and `am`, notice this line in every output:

Plaintext

```
stats: shared=0 unique=0 size=0B
blobs=0 mounts=0 manifests=1
```

**What this means:**

The registry **only copied the lightweight text manifest** (pointers), but **did not pull the underlying binary blobs** from `us-docker.pkg.dev` because of cross-layer optimization or skipped layer flags.

Because the heavy layers weren't written to disk, NFS is sitting at only **37 MB**.

### How to Force a Full Mirror (Blobs + Manifests)

To guarantee that all physical binary layers for **all your images** get written into `/registry` (so your cluster can run completely offline without hitting `us-docker.pkg.dev` again), add the **`--force`** flag to your mirror command:

Bash

```
# 1. Force Mirror AM (using internal service endpoint to avoid 504 timeout)
oc image mirror us-docker.pkg.dev/forgeops-public/images/am:8.1.1 image-registry.openshift-image-registry.svc:5000/png-prod-images/am:8.1.1 --force --insecure=true

# 2. Force Mirror DS
oc image mirror us-docker.pkg.dev/forgeops-public/images/ds:8.0.2 image-registry.openshift-image-registry.svc:5000/png-prod-images/ds:8.0.2 --force --insecure=true

# 3. Force Mirror UI Images
oc image mirror us-docker.pkg.dev/forgeops-public/images/login-ui:8.0.1 image-registry.openshift-image-registry.svc:5000/png-prod-images/login-ui:8.0.1 --force --insecure=true
oc image mirror us-docker.pkg.dev/forgeops-public/images/end-user-ui:8.0.1 image-registry.openshift-image-registry.svc:5000/png-prod-images/end-user-ui:8.0.1 --force --insecure=true
oc image mirror us-docker.pkg.dev/forgeops-public/images/admin-ui:8.1.1 image-registry.openshift-image-registry.svc:5000/png-prod-images/admin-ui:8.1.1 --force --insecure=true
```

### Final Check

After running those with `--force`, check your NFS disk again:

Bash

```
oc exec -it deployment/image-registry -n openshift-image-registry -- df -h /registry
```

You should see your NFS usage jump from **37M** to **2GB - 4GB+**, proving that all image binaries are safely stored on disk!