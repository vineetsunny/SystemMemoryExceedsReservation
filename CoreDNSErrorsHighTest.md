# Reproduce CoreDNSErrorsHigh

Reproduce the OpenShift CoreDNSErrorsHigh alert by intentionally causing CoreDNS to return SERVFAIL responses for .com DNS queries.

Lab Only: This test intentionally affects .com DNS resolution for workloads using the cluster DNS configuration. Perform only on an isolated/test cluster.

---

## Procedure

***1. Save the current DNS configuration***

Create a backup before making any changes.

```bash
oc get dns.operator/default -o yaml > dns-before-test.yaml
```

Check whether custom DNS server entries already exist:

```bash
oc get dns.operator/default \
  -o jsonpath='{.spec.servers}{"\n"}'
```

`Note:` 

The following test assumes that .spec.servers is empty or not configured.
If existing entries are present, do not overwrite them with the test patch. Preserve the existing configuration and add the test entry appropriately.

***2. Configure .com to use an unreachable upstream***

For  test, using the IP 192.0.2.1 as an unreachable DNS upstream.

```bash
oc patch dnses.operator.openshift.io/default \
  --type=merge \
  -p '{
    "spec": {
      "servers": [
        {
          "name": "break-external",
          "zones": ["com"],
          "forwardPlugin": {
            "upstreams": ["192.0.2.1"]
          }
        }
      ]
    }
  }'
```

This causes DNS queries for domains as *.com.

***3. Create the DNS test pod***

```bash
oc run dns-test -n default \
  --image=registry.access.redhat.com/ubi9/ubi-minimal \
  --restart=Never \
  -- sleep 3600
```


***4. Verify the test pod uses OpenShift DNS***

Check /etc/resolv.conf:

```bash
oc exec dns-test -n default -- cat /etc/resolv.conf
```
You should see the OpenShift DNS service IP, for example:
nameserver 172.31.0.10

This confirms that the pod is using the cluster DNS service.

***5. Confirm .com resolution is failing***

```bash
oc exec dns-test -n default -- \
  sh -c 'getent ahosts example.com; echo rc=$?'
```

The return code should be non-zero.

For example: rc=2

***6. Verify that CoreDNS is recording SERVFAIL***

Before starting continuous traffic, check the SERVFAIL rate:

```bash
sum by(namespace) (
  rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])
)
```
Initially, the value may be 0 or very low.

***7. Generate continuous failed DNS requests***

Start continuous DNS resolution requests:

```bash
oc exec dns-test -n default -- \
  sh -c 'while true; do getent ahosts example.com >/dev/null 2>&1; sleep 0.1; done'
```

Leave this command running.

It continuously generates DNS requests for example.com, which should be forwarded by CoreDNS to the unreachable upstream.

***8. Monitor the SERVFAIL percentage***

In Prometheus, run the exact expression used by the alert:

```bash
(
  sum by(namespace) (
    rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])
  )
  /
  sum by(namespace) (
    rate(coredns_dns_responses_total[5m])
  )
)
```
The result must become: > 0.01

***9. Verify CoreDNSErrorsHigh***

Check Console: Observe > Alerting

---

# Recovery / Cleanup

***1. Stop the DNS test traffic***

Return to the terminal running the continuous loop and press:

Ctrl+C

***2. Delete the test pod***

```bash
oc delete pod dns-test -n default
```

***3. Restore the original DNS configuration***

If the original configuration had no spec.servers, remove the test configuration:

```bash
oc patch dnses.operator.openshift.io/default \
  --type=json \
  -p='[{"op":"remove","path":"/spec/servers"}]'
```

Do not use this command if the cluster had existing spec.servers entries. Restore the original configuration from dns-before-test.yaml instead.

***4. Verify Operator:***

```bash
oc get dns.operator/default -o yaml
```

```bash
oc get co dns
```

Wait until:

Available=True
Progressing=False
Degraded=False

***5. Verify DNS resolution has recovered***

Create a temporary recovery test pod:

```bash
oc run dns-recovery-test -n default \
  --image=registry.access.redhat.com/ubi9/ubi-minimal \
  --restart=Never \
  -- sleep 3600
```
Wait for now:

```bash
oc wait --for=condition=Ready \
  pod/dns-recovery-test \
  -n default \
  --timeout=120s
```

You should get:
pod/dns-recovery-test condition met

***6. Test DNS:***

```bash
oc exec dns-recovery-test -n default -- \
  getent ahosts example.com
```
A successful response should contain an IP address.

***7. Delete the pod:***

oc delete pod dns-recovery-test -n default

