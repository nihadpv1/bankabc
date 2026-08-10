**Yes, but do not simply add one static `advertised-listen-address` value to the shared StatefulSet.** Because you have two replicas, each DS pod must advertise its own stable, cross-cluster-reachable address.

## First verify the supported variable

Check whether your ForgeOps DS image already supports the environment variable:

```bash
oc kustomize ../../../vendor/forgeops-2026.1/ds/idrepo \
  | grep -i -A2 -B2 'ADVERTISED\|LISTEN_ADDRESS'
```

Also inspect the running pod:

```bash
oc exec -n pngdr ds-idrepo-0-dr -- \
  env | grep -Ei 'DS_|ADVERTISED|HOSTNAME'
```

If the image supports it, the configuration may look like:

```yaml
- name: DS_ADVERTISED_LISTEN_ADDRESS
  value: "ds-idrepo-0-dr-repl.arabbanking.local"
```

But this value is only correct for `ds-idrepo-0-dr`. The second pod must advertise a different address:

```text
ds-idrepo-1-dr-repl.arabbanking.local
```

PingDS uses the advertised address when publishing the address that other servers should use to connect to it. The bootstrap server setting only provides initial topology discovery. [docs.pingidentity](https://docs.pingidentity.com/pingds/8/config-guide/repl-listen.html)

## Do not use one address for both pods

This would be unsafe:

```yaml
- name: DS_ADVERTISED_LISTEN_ADDRESS
  value: "dr-worker-1.arabbanking.local"
```

with `replicas: 2`, because both DS instances would advertise the same endpoint. Replication traffic could be routed to the wrong DS identity.

You need a design such as:

```text
ds-idrepo-0-dr-repl.arabbanking.local  -> DR DS pod 0 -> 30989
ds-idrepo-1-dr-repl.arabbanking.local  -> DR DS pod 1 -> 30989
```

Each address must:

- resolve from both clusters;
- route to the correct pod;
- use a certificate containing that FQDN in its SAN;
- allow TCP `30989` in both directions;
- optionally expose TCP `30444` for administration and `dsreplication status`.

## Check the effective PingDS setting

Before changing the manifest, check what is currently advertised:

```bash
oc exec -n pngdr ds-idrepo-0-dr -- \
  dsconfig get-global-configuration-prop \
  --property advertised-listen-address \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword '<password>' \
  --trustAll
```

And:

```bash
oc exec -n pngdr ds-idrepo-0-dr -- \
  dsconfig get-replication-server-prop \
  --provider-name "Multimaster Synchronization" \
  --property advertised-listen-address \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword '<password>' \
  --trustAll
```

If the value is still `*.svc.cluster.local`, adding an arbitrary environment variable will not necessarily change it. The ForgeOps startup scripts must explicitly consume that variable, or you must configure the PingDS property through the supported image customization/configuration mechanism.

## Recommended sequence

1. Create stable DNS names and per-pod network endpoints.
2. Ensure the names resolve from both clusters.
3. Ensure the certificate SANs contain those names.
4. Confirm the ForgeOps image’s supported advertised-address variable.
5. Add the variable to the Kustomize patch only if the image consumes it.
6. Render and inspect the final manifest:

```bash
oc kustomize ds-idrepo/ | \
  yq 'select(.kind=="StatefulSet") |
      .spec.template.spec.containers[] |
      select(.name=="ds").env'
```

7. Apply and recreate the pods if the environment is startup-only:

```bash
oc apply -k ds-idrepo/
oc delete pod ds-idrepo-0-dr ds-idrepo-1-dr -n pngdr
```

8. Confirm the effective advertised addresses and rerun replication status.

The safest production approach is to follow the ForgeOps multi-cluster topology pattern, which generates externally reachable per-cluster/per-replica addresses rather than relying on Kubernetes-internal service names. [community.forgerock](https://community.forgerock.com/t/deploying-forgerock-directory-services-on-a-kubernetes-multi-cluster-using-google-cloud-multi-cluster-services-mcs/92)


Your worker-node approach solves only the **initial network path and TLS name**, not the PingDS topology advertisement.

## Why this still happens

You configured DR bootstrap servers as something like:

```text
prbhvspngprw1.arabbanking.local:30989
```

That allows DR to make the first connection to primary. However, after connecting, PingDS exchanges topology information and learns the server addresses that each DS/RS advertises. Your logs show those advertised addresses are still:

```text
ds-idrepo-0.ds-idrepo.pngdr.svc.cluster.local
ds-idrepo-1.ds-idrepo.pngdr.svc.cluster.local
```

and primary has learned equivalent `pngprd.svc.cluster.local` names.

The bootstrap address and the advertised address are different settings:

- `DS_BOOTSTRAP_REPLICATION_SERVERS`: where DS initially connects for discovery.
- `advertised-listen-address`: the address DS tells other DS/RS instances to use afterwards.

PingDS documents the advertised address as the hostname or IP clients should use to connect to that server.  The bootstrap servers only provide topology discovery. [docs.pingidentity](https://docs.pingidentity.com/pingds/8/config-guide/repl-listen.html)

Your certificate for the worker-node names may be perfectly valid, but PingDS is not attempting to connect using those names after discovery. It is attempting to connect using the internal Kubernetes names, so DNS fails before the certificate becomes relevant.

## What must be changed

Each DS/replication-server identity needs a **stable, cross-cluster-reachable advertised hostname**.

For example:

```text
ds-idrepo-0-dr-repl.arabbanking.local
ds-idrepo-1-dr-repl.arabbanking.local
ds-idrepo-0-primary-repl.arabbanking.local
ds-idrepo-1-primary-repl.arabbanking.local
```

These names must resolve from both clusters and route to the correct DS/replication-server instance.

Do not advertise the same generic worker-node name for both StatefulSet replicas. Both replicas are different DS identities, so they need distinct addresses. A normal NodePort that load-balances both pods behind one address can send traffic to the wrong DS replica and is unsafe for a stateful replication topology.

## Configure advertised addresses

First inspect the current values:

```bash
oc exec -n pngdr ds-idrepo-0-dr -- \
  dsconfig get-global-configuration-prop \
  --property advertised-listen-address \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword '<password>' \
  --trustAll
```

Inspect the replication-server value too:

```bash
oc exec -n pngdr ds-idrepo-0-dr -- \
  dsconfig get-replication-server-prop \
  --provider-name "Multimaster Synchronization" \
  --property advertised-listen-address \
  --hostname localhost \
  --port 4444 \
  --bindDN "uid=admin" \
  --bindPassword '<password>' \
  --trustAll
```

For newer PingDS/ForgeOps configurations, the deployment commonly uses an environment variable such as:

```yaml
- name: DS_ADVERTISED_LISTEN_ADDRESS
  value: "ds-idrepo-0-dr-repl.arabbanking.local"
```

But the value must be unique per pod. Do not put the same static value into both `ds-idrepo-0` and `ds-idrepo-1`.

ForgeOps multi-cluster patterns solve this by generating cross-cluster FQDNs and setting the advertised-listen-address dynamically for each DS pod. [community.forgerock](https://community.forgerock.com/t/deploying-forgerock-directory-services-on-a-kubernetes-multi-cluster-using-google-cloud-multi-cluster-services-mcs/92)

## Better network design

The preferred design is:

```text
Primary DS-0  <── dedicated reachable address ──>  DR DS-0
Primary DS-1  <── dedicated reachable address ──>  DR DS-1
```

For each address, provide:

- replication port, for example `30989`;
- administration port, for example `30444`, if remote administration/status is required;
- DNS resolution from both clusters;
- firewall access in both directions;
- TLS certificate SAN containing the exact advertised FQDN;
- routing to the correct DS pod, not an arbitrary replica.

Your certificate preparation is useful, but make sure the certificate SANs contain the **actual names PingDS advertises**, not only the worker-node names.

## Current state

Your current state is:

- DR can bootstrap to primary: yes.
- TLS to the worker-node/NodePort endpoint: apparently yes.
- Initial replication connection: yes.
- Cross-cluster topology names: incorrect.
- `dsreplication status` from the opposite cluster: expected to show DNS errors.
- Automatic reconnect and failover: not reliable until advertised addresses are fixed.

So the key fix is not another bootstrap entry. It is configuring `DS_ADVERTISED_LISTEN_ADDRESS` or the equivalent PingDS `advertised-listen-address` properties with stable, per-replica, cross-cluster-reachable names. Worker-node certificates alone do not change what PingDS advertises.