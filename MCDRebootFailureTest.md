# Alert MCDRebootFailure Reproduce Procedure

This procedure reproduces an MCD reboot failure by making the `systemd-run` binary inaccessible on a worker node. When the Machine Config Daemon (MCD) attempts to apply a MachineConfig that requires a reboot, the reboot command fails and the node is marked as `Degraded`.

> **Lab/Test Environment:** Perform these steps only on a test worker node. Do not use this procedure on production nodes.

## 1. Create the Host-Level Systemd Service

On the target worker node, create an empty file and bind-mount it over `/usr/bin/systemd-run`.

```bash

oc debug node/<node-name> -- chroot /host bash -c '
# 1. Create the persistent empty file in /var/tmp (writable storage)
touch /var/tmp/empty-systemd-run

# 2. Create the systemd unit file in /etc/systemd/system/ (persistent across reboots)
cat << "EOF" > /etc/systemd/system/mask-systemd-run.service
[Unit]
Description=Mask systemd-run binary for MCD test persistence
Before=machine-config-daemon.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/mount --bind /var/tmp/empty-systemd-run /usr/bin/systemd-run
ExecStop=/usr/bin/umount /usr/bin/systemd-run

[Install]
WantedBy=multi-user.target
EOF

# 3. Reload systemd, enable, and start the service
systemctl daemon-reload
systemctl enable --now mask-systemd-run.service
'

```

Replace `<NODE_NAME>` with the name of the worker node being used for testing.

This temporarily hides the actual `systemd-run` binary without deleting or modifying the original binary.

---

## 2. Verify That `systemd-run` Is Inaccessible

Run the following command:

```bash
oc debug node/<NODE_NAME> -- chroot /host systemd-run --version
```

Expected output: 
```bash
bash: /usr/bin/systemd-run: Permission denied 
```

This confirms that the node is in the expected test state.

---

## 3. Trigger MCD Processing Using a MachineConfig

Create the following `test-mc-alert.yaml` file:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 992-worker-test-file
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
        - path: /etc/mco-test4.txt
          mode: 0644
          overwrite: true
          contents:
            source: data:text/plain;charset=utf-8,This%20MachineConfig%20is%20for%20testing.
```

Apply the MachineConfig:

```bash
oc create -f test-mc-alert.yaml
```

The worker node should begin processing the MachineConfig. Because the node requires a reboot and `systemd-run` is inaccessible, MCD should fail to execute the reboot operation.

---

## 4. Verify MCD Logs and Node State

### 4.1 Check MCD Logs

Identify the MCD pod running on the affected node:

```bash
oc get pods -n openshift-machine-config-operator -o wide
```

Then check its logs:

```bash
oc logs -n openshift-machine-config-operator <MCD_POD>
```

Look for messages similar to:

```text
I0819 16:27:18.921129    4904 update.go:1881] Deleting stale data
I0819 16:27:18.921458    4904 update.go:1915] Removing file "/etc/mco-test4.txt" completely
I0819 16:27:18.921595    4904 update.go:1963] Deleting stale config file: /etc/mco-test4.txt
I0819 16:27:18.921672    4904 update.go:1972] Removed stale file "/etc/mco-test4.txt"
E0819 16:27:18.926134    4904 writer.go:231] Marking Degraded due to: "reboot command failed, something is seriously wrong"
I0819 16:27:18.931144    4904 daemon.go:643] Error syncing node worker-cluster-dqdd8-4 (retries 41): reboot command failed, something is seriously wrong
```

The key message confirming the reboot failure is:

```text
Marking Degraded due to: "reboot command failed, something is seriously wrong"
```

---

### 4.2 Check Machine Config Controller (MCC) Logs

Check the Machine Config Controller logs:

```bash
oc logs -n openshift-machine-config-operator <machine-config-controller>
```

Look for messages similar to:

```text
I0819 16:25:36.733821       1 node_controller.go:764] Pool worker: node worker-cluster-dqdd8-4: changed annotation machineconfiguration.openshift.io/state = Degraded
I0819 16:25:36.733960       1 node_controller.go:764] Pool worker: node worker-cluster-dqdd8-4: changed annotation machineconfiguration.openshift.io/reason = reboot command failed, something is seriously wrong
I0819 16:25:41.718278       1 status.go:324] Degraded Machine: worker-cluster-dqdd8-4 and Degraded Reason: reboot command failed, something is seriously wrong
I0819 16:25:46.741409       1 status.go:324] Degraded Machine: worker-cluster-dqdd8-4 and Degraded Reason: reboot command failed, something is seriously wrong
I0819 16:26:20.409385       1 container_runtime_config_controller.go:1024] Applied ImageConfig cluster on MachineConfigPool master
I0819 16:26:20.508931       1 container_runtime_config_controller.go:1024] Applied ImageConfig cluster on MachineConfigPool worker
I0819 16:26:22.452051       1 status.go:324] Degraded Machine: worker-cluster-dqdd8-4 and Degraded Reason: reboot command failed, something is seriously wrong
I0819 16:27:22.498208       1 status.go:324] Degraded Machine: worker-cluster-dqdd8-4 and Degraded Reason: reboot command failed, something is seriously wrong
```

The important MCC messages are:

```text
machineconfiguration.openshift.io/state = Degraded
```

and:

```text
reboot command failed, something is seriously wrong
```

---

### 4.3 Verify the Node State

Check the affected node:

```bash
oc get node <NODE_NAME>
```
Expected Result:

```bash
worker-node         Ready,SchedulingDisabled   worker
```

Also check the MachineConfigPool:

```bash
oc get mcp worker
```


The worker node should show a degraded state, and the worker MCP should reflect the degraded condition.

For additional details:

```bash
oc describe node <NODE_NAME>
```

---

### 4.4 Check the MCD Reboot Failure Metric

Query the Prometheus metric used by the alert:

```bash
mcd_reboots_failed_total > 0
```

![](assets/17871586664517.jpg)



Verify that the metric reports a value greater than `0` for the affected node.


---

## 5. Cleanup and Restore the Node

After completing the test,remove the systemd unit.

```bash
oc debug node/<node-name> -- chroot /host bash -c '
systemctl stop mask-systemd-run.service
systemctl disable mask-systemd-run.service
rm -f /etc/systemd/system/mask-systemd-run.service
rm -f /var/tmp/empty-systemd-run
systemctl daemon-reload
'
```

---

### 5.2 Delete the Test MachineConfig

Delete the MachineConfig created for the test:

```bash
oc delete machineconfig 992-worker-test-file
```

> **Note:** Use the same MachineConfig name that was created in Step 3.

---

### 5.3 Verify the Node Returns to a Healthy State

Check the node:

```bash
oc get node <NODE_NAME>
```

Check the worker MachineConfigPool:

```bash
oc get mcp worker
```

Confirm that the node and MCP return to their expected healthy state before considering the test complete.

