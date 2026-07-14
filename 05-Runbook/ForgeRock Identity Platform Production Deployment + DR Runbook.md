
## Overview

This runbook standardizes deployment, recovery, scaling, and DR onboarding for a production-aligned ForgeRock Identity Platform deployment using:

- ForgeOps
    
- OpenShift
    
- Kubernetes
    
- cert-manager
    
- Kustomize overlays
    
- Centralized nginx reverse proxy
    
- DS multi-master replication
    
- Cross-cluster DR replication
    

This document is intended to serve as:

- Operational deployment standard
    
- Production recovery workbook
    
- DR onboarding guide
    
- GitOps reference architecture
    
- DS replication operations handbook
    

This runbook incorporates validated operational findings from all lab executions and recovery exercises.  
Major findings included:

- DS topology persistence
    
- replication lineage behavior
    
- DR bootstrap workflows
    
- cert-manager PKI integration
    
- WAN replication design
    
- OpenShift SCC handling
    
- StatefulSet operational behavior
    
- replication initialize requirements
    
- AMster lineage sensitivity
    
- DR operational sequencing
    

Derived from validated operational notes and lab execution findings.

---

# 1. Target Architecture

---

# PRIMARY SITE

Platform:

- OpenShift
    

Components:

- ds-idrepo
    
- ds-cts
    
- AM
    
- IDM
    
- login-ui
    
- admin-ui
    
- end-user-ui
    
- fr-proxy
    
- AMster
    

Replication:

- Active DS replication server
    

Ingress:

- Centralized nginx reverse proxy
    
- OpenShift passthrough Route
    

---

# DR SITE

Platform:

- Kubernetes or OpenShift
    

Components:

- ds-idrepo only (warm standby)
    

Optional:

- AM/IDM activated only during failover testing
    

Replication:

- DS replication server enabled
    
- bootstrap from PRIMARY
    

---

# Architectural Principles

Validated production-aligned design:

```text
PRIMARY = full IAM
DR = DS-only warm standby
```

NOT:

- full active-active IAM
    

Reasoning:

- DS is authoritative stateful layer
    
- operational complexity reduced
    
- WAN complexity reduced
    
- failover safer
    
- resource usage significantly lower
    

Validated during lab exercises.

---

# 2. Platform Prerequisites

---

# OpenShift Validation

```bash
oc whoami
oc get nodes
oc get storageclass
oc get ingresscontroller -n openshift-ingress-operator
```

Validate:

- cluster healthy
    
- ingress healthy
    
- storage available
    
- routes operational
    

---

# DNS Validation

```bash
nslookup iam.apps.example.com
```

Validate:

- wildcard DNS working
    
- external DNS resolution functional
    

---

# Required Operators

Install:

- cert-manager
    
- kubernetes-secret-generator
    

---

# cert-manager Validation

```bash
oc get crd | grep cert-manager
```

Expected:

```text
certificates.cert-manager.io
issuers.cert-manager.io
clusterissuers.cert-manager.io
```

---

# cert-manager Controller Validation

```bash
oc get pods -n cert-manager
```

Expected:

```text
Running
```

Deployment must NOT proceed unless cert-manager healthy.

---

# 3. Namespace Foundation

---

# Create Namespace

```bash
oc create namespace forgerock
```

---

# Create Service Account

```bash
oc create sa forgerock-sa -n forgerock
```

---

# Apply SCC

```bash
oc adm policy add-scc-to-user anyuid \
  -z forgerock-sa \
  -n forgerock
```

---

# Apply Namespace Admin

```bash
oc adm policy add-role-to-user admin \
  -z forgerock-sa \
  -n forgerock
```

---

# Validate SCC

```bash
oc adm policy who-can use scc anyuid
```

Important:  
ForgeRock workloads MUST NOT deploy before SCC assignment.  
Validated operationally.

---

# 4. Repository and Overlay Standards

---

# Recommended Repository Layout

```text
forgeops-gitops/
├── base/
├── overlays/
│   ├── primary-site/
│   ├── dr-site/
│   ├── staging/
│   └── production/
```

---

# Required Overlay Components

```text
overlay/
├── am
├── amster
├── admin-ui
├── login-ui
├── end-user-ui
├── idm
├── ds-idrepo
├── ds-cts
├── secrets
├── nginx
├── routes
├── cert-manager
```

---

# Mandatory Workload Standard

ALL workloads MUST contain:

```yaml
serviceAccountName: forgerock-sa
```

Applies to:

- Deployments
    
- StatefulSets
    
- Jobs
    
- CronJobs
    

Validated operationally.

---

# 5. Internal PKI Foundation

---

# Architecture Principle

Internal ForgeRock PKI MUST be separated from public ingress TLS.

Validated architecture:

|Purpose|PKI|
|---|---|
|DS replication|cert-manager internal CA|
|AM ↔ DS|cert-manager internal CA|
|ingress/public TLS|Vault/F5/enterprise PKI|

Validated operationally.

---

# Generate Internal Root CA

```bash
openssl req -x509 -nodes -newkey rsa:4096 \
-keyout forgerock-root-ca.key \
-out forgerock-root-ca.crt \
-days 3650 \
-subj "/CN=forgerock-root-ca"
```

---

# Create Root CA Secret

PRIMARY:

```bash
oc create secret tls forgerock-root-ca \
--cert=forgerock-root-ca.crt \
--key=forgerock-root-ca.key \
-n forgerock
```

DR:

```bash
kubectl create secret tls forgerock-root-ca \
--cert=forgerock-root-ca.crt \
--key=forgerock-root-ca.key \
-n forgerock
```

---

# Create cert-manager Issuer

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: forgerock-ca-issuer
  namespace: forgerock

spec:
  ca:
    secretName: forgerock-root-ca
```

Apply in BOTH clusters.

Validated operationally.

---

# 6. DS Certificate Strategy

---

# Important Rule

DO NOT use:

```text
cluster.local
```

as WAN replication identity.

---

# Recommended Replication DNS

PRIMARY:

```text
ds-idrepo-0.primary.replication.company.lab
```

DR:

```text
ds-idrepo-0.dr.replication.company.lab
```

---

# DS Certificate SAN Example

`ds-ssl-keypair`:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ds-ssl-cert
  namespace: forgerock
spec:
  commonName: ds

  dnsNames:
    - "*.ds"
    - "*.ds-idrepo"
    - "*.ds-cts"
    - "ds-idrepo-0.ds-idrepo.forgerock.svc.cluster.local"
    - "ds-idrepo-0.dr.internal"
    - "ds-idrepo-0.primary.replication.kvm.lab"   

  duration: 43800h
  renewBefore: 720h
  isCA: false

  issuerRef:
    group: cert-manager.io
    kind: Issuer
    name: forgerock-ca-issuer

  privateKey:
    algorithm: ECDSA
    rotationPolicy: Never

  secretName: ds-ssl-keypair

  secretTemplate:
    annotations:
      my-secret-annotation: cert-manager-generated
    labels:
      app: ds

  subject:
    organizations:
      - forgerock.org

  usages:
    - server auth
    - client auth
```
Example san for production:
```yaml
dnsNames:
  - "*.ds"
  - "*.ds-idrepo"
  - "*.ds-cts"

  - "ds-idrepo-0.ds-idrepo.forgerock.svc.cluster.local"
  - "ds-idrepo-1.ds-idrepo.forgerock.svc.cluster.local"
  - "ds-idrepo-2.ds-idrepo.forgerock.svc.cluster.local"

  - "ds-repl-primary.company.com"
```


`ds-master-keypair`:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ds-master-cert
  namespace: forgerock
spec:
  commonName: Master Key

  dnsNames:
    - "*.ds"

  duration: 175200h

  isCA: false

  issuerRef:
    group: cert-manager.io
    kind: Issuer
    name: forgerock-ca-issuer

  privateKey:
    algorithm: RSA
    encoding: PKCS1
    rotationPolicy: Never
    size: 2048

  secretName: ds-master-keypair

  secretTemplate:
    labels:
      app: ds

  subject:
    organizations:
      - ForgeRock.com

  usages:
    - server auth
    - client auth
```


Must include:

- pod FQDN
    
- service FQDN
    
- replication WAN DNS
    

Validated operationally.

---

# 7. Secret Generation

---

# Install Secret Generator

```bash
helm repo add mittwald https://helm.mittwald.de
helm repo update

helm install secret-generator mittwald/kubernetes-secret-generator \
  -n secret-generator
```

---

# Validate Operator

```bash
oc get pods -n secret-generator
```

---

# Validate Secrets

```bash
oc get secret amster-env-secrets -n forgerock
```

Validate:

- IDM_RS_CLIENT_SECRET
    
- IDM_PROVISIONING_CLIENT_SECRET
    

AMster MUST NOT start before secrets exist.  
Validated operationally.

---

# 8. Deployment Sequencing

---

# REQUIRED ORDER

| Order | Component        | Importance and usage                                                          |
| ----- | ---------------- | ----------------------------------------------------------------------------- |
| 1     | cert-manager     | Must for internal certificates                                                |
| 2     | secret-generator | Used for bootstrap secrets(can use manual secret)                             |
| 3     | secrets          | Can either use overlay or secrets manually                                    |
| 4     | keystore-create  | Either run job or use exported keystore                                       |
| 5     | ds-idrepo        | Must run using overlay                                                        |
| 6     | ds-cts           | Must run using overlay                                                        |
| 7     | ds-set passwoprd | Should run first time to sync ds password with internal ds services           |
| 8     | AM               | Must run using overlay                                                        |
| 9     | AMster           | Amster need to run once to populate proper creation of basic oauth and tokens |
| 10    | IDM              | Must run using overlay                                                        |
| 11    | login-ui         | Must run using overlay                                                        |
| 12    | admin-ui         | Must run using overlay                                                        |
| 13    | end-user-ui      | Must run using overlay                                                        |
| 14    | fr-proxy         | Must run with manifest for proper routing                                     |
| 15    | Route            | Must run route manfiest either as transparent or edge                         |

This order was operationally validated and prevents:

- OAuth failures
    
- AMster failures
    
- bootstrap race conditions
    
- TLS sequencing failures
    

---

# 9. PRIMARY DS Deployment

---

# Apply DS Overlay

```bash
oc apply -k overlays/primary-site/ds-idrepo/
```

---

# PLACEHOLDER MANIFEST

Used in lab:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ds-idrepo
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: ds
        env:
        - name: DS_GROUP_ID
          value: primary

        - name: DS_BOOTSTRAP_REPLICATION_SERVERS
          value: "ds-idrepo-0.primary.replication.kvm.lab:8989"
           
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            memory: 1Gi
      initContainers:
      - name: init
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: data
      annotations:
        pv.beta.kubernetes.io/gid: "0"
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: crc-csi-hostpath-provisioner
```

For production:

```yaml
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: ds-idrepo
  namespace: forgerock

spec:
  replicas: 3
  serviceName: ds-idrepo
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: ds-idrepo
  template:
    metadata:
      labels:
        app: ds-idrepo
    spec:
      serviceAccountName: forgerock-sa
      terminationGracePeriodSeconds: 120
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - ds-idrepo
              topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: ds-idrepo
      containers:
        - name: ds
          env:
            # Unique site identity
            - name: DS_GROUP_ID
              value: primary

            # Initial topology discovery seed
            # Use stable VIP/LB in production if available
            - name: DS_BOOTSTRAP_REPLICATION_SERVERS
              value: "ds-repl-primary.company.com:8989"

          resources:
            requests:
              cpu: "1"
              memory: 4Gi
            limits:
              memory: 6Gi
      initContainers:
        - name: init
          resources:
            requests:
              cpu: 250m
              memory: 1Gi
            limits:
              memory: 2Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi

        # Replace with production storage class
        storageClassName: fast-ssd-storage
```


---

# Required DS Variables

```yaml
- name: DS_GROUP_ID
  value: primary

- name: DS_BOOTSTRAP_REPLICATION_SERVERS
  value: "ds-idrepo-0.primary.replication.company.lab:8989"
```

Validated operationally.

---

# Validate DS Startup

```bash
oc get pods -n forgerock
```

Validate:

- Running
    
- no crashloop
    
- PVC mounted
    
- TLS secrets mounted
    

---

# Validate Replication Topology

```bash
ldapsearch \
  --hostname localhost \
  --port 4444 \
  --useSsl \
  --trustAll \
  --bindDN "uid=admin" \
  --bindPassword '<password>' \
  --baseDN "cn=servers,cn=topology,cn=monitor" \
  "(objectClass=*)" \
  ds-mon-server-id
```

Expected:

```text
ds-idrepo-0-primary
```

ONLY.

Validated operationally.

---

# 10. DS External Exposure

---

# Important Principle

Do NOT expose internal headless services externally.

Create separate NodePort/LB service.

---

# PLACEHOLDER MANIFEST

```yaml
# PLACEHOLDER:
# Insert ds-idrepo-external NodePort or LoadBalancer manifest
```

---

# Required Ports

|Purpose|Port|
|---|---|
|Replication|8989|
|Admin Connector|4444|
|LDAPS|1636|

Validated operationally.

---

# 11. PRIMARY IAM Deployment

---

# Deploy DS CTS

```bash
oc apply -k overlays/primary-site/ds-cts/
```

---

# Deploy AM

```bash
oc apply -k overlays/primary-site/am/
```

Wait for:

```bash
oc get pods -n forgerock
```

AM MUST be healthy before AMster.

---

# Deploy AMster

```bash
oc apply -k overlays/primary-site/amster/
```

---

# Deploy IDM + UIs

```bash
oc apply -k overlays/primary-site/idm/
oc apply -k overlays/primary-site/login-ui/
oc apply -k overlays/primary-site/admin-ui/
oc apply -k overlays/primary-site/end-user-ui/
```

---

# Deploy fr-proxy

```bash
oc apply -k overlays/primary-site/nginx/
```

---

# 12. OpenShift Route Deployment

---

# Recommended Route Mode

```yaml
tls:
  termination: passthrough
```

Validated operationally.

---

# PLACEHOLDER MANIFEST

```yaml
# PLACEHOLDER:
# Insert OpenShift Route manifest here
```

---

# 13. PRIMARY Validation

---

# Health Endpoint

```bash
curl -kI https://iam.apps.example.com/healthz
```

Expected:

```text
HTTP/1.1 200 OK
```

---

# Validate AM

```bash
curl -kI https://iam.apps.example.com/am/
```

---

# Validate IDM

```bash
curl -k https://iam.apps.example.com/openidm/info/uiconfig
```

Expected:

- HTTP 200
    
- valid JSON
    

Validated operationally.

---

# 14. DR Site Deployment

---

# Deploy Namespace + SA

Repeat:

- namespace creation
    
- SCC/RBAC
    
- cert-manager issuer
    
- secret import
    

---

# Deploy DR DS ONLY

```bash
kubectl apply -k overlays/dr-site/ds-idrepo/
```

---

# Required DR Variables

```yaml
- name: DS_GROUP_ID
  value: dr

- name: DS_BOOTSTRAP_REPLICATION_SERVERS
  value: "ds-idrepo-0.primary.replication.company.lab:8989"
```

Validated operationally.

---

# 15. Replication Bootstrap Workflow

---

# IMPORTANT

Validated working onboarding sequence:

1. Deploy fresh DR DS
    
2. DR bootstraps to PRIMARY
    
3. topology converges
    
4. initialize from PRIMARY
    
5. replication converges
    

DO NOT:

```text
dual-bootstrap primary ↔ dr
```

This caused topology contamination during testing.

Validated operationally.

---

# 16. Replication Validation

---

# Validate Replication Status

```bash
dsrepl status --showReplicas --showChangeLogs
```

---

# Validate Replication Servers

```bash
dsconfig list-replication-server
```

---

# Validate Topology Monitor

```bash
ldapsearch \
  --baseDN "cn=servers,cn=topology,cn=monitor"
```

Expected:

```text
ds-idrepo-0-primary
ds-idrepo-0-dr
```

Validated operationally.

---

# 17. Replication Initialize Procedure

---

# IMPORTANT

Replication topology GOOD does NOT guarantee historical convergence.

Explicit initialize required.

Validated operationally.

---

# Initialize Replication

Run from PRIMARY:

```bash
dsrepl initialize \
  --baseDn ou=identities \
  --baseDn ou=am-config \
  --baseDn dc=openidm,dc=forgerock,dc=io \
  --toAllServers \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword '<password>' \
  --trustAll
```

---

# Validate Entry Counts

```bash
dsrepl status --showReplicas
```

Validate:

- identical entry counts
    
- replay delay = 0
    
- GOOD status
    

---

# Validate Historical Objects

```bash
ldapsearch \
  --hostname localhost \
  --port 1636 \
  --useSsl \
  --trustAll \
  --bindDN "uid=admin" \
  --bindPassword '<password>' \
  --baseDN "ou=identities" \
  "(uid=testuser)"
```

Validated operationally.

---

# 18. Backup Strategy

---

# List Backends

```bash
backendstat list-backends
```

Expected:

```text
amIdentityStore
cfgStore
idmRepo
```

---

# Export Backends

```bash
export-ldif \
  --backendID amIdentityStore \
  --ldifFile /tmp/amIdentityStore_backup.ldif
```

```bash
export-ldif \
  --backendID idmRepo \
  --ldifFile /tmp/idmRepo_backup.ldif
```

---

# IMPORTANT

Avoid restoring dirty cfgStore lineage directly.

Preferred strategy:

- restore identities
    
- restore idmRepo
    
- replay AM config logically using AMster
    

Validated operationally.

---

# 19. Recovery Workflow

---

# Validated Recovery Sequence

1. clean topology rebuild
    
2. deploy PRIMARY DS only
    
3. validate topology clean
    
4. restore identities
    
5. restore idmRepo
    
6. deploy ds-cts
    
7. deploy AM
    
8. deploy IDM
    
9. deploy UIs
    
10. replay AMster
    
11. validate IAM
    
12. onboard DR later
    

Validated operationally.

---

# 20. Production Scaling Procedure

---

# IMPORTANT

Scaling StatefulSet alone is NOT sufficient.

Explicit initialize required.

Validated operationally.

---

# Scale Example

```bash
oc scale sts ds-idrepo --replicas=3 -n forgerock
```

---

# Required Validation

After scale:

- replication status
    
- initialize
    
- entry counts
    
- historical object validation
    
- bidirectional replication validation
    

---

# 21. Production Hardening

---

# Mandatory Requirements

- anti-affinity
    
- topology spread constraints
    
- immutable image tags
    
- PVC monitoring
    
- backup automation
    
- GitOps deployment
    
- TLS everywhere
    
- no mutable configs in cluster
    

---

# PLACEHOLDER MANIFESTS

```yaml
# PLACEHOLDER:
# Insert anti-affinity policy
```

```yaml
# PLACEHOLDER:
# Insert topology spread constraints
```

```yaml
# PLACEHOLDER:
# Insert production resource limits
```

---

# 22. Operational Failure Patterns

|Symptom|Likely Cause|
|---|---|
|Init:0/1|SCC restriction|
|TLS failures|cert-manager issue|
|replication GOOD but missing data|initialize not run|
|OAuth failure|AMster sequencing|
|route redirect loops|forwarded headers|
|topology contamination|dual bootstrap|
|stale replication members|dirty cfgStore lineage|
|AMster SMS failures|topology inconsistency|

Validated operationally across recovery exercises.

---

# 23. Production Readiness Gates

Before go-live validate:

- SCC validated
    
- cert-manager healthy
    
- DS replication healthy
    
- replay delay = 0
    
- backups tested
    
- DR initialize tested
    
- Route failover tested
    
- OAuth validated
    
- IDM validated
    
- AMster replay tested
    
- TLS SAN validation tested
    
- failback documented
    
- GitOps sync tested
    

---

# 24. Final Operational Principles

---

# DS Is Stateful Infrastructure

Do NOT treat DS like stateless Kubernetes workloads.

Kubernetes validates:

```text
pod running
```

ForgeRock additionally requires:

```text
replication convergence
dataset consistency
lineage consistency
```

Validated operationally.

---

# Bootstrap Servers Are Discovery Seeds

`DS_BOOTSTRAP_REPLICATION_SERVERS` is:

```text
initial discovery only
```

NOT:

- permanent topology map
    
- authoritative replication config
    

Validated operationally.

---

# Final Architecture Summary

Validated production-aligned model:

```text
PRIMARY (Full IAM)
        ↓
DS Replication
        ↓
DR (DS Warm Standby)
```

Validated across:

- OpenShift
    
- Kubernetes
    
- WAN-style networking
    
- cert-manager PKI
    
- cross-cluster replication
    
- topology rebuilds
    
- DR onboarding
    
- recovery workflows