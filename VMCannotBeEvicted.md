### VMCannotBeEvicted

Prometheus source: prometheus-kubevirt-rules
For: 1m
Runbook: https://github.com/openshift/runbooks/blob/master/alerts/openshift-virtualization-operator/VMCannotBeEvicted.md


# Meaning

This alert indicates that a virtual machine (VM) is configured to use Live Migration as its eviction strategy, but the VM cannot be live migrated due to its current configuration or runtime conditions. Common reasons include unsupported network or storage configurations, or attached devices that do not support live migration.

# Impact

Prevents the VM from being evicted during node drain.
Blocks node maintenance and cluster upgrade operations that require node evacuation.
Delays administrative tasks until the VM becomes live migratable or its eviction strategy is updated.

# Diagnosis

```bash
VM_NAME=labels.name
NAMESPACE=labels.namespace
NODE_NAME=labels.node
```

## Step 1: Check the VMI configuration to determine whether the value (True/False) of evictionStrategy is LiveMigrate of the VMI:

```bash
oc get vmi -n <Namespace> -o wide
```


## Step 2 : Obtain the details of the VMI and check spec.conditions to identify the issue:

```bash
oc get vmi <VMI_Name> -n <Namespace> -o jsonpath='{"STATUS: "}{.status.conditions[?(@.type=="LiveMigratable")].status}{"\n"}{"REASON: "}{.status.conditions[?(@.type=="LiveMigratable")].reason}{"\n"}{"MESSAGE: "}{.status.conditions[?(@.type=="LiveMigratable")].message}{"\n"}'

It will show the output as live migrate fail reason.
```

# Remediation

There are two supported approaches to resolve this alert:

## Option 1: Restore Live Migratability (Preferred)

The default eviction strategy is `LiveMigrate`, which ensures that a VirtualMachineInstance (VMI) is live migrated when the node is drained or placed into maintenance.

If the VM is no longer live migratable, resolve the underlying issue preventing live migration. Once the VM becomes live migratable again, the alert is resolved and node maintenance can proceed without interrupting the VM.

---

## Option 2: Change the Eviction Strategy

If restoring live migratability is not feasible, update the VM to use the `LiveMigrateIfPossible` (or `None`) eviction strategy. This allows node maintenance to continue even when the VM cannot be live migrated. During node drain, the VM will be stopped and restarted on another node instead of being live migrated.

```bash
oc patch vm <VM_Name> -n <Namespace> --type=merge -p '{"spec":{"template":{"spec":{"evictionStrategy":"LiveMigrateIfPossible"}}}}'
```
```

# Decision Flow


flowchart TD
    A["Alert: VMCannotBeEvicted"] --> B["Check the affected VMI"]

    B --> C["Verify:<br/>evictionStrategy = LiveMigrate<br/>LIVE-MIGRATABLE = False"]

    C --> D["Review the LiveMigratable<br/>Reason and Message"]

    D --> E{"Can the underlying issue be resolved?"}

    E -- Yes --> F["Resolve the issue<br/>(network, storage,<br/>attached devices, etc.)"]

    F --> G["Verify<br/>LiveMigratable = True"]

    E -- No --> H["Update the VM eviction strategy<br/>to LiveMigrateIfPossible or None"]

    H --> I["Restart the VM<br/>to apply the new configuration"]

    G --> J["Verify the alert is cleared"]

    I --> J

    J --> K["Node drain, maintenance,<br/>and cluster upgrades can proceed"]
