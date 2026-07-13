

---

## Modify DS password

Update LDAP service account password

```bash
ldappasswordmodify \
--useSsl \
--trustAll \
--hostname localhost \
--port 1636 \
--bindDn "uid=admin" \
--bindPassword:file /var/run/secrets/admin/dirmanager.pw \
--authzId "dn:uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens" \
--newPassword "<password>" \
--no-prompt
```

Expected:

- Password modification succeeds
    
- No LDAP authorization errors
    

Important:

- Commonly used for CTS/OpenAM service users
    
- Update Kubernetes secrets after password changes
    

---

# Replication Operations

## Check replication status

Verify replication topology and convergence

```shell
# exec into DS pod
oc exec -it ds-idrepo-0 -n forgerock -- bash

/opt/opendj/bin/dsrepl status \
  --showReplicas \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

Expected:

- Replica status = GOOD
    
- Entry counts match
    
- Replay delay near 0 ms
    

Important:

- GOOD status alone does NOT guarantee historical convergence
    
- Always verify entry counts
    

---

## Check replication changelog status

Inspect replication replay/changelog state

```shell
oc exec -it ds-idrepo-0 -n forgerock -- bash

/opt/opendj/bin/dsrepl status \
  --showReplicas \
  --showChangeLogs \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

```shell
/opt/opendj/bin/dsrepl status \
  --showReplicas \
  --hostname ds-idrepo-0.primary.replication.kvm.lab \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

DR site:
```shell
/opt/opendj/bin/dsrepl status \
  --showReplicas \
  --hostname ds-idrepo-0.dr.replication.kvm.lab \
  --port 30444 \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

```shell
ldapsearch \
  --hostname localhost \
  --port 1636 \
  --useSsl \
  --trustAll \
  --bindDn "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --baseDn "cn=servers,cn=topology,cn=monitor" \
  "(objectClass=ds-monitor-topology-server)" \
  ds-mon-server-id \
  ds-mon-group-id \
  ds-mon-last-seen
```


Expected:

- Changelog replay information displayed
    
- No replication backlog errors
    

Important:

- Useful during replication lag investigations
    
- Helps diagnose BAD TOO LATE situations
    

---

## Search for specific user/data

Example search for uid `test2`

```shell
/opt/opendj/bin/ldapsearch \
  --hostname localhost \
  --port 1636 \
  --useSsl \
  --trustAll \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --baseDN "ou=identities" \
  "(uid=test2)"
```

Expected:

- Matching LDAP entry returned
    

Important:

- Best validation method for replication consistency
    
- Useful after replica initialization
    

---

## List replication domains

Show all replicated base DNs

```shell
/opt/opendj/bin/dsconfig list-replication-domains \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

Expected:

```text
ou=identities
ou=am-config
dc=openidm,dc=forgerock,dc=io
```

Important:

- These base DNs must be initialized during replica bootstrap
    
- Replication initialize works on base DNs, not backend IDs
    

---

## List replication server

Show DS replication infrastructure

```shell
/opt/opendj/bin/dsconfig list-replication-server \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

Expected:

```text
replication-server : 8989
```

Important:

- Port 8989 is DS-to-DS replication traffic
    
- Not used for LDAP client authentication
    

---

## Initialize replication

Force historical data synchronization

```shell
/opt/opendj/bin/dsrepl initialize \
  --baseDn ou=identities \
  --baseDn ou=am-config \
  --baseDn dc=openidm,dc=forgerock,dc=io \
  --toAllServers \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

Expected:

- Entries processed successfully
    
- Initialization completes without errors
    

Important:

- Required after StatefulSet scale-up
    
- Replica join alone may NOT synchronize historical data
    

---

# Backend Operations

## List DS backends

Show backend IDs used by DS

```shell
/opt/opendj/bin/backendstat list-backends
```

Expected:

```text
amIdentityStore
cfgStore
idmRepo
```

Important:

- Backend IDs are used for export/import
    
- Replication initialize uses base DNs instead
    

---

## Export backend (LDIF backup)

Create logical LDAP export

```shell
/opt/opendj/bin/export-ldif \
  --backendID amIdentityStore \
  --ldifFile /tmp/amIdentityStore.ldif
```

Expected:

- Export task completes successfully
    
- LDIF file created
    

Important:

- Useful for migration and recovery
    
- Safer than raw filesystem copy
    

---

## Show all sites

```bash
/opt/opendj/bin/ldapsearch \
  --hostname localhost \
  --port 1636 \
  --useSsl \
  --trustAll \
  --bindDn "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --baseDn "cn=replication,cn=monitor" \
  "(|(objectClass=ds-monitor-replica)(ds-mon-server-id=ds-idrepo-0-primary))" \
  ds-mon-server-id ds-mon-domain-name ds-mon-status ds-mon-receive-delay
```

```shell
/opt/opendj/bin/ldapsearch \
  --hostname localhost \
  --port 1636 \
  --useSsl \
  --trustAll \
  --bindDn "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --baseDn "cn=replication,cn=monitor" \
  "(ds-mon-server-id=ds-idrepo-0-dr)" \
  ds-mon-server-id ds-mon-connection-status ds-mon-sent-updates ds-mon-received-updates
```

---
## Import backend (restore)

First copy the file to pod:
```shell
oc cp ldif_backup/<file>.ldif forgerock/ds-idrepo-0:/tmp/<file>.ldif
```


Restore backend from LDIF

```shell
/opt/opendj/bin/import-ldif \
  --backendID amIdentityStore \
  --ldifFile /tmp/amIdentityStore.ldif
```

Expected:

- Import completes successfully
    

Important:

- Usually performed offline
    
- Used for DR and backend recovery


```shell
/opt/opendj/bin/import-ldif \
  --backendID amIdentityStore \
  --ldifFile /tmp/amIdentityStore_backup.ldif \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

```shell
/opt/opendj/bin/import-ldif \
--backendID idmRepo \
--ldifFile /tmp/idmRepo_backup.ldif \
--hostname localhost \
--port 4444 \
--bindDN "uid=admin" \
--bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
--trustAll
```



---

## Inspect topology visibility


```shell
  ldapsearch \
  --hostname localhost \
  --port 4444 \
  --useSsl \
  --trustAll \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --baseDN "cn=servers,cn=topology,cn=monitor" \
  "(objectClass=*)" \
  ds-mon-server-id
```



# Health & Performance

## Check DS status

Quick DS operational health check

```shell
/opt/opendj/bin/status \
  --bindDN "uid=admin" \
  --bindPassword 'meWIV0No0slpueta6UNcy0jq804YWXvj' \
  --trustAll
```

Expected:

- Server status ONLINE
    
- Ports/listeners active
    
- Backends healthy
    

Important:

- Fastest overall DS health validation command
    

---

## Show changelog statistics

Inspect replication activity

```shell
/opt/opendj/bin/changelogstat
```

Expected:

- Replication throughput statistics
    
- Changelog activity displayed
    

Important:

- Useful for replication lag analysis
    

---

## Verify indexes

Check LDAP index consistency

```shell
/opt/opendj/bin/verify-index \
  --baseDN ou=identities
```

Expected:

- Index verification successful
    

Important:

- Useful during performance troubleshooting
    

---

## Rebuild indexes

Rebuild LDAP indexes

```shell
/opt/opendj/bin/rebuild-index \
  --baseDN ou=identities
```

Expected:

- Index rebuild completes successfully
    

Important:

- Resource intensive operation
    
- May impact production performance