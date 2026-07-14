
## Issue

Running ForgeOps prereqs failed with:

```bash
error: current-context must exist in order to minify
```

---

## Cause

`kubectl` kubeconfig had:

- valid context configured
    
- but no active `current-context` selected
    

---

## Verification

Check current context:

```bash
kubectl config current-context
```

Returned:

```bash
error: current-context is not set
```

List available contexts:

```bash
kubectl config get-contexts
```

Output:

```bash
kubernetes-admin@kubernetes
```

---

## Concept

Kubernetes kubeconfig stores:

- cluster information
    
- authentication info
    
- contexts
    

A context binds:

- cluster
    
- user
    
- optional namespace
    

`current-context` defines the active/default context used by:

- kubectl
    
- Helm
    
- ForgeOps
    
- Kubernetes tools
    

ForgeOps internally uses:

```bash
kubectl config view --minify
```

This command requires an active `current-context`.

Even though some `kubectl` commands may still work, ForgeOps fails because it cannot safely determine the active cluster context.

---

## Fix

Set the active context manually:

```bash
kubectl config use-context kubernetes-admin@kubernetes
```

---

## Verification After Fix

```bash
kubectl config current-context
```

Expected output:

```bash
kubernetes-admin@kubernetes
```

Result:

- ForgeOps prereqs command worked successfully afterward.
    

---

## Additional Finding

ForgeOps prereqs uses Helm internally.

Default command:

```bash
./bin/forgeops prereqs
```

Installs:

- ingress
    
- cert-manager
    
- secret-agent
    

Using:

```bash
./bin/forgeops prereqs --secret-generator secrets
```

Installed:

- mittwald/kubernetes-secret-generator
    

without reinstalling:

- ingress
    
- cert-manager