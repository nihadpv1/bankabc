---

date: 2026-05-13  
type: workbook  
domain:

- identity-access-management
    
- openshift
    
- kubernetes  
topics:
    
- forgerock-forgeops
    
- openshift-scc
    
- nginx-reverse-proxy
    
- cert-manager
    
- gitops  
source: self-study  
tags:
    
- forgerock
    
- openshift
    
- kubernetes
    
- nginx
    
- gitops
    

---

# Summary

This workbook standardizes ForgeRock Identity Platform deployments on OpenShift using:

- ForgeOps
    
- Kustomize overlays
    
- OpenShift Routes
    
- Centralized nginx reverse proxy
    
- GitOps deployment model
    
- cert-manager internal PKI
    

Applicable for:

- CRC labs
    
- Staging
    
- Production
    

---

# Objective

- Deploy ForgeOps on OpenShift
    
- Resolve OpenShift SCC restrictions
    
- Standardize service account usage
    
- Enable ForgeRock secret generation
    
- Stabilize DS, AM, IDM, and UI deployments
    
- Replace ForgeOps ingress sprawl with centralized nginx reverse proxy
    
- Implement production-style routing architecture
    
- Prepare overlays for GitOps deployment
    
- Standardize operational deployment sequencing
    

---

# Environment

- OpenShift CRC
    
- Namespace: `forgerock`
    
- Secret generator:
    
    - mittwald/kubernetes-secret-generator
        
- Internal PKI:
    
    - cert-manager
        
- Reverse proxy:
    
    - Bitnami nginx
        
- Deployment model:
    
    - Kustomize overlays
        
    - OpenShift Route passthrough TLS
        

---

# Architecture Standard

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
|cert-manager internal PKI|Internal ForgeRock TLS automation|

---

# Environment Preparation Checklist

## Validate OpenShift Environment

### Command

```bash
oc whoami
oc get nodes
oc get storageclass
oc get ingresscontroller -n openshift-ingress-operator
```

### Notes

- Validate cluster health
    
- Validate storage availability
    
- Validate Route infrastructure
    

---

## Validate DNS

### Command

```bash
nslookup iam.apps.example.com
```

### Notes

- External FQDN must resolve correctly
    
- Route wildcard DNS must function properly
    

---

# Certificate Management Foundation

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

```bash
oc get crd | grep cert-manager
```

### Expected

```text
certificates.cert-manager.io
clusterissuers.cert-manager.io
issuers.cert-manager.io
```

---

## Validate cert-manager Controllers

### Command

```bash
oc get pods -n cert-manager
```

### Expected

```text
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

```bash
oc get secret -n forgerock
```

### Expected Secrets

```text
ds-master-keypair
ds-ssl-keypair
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

# Namespace Foundation Standard

## Create Namespace

### Command

```bash
oc create namespace forgerock
```

---

## Create Service Account

### Command

```bash
oc create sa forgerock-sa -n forgerock
```

---

## Apply SCC and Namespace Admin Role

### Command

```bash
oc adm policy add-scc-to-user anyuid \
  -z forgerock-sa \
  -n forgerock

oc adm policy add-role-to-user admin \
  -z forgerock-sa \
  -n forgerock
```

### Notes

- `anyuid` SCC required for ForgeRock workloads
    
- Namespace admin role required for operational workload behavior
    
- Apply before deploying workloads
    

---

## Validate SCC

### Command

```bash
oc adm policy who-can use scc anyuid
```

### Notes

- Ensure workloads inherit correct SCC behavior
    

---

# Overlay Design Standard

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

### CONFIG

```yaml
serviceAccountName: forgerock-sa
```

### Applies To

- Deployments
    
- StatefulSets
    
- Jobs
    
- CronJobs
    

---

## Ingress Policy

Do NOT use ForgeOps generated ingress resources.

Required:

- ingress removal patches
    
- centralized nginx routing
    

---

# Secret Generation Phase

## Install Secret Generator

### Command

```bash
helm repo add mittwald https://helm.mittwald.de
helm repo update

helm install secret-generator mittwald/kubernetes-secret-generator \
  -n secret-generator
```

---

## Validate Secret Generator

### Command

```bash
oc get pods -n secret-generator
```

### Expected

```text
Running
```

---

## Validate Critical Secrets

### Command

```bash
oc get secret amster-env-secrets -n forgerock -o yaml
```

### Required Values

- IDM_RS_CLIENT_SECRET
    
- IDM_PROVISIONING_CLIENT_SECRET
    

---

## Deployment Gate

AMster must NOT start before:

- secret generation completed
    
- secrets populated correctly
    
- OAuth secrets verified
    

---

# Deployment Sequencing Standard

## Correct Deployment Order

|Order|Component|
|---|---|
|1|cert-manager|
|2|secret-generator|
|3|secrets|
|4|keystore-create|
|5|ds-idrepo|
|6|ds-cts|
|7|AM|
|8|AMster|
|9|IDM|
|10|login-ui|
|11|admin-ui|
|12|end-user-ui|
|13|fr-proxy|

---

## Stateful Validation Gate

### Command

```bash
oc get pods -n forgerock
oc logs <ds-pod> -n forgerock
```

### Validate

- replication healthy
    
- PVC mounted correctly
    
- no crashloops
    

---

# Reverse Proxy Standard

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

### CONFIG

```nginx
location /healthz {
    access_log off;
    return 200;
}
```

---

## Mandatory Forwarded Headers

### CONFIG

```nginx
proxy_set_header X-Forwarded-Proto https;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port 443;
```

---

# OpenShift Route Standard

## Required Route Type

### CONFIG

```yaml
tls:
  termination: passthrough
```

### Reason

- nginx manages TLS internally
    

---

# Production Hardening Checklist

## Images

Do NOT use:

### CONFIG

```yaml
image: latest
```

Use:

- immutable digest
    
- tested version tag
    

---

## Resources

### Minimum Baseline

```yaml
requests:
  cpu: 100m
  memory: 128Mi

limits:
  cpu: 500m
  memory: 512Mi
```

### Notes

- Tune production resources using metrics
    
- Avoid arbitrary sizing
    

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

# Verification Workbook

## Validate Health Endpoint

### Command

```bash
curl -kI https://iam.apps.example.com/healthz
```

### Expected

```text
HTTP/1.1 200 OK
```

---

## Validate AM

### Command

```bash
curl -kI https://iam.apps.example.com/am/
```

---

## Validate Login UI

### Command

```bash
curl -kI https://iam.apps.example.com/am/XUI/
```

---

## Validate Platform UI

### Command

```bash
curl -kI https://iam.apps.example.com/platform/
```

---

## Validate IDM

### Command

```bash
curl -k https://iam.apps.example.com/openidm/info/uiconfig
```

### Expected

- HTTP 200
    
- valid JSON
    
- no 401
    

---

# Failure Pattern Reference

|Symptom|Likely Cause|
|---|---|
|Init:0/1|SCC restriction|
|401 on IDM|OAuth introspection failure|
|Blank UI|Incorrect platform-config URLs|
|Route loops|Wrong forwarded headers|
|Random pod permission issues|Missing serviceAccountName|
|AMster failures|Secret generation incomplete|
|Broken redirects|Incorrect X-Forwarded-* headers|
|TLS secret missing|cert-manager not ready|

---

# GitOps Standard

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

# Operational Rules

## Never Do

- Bulk deploy everything at once
    
- Use default service accounts
    
- Trust ForgeOps ingress defaults on OpenShift blindly
    
- Skip secret validation
    
- Deploy AMster before AM healthy
    
- Use mutable image tags in production
    
- Skip cert-manager validation
    

---

# Production Readiness Gates

Before production go-live:

- SCC validated
    
- Namespace admin role validated
    
- cert-manager healthy
    
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

# Key Discovery Summary

Most ForgeRock/OpenShift deployment failures were caused by:

1. SCC inheritance behavior
    
2. Incorrect deployment sequencing
    
3. OAuth bootstrap timing
    
4. Overly fragmented ingress architecture
    
5. cert-manager dependency ordering
    

Centralized nginx reverse proxy + sequential deployment resolved the majority of operational instability.

Additional discoveries:

- cert-manager CRDs must exist before ForgeRock secret generation
    
- Internal TLS generation is a hard dependency for DS and nginx
    
- Missing cert-manager components can silently break bootstrap sequencing
    
- Namespace admin role improved workload operational consistency
    

---

# Related Concepts

- OpenShift SCC
    
- cert-manager
    
- ForgeOps overlays
    
- Kustomize patching
    
- StatefulSet deployment sequencing
    
- Reverse proxy architecture
    
- OpenShift passthrough Routes
    
- ForgeRock AM/IDM OAuth integration
    
- GitOps overlay management