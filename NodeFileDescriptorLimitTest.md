# Reproducing `NodeFileDescriptorLimit` Alert
---

## Step 1: Reduce the System File Descriptor Limit

> **Note:** Perform this only in a non-production/lab environment.

Log in to the node and check the current file descriptor settings.

```bash
oc debug node/<node-name>
chroot /host

cat /proc/sys/fs/file-nr
```

Example output:

```text
2624    0    9223372036854775807
```

Reduce the system-wide file descriptor limit:

```bash
sysctl -w fs.file-max=60000
```

Note: In my node, the original system-wide file descriptor limit was `9223372036854775807`. For testing purposes, it was temporarily reduced to 60000 to make reproducing the alert practical.

---

## Step 2: Verify the Configuration

Verify that the new limit has been applied.

```bash
cat /proc/sys/fs/file-nr
```

Example output:

```text
2624    0    60000
```

Where:

- **2624** – Currently allocated file descriptors
- **0** – Unused allocated descriptors
- **60000** – Maximum file descriptors allowed

---

## Step 3: Create the Test Script

Create a file named `fd_test.py`.

```python
import time
import os

TARGET = 40000          # Open ~40K additional FDs
BATCH = 500             # Open in batches
SLEEP = 0.2             # Pause between batches

fds = []

print(f"PID: {os.getpid()}")
print(f"Target: {TARGET} file descriptors")

try:
    while len(fds) < TARGET:
        for _ in range(BATCH):
            fds.append(open("/dev/null", "r"))

        print(f"Opened {len(fds)} FDs")
        time.sleep(SLEEP)

except OSError as e:
    print(f"Stopped at {len(fds)} FDs")
    print(f"Error: {e}")

print("Holding file descriptors open...")

while True:
    time.sleep(60)
```

---

## Step 4: Run the Script

Execute the script and keep it running.

```bash
python3 fd_test.py
```

Example output:

```text
Opened 39000 FDs
Opened 39500 FDs
Opened 40000 FDs
Holding file descriptors open...
```

---

## Step 5: Verify File Descriptor Utilization

In another terminal/session, verify the file descriptor allocation.

```bash
cat /proc/sys/fs/file-nr
```

Example output:

```text
42656    0    60000
```

The utilization is:

```text
42656 / 60000 × 100 = 71.09%
```

This satisfies the alert threshold (>70%).

---

## Step 6: Wait for the Alert

Keep the Python script running for **at least 15 minutes**.

The alert rule is:

```promql
(node_filefd_allocated * 100 / node_filefd_maximum) > 70
```

After the condition remains true continuously for **15 minutes**, the alert transitions.

---

## Verification

Check the alert status:

```promql
ALERTS{alertname="NodeFileDescriptorLimit"}
```

Expected states:

- `pending` – Condition has crossed the threshold but has not yet met the 15-minute duration.
- `firing` – Condition has remained above the threshold for 15 continuous minutes.

---

## Cleanup

Stop the Python script:

```text
Ctrl+C
```

Restore the original file descriptor limit.

```bash
sysctl -w fs.file-max=<original-value>
```

Verify:

```bash
cat /proc/sys/fs/file-nr
```





