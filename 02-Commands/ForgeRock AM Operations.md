
## Check AM logs
Monitor runtime logs

```bash
kubectl logs -n forgerock deploy/am -f
````

---

## Check AM environment

Verify CTS configuration

```bash
kubectl exec -it -n forgerock deploy/am -- env | grep CTS
```

---

## Debug AM errors

Search for failures in logs

```bash
kubectl logs deploy/am -n forgerock | grep -Ei "error|warn|invalid|failed|credential"
```

---

## Verify AM health

Test ingress access

```bash
curl -kv -H "Host: iam.apps.kvm.lab" https://192.168.100.11/am
```

---
