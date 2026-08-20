# MCDRebootError

PrometheusRuleSource: `machine-config-daemon` · Alert Severity `Critical` ·  Pending Period:`5m` ·  
[Runbook](https://github.com/openshift/runbooks/blob/master/alerts/machine-config-operator/MachineConfigDaemonRebootError.md)
 
---
 
## Meaning

The `MCDRebootError` alert indicates that the Machine Config Daemon (MCD) encountered an error while attempting to reboot a node.

The alert fires when the MCD detects a reboot failure that persists for 5 minutes, indicating that the node has not successfully completed the required reboot and the MachineConfig update may be blocked.

Alert Expr:
```bash
mcd_reboots_failed_total > 0
```

---

## Impact

A failed reboot can prevent a MachineConfig update from completing. The affected node may remain on the previous configuration, causing the MachineConfigPool to remain updating or become degraded.

The alert itself indicates a failed reboot, not necessarily that the node is currently down.

- The affected node may remain on the previous MachineConfig.
- The MachineConfigPool (MCP) may remain in `UPDATING` state.
- The MCP may become `DEGRADED`.
- Further MachineConfig updates may be blocked on the affected node.
- Configuration changes may not be applied consistently across nodes.
- Workloads running on the affected node may be impacted if the node becomes `NotReady`.

---

## Diagnosis

###  Variables (From Alert)

```bash
Node=<labels.node>
Namespace=<labels.namespace>
Pod=<labels.pod>
```

If a node is stuck and does not reboot, follow these steps to check whether the **Machine Config Daemon (MCD)** detected a reboot failure.

***1. Check the MCD Logs***

Find the `machine-config-daemon-*` pod running on the affected node and set its name in `$Pod`.

Run:

```bash
oc logs -f -n openshift-machine-config-operator $Pod -c machine-config-daemon
```

***2. Check for the Reboot Attempt***

When the MCD starts a node reboot, it logs:

```text
update.go: Rebooting node
```
This confirms that the MCD reached the reboot step.

The MCD uses `systemd-run` to create a systemd unit, which then executes:

```bash
systemctl reboot
```

***3. Check for a Reboot Failure***

If the reboot command fails, the MCD logs an error similar to:

```text
"failed to run reboot: %v", err
```

The `%v` value contains the actual error returned by the reboot command.

For example:

```
update.go:2820] "failed to run reboot: exec: \"systemd-run\": executable file not found in $PATH"
```

Review this error carefully because it usually indicates why the node could not be rebooted.

***4. Confirm the Node Was Marked Degraded***

When the reboot command fails, the error is returned to the MCD `writer.go` service. The MCD then marks the node as **Degraded** and logs a message similar to:

```text
E0219 01:25:20.890006    8301 writer.go:226] Marking Degraded due to: reboot command failed, something is seriously wrong
```

This will increment the mcd_reboots_failed_total value by 1.

***5. Verify the node's journal logs to determine the underlying cause***

After identifying the MCD error, check the node's journal logs to determine the underlying cause of the reboot failure.

```text
journalctl -b | grep -iE "reboot|shutdown|poweroff|systemd|failed|timeout|error"
```

---

## Mitigation:


Review the exact error reported by MCD.The error message will change depending on what is preventing the reboot. Based on the identified scenario, follow the corresponding mitigation procedure.


| Scenario | Mitigation Action | Expected Result |
|---|---|---|
| MCD reports a specific reboot failure | Review the exact MCD error, identify the node component or service causing the failure, and apply the corresponding remediation. | Underlying node issue is resolved. |
| Required reboot command or executable is unavailable | Restore the node to a valid and consistent OS state using the approved node recovery procedure. | Reboot functionality is restored. |
| Systemd or reboot operation is failing | Correct the identified systemd or reboot-related issue. If required, perform a controlled reboot using the approved node maintenance procedure. | Node successfully reboots. |
| Node fails to complete the reboot | Use the server's physical console or BMC (such as iDRAC, iLO, or equivalent) to access the node and recover the server. | Node completes the boot process successfully. |
| Node remains `NotReady` after reboot | Verify node and kubelet availability and restore the node to a healthy state. Use BMC/console access if the node cannot be recovered through OpenShift. | Node returns to `Ready`. |
| Node repeatedly fails to reboot | Avoid repeated forced reboots. Investigate the underlying node or hardware issue and escalate to the bare-metal/infrastructure team if required. | Node can reboot successfully. |
| MCD does not resume after the node issue is resolved | Allow MCD to reconcile. If MCD is confirmed to be stuck, restart the affected MCD pod. | MCD resumes normal processing. |
| Node cannot be recovered | Follow the approved bare-metal node replacement or decommission procedure. | Replacement or recovered node becomes healthy. |
| Reboot succeeds but the associated MachineConfig operation does not complete | Allow MCD to retry the operation and verify the MachineConfig/MCP state. If it remains stuck, continue with MCD/MCP investigation. | MachineConfig operation completes successfully. |



The mitigation is complete when:
- Node is Ready.
- Node has successfully completed the reboot.
- Affected node's MCD pod is Running.
- CurrentConfig matches DesiredConfig.
- MCP shows UPDATED=True.
- MCP shows UPDATING=False.
- MCP shows DEGRADED=False.
