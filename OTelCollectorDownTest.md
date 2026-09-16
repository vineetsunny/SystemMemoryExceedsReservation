# Reproduce OTelCollectorDown Alert

Simulate a process freeze (unresponsive metrics endpoint) to validate alert firing without destroying the container or triggering unintended Kubernetes restarts.


1. Find Target Collector pod Name & Node:

```bash
oc -n observability get pod -l app=otel-collector -o wide
```

2. Login into the node where pod is running using SSH/debug pod method.

3. Identify Target Pod ID
```bash
POD_ID=$(crictl pods --name <podName> -q)
```

4. Extract Container ID
```bash
CONTAINER_ID=$(crictl ps --pod $POD_ID --name otc-container -q)
```

5. Get the PID:
```bash
CONTAINER_PID=$(crictl inspect "$CONTAINER_ID" | jq -r '.info.pid')
```

6. Trigger Fault Injection (Freeze)
```bash
kill -STOP $CONTAINER_PID
```

7. Verify if process is suspended.

```bash
 ps -o pid,stat,cmd -p "$CONTAINER_PID"
```

Sample output:

```bash
PID STAT CMD
  12402 Tsl  /usr/bin/opentelemetry-collector --config=/conf/collector.yaml
```

T = stopped/suspended by a job-control signal.

---

# Restore

1. Restore target process

```bash
kill -CONT $CONTAINER_PID
```

2. Verify if process is Running

```bash
ps -o pid,stat,cmd -p "$CONTAINER_PID"
    PID STAT CMD
  12402 Ssl  /usr/bin/opentelemetry-collector --config=/conf/collector.yaml
```
S = Process is running/sleeping.

**OR**

Restarting the collector pod can restore it's healthy state. 
