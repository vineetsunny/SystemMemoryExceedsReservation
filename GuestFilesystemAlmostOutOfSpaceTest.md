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
