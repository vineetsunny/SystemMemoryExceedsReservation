# CoreDNSHealthCheckSlow

PrometheusRule Source: `dns` · Alert Severity: `Warning` · Pending Period: `5m`

---

## Meaning

CoreDNS health-check requests are taking longer than the configured threshold at the 95th percentile.

***Expression:***

```bash
histogram_quantile(0.95, sum by (instance, le) (rate(coredns_health_request_duration_seconds_bucket[5m]))) > 10
```
The p95 means that approximately 95% of the observed /health requests completed within the calculated latency, while approximately 5% took longer.

---

## Impact

- Application DNS resolution may be delayed, resulting in slower application requests or connection establishment.
- Service discovery may be affected, causing delays when applications resolve Kubernetes Services or internal hostnames.
- External hostname resolution may be impacted, potentially delaying connections to external dependencies.
- If multiple CoreDNS pods are affected or DNS queries begin timing out/failing, it can lead to broader application and platform-service disruptions.

---

## Diagnosis

| Scenario | What to Check | Commands / Expected Result | Action |
|---|---|---|---|
| **1. Identify affected CoreDNS pod(s)** | Check all CoreDNS pods and the nodes where they are running. | `oc get pods -n openshift-dns -o wide` | Identify affected pod(s), node(s), `READY`, `STATUS`, and restart count. |
| **2. Compare CPU usage across CoreDNS pods** | Determine whether one CoreDNS pod is consuming significantly more CPU than the others. | `oc adm top pod -n openshift-dns` | If one pod has significantly higher CPU, investigate that pod and its node. |
| **3. Check p95 latency per CoreDNS instance** | Determine whether the latency is isolated to one instance or affects multiple instances. | `histogram_quantile(0.95, sum by (instance, le) (rate(coredns_health_request_duration_seconds_bucket[5m])))` | Identify which instance(s) have elevated p95 latency and correlate them with the corresponding pod(s). |
| **4. One CoreDNS pod has high latency** | Check the node hosting that pod. | `oc get pod -n openshift-dns <pod> -o wide` | Investigate CPU utilization, memory pressure, node conditions, events, and competing workloads on that node. |
| **5. Multiple CoreDNS pods have high latency** | Determine whether the issue is cluster-wide. | `oc adm top nodes`<br>`oc get pods -n openshift-dns -o wide` | Investigate cluster-wide CPU/memory utilization, DNS traffic, networking, and DNS Operator status. |
| **6. CoreDNS CPU is high and node CPU is also high** | Determine whether other workloads are consuming the node's CPU capacity. | `oc adm top node <node>` | Identify CPU-intensive workloads and investigate node-level CPU contention. |
| **7. CoreDNS CPU is high but node has available CPU** | Determine whether the high CPU is related to increased DNS workload. | `oc adm top pod -n openshift-dns`<br>Check CoreDNS metrics/logs | Investigate DNS request volume, query patterns, and recent workload changes. |
| **8. CoreDNS pod has frequent restarts** | Check pod events and previous logs. | `oc describe pod -n openshift-dns <pod>`<br>`oc logs -n openshift-dns <pod> -c dns --previous` | Investigate the reason for the restart before taking remediation. |
| **9. CoreDNS pod is not Ready** | Check readiness/liveness probes and events. | `oc describe pod -n openshift-dns <pod>` | Investigate probe failures, container errors, resource pressure, or node conditions. |
| **10. `/health` is slow but DNS resolution is normal** | Compare health endpoint latency with actual DNS resolution. | Test `/health` and perform DNS lookup from a test pod. | Investigate the health endpoint separately; do not assume that DNS resolution is impacted. |
| **11. `/health` is slow and DNS queries are also slow/failing** | Determine whether actual DNS service is affected. | Perform DNS resolution tests and check CoreDNS metrics/logs. | Treat as a potential DNS service-impacting condition and investigate CoreDNS, node, and network conditions. |
| **12. CoreDNS logs show errors/timeouts** | Check recent CoreDNS logs. | `oc logs -n openshift-dns <pod> -c dns --since=15m` | Investigate errors/timeouts and correlate them with the alert. |
| **13. All CoreDNS pods are healthy and resource usage is normal** | Verify current p95 latency and alert state. | Check the p95 PromQL result and alert state. | Continue monitoring and determine whether the condition has cleared. |

## Mitigation

1. If node CPU/resource contention is identified: Reduce competing workload pressure or redistribute workloads so the affected CoreDNS pod has sufficient CPU and memory resources.

2. If only one CoreDNS pod is affected: Investigate the node hosting that pod for CPU, memory, network, or other node-level issues. Restore the node condition or workload placement as appropriate.

3. If multiple CoreDNS pods are affected: Investigate cluster-wide resource pressure or common networking conditions rather than restarting individual pods.

4. If DNS queries are also slow or failing: Prioritize restoring DNS service capacity and investigate the underlying CoreDNS/node condition. Verify DNS resolution from affected workloads after remediation.

5. If a CoreDNS pod is unhealthy or repeatedly restarting: Check pod events and previous logs, and allow the DNS Operator to reconcile the DaemonSet. Restarting a specific unhealthy pod can be considered if required by the identified issue.

## Validation

Confirm all CoreDNS pods are Running/Ready, DNS resolution is successful, health-check latency has returned to normal, and the alert clears after the configured 5-minute duration.

**Note:** Do not treat the alert itself as proof of DNS failure. First determine whether the elevated /health latency is also affecting actual DNS query processing.


