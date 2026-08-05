# NodeFilesystemAlmostOutOfSpace

**Prometheus Source:** `NodeFilesystemAlmostOutOfSpace` · **Severity:** `Warning / Critical` · **Duration For:** `30m` · [**Runbook**](https://github.com/openshift/runbooks/blob/master/alerts/cluster-monitoring-operator/NodeFilesystemAlmostOutOfSpace.md)

---

## Meaning

This alert indicates that a node's filesystem is running low on available disk space. This alert is triggered based on fixed free space thresholds.<br>As the available space decreases, the risk of the filesystem becoming completely full increases. If no action is taken, the node may no longer be able to write new data, logs, or temporary files required by the operating system and workloads.
The alert is triggered when the percentage of available disk space on a writable filesystem falls below the configured threshold.

### Warning

Triggered when **less than 5%** of disk space remains available.(Utilization > 95 %)

**Calculation**

```text
(Available Disk Space / Total Filesystem Size) × 100 < 5
```

### Critical

Triggered when **less than 3%** of disk space remains available.(Utilization > 97 %)

**Calculation**

```text
(Available Disk Space / Total Filesystem Size) × 100 < 3
```

### Metrics Used

| Metric | Description |
|--------|-------------|
| `node_filesystem_avail_bytes` | Amount of disk space available to non-root users (in bytes). |
| `node_filesystem_size_bytes` | Total size of the filesystem (in bytes). |
| `node_filesystem_readonly == 0` | Evaluates only writable filesystems and ignores read-only filesystems. |
| `mountpoint!~"/var/lib/ibmc-s3fs.*"` | Excludes IBM S3FS mount points from evaluation. |
| `job="node-exporter"` | Uses filesystem metrics collected by the node-exporter. |
| `fstype!=""` | Excludes entries that do not have a valid filesystem type. |

---

## Impact

* Applications may fail to write data, logs, or temporary files.
* Workloads running on the affected node may experience degraded performance or unexpected failures.
* System services on the node may fail or behave unexpectedly due to insufficient disk space.
* New pods or workloads may fail to start if required disk space is unavailable.
* If the filesystem becomes completely full, the node may become unstable or enter a degraded state.
* Prolonged disk space exhaustion can lead to application downtime and impact overall cluster reliability.

---

## Diagnosis

## Variables (From Alert)

```bash
Filesystem_Device=<labels.device>
Mountpoint=<labels.mountpoint>
Node=<labels.instance>
Free_Space=<value>
```
Follow these steps to identify why the filesystem is running low on disk space.

***1. Identify the affected filesystem***

From the alert, note the following information:

- **Node**
- **Mountpoint** (for example: `/`, `/var`, `/var/lib/containers`)

The **mountpoint** indicates which filesystem is running out of space and should be the focus of your investigation.

***2. Verify the available disk space***

Access the affected node.

```bash
oc debug node/<node-name>
chroot /host
```

Check the disk usage of all mounted filesystems:

```bash
df -h
```

Or check only the affected mountpoint:

```bash
df -h <mountpoint>
```

**Example:**

```bash
df -h /var
```

Verify that the available space matches the alert condition.

***3. Identify the directories consuming the most space***

Find the largest directories under the affected mountpoint:

```bash
du -xh --max-depth=1 <mountpoint> | sort -hr
```

**Example:**

```bash
du -xh --max-depth=1 /var | sort -hr
```

If a directory is consuming most of the space, continue checking inside it.

***4. Identify large files***

List the largest files in the affected filesystem:

```bash
find <mountpoint> -type f -exec du -h {} + | sort -hr | head -20
```

Look for unusually large log files, core dumps, backup files, or temporary files that may be consuming disk space.

***5. Review the filesystem free space trend***

Monitor the following Prometheus metric:

```promql
node_filesystem_free_bytes
```
Example:

Check for affected mountpoint:
```promql
node_filesystem_free_bytes{mountpoint="/<mountpoint>"}
```
Review the trend over time to determine whether:

- Free space is **steadily decreasing**, indicating continuous disk usage growth.
- Free space **drops and then recovers periodically**, indicating temporary usage that is cleaned up automatically.

***6. Determine the root cause***

Based on the findings, determine whether the disk usage is caused by:

- **Unexpected growth**, such as log files, temporary files, core dumps, or a process that is not cleaning up old data.
- **Organic growth**, where applications are generating data faster than the available storage can accommodate.

Identifying the root cause will help determine the appropriate remediation, such as removing unnecessary files, rotating logs, or expanding the filesystem.

---

# Mitigation

If the affected **mountpoint** is **`/`**, **`/sysroot`**, or **`/var`**, the issue may be caused by unused container images consuming disk space. Remove unused images to free up space.

> **Note**

Before removing images, verify that they are no longer required by any running workloads.

***1. Access the affected node***

Use the **instance** value from the alert to identify the affected node.

```bash
NODE_NAME=<instance-from-alert>
```

Start a debug session on the node:

```bash
oc debug node/<NODE_NAME>
```

Access the host filesystem:

```bash
chroot /host
```

***2. Remove dangling images***

Dangling images are untagged images that are no longer referenced and can be safely removed.

```bash
podman images -q -f dangling=true | xargs --no-run-if-empty podman rmi
```

***3. Remove unused images***

Remove container images that are no longer required while preserving commonly used Red Hat and OpenShift images.

```bash
podman images \
| grep -v -e registry.redhat.io \
          -e quay.io/openshift \
          -e registry.access.redhat.com \
          -e docker-registry.usersys.redhat.com \
          -e docker-registry.ops.rhcloud.com \
          -e rhmap \
| xargs --no-run-if-empty podman rmi 2>/dev/null
```

> **Note**
>
> Review the list of images before removing them to ensure that no required images are deleted.

***4. Verify disk space***

Confirm that sufficient disk space has been reclaimed.

```bash
df -h
```

Or check only the affected mountpoint:

```bash
df -h <mountpoint>
```

***5. Exit the debug session***

Once the cleanup is complete, exit the debug session.

```bash
exit
```

After the available disk space rises above the alert threshold, the **NodeFilesystemAlmostOutOfSpace** alert should automatically clear during the next Prometheus evaluation cycle.

---

## Related Alerts

* `NodeFilesystemSpaceFillingUp` (Warning/Critical)<br>
Warning:  Mountpoint utilization > 85% <br>
Critical: Mountpoint utilization > 90% <br>
