# OTelCollectorDown

PrometheusRule Source: `otel-collector-alerts` · Alert Severity: `Critical` · Pending Period: `5m`

---

## Meaning

Prometheus has been unable to successfully scrape the OpenTelemetry Collector's metrics endpoint for 5 consecutive minutes.
The target is expected to expose Prometheus metrics, typically on port 8888. An up value of 0 indicates that the Prometheus scrape of this target is failing.

***Alert contdition***

The alert is triggered when the OpenTelemetry Collector monitoring target is unavailable to Prometheus:

```bash
up{job="otel-collector-monitoring"} == 0
```

---

## Impact

If the OpenTelemetry Collector is unavailable or its metrics endpoint cannot be scraped:

- Traces, metrics, or logs may not be delivered to configured downstream exporters if the Collector is unavailable.
- Monitoring of the Collector becomes unavailable, reducing visibility into telemetry-processing health.
- If the Collector is completely unavailable, applications sending telemetry to that Collector may experience telemetry loss or failed delivery depending on their retry/buffering configuration.

---

## Diagnosis

| Command | What to look for | Purpose |
|---|---|---|
| `oc -n openshift-monitoring exec prometheus-k8s-0 -- curl -sG 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="otel-collector-monitoring"}'` | `up = 0` | Confirm Prometheus cannot successfully scrape the Collector metrics target. |
| `oc -n openshift-monitoring exec prometheus-k8s-0 -- curl -s http://localhost:9090/api/v1/targets` | Collector target; `health`; `scrapeUrl`; `lastError` | Identify **why Prometheus considers the Collector target down**. This should be the main starting point. |
| `oc -n observability get pod -l app=otel-collector -o wide` | Pod is `Running/Ready`; pod IP; restart count | Confirm the Collector pod itself is still running and identify the pod being investigated. |
| `oc -n observability exec <collector-pod> -c otc-container -- sh -c 'grep "^State:" /proc/1/status'` | OpenTelemetry Collector process is running OR stopped (T) | Determine whether the process inside the running pod is stopped, paused, or abnormal. |
| `oc -n observability port-forward pod/<collector-pod> 8888:8888 >/tmp/otel-port-forward.log 2>&1 & PF_PID=$!` followed by `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8888/metrics` | HTTP `200` vs connection refused/timeout/error | Confirm whether the Collector's **metrics endpoint itself is responding**. |
| `oc -n observability logs <collector-pod> -c otc-container --tail=200` | `error`, `panic`, `fatal`, `timeout`, `shutdown`, `bind`, `listen`, etc. | Identify Collector-side problems that could make the metrics endpoint unavailable. |
| `oc -n observability get svc <collector-svc>` | Service exists and exposes `8888/TCP` | Confirm the monitoring Service exposes the Collector metrics port. |
| `oc -n observability get endpoints <collector-endpoint> -o wide` | `<pod-IP>:8888`, e.g. `10.233.0.17:8888` | Confirm the monitoring Service points to the **correct Collector pod and port**. |
| `oc -n observability get servicemonitor <collector-servicemonitor> -o yaml` | Correct Service, port, and metrics path | Confirm Prometheus is configured to scrape the intended Collector monitoring Service. |
| `oc -n observability get networkpolicy` | Policies affecting Prometheus → Collector traffic | Investigate whether NetworkPolicy can prevent Prometheus from reaching port `8888`. |
| `oc -n observability describe pod <collector-pod>` | Events, `OOMKilled`, resource pressure, readiness/lifecycle issues | Investigate container, resource, or node-related problems when the Collector process is abnormal. |
| `oc describe node <node>` | Node conditions/resource pressure | Investigate node-level problems if the Collector itself appears healthy but scraping still fails. |

---

## Mitigation

The following actions address the most commonly encountered causes of this alert. Based on the diagnosis results, apply the appropriate remediation. If the identified cause is not covered by these scenarios, continue troubleshooting using the diagnosis results before taking corrective action.

- If the Collector pod or process is unhealthy: restart the affected Collector pod and allow the OpenTelemetry Operator/controller to recreate it.
- If the Service, Endpoint, or ServiceMonitor configuration is incorrect, correct the source-of-truth configuration rather than modifying generated resources directly.
- If NetworkPolicy is blocking Prometheus-to-Collector traffic, update the applicable policy to permit the required Prometheus monitoring traffic to the Collector metrics port.
- If node or resource pressure is identified, resolve the underlying node/resource condition and allow the Collector to recover or reschedule.
- If no Collector-side issue is found,  use the Prometheus target lastError and target configuration to investigate the Prometheus-to-Collector scrape path before taking further action.


## Verification

After applying the applicable remediation:

- Prometheus reports up{job="otel-collector-monitoring"} = 1.
- Prometheus target shows health="up" with no scrape error.

```bash
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
  curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.labels.job=="<ServiceMonitor>") | {health, scrapeUrl, lastError}'
``` 
- The alert clears after its configured evaluation period.
