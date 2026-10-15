# KubeletHealthState

PrometheusRule Source: `machine-config-daemon`· Alert Severity: `Warning`· [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/machine-config-operator/KubeletHealthState.md)

---

## Meaning

The alert is triggered when the Machine Config Daemon (MCD) detects repeated failures in the kubelet health check. The MCD checks the kubelet health endpoint at `http://localhost:10248/healthz` every 30 seconds. 
If the endpoint does not return the expected `ok` response, the failure counter `mcd_kubelet_state` is incremented.

**Expression:**

```promql
mcd_kubelet_state > 2
```
---

## Impact

- Applications running on the affected node may experience interruptions or become unavailable.
- The cluster may have fewer healthy nodes available to schedule workloads.

---

## Diagnosis

| Command | What to Look For | Action |
|---|---|---|
| `oc get nodes` | Check whether the affected node is `NotReady` or experiencing other issues. | Identify the affected node and investigate its kubelet health. |
| `oc get pods -n openshift-machine-config-operator -o wide` | Locate the Machine Config Daemon (MCD) pod running on the affected node and check its status. | Identify the corresponding MCD pod and review its logs. |
| `oc logs -n openshift-machine-config-operator <mcd-pod-name> -c machine-config-daemon --since=30m` | Look for `Failed kubelet health check`, `kubelet health failure threshold reached`, or localhost resolution errors. | Investigate the kubelet health endpoint and the underlying error reported in the logs. |
| `systemctl status kubelet.service` | Check whether kubelet is `active`, `inactive`, or `failed`. | If inactive or failed, review the service logs and resolve the underlying issue before restarting kubelet if required. |
| `journalctl -u kubelet.service --since "30 minutes ago"` | Look for service startup failures, runtime communication issues, resource exhaustion, or other kubelet errors. | Address the identified cause and restart kubelet if necessary and approved. |
| `curl -v --max-time 30 http://localhost:10248/healthz` | Verify that the health endpoint responds with `ok`. Check for connection errors, timeouts, or resolution failures. | If the endpoint fails, investigate kubelet health, port accessibility, and localhost resolution. |
| `ss -lntp \| grep ':10248'` | Check whether a process is listening on port `10248`. | If no listener is present, investigate kubelet service status and configuration. |
| `df -h` | Check filesystem utilization for capacity constraints. | Investigate full or nearly full filesystems and reclaim space according to operational procedures. |
| `free -h` | Check available memory and overall memory pressure indicators. | Investigate memory exhaustion and identify resource-intensive processes if necessary. |
| `top` | Check CPU utilization, memory consumption, and resource-intensive processes. | Investigate sustained resource pressure that may affect kubelet health. |
| `journalctl -u kubelet.service --since "30 minutes ago"` | Correlate kubelet errors with resource pressure or other node-level failures. | Resolve the identified issue or escalate to the OpenShift platform team if the root cause remains unclear. |

---

## Mitigation

1. Address the underlying cause identified during diagnosis, such as kubelet service failure, localhost resolution issues, resource exhaustion, or node-level problems.

2. If kubelet is inactive or failed, review the service logs and correct the underlying issue before attempting a restart.

3. If a kubelet restart is necessary and approved, execute it on the affected node during the appropriate operational window:

   ```bash
   systemctl restart kubelet.service
   ```
   Monitor the service and node status after the restart.

4. If the node remains unhealthy, investigate broader node-level issues, including storage, CPU, memory, and network connectivity. 

5. Reboot the node only when justified by the diagnosis and approved through the applicable change process.

---

## Validation

1. Verify that the kubelet service is active:

   ```bash
   systemctl is-active kubelet.service
   ```

2. Confirm that the kubelet health endpoint returns `ok`:
   ```bash
   curl -fsS --max-time 30 http://localhost:10248/healthz
   ```

3. Verify that the affected node has returned to `Ready`:

   ```bash
   oc get node <node-name>
   ```

4. Review the MCD logs and confirm that no new kubelet health-check failures are being reported:
 
   ```bash
   oc logs -n openshift-machine-config-operator <mcd-pod-name> -c machine-config-daemon --since=15m
   ```

5. Verify that the `KubeletHealthState` alert is no longer firing in the cluster monitoring system.
