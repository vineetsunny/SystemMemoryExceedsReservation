# Reproduce CoreDNSHealthCheckSlow Alert

---

This procedure reproduces the CoreDNSHealthCheckSlow alert by generating CPU pressure on a CoreDNS DNS pod and sending a high volume of parallel requests to its /health endpoint.



***Step 1: Apply a CPU Limit to the CoreDNS DaemonSet***


```bash
oc edit ds dns-default -n openshift-dns
```

Add the following CPU limit to the DNS container:- 
```bash
resources:
          limits:
            cpu: "1"
```

Without a CPU limit, processes consume more CPU whenever it is available, making it harder to control the pressure on that pod.

***Step 2:Generate CPU Load in one of the CoreDNS Pods : (16 cpu in worker node)***

Check pods:
```bash
oc get pods -n openshift-dns -o wide
```

```bash
oc exec -n openshift-dns <pod>  -c dns -- \
  sh -c 'for i in $(seq 1 14); do while true; do :; done & done; wait'
```

Note:  The command starts 14 CPU-intensive background loops inside the dns container

***Step 3: Port-Forward the CoreDNS Health Endpoint:***

```bash
oc port-forward -n openshift-dns pod/<pod> 8080:8080
```

***Step 4: Install `hey` utility on local system, Generate Concurrent Health Requests (for 15 min):***

```bash
hey -z 15m -c 1000 http://127.0.0.1:8080/health
```

***Step 5: Temporarily Lower the Alert Threshold: (From 10 sec to 2 sec)***

```bash
#oc edit prometheusrule dns -n openshift-dns-operator

 - alert: CoreDNSHealthCheckSlow
      annotations:
        description: CoreDNS Health Checks are slowing down (instance {{ $labels.instance
          }})
        summary: CoreDNS health checks
      expr: histogram_quantile(.95, sum(rate(coredns_health_request_duration_seconds_bucket[5m]))
        by (instance, le)) > 2
      for: 5m
      labels:
        severity: warning
```

`Note:` The metric has a maximum bucket boundary of approximately 2.5 seconds. Therefore, a p95 value above 10 seconds cannot be observed from this metric in the environment. For alert-reproduction purposes, the threshold was temporarily reduced to 2 seconds, which is within the measurable range of the histogram.

***Track the status using query:***

```bash
histogram_quantile(0.95, sum by (instance, le) (rate(coredns_health_request_duration_seconds_bucket[5m])))
```

***2.5s bucket cap:***

![](assets/17904427758806.jpg)



***Step 6: Verify the alert: (Observe > Alerting)***

![](assets/17904428230053.jpg)


---

# Recovery

1. Stop the port-forward (ctrl+c)
2. Restore the original alert threshold ( 2 sec to 10 sec)

```bash
oc edit prometheusrule dns -n openshift-dns-operator
```
3. Remove the CPU limit section :

```bash
oc edit ds dns-default -n openshift-dns
```
