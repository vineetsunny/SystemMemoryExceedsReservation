# Reproducing VMCannotBeEvicted Alert

## Testing environment:
OCP Version: 4.21.10
OpenShift Virtualization Version: 4.21.13

## 1. Verify that the VM is Live Migratable

Run the following command to verify the current migration status of the VM:

```bash
oc get vmi -n <Namespace> -o wide 
```
Example output:

```bash
NAME                   AGE   PHASE     IP           NODENAME                 READY   LIVE-MIGRATABLE   PAUSED
fedora-beige-fowl-40   54m   Running   10.133.2.6   worker-cluster-4p54c-1   True    True
```

---

## 2. Verify the Current Network Interface Configuration

Check whether the VM is using the `bridge` network interface.

```bash
oc get vm <VM_Name> -n <Namespace> -o yaml | grep -A10 "interfaces:"
```

Example output:

```yaml
interfaces:
- macAddress: 02:59:7e:a0:1e:bc
  bridge: {}
  model: virtio
  name: default
rng: {}
features:
  acpi: {}
  smm:
    enabled: true
firmware:
```

---

## 3. Change the VM Interface from `bridge` to `masquerade`

Apply the following patch:

```bash
oc patch vm <VM_Name> -n <Namespace> --type=json -p='[{"op":"remove","path":"/spec/template/spec/domain/devices/interfaces/0/bridge"},{"op":"add","path":"/spec/template/spec/domain/devices/interfaces/0/masquerade","value":{}}]'
```

---

## 4. Verify the Updated Interface Configuration

Confirm that the interface has been changed to `masquerade`.

```bash
oc get vm <VM_Name> -n <Namespace> -o yaml | grep -A10 "interfaces:"
```

Example output:

```yaml
interfaces:
- masquerade: {}
  macAddress: 02:59:7e:a0:1e:bc
  model: virtio
  name: default
rng: {}
features:
  acpi: {}
  smm:
    enabled: true
firmware:
```

---

## 5. Restart the VM

Restart the VM for the configuration change to take effect.



```bash
virtctl restart <VM_Name> -n <Namespace>
```

---

## 6. Verify That the VM Is No Longer Live Migratable

Check the VM status again.

```bash
oc get vmi -n <Namespace> -o wide
```

Example output:

```bash
NAME                   AGE   PHASE     IP            NODENAME                 READY   LIVE-MIGRATABLE   PAUSED
fedora-beige-fowl-40   76s   Running   10.133.2.13   worker-cluster-4p54c-1   True    False
```

The `LIVE-MIGRATABLE` field should now display `False`.

---

## 7. Verify That the Alert Is Triggered

After the VM becomes non-live-migratable, corresponding alert should be generated `VMCannotBeEvicted`.
---

## 8. Apply a MachineConfig

Apply a MachineConfig to initiate node draining.

```bash
oc apply -f test-machine-config.yaml
```

Example MachineConfig:

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
      version: 3.5.0
    storage:
      files:
      - path: /etc/mco-test3.txt
        mode: 0644
        overwrite: true
        contents:
          source: data:text/plain;charset=utf-8,This%20MachineConfig%20is%20for%20testing.
```

---

## 9. Verify That Node Drain Fails

The Machine Config Operator applies the MachineConfig to worker nodes one at a time. When it reaches the node hosting the non-live-migratable VM, the drain operation should fail because the VM cannot be migrated.

Check the `machine-config-controller` logs:

```bash
oc logs <machine-config-controller-*> -n openshift-machine-config-operator
```

Example output:

```bash
node worker-cluster-4p54c-1: Drain failed. Drain has been failing for more than 10 minutes. Waiting 5 minutes then retrying. Error message from drain: error when evicting pods
```

---

# Remediation

**Option 1**
## A. Revert the Interface Back to `bridge`

Restore the original network interface configuration.

```bash
oc patch vm <VM_NAME> --type=json -p='[{"op":"remove","path":"/spec/template/spec/domain/devices/interfaces/0/masquerade"},{"op":"add","path":"/spec/template/spec/domain/devices/interfaces/0/bridge","value":{}}]'
```

---

## B. Restart the VM

Restart the VM to apply the configuration change.

```bash
virtctl restart <VM_NAME> -n <Namespace>
```

## C. Verify That the VM Is Live Migratable Again

Check the VM status.

```bash
oc get vmi -n <Namespace> -o wide
```

Example output:

```bash
NAME                   AGE   PHASE     IP            NODENAME                 READY   LIVE-MIGRATABLE   PAUSED
fedora-beige-fowl-40   73s   Running   10.132.2.49   worker-cluster-4p54c-2   True    True
```

The `LIVE-MIGRATABLE` field should now display `True`.

`Note`: At this stage, the alert is resolved.

---

## D. Verify That the Node Drain Completes Successfully

Once the VM becomes live migratable again, verify that the Machine Config Operator successfully drains the node and resumes applying the MachineConfig.


**Option 2**

## A. Change the eviction strategy to "LiveMigrateIfPossible"

```bash
oc patch vm <VM_Name> -n <Namespace> --type=merge -p '{"spec":{"template":{"spec":{"evictionStrategy":"LiveMigrateIfPossible"}}}}'
```

## B. Restart the VM

Restart the VM to apply the configuration change.

```bash
virtctl restart fedora-beige-fowl-40
```

## C. Verify That the VM Is Live Migratable Again

Check the VM status.

```bash
oc get vmi -n <Namespace> -o wide
```

Example output:

```bash
NAME                   AGE   PHASE     IP            NODENAME                 READY   LIVE-MIGRATABLE   PAUSED
fedora-beige-fowl-40   85s   Running   10.132.2.49   worker-cluster-4p54c-2   True    False
```

The `LIVE-MIGRATABLE` field should now display `False`.

`Note`: At this stage, the alert is resolved.

---

`Note` : During node maintenance, draining the node will cause the VM to stop and migrate, instead of being live migrated. As a result, the maintenance operation is not blocked.
