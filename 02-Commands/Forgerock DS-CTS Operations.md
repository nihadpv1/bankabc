## Verify DS base data

Query root DSE and verify DS is responding

```bash
kubectl exec -it -n forgerock ds-idrepo-0 -- sh -c 'ldapsearch --useSsl --trustAll -h localhost -p 1636 -D "uid=admin" -w "$(cat /var/run/secrets/admin/dirmanager.pw)" -b "" -s base "(objectclass=*)"'
```

Expected:

- Naming contexts returned
    
- DS responds successfully
    
- No bind/TLS errors
    

Important:

- Good for quick DS connectivity validation
    
- Useful during startup/TLS troubleshooting
    

---

## Query identity store

Check AM identity data

```bash
kubectl exec -it -n forgerock ds-idrepo-0 -- sh -c 'ldapsearch --useSsl --trustAll -h localhost -p 1636 -D "uid=admin" -w "$(cat /var/run/secrets/admin/dirmanager.pw)" -b "ou=identities" "(objectclass=*)"'
```

Expected:

- User entries returned
    
- LDAP search succeeds
    

Important:

- Verifies AM identity backend health
    
- Useful for checking missing users/replication issues
    

---

## Query CTS tokens

Inspect AM session/token store

```bash
kubectl exec -it -n forgerock ds-cts-0 -- sh -c 'ldapsearch --useSsl --trustAll -h localhost -p 1636 -D "uid=admin" -w "$(cat /var/run/secrets/admin/dirmanager.pw)" -b "ou=tokens" "(objectclass=*)"'
```

Expected:

- CTS token entries visible
    
- Sessions/tokens present
    

Important:

- Useful for AM login/session troubleshooting
    
- Large environments may return many entries
    
