
### Step 1: Discover and Extract the Registry FQDN

Log into your cluster from the jump server and fetch the exposed external route for the image registry.

Bash

```
# Ensure you are logged in with cluster-admin or equivalent registry access privileges
oc project openshift-image-registry

# Extract the exact external FQDN of your cluster registry
export OCP_REGISTRY_HOST=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
echo "Target Cluster Registry: $OCP_REGISTRY_HOST"
```

### Step 2: Extract the CA Certificate Directly from the Cluster

Instead of guessing network paths via `openssl`, pull the exact ingress/router signing CA bundle straight out of the OpenShift API config map. This guarantees you capture the actual root/intermediate anchor.

Bash

```
# Extract the cluster's ingress/router CA certificates to a local PEM file
oc get configmap router-ca -n openshift-config -o jsonpath='{.data.ca-bundle\.crt}' > ocp-registry-ca.crt

# Verify the certificate format is valid
openssl x509 -in ocp-registry-ca.crt -text -noout | grep "Issuer:"
```

### Step 3: Trust the CA at the OS / Podman Layer

Executing these commands will register the certificate system-wide, allowing `curl`, `wget`, `podman`, and `skopeo` to push and pull cleanly without utilizing risk-prone `--tls-verify=false` flags.

#### For RHEL / CentOS / Rocky Linux / Fedora Jump Servers:

Bash

```
# Copy the certificate to the shared anchors directory
sudo cp ocp-registry-ca.crt /etc/pki/ca-trust/source/anchors/ocp-registry-ca.crt

# Force the OS to re-evaluate and rebuild the system trust store
sudo update-ca-trust extract
```

#### For Ubuntu / Debian Jump Servers:

Bash

```
# Copy the certificate to the local share directory (must end in .crt)
sudo cp ocp-registry-ca.crt /usr/local/share/ca-certificates/ocp-registry-ca.crt

# Rebuild the system cert store
sudo update-ca-certificates
```

### Step 4: Trust the CA at the Docker Daemon Layer (If using Docker)

Docker isolation relies on explicit, host-mapped directory namespaces matching the destination registry domain.

Bash

```
# 1. Create Docker's dedicated configuration directory for this specific FQDN
# NOTE: Do NOT append port :443 if you are hitting the default HTTPS route.
sudo mkdir -p /etc/docker/certs.d/${OCP_REGISTRY_HOST}

# 2. Copy the extracted CA directly into that folder under the required name 'ca.crt'
sudo cp ocp-registry-ca.crt /etc/docker/certs.d/${OCP_REGISTRY_HOST}/ca.crt

# 3. Verify directory layout structure
ls -la /etc/docker/certs.d/${OCP_REGISTRY_HOST}/
```

_(Note: Docker dynamically monitors this directory tree; a daemon restart is **not** required to execute the next transaction.)_

### Step 5: Verification Phase

Execute a clean authentication loop using your chosen toolset to prove the secure TLS handshake completes flawlessly.

Bash

```
# Obtain your current active login token
export OCP_TOKEN=$(oc whoami -t)

# Scenario A: If testing via Podman
podman login -u kubeadmin -p ${OCP_TOKEN} ${OCP_REGISTRY_HOST}

# Scenario B: If testing via Docker
docker login -u kubeadmin -p ${OCP_TOKEN} ${OCP_REGISTRY_HOST}
```

If the terminal returns `Login Succeeded` without throwing x509 unknown authority errors, your network chain is secure. You can now execute `forgeops` builds and safely push the resulting customized ForgeRock IDM images directly to OpenShift.