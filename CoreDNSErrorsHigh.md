# CoreDNSErrorsHigh

PrometheusRule Source: `dns` · Alert Severity: `Warning` · Pending Period: `5m` · [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-dns-operator/CoreDNSErrorsHigh.md)

---

## Meaning

The CoreDNSErrorsHigh alert indicates that CoreDNS is returning SERVFAIL responses for more than 1% of DNS requests, sustained for 5 minutes.
The alert is based on the ratio of SERVFAIL responses to all DNS responses:

```bash
(
  sum by(namespace) (
    rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])
  )
  /
  sum by(namespace) (
    rate(coredns_dns_responses_total[5m])
  )
) > 0.01
```

A SERVFAIL response means that CoreDNS could not successfully complete a DNS query. This can occur due to an upstream DNS failure, DNS forwarding issues, or problems resolving records.

---

## Impact 

- Workloads may be unable to resolve internal or external service names, depending on which DNS queries are failing.
- DNS lookup failures can result in connection errors, retries, increased response times, or temporary application unavailability.
- Applications that rely on repeated DNS lookups may experience intermittent failures until DNS resolution is restored.

---

## Diagnosis 

Use the following table to identify the source of `SERVFAIL` responses and determine the appropriate next action.

| Command | What to look for | Action |
|---|---|---|
| `oc get co dns` | DNS Operator is `Degraded=True`, `Progressing=True`, or `Available=False`. | Investigate the DNS Operator conditions and events. Check for reconciliation or configuration errors. |
| `oc describe dns.operator/default` | Degraded conditions, reconciliation errors, or unexpected DNS configuration. | Identify and correct the reported configuration issue through the approved change process. |
| `oc get pods -n openshift-dns -o wide` | DNS pods are not `Running` or `Ready`, or have frequent restarts. | Inspect affected pods, their events, node health, and resource usage. |
| `oc get ds dns-default -n openshift-dns` | Desired, current, ready, or available pod counts do not match. | Investigate why DNS pods are not scheduled or ready, and resolve the underlying issue. |
| `oc logs -n openshift-dns <dns-pod-name> -c dns --since=15m` | Upstream timeouts, forwarding errors, or repeated DNS resolution failures. | Identify the affected upstream or DNS zone and investigate connectivity or configuration. |
| `oc exec -n openshift-dns <dns-pod-name> -c dns -- dig -q <domain-name> @<upstream-dns-ip>` | CoreDNS pod cannot reach the upstream nameserver, or the upstream query returns `SERVFAIL`/timeout. | Verify connectivity from the CoreDNS pod to the upstream nameserver. If the query fails, investigate upstream DNS health, routing, or firewall connectivity. |
| `oc exec -n <namespace> <pod-name> -- cat /etc/resolv.conf` | Incorrect nameserver or unexpected DNS search domains. | Verify the workload's DNS policy and configuration. |
| `oc exec -n <namespace> <pod-name> -- getent ahosts <domain-name>` | DNS lookup fails or returns no address. | Identify whether the issue affects a particular domain or multiple domains. Test from another workload to compare. |
| `dig @<upstream-dns-ip> <domain-name> +time=2 +tries=1` | Upstream query times out, returns `SERVFAIL`, or resolves successfully. | If it fails, investigate upstream DNS health and network connectivity. If it succeeds, investigate the CoreDNS forwarding path. Run from a diagnostic pod with `dig` installed and network access to the upstream. |
| `oc exec -n openshift-dns <dns-pod-name> -c dns -- dig -q <domain-name> @<upstream-dns-ip>` | Query from the CoreDNS pod consistently fails while the same query works from another location. | Investigate network connectivity, firewall rules, routing, or policy between the CoreDNS pods and the upstream nameserver. |
| `oc patch dnses.operator.openshift.io/default --type=merge -p '{"spec":{"logLevel":"Debug"}}'` | Additional DNS forwarding or resolution details are required to identify the source of SERVFAIL responses. | Temporarily enable Debug logging through the approved change process, collect the relevant CoreDNS logs, and revert the log level after troubleshooting. |
| `oc logs -n openshift-dns -c dns -l dns.operator.openshift.io/daemonset-dns=default --follow --max-log-requests=<number-of-coredns-pods> --timestamps` | Detailed DNS request, forwarding, or upstream error information. | Correlate errors with the affected domain, upstream server, and alert time. Avoid leaving Debug logging enabled longer than necessary. |
| `oc exec -n <namespace> <pod-name> -- cat /etc/resolv.conf` | Incorrect nameserver or unexpected DNS search domains. | Verify the workload's DNS policy and configuration. |
| `oc get events -n openshift-dns --sort-by=.lastTimestamp` | Scheduling failures, probe failures, resource pressure, or other DNS pod events. | Address the reported scheduling, node, resource, or pod health issue. |
| `oc get nodes` | Nodes hosting DNS pods are `NotReady` or otherwise unhealthy. | Investigate node health and restore node availability before taking DNS-specific recovery actions. |
| Prometheus: `(sum by (namespace) (rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])) / sum by (namespace) (rate(coredns_dns_responses_total[5m]))) > 0.01` | `SERVFAIL` percentage is above 1%, and identify which namespace has the elevated ratio. | Correlate the affected namespace and time range with workload failures, DNS logs, forwarding configuration, and upstream health. |

---

## Mitigation

Apply the mitigation based on the findings from the diagnosis. Avoid restarting DNS pods or changing forwarding configuration until the underlying cause is identified.

- Identify the affected domain and check whether the SERVFAIL responses are caused by incorrect DNS configuration, upstream DNS failures, or network connectivity issues.

- Verify the DNS Operator status and the health of CoreDNS pods in the openshift-dns namespace.

- If an incorrect forwarding rule or upstream DNS address is identified, correct the affected configuration through the approved change process.

- If upstream DNS is unreachable or returning SERVFAIL, restore connectivity or engage the upstream DNS team.

- If the upstream nameservers is not healthy to respond to the queries by the CoreDNS pods, apply a silence to the alert, until these servers are troubleshooted.

---

## Validation:

- Verify DNS Operator health:

```bash
oc get co dns
```
- Verify DNS pods are ready:

```bash
oc get pods -n openshift-dns -o wide
```

- Test resolution of the affected domain:

```bash
oc exec -n <namespace> <pod> -- getent ahosts <domain-name>
```

- Recheck the SERVFAIL percentage using the alert expression.Confirm that it has fallen below the 1% threshold and remains below it for at least 5 minutes..

```bash
(
  sum by (namespace) (rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m]))
  /
  sum by (namespace) (rate(coredns_dns_responses_total[5m]))
) < 0.01
```

- Confirm that CoreDNSErrorsHigh has returned to the resolved state and that affected applications have recovered.
