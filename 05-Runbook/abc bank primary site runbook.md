

Cluster role sett


```bash
# 1. Create SA
oc apply -f bootstrap/cluster/forgerock-sa.yaml

# 2. Create SCC
oc apply -f bootstrap/cluster/forgerock-scc.yaml

# 3. Bind SCC to SA (THIS WAS MISSING!)
oc adm policy add-scc-to-user forgerock-scc -z forgerock-sa -n forgerock

# 4. Grant namespace admin (optional, or use custom role)
oc adm policy add-role-to-user admin -z forgerock-sa -n forgerock



```

# 1