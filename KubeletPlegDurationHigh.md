# KubeletPlegDurationHigh

PrometheusRule Source: `kubernetes-monitoring-rules`  · Alert Severity: `Warning` · Pending Period: `5m`

---

## Meaning

The alert indicates that the kubelet's Pod Lifecycle Event Generator (PLEG) is taking too long to relist pod states from the container runtime.

PLEG monitors pod and container lifecycle changes and helps kubelet maintain the desired state of workloads on the node.

**Alert condition:**
```promql
node_quantile:kubelet_pleg_relist_duration_seconds:histogram_quantile{quantile="0.99"} >= 10
```

**Trigger criteria:**
- The 99th percentile (P99) PLEG relisting duration is greater than or equal to 10 seconds.
- The condition must persist for 5 minutes before the alert fires.


---

## Impact

- Delayed detection of pod and container lifecycle changes on the affected node.
- Delayed pod startup, termination, or recovery operations.
- Potential delays in workload rescheduling or recovery if the node becomes unhealthy.
- If the underlying issue persists, the node may transition to `NotReady`, affecting workload availability.
- Repeated or prolonged container runtime communication failures can affect node stability.

The actual impact depends on the duration of the issue, node health, and workloads running on the affected node.

---

## Diagnosis

| Check | Command | What to Look For / Action |
|---|---|---|
| Node status | `oc get nodes` | Identify whether the affected node is `Ready` or `NotReady`. |
| Node conditions and events | `oc describe node <NODE_NAME>` | Check `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`, and events indicating node or runtime issues. |
| Access the affected node | `oc debug node/<NODE_NAME>` | Use this method when node debugging is available. If the node is `NotReady` and the debug session cannot start, access the node through SSH if connectivity is available. |
| CRI-O service status | `systemctl status crio` | Check whether CRI-O is active, failed, or unresponsive. Execute on the affected node through the debug session after `chroot /host`, or directly through SSH. |
| CRI-O logs | `journalctl -u crio --since "30 minutes ago" --no-pager` | Look for runtime errors, timeouts, slow operations, or failures communicating with the container runtime. |
| Kubelet logs | `journalctl -u kubelet --since "30 minutes ago" --no-pager` | Look for PLEG relisting delays, runtime communication errors, timeouts, and pod synchronization issues. |
| Node resource utilization | `oc adm top node <NODE_NAME>` | Check CPU and memory utilization. If metrics are unavailable, inspect resource utilization directly on the node through SSH. |



## Mitigation

Perform the following actions based on the diagnosis.

***Scenario A: CRI-O is unresponsive or unhealthy***

1. Confirm CRI-O status and review its logs.
2. Follow the approved OpenShift node-recovery procedure for the identified runtime failure. See - Troubleshooting [Documentation](https://docs.okd.io/4.22/support/troubleshooting/troubleshooting-crio-issues.html)

***Scenario B: Node resource pressure***

1. Identify the resource causing pressure using node conditions, resource metrics, and logs.
2. Identify resource-intensive or abnormal workloads on the affected node.
3. Apply the approved workload or resource remediation procedure.
4. Confirm that node resource pressure has cleared and kubelet is responding normally.


**Note:** Do not force-delete pods, reboot the node, or restart services without assessing workload availability and following the approved recovery procedure.

## Validation

***Step 1: Verify node status***

```bash
oc get nodes
```

**Expected:** The affected node is in the `Ready` state.

***Step 2: Verify CRI-O and kubelet health***

From the affected node:

```bash
systemctl is-active crio
systemctl is-active kubelet
```

**Expected:** Both services report `active`.

***Step 3: Verify workload health***

```bash
oc get pods -A -o wide
```

Check for pods in `Pending`, `Unknown`, or unexpected `CrashLoopBackOff` states, especially those associated with the affected node.

***Step 4: Verify PLEG relisting duration***

Run the following PromQL query in the OpenShift monitoring console:

```promql
node_quantile:kubelet_pleg_relist_duration_seconds:histogram_quantile{quantile="0.99"}
```

**Expected:** The P99 PLEG relisting duration for the affected node is below 10 seconds.

***Step 5: Confirm alert recovery***

Verify that `KubeletPlegDurationHigh` is no longer firing.




