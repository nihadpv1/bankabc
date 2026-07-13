I reviewed all of the uploaded notes, workbooks, recovery exercises, replication labs, scaling runbooks, and deployment notes. The biggest change I would make before turning this into a production runbook is:

### What I would change from the lab design

Your lab was constrained by resources, CRC limitations, and lack of enterprise infrastructure.

For a production runbook, I would **separate "Lab Findings" from "Production Standard."**

Examples:

|Lab|Production|
|---|---|
|CRC|OpenShift HA cluster|
|Single DS replica|3 DS replicas per site|
|socat + NodePort|Load Balancer / F5 / VIP|
|`ds-idrepo-0.primary.replication.kvm.lab`|`ds-repl-primary.company.com`|
|Bitnami nginx latest|Pinned image digest|
|Manual apply sequence|GitOps later|
|Route passthrough TLS|Edge TLS (per your requirement)|
|Internal cert-manager CA|Enterprise PKI preferred|
|Single node validation|Anti-affinity and spread constraints|

---

## Production Architecture Recommended

Based on everything validated in the labs:

```text
                         Internet
                             |
                     External DNS
                     iam.company.com
                             |
                     OpenShift Route
                     (Edge TLS)
                             |
                        fr-proxy
                          (HTTP)
                             |
 ---------------------------------------------------
 |         |         |         |         |          |
 AM     LoginUI     IDM    AdminUI   EndUserUI   APIs
 |
 |
 DS-IDRepo (3 replicas)
 |
 DS-CTS (3 replicas)
 |
 ---------------------------------------------------
                     DS Replication
 ---------------------------------------------------
 |
 DR Site
 |
 DS-IDRepo (3 replicas)
 |
 Warm Standby
```

---

# Production Design Decisions

### External URL

Users always access:

```text
https://iam.company.com
```

Same hostname during failover.

---

### Replication DNS

Separate from application URL.

Example:

Primary:

```text
ds-repl-primary.company.com
```

DR:

```text
ds-repl-dr.company.com
```

This follows one of the biggest discoveries from the replication labs: application hostname and replication identity are separate concerns.

---

### DS Group IDs

Primary:

```yaml
DS_GROUP_ID=primary
```

DR:

```yaml
DS_GROUP_ID=dr
```

Never use identical group IDs.

This directly caused topology contamination during lab testing.

---

### Bootstrap Rule

Primary:

```yaml
DS_BOOTSTRAP_REPLICATION_SERVERS=ds-repl-primary.company.com:8989
```

DR:

```yaml
DS_BOOTSTRAP_REPLICATION_SERVERS=ds-repl-primary.company.com:8989
```

Important:

`DS_BOOTSTRAP_REPLICATION_SERVERS` is only a discovery seed.

It is not a topology definition.

---

# Production Deployment Sequence

This was repeatedly validated throughout the recovery exercises.

```text
1. OpenShift validation
2. DNS validation
3. Namespace creation
4. Service account creation
5. SCC assignment
6. cert-manager validation
7. Secret Generator deployment
8. Secrets deployment
9. Keystore creation
10. DS-IDRepo
11. DS-CTS
12. DS password sync
13. AM
14. AMster
15. IDM
16. Login UI
17. Admin UI
18. End User UI
19. fr-proxy
20. Route
21. Validation
```

Never bulk deploy everything.

---

# OpenShift Foundation

```bash
oc create namespace forgerock

oc create sa forgerock-sa -n forgerock

oc adm policy add-scc-to-user anyuid \
-z forgerock-sa \
-n forgerock

oc adm policy add-role-to-user admin \
-z forgerock-sa \
-n forgerock
```

Every workload must contain:

```yaml
serviceAccountName: forgerock-sa
```

This was one of the major findings from the OpenShift deployment exercises.

---

# Internal TLS Design

Since you explicitly stated:

```text
No Vault
No Internal PKI
No Service Mesh
```

Recommended:

```text
cert-manager
      +
Self-managed internal CA
```

Used only for:

```text
DS ↔ DS
AM ↔ DS
Internal platform communication
```

External ingress certificate:

```text
iam.company.com
```

Should come from:

```text
Public CA
Enterprise Certificate
```

and terminate at Route (Edge TLS).

---

# Route Design

You mentioned:

```text
External TLS
Route Edge TLS
HTTP between Route and nginx
```

Therefore the lab passthrough design should be replaced.

Production Route:

```yaml
tls:
  termination: edge
```

Traffic flow:

```text
Client HTTPS
      |
OpenShift Route
(Edge TLS)
      |
HTTP
      |
fr-proxy
      |
Platform Services
```

This differs from the CRC lab architecture.

---

# Nginx Design

Production nginx should:

### Keep

```nginx
/healthz

X-Forwarded-Host
X-Forwarded-Proto
X-Forwarded-Port

Websocket support

OAuth routing
```

Validated in lab.

### Remove

```nginx
ssl_certificate
ssl_certificate_key
listen 8443 ssl
```

Because Route performs TLS termination.

Use:

```nginx
listen 8080;
```

or

```nginx
listen 80;
```

internally.

---

# DS Production Standard

Instead of:

```yaml
replicas: 1
```

Use:

```yaml
replicas: 3
```

And add:

```yaml
podAntiAffinity
topologySpreadConstraints
```

This aligns with the production topology planning notes.

---

# DR Standard

From all replication testing, the most operationally sane model was:

```text
PRIMARY
Full IAM

DR
DS only
```

Do not run:

```text
AM
IDM
UIs
```

continuously in DR unless business requirements demand active-active.

This was one of the strongest conclusions from the DR exercises.

---

# DR Onboarding Procedure

```text
1. Deploy DR namespace
2. Deploy DR DS
3. Bootstrap to PRIMARY
4. Validate topology
5. Initialize replication
6. Validate entry counts
7. Validate historical objects
8. Validate replay delay
```

Never:

```text
PRIMARY <-> DR dual bootstrap
```

This caused topology contamination in testing.

---

# Backup Standard

Back up:

```text
amIdentityStore
idmRepo
```

Commands:

```bash
backendstat list-backends
```

```bash
export-ldif \
--backendID amIdentityStore \
--ldifFile amIdentityStore.ldif
```

```bash
export-ldif \
--backendID idmRepo \
--ldifFile idmRepo.ldif
```

Avoid restoring historical cfgStore blindly.

Instead:

```text
Restore identities
Restore idmRepo
Replay AMster
```

This was one of the most important recovery findings.

---

# Production Readiness Gates

Before go-live:

```text
✓ SCC validated
✓ cert-manager healthy
✓ Secrets generated
✓ DS healthy
✓ DS replication healthy
✓ Replay delay = 0
✓ AM healthy
✓ IDM healthy
✓ OAuth validated
✓ AMster tested
✓ Backup tested
✓ Restore tested
✓ DR bootstrap tested
✓ DR initialize tested
✓ Route failover tested
✓ Monitoring enabled
✓ Resource sizing validated
✓ Immutable images used
```

---

### One architectural recommendation

For a real production deployment, I would not keep the runbook tied to:

```text
ds-idrepo-0.primary.replication.company.com
```

Instead I would introduce:

```text
ds-repl-primary.company.com
```

behind an F5/VIP/load balancer and let DS replicas sit behind that stable endpoint.

That removes pod-specific DNS from the DR onboarding process and is much closer to how large ForgeRock/Ping Identity deployments are operated. This is the main architectural improvement I would make beyond what was validated in the lab.