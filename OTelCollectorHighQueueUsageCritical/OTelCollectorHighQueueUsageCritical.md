# OTelCollectorHighQueueUsageCritical

PrometheusRule Source: `otel-collector-alerts` · Alert Severity: `Critical` · Pending Period: `5m` · 

---

## Meaning

This alert triggers when the OpenTelemetry Collector’s exporter queue exceeds 90% of its configured capacity for continuously 5 minutes.

The collector uses an in-memory queue (sending_queue) to temporarily buffer telemetry data (traces, metrics, or logs) before transmitting it to a downstream destination (e.g. Jaeger, Prometheus, Thanos, OpenSearch, Splunk etc). Crossing the 90% threshold indicates that the collector is accepting incoming telemetry much faster than the exporter can send it, bringing the pipeline close to a total buffer overflow.

***Expression:***

```bash
(otelcol_exporter_queue_size{namespace="observability"} / otelcol_exporter_queue_capacity{namespace="observability"}) > 0.9
```
- `otelcol_exporter_queue_size` → How much data is currently in the queue<br/>
- `otelcol_exporter_queue_capacity` → How much data the queue can hold

---

## Impact 

- This can lead to loss or delay of observability data, including traces, metrics, or logs, and may reduce the ability to effectively monitor and troubleshoot applications.

- Newly received telemetry may be dropped because there is no available queue capacity. Telemetry already retained in the queue remains pending until it is successfully exported or discarded according to the Collector's queue/retry behavior.

---

## Diagnosis

| Step | What to Check | Command / Metric | What It Tells You |
|---|---|---|---|
| **1. Confirm queue is actually high** | Queue size vs. capacity | `otelcol_exporter_queue_size` / `otelcol_exporter_queue_capacity` | Confirms which exporter queue is filling and approaching maximum capacity limit |
| **2. Identify the affected exporter** | Queue broken down by exporter | `otelcol_exporter_queue_size{exporter="..."}` | A single exporter may be responsible while others are healthy |
| **3. Check export failures** | Failed exports | `otelcol_exporter_send_failed_*` metrics (`spans`, `metric_points`, `log_records`) | Measures telemetry items that failed to send to the destination |
| **4. Check Collector logs** | Export/retry errors | `oc logs ... \| grep -iE 'error\|failed\|timeout\|retry\|refused'` | Provides the actual reason for failed exports (gRPC/HTTP codes, TLS issues) |
| **5. Check destination** | Endpoint/service health | `oc get pods,svc,endpoints -n <namespace>`| Destination may be down, unreachable, overloaded, or refusing connections |
| **6. Check Collector resources** | CPU/memory/throttling | `oc adm top pod -n observability`| Collector itself may be unable to process/export telemetry fast enough |
| **7. Check incoming telemetry** | Receiver traffic & rejections | `otelcol_receiver_accepted_*` <br>`otelcol_receiver_refused_*` | Measures incoming traffic successfully received or rejected by the Collector |
| **8. Check queue trend** | Is the queue increasing or decreasing? | `otelcol_exporter_queue_size` over time / `rate()` | Increasing = producer rate > exporter drain rate; decreasing = recovery |

---

## Mitigation

If the issue is within the OpenShift/Collector environment, apply the appropriate remedy based on the diagnosis. 

Issues that require investigation outside OpenShift:

| Finding | Other Side to Check |
|---|---|
| **Destination is unavailable** | Destination / Platform team |
| **Destination returns `503 Service Unavailable`** | Destination / Platform team |
| **Destination is accepting telemetry slowly** | Destination team |
| **External connectivity failure** | Network team |
| **TLS / certificate errors with external destination** | Destination team |
| **Telemetry volume suddenly increases** | Application team |
| **Single application generates excessive telemetry** | Application / Development team |
| **Destination ingestion quota is reached** | Destination / Service owner |
| **Collector is healthy but destination cannot keep up** | Destination team |
| **Queue repeatedly fills during normal traffic** | Application / Destination / Architecture team |
| **External proxy or load balancer is rejecting requests** | Network / Infrastructure team |
| **DNS resolution fails for external destination** | Network / DNS team |
