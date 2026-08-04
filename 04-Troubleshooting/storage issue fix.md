
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