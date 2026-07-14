
Let’s verify properly first.

Run:

```
curl -k -i \-u openidm-admin:'rSLOvsc0wZdyup5ebyrSlGLw' \https://iam.apps.kvm.lab/openidm/info/ping
```

and:

```
curl -k -i \-u openidm-admin:'rSLOvsc0wZdyup5ebyrSlGLw' \'https://iam.apps.kvm.lab/openidm/managed/user?_queryFilter=true'
```

The `-i` helps show:

- HTTP status
- headers
- auth behavior