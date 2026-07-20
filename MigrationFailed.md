**Meaning**
This alert indicates that an MTV (Migration Toolkit for Virtualization) migration plan has failed during one of its execution phases.
The alert is triggered when the following metric reports a failed migration status:
mtv_plan_alert_status{status="Failed"}
An MTV migration progresses through multiple execution phases, such as Initialize, Disk Allocation, Image Conversion, Disk Transfer, V2V Conversion, and Virtual Machine Creation. If the migration encounters an error during any of these phases, the migration status changes to Failed, triggering this alert.
The alert indicates that the migration did not complete successfully but does not identify the underlying cause. Before performing remediation, determine the phase in which the migration failed and review the associated migration logs and events to identify the root cause.


**OpenShift Impact**
	•	Locked Storage Capacity: A failed migration leaves behind provisioned Persistent Volumes (PVs). This traps storage capacity in OpenShift that other applications cannot use until cleaned up. [1]
	•	Stale Resource Objects: OpenShift keeps failed migration objects, pods, and tracking metadata in the cluster. This clutter can cause confusion for automation tools and other cluster administrators. [1, 2]
	•	Blocked Retry Attempts: You cannot restart or retry the migration for that specific Virtual Machine until the previous failed execution object is manually deleted from OpenShift.


**Impact**
VM Migration
One or more VMs were not migrated successfully.
Migration Plan
Migration workflow stops until the failure is resolved.
Application Availability
Depends on the migration phase. Cold migration may leave the VM powered off; warm migration may leave synchronization incomplete.
Cutover Activities
Migration cannot proceed until the failed VM or plan is recovered.
Business Impact


Planned migration windows may be delayed.

Diagnosis:


| Migration Phase | Responsible Component| Common Issues | Primary Logs | Investigation Commands| Next Checks|
|-----------------|----------------------|---------------|---------------|------------------------|------------|
|Initialize	| Forklift Controller | Provider authentication failure, inventory collection failure, provider API unreachable, invalid migration plan, missing network/storage mapping | forklift-controller | oc describe migration <migration> -n openshift-mtv, 
oc logs deployment/forklift-controller -n openshift-mtv	| Verify Provider, Plan, NetworkMap, StorageMap | 
| DiskAllocation | Forklift Controller + Kubernetes Storage	| PVC Pending, StorageClass missing, insufficient storage, CSI provisioning failure, quota exceeded	| forklift-controller| oc get pvc -n <targe-ns>, oc describe pvc <pvc> -n <target-ns>,
oc logs deployment/forklift-controller -n openshift-mtv	| Verify PV/PVC, StorageClass, CSI driver
| ImageConversion| virt-v2v Pod | Guest OS conversion failure, unsupported OS, missing drivers, conversion script errors | virt-v2v pod	| oc get pods -n <target-ns> /| grep virt-v2v , oc logs <virt-v2v-pod> -n <target-ns> | Review conversion errors, guest OS compatibility|
| DiskTransferV2v | virt-v2v Pod | VMware disk read failure, upload failure, network timeout, authentication failure, storage write error | virt-v2v pod | oc logs <virt-v2v-pod> -n <target-ns>, oc describe pod <virt-v2v-pod> -n <target-ns> | Verify provider connectivity, storage, network throughput|
|VirtualMachineCreation	| MTV Controller + KubeVirt + CDI | VM validation failed, DataVolume not ready, PVC not bound, scheduling failure, insufficient resources | VM Events, DataVolume, KubeVirt | oc get vm,vmi -n <target-ns> | oc describe vm <vm>, oc get dv -n <target-ns>, oc describe dv <dv> | Verify DataVolumes, PVCs, node resources
Completed	—	No issue	—	oc get migration -n openshift-mtv	Verify VM boots successfully








Diagnosis:
Primary file to identify the failed phase (Recommended)
oc describe migration <migration-name> -n openshift-mtv

Migration Phase
What Happens
Common Symptoms
Diagnosis (What to Check)
Commands
Possible Remediation
Initialize
MTV validates the migration plan, inventory, providers, storage mappings, network mappings, and creates the migration workflow.
Migration never starts or remains in Initialize.
Verify the migration plan, Forklift controller, provider connection, and migration CR status.
oc get plan -n openshift-mtv
oc get migration -n openshift-mtv
oc describe migration <migration-name> -n openshift-mtv
oc get pods -n openshift-mtv
oc logs deployment/forklift-controller -n openshift-mtv





Fix invalid provider credentials, storage/network mappings, or restart failed MTV controller pods.
DiskAllocation
Creates the target PVC(s) and allocates storage for VM disks.
Stuck in DiskAllocation or PVC remains Pending.
Check whether PVCs were created and whether the StorageClass can provision volumes.
oc get pvc -A
oc describe pvc <pvc-name>
oc get storageclass
oc get events --sort-by=.lastTimestamp
Ensure enough storage capacity, StorageClass is correct, CSI provisioner is healthy, and PVs can be created.
ImageConversion
Converts the source VM disk into a KubeVirt-compatible image (for example, VMware → qcow2/raw).
Conversion takes unusually long or fails.
Check the conversion pod and conversion logs for disk/image conversion errors.





oc get pods -n openshift-mtv
oc logs <conversion-pod> -n openshift-mtv
oc describe pod <conversion-pod>
Resolve image conversion errors, increase available CPU/memory if constrained, or verify source disk integrity.
DiskTransferV2v
Copies the converted VM disk to the destination PVC using virt-v2v/populator.
Transfer is slow, stuck, or fails.
Verify transfer pod status, PVC usage, storage throughput, and transfer logs.




oc get pods -n openshift-mtv
oc logs <disk-transfer-pod> -n openshift-mtv
oc describe pod <disk-transfer-pod>
oc get pvc -A
Check storage performance, available capacity, network connectivity (if applicable), and retry the migration if the transfer failed.
VirtualMachineCreation
MTV creates the KubeVirt VirtualMachine object after all disks are available.
VM object is not created or remains Pending.
Verify the VM, DataVolumes, PVC binding, and KubeVirt controllers.




oc get vm -A
oc describe vm <vm-name>
oc get dv -A
oc get pvc -A
oc get pods -n openshift-cnv
Resolve DataVolume import issues, PVC binding failures, insufficient resources, or KubeVirt controller problems.
WaitForGuestReboots
Waits for the Windows guest OS to reboot after virt-v2v installs drivers and completes conversion.
Migration waits indefinitely or eventually times out.
Verify the VM is running, guest OS has booted successfully, and the reboot completed.





oc get vmi -A                
oc describe vmi <vm-    name>
virtctl console <vm-name> (or use VNC)
oc get events


(Last column)Verify VirtIO drivers were installed, check Windows boot logs, resolve boot failures (for example, missing drivers or incorrect boot device), and reboot the VM if necessary.


During migration, check the migration logs using migration pod running under destination namespace where VMs are being imported.

Variable
Source Label
Purpose
MTV_NAMESPACE
namespace
Locate Forklift components
CONTROLLER_POD
pod
Collect Forklift controller logs
PLAN_NAME
plan_name
Find the Migration CR
FAILED_PHASE
phase
Decide the next diagnostic step
Cluster name 
Description
Diagnosis:
Step 1 – Validate the Migration
Use the plan name to identify the migration object:

oc get migration -n ${MTV_NAMESPACE}


oc describe migration <migration-name> -n ${MTV_NAMESPACE}

From the Migration CR, identify:
	•	Destination namespace
	•	VM name
	•	Migration status
	•	Failure reason

Step 2 – Collect Forklift Controller Logs
Since the alert already provides the controller pod, automation can directly collect:

oc logs ${CONTROLLER_POD} -n ${MTV_NAMESPACE}

This is the first log that should always be collected, regardless of the failure phase.


Step 3 – Branch Based on Failed Phase
The phase label tells the automation which component is most likely responsible.
Phase
Next Diagnostic
AllocateDisks
Check PVCs and StorageClasses
CopyDisks
Check DataVolume and importer pod logs
ConvertGuest
Check virt-v2v/conversion pod logs in the destination namespace
CreateVM
Check VM events and VM launcher pod (logs - forklift-wait-reboot-) 
Completed
No further diagnosis

Step 4 – Collect Migration Pod Logs
Once the destination namespace is identified from the Migration CR:
Collect logs from the migration pod:

oc logs <virt-v2v-pod> -n <target-namespace>

These logs usually contain the actual conversion or import error.

—








