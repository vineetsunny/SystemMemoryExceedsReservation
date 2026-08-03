# NodeFilesystemAlmostOutOfFiles

Prometheus Source: node-exporter-rules
Severity: Warning / Critical
Runbook:https://github.com/openshift/runbooks/blob/master/alerts/cluster-monitoring-operator/NodeFilesystemAlmostOutOfFiles.md
For:1h

## Meaning

This alert indicates that a writable filesystem on an OpenShift node is approaching **inode exhaustion**.

An inode stores metadata about a file or directory. Once all available inodes are consumed, the filesystem can no longer create new files or directories, even if sufficient disk space remains available.

This condition is typically caused by a very large number of small files, such as container logs, temporary files, cached images, or application-generated files.

The alert is triggered when the percentage of available inodes falls below the configured threshold.

### Warning

Triggered when fewer than **5%** of inodes remain available.

**Calculation**

```text
(Available Inodes / Total Inodes) × 100 < 5
```

### Critical

Triggered when fewer than **3%** of inodes remain available.

**Calculation**

```text
(Available Inodes / Total Inodes) × 100 < 3
```

### Metrics Used

| Metric | Description |
|--------|-------------|
| `node_filesystem_files_free` | Number of available (free) inodes |
| `node_filesystem_files` | Total number of inodes |
| `node_filesystem_readonly == 0` | Evaluates only writable filesystems |
| `mountpoint!~"/var/lib/ibmc-s3fs.*"` | Excludes IBM S3FS mount points |
| `job="node-exporter"` | Filesystem metrics collected by node-exporter |

---

## Impact

If inode exhaustion occurs, workloads running on the affected node may experience failures even when sufficient disk capacity is available.

Possible impacts include:

- Inability to create new files or directories.
- Applications failing to write logs, temporary files, or application data.
- Pods or system services failing to start.
- Reduced stability of workloads running on the affected node.
- Potential degradation of overall OpenShift cluster health if the issue is not resolved.

---

# Diagnosis

## Variables

```bash
NODE=<labels.node>
```

## Step 1. Identify the Affected Filesystem

Query Prometheus to determine inode utilization across all node filesystems.

```bash
oc exec -it -n openshift-monitoring prometheus-k8s-0 -c prometheus -- \
curl -s 'http://localhost:9090/api/v1/query?query=(1-(node_filesystem_files_free/node_filesystem_files))*100'
```

Identify:

- The affected node.
- The affected filesystem (for example, `/`, `/var`, or `/var/lib/containers`).

---

## Step 2. Open a Debug Session

```bash
oc debug node/$NODE

chroot /host
```

---

## Step 3. Verify Filesystem Inode Utilization

Display inode usage for all mounted filesystems.

```bash
df -i
```

Verify that the filesystem identified in Prometheus has the highest **IUse%**.

---

## Step 4. Identify Directories Consuming Large Numbers of Inodes

Inspect the affected filesystem to determine which directories contain the largest number of files.

For example, if `/var` is affected:

```bash
find /var -xdev -printf '%h\n' | sort | uniq -c | sort -nr | head -20
```

> **Note:** If another filesystem is affected, replace `/var` with the appropriate mount point.

---

## Step 5. Inspect CRI-O Container Storage

If the affected filesystem contains CRI-O storage, inspect the overlay directories.

```bash
find /var/lib/containers/storage/overlay -maxdepth 2 | wc -l
```

A large number of overlay directories may indicate unused container images or stale container layers.

---

## Step 6. Inspect Log and Temporary Directories

Review common locations for excessive numbers of files.

```text
/var/log
/tmp
/var/tmp
```

Look for applications or processes generating unusually large numbers of files.

---

# Remediation

> **Note**
>
> Before removing files, verify that they are no longer required. Avoid manually deleting OpenShift-managed files or directories unless instructed by Red Hat documentation or support.

## Step 1. Remove Unused Container Images

Prune all unused CRI-O images.

```bash
crictl rmi --prune
```

This removes cached images that are no longer referenced and helps reclaim inodes used by container storage.

---

## Step 2. Remove Exited Containers

Delete exited containers and their writable layers.

```bash
crictl ps -a -q --state Exited | xargs -r crictl rm
```

---

## Step 3. Remove Unnecessary Temporary Files

Delete temporary files that have not been accessed within the last two days.

```bash
find /tmp /var/tmp -type f -atime +2 -delete
```

This removes aged temporary files while preserving recently used files.

---

## Step 4. Remove Application-Specific Files

If the diagnosis identifies an application generating excessive numbers of files, remove or archive only those files after confirming they are no longer required.

Examples include:

- Application cache files
- Temporary files
- Old backups
- Application-generated logs

Avoid manually deleting OpenShift-managed directories such as `/var/log/pods` unless specifically directed by Red Hat Support.

---

## Step 5. Verify Inode Utilization

After completing the cleanup, verify that inode utilization has decreased.

```bash
df -i
```

Or verify a specific filesystem, for example:

```bash
df -i /var
```

Confirm that **IUse%** has dropped below the alert threshold and verify that the alert has cleared in Alertmanager or the OpenShift monitoring dashboard.
