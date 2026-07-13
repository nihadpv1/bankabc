

## Label Nodes Worker node 1 and 2

```shell
oc label node worker1 node-role.kubernetes.io/forgerock=""  
oc label node worker2 node-role.kubernetes.io/forgerock=""
```

## Taint nodes (Optional)

```shell
oc adm taint node worker1 dedicated=forgerock:NoSchedule
oc adm taint node worker2 dedicated=forgerock:NoSchedule
```

## Create New project or namespace

```shell
oc new-project forgerock
```

## Create new sa and apply role and custom SCC

```shell
# 1. Create SA
oc apply -f bootstrap/cluster/forgerock-sa.yaml

# 2. Create SCC
oc apply -f bootstrap/cluster/forgerock-scc.yaml

# 3. Bind SCC to SA (THIS WAS MISSING!)
oc adm policy add-scc-to-user forgerock-scc -z forgerock-sa -n forgerock

# 4. Grant namespace admin (optional, or use custom role)
oc adm policy add-role-to-user admin -z forgerock-sa -n forgerock
```