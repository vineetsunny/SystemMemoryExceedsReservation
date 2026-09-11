# Reproduce OTelCollectorHighQueueUsageCritical Alert

Before starting ensure the OpenTelemetry Collector and Prometheus monitoring are already configured and healthy, and run this test only in a non-production environment with synthetic telemetry.

***1. Take the backup of OpenTelemetryCollector configuration***

```bash
oc get opentelemetrycollector -n observability
oc get opentelemetrycollector <collectorName> -n observability -o yaml > OpenTelemetrycollector_backup.yaml
```

***2. Update the OpenTelemetryCollector configuration to block queue flushes*** 


Changes in the collector configuration:

| Area | Normal configuration | Test configuration | Purpose |
|---|---|---|---|
| **1. Add `otlp/test` exporter** | Not present | `otlp/test` with endpoint `192.0.2.1:4317` | Creates a deliberately unreachable export destination to cause export failures/backpressure |
| **2. Enable sending queue on `otlp/test`** | No queue on `otlp/test` | `sending_queue.enabled: true`, `queue_size: 1000` | Allows failed/pending traces to accumulate in the exporter queue |
| **3. Change `service.pipelines.traces`** | `exporters: [debug]` | `exporters: [otlp/test, debug]` | Routes traces to the test exporter while retaining `debug` for visibility |

Note: Before running the test in the existing environment, check the current queue size and adjust the simulation values accordingly.

```bash
oc edit opentelemetrycollector otel-collector -n observability
```

****Sample configuration:**** 
```bash
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  generation: 13
  name: otel-collector
  namespace: observability
spec:
  config:
    exporters:
      debug:
        sending_queue:
          enabled: true
          queue_size: 1000
        verbosity: detailed
      otlp/test:
        endpoint: 192.0.2.1:4317
    processors:
      batch: {}
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    service:
      pipelines:
        traces:
          exporters:
          - otlp/test
          - debug
          processors:
          - batch
          receivers:
          - otlp                                 
      telemetry:
        metrics:
          readers:
          - pull:
              exporter:
                prometheus:
                  host: 0.0.0.0
                  port: 8888
  configVersions: 3
  daemonSetUpdateStrategy: {}
  deploymentUpdateStrategy: {}
  httpRoute:
    enabled: false
    gateway: ""
  ingress:
    route: {}
  ipFamilyPolicy: SingleStack
  managementState: managed
  mode: deployment
  networkPolicy:
    enabled: true
  observability:
    metrics:
      enableMetrics: true
  podDnsConfig: {}
  replicas: 1
  resources: {}
  targetAllocator:
    allocationStrategy: consistent-hashing
    collectorNotReadyGracePeriod: 30s
    collectorTargetReloadInterval: 30s
    filterStrategy: relabel-config
    observability:
      metrics: {}
    prometheusCR:
      evaluationInterval: 30s
      podMonitorNamespaceSelector: {}
      probeNamespaceSelector: {}
      scrapeConfigNamespaceSelector: {}
      scrapeInterval: 30s
      serviceMonitorNamespaceSelector: {}
    resources: {}
  upgradeStrategy: automatic

```
***3. Create Service Monitor*** 

Perform this step only if the OpenTelemetry Collector is newly configured and a ServiceMonitor does not already exist.

```bash
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: otel-collector-monitoring-collector
  namespace: observability
spec:
  endpoints:
    - port: monitoring
  namespaceSelector:
    matchNames:
      - observability
  selector:
    matchLabels:
      app.kubernetes.io/component: opentelemetry-collector
      app.kubernetes.io/instance: observability.otel-collector
      app.kubernetes.io/managed-by: opentelemetry-operator
      app.kubernetes.io/part-of: opentelemetry
      operator.opentelemetry.io/collector-service-type: monitoring
```


***4. Create a job to generates synthetic OTLP trace data***

```bash
cat <<EOF | oc create -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: telemetrygen-high-rate
  namespace: observability
spec:
  ttlSecondsAfterFinished: 60
  template:
    spec:
      restartPolicy: Never
      securityContext:
        runAsNonRoot: true
      containers:
      - name: telemetrygen
        image: ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest
        args:
        - traces
        - --otlp-endpoint=otel-collector-collector.observability.svc.cluster.local:4317
        - --otlp-insecure=true
        - --duration=10m
        - --rate=1000
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
EOF
```
The Job generates synthetic OTLP trace data at a high rate to push your OpenTelemetry Collector's exporter queue to its maximum capacity (1000).
Job is sending traffic to collector service: 
`--otlp-endpoint=otel-collector-collector.observability.svc.cluster.local:4317`

***5. Confirm the Job, and pod is running***

```bash
oc get job -n observability
oc get pods -n observability -l job-name=telemetrygen-high-rate
```

***6. Verify the metrics and alert:***

Monitor the queue utilization: ( it will take 5-10 min to reach the capacity on queue size 1000 ) 

```bash
otelcol_exporter_queue_size / otelcol_exporter_queue_capacity
```

To check alert : Go to Observe -> Alerting

---

# Revert the changes:

1. Make the configuration changes back to the original by referring to the backup file.

2. Remove the Job.

```bash
oc delete job telemetrygen-high-rate
```
3. Ensure the alert has been resolved.
