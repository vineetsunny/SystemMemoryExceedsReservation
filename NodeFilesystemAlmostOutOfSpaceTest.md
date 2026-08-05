# Reproduce NodeFilesystemAlmostOutOfSpace Alert

## Procedure

> **Note**
>
> This procedure uses the **`/tmp`** filesystem because it is easy to clean up after testing.
>
> If you perform this test on any other filesystem (for example, `/`, `/var`, or `/var/lib/containers`), ensure you have **SSH access** to the node. Filling critical filesystems may prevent `oc debug` from working, making cleanup difficult without direct node access.

---

***1. Create the disk fill script***

Create a file named **`disk_fillup.sh`** with the following contents:

```bash
#!/bin/bash

if [ $# -ne 1 ]; then
    echo "Usage: $0 <target_percentage>"
    echo "Example: $0 90"
    exit 1
fi

TARGET="$1"
FILESYSTEM="/tmp"
TEST_DIR="${FILESYSTEM}/diskfill-test"

# Validate input
if ! [[ "$TARGET" =~ ^[0-9]+$ ]] || [ "$TARGET" -lt 1 ] || [ "$TARGET" -gt 99 ]; then
    echo "Target must be an integer between 1 and 99."
    exit 1
fi

mkdir -p "${TEST_DIR}"

echo "Filesystem : ${FILESYSTEM}"
echo "Target     : ${TARGET}%"
echo

while true; do
    USED=$(df -P "${FILESYSTEM}" | awk 'NR==2 {gsub("%","",$5); print $5}')

    if [ "$USED" -ge "$TARGET" ]; then
        break
    fi

    FILE="${TEST_DIR}/fill-$(date +%s)-$RANDOM.bin"

    # Create a 100 MB file
    dd if=/dev/zero of="${FILE}" bs=1M count=100 status=none

    df -h "${FILESYSTEM}" | awk 'NR==2 {printf "Used: %s | Available: %s | Usage: %s\n",$3,$4,$5}'
done

echo
echo "Target reached!"
df -h "${FILESYSTEM}"
echo
echo "Space occupied by test files:"
du -sh "${TEST_DIR}"
```

Make the script executable:

```bash
chmod +x disk_fillup.sh
```

---

***2. Trigger the Warning alert***

Increase the filesystem usage to approximately **96%**, which leaves less than **5% free space**.

Run the script:

```bash
./disk_fillup.sh 96
```

Verify the filesystem usage:

```bash
df -h /tmp
```

**Example output**

```text
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            12G   12G  513M  96% /tmp
```

Wait a few minutes for Prometheus to evaluate the metrics.

The **NodeFilesystemAlmostOutOfSpace** alert should transition to the **Warning** state.

![](assets/17858955392400.jpg)

---

***3. Trigger the Critical alert***

Increase the filesystem usage to approximately **98%**, which leaves less than **3% free space**.

Run the script:

```bash
./disk_fillup.sh 98
```

Verify the filesystem usage:

```bash
df -h /tmp
```

**Example output**

```text
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            12G   12G  250M  98% /tmp
```

After Prometheus evaluates the metrics, the alert should transition from **Warning** to **Critical**.

![](assets/17858961467466.jpg)

---

***4. Clean up the test files***

Remove the files created during the test:

```bash
rm -rf /tmp/diskfill-test
```

Verify that the disk space has been released:

```bash
df -h /tmp
```

**Example output**

```text
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            12G  8.0K   12G   1% /tmp
```

Once the disk usage drops below the alert thresholds, the alert will automatically clear after the next Prometheus evaluation cycle.
