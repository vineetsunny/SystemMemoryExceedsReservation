# Runbook: Reproducing and Remediating **GuestFilesystemAlmostOutOfSpace**

## Objective

This runbook describes how to:

* Reproduce the **GuestFilesystemAlmostOutOfSpace** alert.
* Generate both **Warning** and **Critical** alert severities.
* Resolve the alert by either expanding the virtual disk or freeing disk space.

---

# Reproducing the Alert

## Prerequisites

* A Linux Virtual Machine (Fedora is used in this example).
* The VM must use a PersistentVolume that supports online expansion (if testing disk expansion).
* Sufficient storage available in the backend storage system.
* OpenShift Version v4.22.x and Virtualization operator v4.22.x

> **Note**
>
> This runbook uses Fedora with a **30 GiB** root filesystem. Adjust the file sizes if your VM uses a different disk size.

---

## Step 1: Create a Linux Virtual Machine

Create a new Virtual Machine using the **Fedora** template.

Wait until the VM is in the **Running** state.

---

## Step 2: Connect to the VM

Access the VM using SSH/virtctl or GUI.

Using ssh:
```bash
oc get vmi -n <namespace> -o wide
ssh <username>@<vm-ip>
```

The VM must have:
An SSH server (sshd) running.
Network connectivity to the client machine.
A reachable IP address.
Appropriate firewall rules allowing SSH (TCP port 22).

`OR`

```bash
oc get vmi -n <namespace> -o wide
virtctl console <vmiName>
```
---

## Step 3: Verify the Initial Disk Usage

Run:

```bash
df -h
```

Expected output (example):

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda3        30G  681M   29G   3% /
```

The root filesystem should initially have very low utilisation.

---

## Step 4: Increase Disk Usage Above 85%

Create a large file to consume disk space.

```bash
sudo fallocate -l 25G /testfile
```

Verify the utilisation:

```bash
df -h
```

Expected output:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda3        30G   26G  3.8G  88% /
```

The root filesystem is now approximately **88% utilised**.

> **Note**
>
> `fallocate` allocates space almost instantly and is useful for reproducing this alert. If `fallocate` is unavailable, `dd` can be used instead.

---

## Step 5: Verify the Warning Alert

After the filesystem remains above **85% utilisation** for approximately **5 minutes**, the following alert should be generated:

* **Severity:** Warning
* **Alert:** GuestFilesystemAlmostOutOfSpace

> **Expected Result**
>
The alert enters the **Warning** state once utilisation remains between **85% and 95%** for the configured duration.

<img width="3126" height="750" alt="image" src="https://github.com/user-attachments/assets/16ab49fa-01cc-42ba-bbb7-c2972e1e0f71" />

￼￼

￼

---

## Step 6: Increase Disk Usage Above 95%

Increase the allocated file size.

```bash
sudo fallocate -l 28G /testfile
```

Verify the filesystem usage.

```bash
df -h
```

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda3        30G   29G  804M  98% /
```

The filesystem is now approximately **98% utilised**.

---

## Step 7: Verify the Critical Alert

Once utilisation exceeds **95%**, the alert should transition to:

* **Severity:** Critical

> **Expected Result**
>
> The alert changes from **Warning** to **Critical** after the filesystem usage exceeds the critical threshold.

<img width="3126" height="750" alt="image" src="https://github.com/user-attachments/assets/b0422e07-2c24-4fb6-a1c7-0f920fd12502" />

---

# Remediation

There are two supported remediation options:

1. Increase the virtual disk size.
2. Remove unnecessary files to free disk space.

---

# Option 1: Expand the Virtual Disk

## Prerequisites

Before resizing the disk, verify the following:

* The StorageClass supports volume expansion.
* The PVC is in the **Bound** state.
* The VM disk is backed by a PVC/DataVolume.

> **Important**
>
> This example expands a **DataVolume** attached to the VM.
>
> If the VM uses a **containerDisk**, it cannot be resized.

---

## Step 1: Verify StorageClass Supports Expansion

```bash
oc get sc
```

Example:

```text
NAME                                             ALLOWVOLUMEEXPANSION
ocs-external-storagecluster-ceph-rbd             true
```

Ensure:

```text
ALLOWVOLUMEEXPANSION = true
```

---

## Step 2: Verify the PVC is Bound

```bash
oc get pvc
```

Example:

```text
NAME                        STATUS   CAPACITY
fedora-scarlet-haddock-20   Bound    30Gi
```

---

## Step 3: Expand the PVC

Increase the PVC size from **30 GiB** to **50 GiB**.

```bash
oc patch pvc fedora-scarlet-haddock-20 \
-n testing \
--type merge \
-p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

Expected output:

```text
persistentvolumeclaim/fedora-scarlet-haddock-20 patched
```

---

## Step 4: Verify the PVC Size

```bash
oc get pvc
```

Expected output:

```text
CAPACITY
50Gi
```

At this point, the block device has grown, but the filesystem inside the VM still needs to be resized.

---

## Step 5: Determine the Filesystem Type

Check the filesystem type.

```bash
df -Th
```

or

```bash
lsblk
```

Example:

```text
Filesystem     Type
/dev/vda3      btrfs
```

> **Important**
>
> The filesystem resize command depends on the filesystem type.
>
> * **Btrfs** → `btrfs filesystem resize`
> * **XFS** → `xfs_growfs`
> * **ext4** → `resize2fs`

---

## Step 6: Resize the Filesystem

For **Btrfs**, run:

```bash
sudo btrfs filesystem resize max /
```

Verify the result.

```bash
lsblk
```

Expected:

```text
vda3 49.9G
```

Then verify filesystem utilisation.

```bash
df -h
```

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda3        50G   29G   21G  58% /
```

The filesystem has successfully expanded and utilisation has dropped to approximately **58%**.

> **Expected Result**
>
> Once the filesystem usage falls below the alert threshold, the **GuestFilesystemAlmostOutOfSpace** alert automatically clears.

---

# Option 2: Free Disk Space

Instead of expanding the disk, reclaim space by removing unnecessary data, such as:

* Temporary files
* Old log files
* Archived backups
* Unused application data
* Crash dumps

After deleting files, verify the utilisation.

```bash
df -h
```

Once the filesystem usage falls below the configured threshold, the alert resolves automatically.

---

# Validation

After remediation, verify:

```bash
df -h
```

Confirm that:

* Filesystem utilisation is below **85%**
* The VM has sufficient free space for normal operation
* The alert is no longer active in Alertmanager or the monitoring dashboard

---

# Caveats

* `fallocate` permanently consumes disk space until the file is removed.
* Delete the test file after completing validation:

```bash
sudo rm -f /testfile
```

* Online volume expansion requires:

  * A StorageClass with `allowVolumeExpansion: true`
  * CSI driver support for expansion
  * Filesystem resize within the guest operating system
* Filesystem resize commands differ depending on the filesystem type (Btrfs, XFS, ext4, etc.).
* Expanding the PVC alone does **not** automatically increase the filesystem size inside the guest VM; the guest filesystem must also be resized.


**Diagnosis**
| Diagnosis (Commands) | What to Look For | Action |
|----------------------|------------------|--------|
| Connect to the VM using the OpenShift Console, `virtctl console <vm-name> -n <namespace>`, or `ssh <user>@<vm-ip>`. | Verify the VM is running and you can access the guest operating system. | Proceed with filesystem investigation. |
| `df -h` | Identify the affected filesystem and confirm that its utilization matches the alert threshold (Warning: 85–95%, Critical: >95%). | Continue investigating the cause of the high disk utilization. |
| `df -Th`<br/>or<br/>`lsblk -f` | Determine the filesystem type (for example, **btrfs**, **xfs**, or **ext4**) and identify the affected mount point. | Record the filesystem type for use during remediation if disk expansion is required. |
| `du -xh / --max-depth=1 2>/dev/null \| sort -h` | Identify the top-level directories (for example, `/var`, `/home`, `/tmp`) consuming the most disk space. | Determine whether the usage is caused by logs, temporary files, backups, or application data. |
| `find / -xdev -type f -size +100M -exec ls -lh {} \; 2>/dev/null` | Identify unusually large files contributing to high disk utilization. | Record the large files and determine whether they are expected or require cleanup. |
| `oc get sc` | Verify whether the StorageClass backing the VM supports volume expansion (`ALLOWVOLUMEEXPANSION=true`). | Determine whether online volume expansion is supported if additional storage is required. |
| `oc get pvc -n <namespace>` | Verify the VM's PVC is **Bound** and note its current capacity. | Confirm the PVC is healthy and identify the current disk size before planning any expansion. |

**Remediation:**

Here's a sequential **Remediation** section that is suitable for a runbook.

## Remediation

There are two possible remediation approaches:

* **Option 1 (Recommended):** Free disk space by removing unnecessary files.
* **Option 2:** Expand the virtual disk if additional storage is required.

### Option 1: Free Disk Space

1. Identify unnecessary files consuming disk space, such as:

   * Temporary files
   * Old log files
   * Application caches
   * Backup or archive files
   * Unused application data

2. Remove only the files that are no longer required.

3. Verify that the filesystem utilization has dropped below the alert threshold.

   ```bash
   df -h
   ```

4. Confirm that the **GuestFilesystemAlmostOutOfSpace** alert clears automatically.

---

### Option 2: Expand the Virtual Disk

Yes. This is an important caveat to include because **not every VM disk is expandable**. The operator should first identify the type of disk attached to the VM.

You can add the following section before expanding the PVC.

---

### Step 2: Verify that the VM Disk Can Be Expanded

Before attempting to resize the disk, identify the type of disk attached to the Virtual Machine.

Only **PersistentVolumeClaim (PVC)-backed DataVolumes** support online expansion (provided the StorageClass supports volume expansion).

The following disk types **cannot** be resized using this procedure:

* **containerDisk** (read-only container image)
* **Ephemeral disks**
* Other non-expandable disk types

You can verify the VM disk configuration by running:

```bash
oc get vm <vm-name> -n <namespace> -o yaml
```

Look for the `volumes` section.

**Expandable (PVC/DataVolume-backed disk):**

```yaml
volumes:
- dataVolume:
  name: rootdisk
```

or

```yaml
volumes:
- persistentVolumeClaim:
    claimName: fedora-scarlet-haddock-20
  name: rootdisk
```

These disks can be expanded by increasing the associated PVC size.

**Not Expandable (containerDisk):**

```yaml
volumes:
- containerDisk:
  name: rootdisk
```

Since a `containerDisk` is read-only, its size cannot be increased. In this case, create a new VM with a larger disk or migrate the workload to a VM backed by a PVC/DataVolume.


#### Step 1: Verify that the StorageClass supports volume expansion

```bash
oc get sc
```

Confirm that the StorageClass used by the VM has:

```text
ALLOWVOLUMEEXPANSION=true
```

---

#### Step 2: Verify the PersistentVolumeClaim (PVC)

```bash
oc get pvc -n <namespace>
```

Ensure the PVC is in the **Bound** state.

---

#### Step 3: Expand the PVC

Increase the requested storage size.

Example:

```bash
oc patch pvc <pvc-name> -n <namespace> \
--type merge \
-p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

---

#### Step 4: Verify the PVC has been expanded

```bash
oc get pvc -n <namespace>
```

Confirm that the PVC reflects the new capacity.

---

#### Step 5: Determine the guest filesystem type

```bash
df -Th
```

or

```bash
lsblk -f
```

Identify whether the filesystem is **Btrfs**, **XFS**, or **ext4**.

> **Note:** The filesystem resize command depends on the filesystem type.

---

#### Step 6: Resize the guest filesystem

For **Btrfs**:

```bash
sudo btrfs filesystem resize max /
```

For **XFS**:

```bash
sudo xfs_growfs /
```

For **ext4**:

```bash
sudo resize2fs <device>
```

---

#### Step 7: Validate the filesystem

Verify that the additional storage is available and the utilization has decreased.

```bash
df -h
```

The filesystem should reflect the new capacity, and the usage should be below the alert threshold.

---

#### Step 8: Verify the alert is resolved

Confirm that the **GuestFilesystemAlmostOutOfSpace** alert has cleared in Alertmanager or your monitoring dashboard.

### Important Notes

* Always consider **freeing disk space** before expanding the virtual disk.
* Ensure the StorageClass supports **online volume expansion** (`ALLOWVOLUMEEXPANSION=true`) before resizing the PVC.
* Expanding the PVC does **not** automatically expand the guest filesystem; the filesystem must also be resized from within the VM.
* The appropriate resize command depends on the filesystem type (Btrfs, XFS, ext4, etc.).

### Expression Logic

1. Collect the used filesystem bytes

```promql
kubevirt_vmi_filesystem_used_bytes{...}
```

2. Collect the filesystem capacity

```promql
kubevirt_vmi_filesystem_capacity_bytes{...}
```

3. Calculate usage percentage

```promql
(used_bytes / capacity_bytes) * 100
```

4. Trigger a warning when usage is between 85% and 95%.


**Dicision Flow**

flowchart TD

A[GuestFilesystemAlmostOutOfSpace Alert Triggered] --> B[Check filesystem usage inside the VM]

B --> C{Can unnecessary files be removed?}

C -->|Yes| D[Remove unnecessary files]

C -->|No| E[Expand the virtual disk]

D --> F{Usage below threshold?}

F -->|Yes| G[Alert resolves automatically]

F -->|No| E

E --> H{Storage supports volume expansion?}

H -->|Yes| I[Expand the disk and resize the filesystem]

H -->|No| J[Provision a larger disk or migrate data]

I --> K[Validate filesystem usage]

K --> G
