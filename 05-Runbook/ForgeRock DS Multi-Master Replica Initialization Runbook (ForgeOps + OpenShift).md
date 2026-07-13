
This runbook documents the correct operational procedure for scaling ForgeRock DS (`ds-idrepo`) replicas in ForgeOps on OpenShift.

Environment:

- ForgeOps on OpenShift CRC
    
- StatefulSet-based DS deployment
    
- OpenShift headless service
    
- cert-manager internal PKI
    
- Multi-master replication
    

---

# Problem Statement

Simply scaling the StatefulSet:

```bash
oc scale sts ds-idrepo --replicas=2 -n forgerock
```

DOES NOT guarantee historical data synchronization.

Observed behavior:

|Validation|Result|
|---|---|
|Replication topology|GOOD|
|New writes replication|Working|
|Historical entries|Missing|
|Entry counts|Mismatch|

This causes incomplete DS convergence.

---

# Root Cause

New DS replicas:

- join replication topology
    
- receive NEW changes only
    

BUT:

- historical backend initialization is NOT automatically guaranteed
    

Explicit replication initialization is required.

---

# Correct Operational Lifecycle

```text
Scale Replica
→ Verify Replication Topology
→ Initialize Replication Domains
→ Validate Entry Counts
→ Validate Historical Entries
→ Validate Bidirectional Replication
→ Only Then Trust Replica
```

---

# Step 1 — Scale StatefulSet

```bash
oc scale sts ds-idrepo --replicas=2 -n forgerock
```

Verify pod creation:

```bash
oc get pods -n forgerock -w
```

Expected:

```text
ds-idrepo-0   Running
ds-idrepo-1   Running
```

---

# Step 2 — Verify Replication Topology

Enter DS pod:

```bash
oc rsh -n forgerock ds-idrepo-0
```

Run replication status:

```bash
/opt/opendj/bin/dsrepl status \
  --showReplicas \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --trustAll
```

---

# Expected Initial Result (Before Initialize)

Example broken state:

```text
ou=identities

ds-idrepo-0   GOOD   Entry count: 23
ds-idrepo-1   GOOD   Entry count: 1
```

Symptoms:

- Status = GOOD
    
- Historical entries missing
    
- Entry count mismatch
    

IMPORTANT:  
`GOOD` does NOT guarantee historical convergence.

---

# Step 3 — Identify Replication Domains

The `dsrepl status` output shows all replicated base DNs.
Command:
```shell
/opt/opendj/bin/dsconfig list-replication-domains \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --trustAll
```


Example:

| Replication Domain              | base-dn                       |
| ------------------------------- | ----------------------------- |
| `cn=schema `                    | cn=schema                     |
| `ou=am-config`                  | dc=openidm,dc=forgerock,dc=io |
| `dc=openidm,dc=forgerock,dc=io` | ou=am-config                  |
| `ou=identities `                | ou=identities                 |
| `uid=monitor`                   | uid=monitor                   |

Critical application domains in ForgeOps:

| Base DN                         | Purpose             |
| ------------------------------- | ------------------- |
| `ou=identities`                 | AM users/identities |
| `ou=am-config`                  | AM configuration    |
| `dc=openidm,dc=forgerock,dc=io` | IDM repository      |

These must be initialized.

---

# Step 4 — Initialize Replication Domains

Run initialization from authoritative replica (`ds-idrepo-0`):

```bash
/opt/opendj/bin/dsrepl initialize \
  --baseDn ou=identities \
  --baseDn ou=am-config \
  --baseDn dc=openidm,dc=forgerock,dc=io \
  --toAllServers \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --trustAll
```

---

# Expected Initialize Output

Example:

```text
Starting initialization from 'ds-idrepo-0-default'

Starting initialization for base DN: 'ou=identities'
23 entries processed (100 % complete)

Starting initialization for base DN: 'ou=am-config'
126 entries processed (100 % complete)

Starting initialization for base DN: 'dc=openidm,dc=forgerock,dc=io'
66 entries processed (100 % complete)

Done
```

---

# Step 5 — Validate Replication Convergence

Run again:

```bash
/opt/opendj/bin/dsrepl status \
  --showReplicas \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --trustAll
```

Expected healthy state:

```text
ou=identities

ds-idrepo-0   GOOD   Entry count: 23
ds-idrepo-1   GOOD   Entry count: 23
```

Validation criteria:

|Validation|Expected|
|---|---|
|Status|GOOD|
|Entry counts|Equal|
|Replay delay|0 ms|
|Receive delay|0 ms|

---

# Step 6 — Validate Historical Entries

Check old user exists on new replica.

Enter replica:

```bash
oc rsh -n forgerock ds-idrepo-1
```

Run:

```bash
/opt/opendj/bin/ldapsearch \
  --hostname localhost \
  --port 1636 \
  --useSsl \
  --trustAll \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --baseDN "ou=identities" \
  "(uid=nihad)"
```

Expected:

- User entry returned successfully
    

If historical user missing:

- initialization incomplete
    
- re-run `dsrepl initialize`
    

---

# Step 7 — Validate Bidirectional Replication

Create a test user through IDM/AM/UI or LDAP.

Then verify:

## On ds-idrepo-1

```bash
ldapsearch ...
(uid=test2)
```

## On ds-idrepo-0

```bash
ldapsearch ...
(uid=test2)
```

Expected:

- User exists on BOTH replicas
    

This confirms:

- multi-master replication healthy
    
- bidirectional synchronization working
    

---

# Important Operational Lessons

## 1. StatefulSet Scale-Up Alone Is NOT Enough

Incorrect assumption:

```text
Scale StatefulSet
→ replication automatically converges
```

Reality:

```text
Scale StatefulSet
→ replica joins topology
→ explicit initialize required
```

---

# 2. Replication GOOD != Data Converged

`dsrepl status` showing:

```text
GOOD
```

does NOT guarantee:

- historical entries present
    
- entry counts equal
    
- application consistency
    

Always validate:

- entry counts
    
- historical objects
    
- bidirectional replication
    

---

# 3. ForgeRock DS Is Stateful Infrastructure

Kubernetes only validates:

```text
pod running
```

ForgeRock DS additionally requires:

```text
replication convergence
backend initialization
data consistency
```

---

# 4. Important ForgeOps/OpenShift Behavior

Running:

```bash
/opt/opendj/bin/stop-ds
```

inside DS container terminates container lifecycle.

Reason:

- DS process is tied to container PID1
    

Implication:

- traditional VM operational procedures differ from containerized DS behavior
    

---

# Recommended Production Procedure

## Standard Replica Expansion

```text
1. Scale StatefulSet
2. Wait for replica pod healthy
3. Run dsrepl initialize
4. Validate entry counts
5. Validate historical entries
6. Validate replay delay
7. Validate bidirectional writes
8. Only then trust replica for production traffic
```

---

# Useful Commands Summary

## Replication Status

```bash
/opt/opendj/bin/dsrepl status \
  --showReplicas \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --trustAll
```

## Initialize Replication

```bash
/opt/opendj/bin/dsrepl initialize \
  --baseDn ou=identities \
  --baseDn ou=am-config \
  --baseDn dc=openidm,dc=forgerock,dc=io \
  --toAllServers \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --trustAll
```

## LDAP Validation

```bash
/opt/opendj/bin/ldapsearch \
  --hostname localhost \
  --port 1636 \
  --useSsl \
  --trustAll \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --baseDN "ou=identities" \
  "(uid=testuser)"
```

## List Backends

```bash
/opt/opendj/bin/backendstat list-backends
```

## List Replication Domains

```bash
/opt/opendj/bin/dsconfig list-replication-domains \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'PASSWORD' \
  --trustAll
```