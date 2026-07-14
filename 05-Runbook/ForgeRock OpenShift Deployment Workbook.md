
## 1. Workbook Purpose

This workbook standardizes ForgeRock Identity Platform deployments on OpenShift using:

- ForgeOps
    
- Kustomize overlays
    
- OpenShift Routes
    
- Centralized nginx reverse proxy
    
- GitOps deployment model
    

Applicable for:

- CRC labs
    
- Staging
    
- Production
    

---

# 2. Architecture Standard

## Recommended Architecture

```text
Internet
   |
OpenShift Route (Passthrough TLS)
   |
fr-proxy (nginx reverse proxy)
   |
------------------------------------------------
|        |          |          |               |
AM     login-ui   IDM      admin-ui      end-user-ui
   |
DS CTS + DS IDRepo
```

---

## Architectural Decisions

|Decision|Reason|
|---|---|
|Central nginx reverse proxy|Single ingress boundary|
|OpenShift passthrough Route|TLS handled inside nginx|
|Explicit serviceAccountName everywhere|Avoid SCC inconsistency|
|Sequential deployment|Prevent bootstrap race conditions|
|DS deployed first|Stateful dependency foundation|
|AM before AMster|Prevent OAuth bootstrap failures|
|Secret generation before AMster|Required for IDM OAuth integrity|

---

# 3. Environment Preparation Checklist

## OpenShift Validation

```bash
oc whoami
oc get nodes
oc get storageclass
oc get ingresscontroller -n openshift-ingress-operator
```

Checklist:

- Cluster healthy
    
- StorageClass available
    
- Routes functional
    
- DNS functional
    
- SCC permissions validated
    

---

## DNS Validation

```bash
nslookup iam.apps.example.com
```

Checklist:

- External FQDN resolves correctly
    
- Route wildcard DNS operational
    

---

# 4. Namespace Foundation Standard

## Create Namespace

```bash
oc create namespace forgerock
```

---

## Create Service Account

```bash
oc create sa forgerock-sa -n forgerock
```

---

## Apply SCC

```bash
oc adm policy add-scc-to-user anyuid \
  -z forgerock-sa \
  -n forgerock
```

---

## Validate SCC

```bash
oc adm policy who-can use scc anyuid
```

Critical Rule:

> Do NOT deploy workloads before SCC assignment.

---

# 5. Overlay Design Standard

## Required Overlay Structure

```text
overlay/openshift/
├── am
├── amster
├── admin-ui
├── login-ui
├── end-user-ui
├── idm
├── ds-idrepo
├── ds-cts
├── secrets
├── base
├── nginx
```

---

## Mandatory Patch Standards

Every workload must include:

```yaml
serviceAccountName: forgerock-sa
```

Applies to:

- Deployments
    
- StatefulSets
    
- Jobs
    
- CronJobs
    

---

## Ingress Policy

### DO NOT use ForgeOps generated ingress resources

Required:

- ingress removal patches
    
- centralized nginx routing
    
---
# 5A. Certificate Management Foundation

## Purpose

ForgeRock internal TLS certificates depend on cert-manager resources and issuers.

This layer must be operational before:

- secret generation
- DS deployment
- AM deployment
- nginx TLS mounting

---

## Validate cert-manager Installation

### Command

```
oc get crd | grep cert-manager
```

### Expected

```
certificates.cert-manager.ioclusterissuers.cert-manager.ioissuers.cert-manager.io
```

---

## Validate cert-manager Controllers

### Command

```
oc get pods -n cert-manager
```

### Expected

```
Running
```

---

## Deployment Gate

Do NOT continue unless:

- cert-manager CRDs exist
- cert-manager controllers are healthy
- issuer resources reconcile successfully

---

## Verify Generated TLS Secrets

### Command

```
oc get secret -n forgerock
```

### Expected Secrets

```
ds-master-keypairds-ssl-keypair
```

---

## Architecture Note

The deployment depends on internally generated TLS assets for:

- DS secure replication
- AM ↔ DS trust
- nginx TLS termination
- internal service communication

Failure in cert-manager propagation can cascade into:

- DS startup failures
- TLS handshake failures
- nginx certificate mount failures
- OAuth trust issues

---

# 6. Secret Generation Phase

## Install Secret Generator

```bash
helm repo add mittwald https://helm.mittwald.de
helm repo update

helm install secret-generator mittwald/kubernetes-secret-generator \
  -n secret-generator
```

---

## Validate Operator

```bash
oc get pods -n secret-generator
```

Expected:

```text
Running
```

---

## Validate Critical Secrets

```bash
oc get secret amster-env-secrets -n forgerock -o yaml
```

Critical values:

- IDM_RS_CLIENT_SECRET
    
- IDM_PROVISIONING_CLIENT_SECRET
    

Deployment Gate:

> AMster must NOT start before these secrets exist.

---

# 7. Deployment Sequencing Standard

## Correct Deployment Order

|Order|Component|
|---|---|
|1|secrets|
|2|keystore-create|
|3|ds-idrepo|
|4|ds-cts|
|5|AM|
|6|AMster|
|7|IDM|
|8|login-ui|
|9|admin-ui|
|10|end-user-ui|
|11|fr-proxy|

---

## Stateful Validation Gate

Before proceeding beyond DS:

```bash
oc get pods -n forgerock
oc logs <ds-pod> -n forgerock
```

Must verify:

- replication healthy
    
- PVC mounted
    
- no crashloops
    

---

# 8. Reverse Proxy Standard

## Reverse Proxy Responsibilities

nginx handles:

- AM
    
- XUI
    
- IDM
    
- admin-ui
    
- end-user-ui
    
- OAuth routing
    
- websocket upgrade headers
    
- TLS termination
    

---

## Mandatory nginx Features

Required:

- health endpoint
    
- readiness probes
    
- liveness probes
    
- forwarded headers
    
- websocket headers
    
- timeout tuning
    

---

## Mandatory Health Endpoint

```nginx
location /healthz {
    access_log off;
    return 200;
}
```

---

## Mandatory Headers

```nginx
proxy_set_header X-Forwarded-Proto https;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port 443;
```

---

# 9. OpenShift Route Standard

## Required Route Type

```yaml
tls:
  termination: passthrough
```

Reason:

- nginx manages TLS internally
    

---

# 10. Production Hardening Checklist

## Images

DO NOT use:

```yaml
image: latest
```

Use:

- immutable digest
    
- tested version tag
    

---

## Resources

Minimum baseline:

```yaml
requests:
  cpu: 100m
  memory: 128Mi

limits:
  cpu: 500m
  memory: 512Mi
```

Production:

- tune using metrics
    
- never guess
    

---

## Persistence

DS requirements:

- backup policy
    
- snapshot policy
    
- PVC monitoring
    
- replication validation
    

---

## Security

Required:

- TLS everywhere
    
- SCC reviewed
    
- secrets externalized
    
- no hardcoded credentials
    

---

# 11. Verification Workbook

## Platform Health

```bash
curl -kI https://iam.apps.example.com/healthz
```

Expected:

```text
HTTP/1.1 200 OK
```

---

## AM Validation

```bash
curl -kI https://iam.apps.example.com/am/
```

---

## Login UI Validation

```bash
curl -kI https://iam.apps.example.com/am/XUI/
```

---

## Platform UI Validation

```bash
curl -kI https://iam.apps.example.com/platform/
```

---

## IDM Validation

```bash
curl -k https://iam.apps.example.com/openidm/info/uiconfig
```

Expected:

- HTTP 200
    
- valid JSON
    
- no 401
    

---

# 12. Failure Pattern Reference

|Symptom|Likely Cause|
|---|---|
|Init:0/1|SCC restriction|
|401 on IDM|OAuth introspection failure|
|Blank UI|Incorrect platform-config URLs|
|Route loops|Wrong forwarded headers|
|Random pod permission issues|Missing serviceAccountName|
|AMster failures|Secret generation incomplete|
|Broken redirects|Incorrect X-Forwarded-* headers|

---

# 13. GitOps Standard

## ArgoCD Principles

- Git is source of truth
    
- No manual patching in cluster
    
- Separate overlays:
    
    - lab
        
    - staging
        
    - production
        

---

## Recommended Repo Layout

```text
forgeops-gitops/
├── base/
├── overlays/
│   ├── lab/
│   ├── staging/
│   └── production/
```

---

# 14. Operational Rules

## Never Do

- Bulk deploy everything at once
    
- Use default service accounts
    
- Trust ForgeOps ingress defaults on OpenShift blindly
    
- Skip secret validation
    
- Deploy AMster before AM healthy
    
- Use mutable image tags in production
    

---

# 15. Production Readiness Gates

Before production go-live:

- SCC validated
    
- DS replication validated
    
- Backups tested
    
- Route failover tested
    
- OAuth flows validated
    
- IDM introspection validated
    
- TLS certificates validated
    
- Resource limits tested under load
    
- GitOps sync tested
    
- Rollback tested
    

---

# 16. Key Discovery Summary

Most ForgeRock/OpenShift deployment failures were caused by:

1. SCC inheritance behavior
    
2. Incorrect deployment sequencing
    
3. OAuth bootstrap timing
    
4. Overly fragmented ingress architecture
    

Centralized nginx reverse proxy + sequential deployment resolved the majority of operational instability.

Source workbook derived from lab execution notes.