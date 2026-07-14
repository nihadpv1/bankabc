## Initialize ForgeOps environment
Create lab overlay configuration

```bash
./bin/forgeops env \
-e lab \
-n forgerock \
-i nginx \
-f iam.apps.kvm.lab \
--single-instance \
--small \
--skip-issuer
````

---

## Configure ForgeOps tooling

Install dependencies and setup environment

```bash
./bin/forgeops configure
```

---

## Verify ForgeOps version

Check installed version

```bash
./bin/forgeops version
```

---