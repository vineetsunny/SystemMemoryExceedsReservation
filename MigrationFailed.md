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




flowchart TD

A[MigrationFailed] --> B{Failed Phase?}

B -->|Initialize| C[Forklift Logs]
B -->|DiskAllocation| D[Storage + Forklift Logs]
B -->|ImageConversion| E[virt-v2v Logs]
B -->|DiskTransferV2v| F[virt-v2v Logs]
B -->|VM Creation| G[VM/DV/PVC Events]
B -->|Unknown| H[Migration Events]

C --> I[Fix Provider/Mapping]
D --> J[Fix Storage/PVC]
E --> K[Fix Conversion]
F --> L[Fix Transfer]
G --> M[Fix VM Creation]
H --> N[Identify Root Cause]

I --> O[Retry Migration]
J --> O
K --> O
L --> O
M --> O
N --> O

—








