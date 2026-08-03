## Reproduce NodeFilesystemAlmostOutOfFiles Alert

This procedure reproduces the inode exhaustion alert by creating a large number of empty files on a selected filesystem. In this example, the /tmp filesystem is used.

## Procedure: 
***1. Access the affected node***

> Log in to the OpenShift node.

```bash
oc debug node/<node-name>
chroot /host
```

***2. Verify the target filesystem***

Check the current inode utilization of the target filesystem. In this example, the /tmp filesystem is used.

```bash
df -ih /tmp
```

***3. Create a test directory***

Create a dedicated directory to store the test files.

```bash
mkdir -p /tmp/inode_test
```

***4. Create the inode generation script***

Create the following script (for example, inode_fillup.sh) to generate empty files until the desired inode utilization is reached.

Usage

./inode_fillup.sh <target_inode_utilization_percentage>

Example:
```bash
./inode_fillup.sh 96
```

`Note`: The target utilization percentage can be adjusted as required to reproduce the desired alert severity.

```bash

#!/bin/bash

# Usage:
# ./inode_fillup.sh <target_percent> [mount_point] [test_directory]

TARGET="${1:-97}"
MOUNT_POINT="${2:-/tmp}"
TEST_DIR="${3:-/tmp/inode_test}"

if ! [[ "$TARGET" =~ ^[0-9]+$ ]] || [ "$TARGET" -lt 1 ] || [ "$TARGET" -gt 100 ]; then
    echo "Error: Target must be an integer between 1 and 100."
    echo "Usage: $0 <target_percent> [mount_point] [test_directory]"
    exit 1
fi

mkdir -p "${TEST_DIR}"

echo "Creating files in ${TEST_DIR} until inode usage on ${MOUNT_POINT} reaches ${TARGET}%..."

while true; do
    USED=$(df -i "${MOUNT_POINT}" | awk 'NR==2 {gsub("%","",$5); print $5}')

    if [ "$USED" -ge "$TARGET" ]; then
        echo "Target reached: ${USED}% inode usage."
        break
    fi

    for i in $(seq 1 10000); do
        touch "${TEST_DIR}/file_${RANDOM}_$(date +%s%N)_$i"
    done

    echo "Current inode usage: $(df -i "${MOUNT_POINT}" | awk 'NR==2 {print $5}')"
done
echo "Done."

```

***5. Trigger the Warning alert***

Run the script with a target inode utilization greater than 95%.

```bash
./inode_fillup.sh 96
```
Monitor the inode usage:

```bash
df -ih /tmp
```

Once the inode utilization exceeds 95%, the Warning alert should be generated.

***6. Trigger the Critical alert***

Increase the inode utilization to more than 97% to trigger the Critical alert.

```bash
./inode_fillup.sh 98
```
Verify the inode utilization:

```bash
df -ih /tmp
```
Once the inode utilization exceeds 97%, the Critical alert should be generated.

***7. Clean up the test files***

After completing the validation, remove the test directory and all generated files to release the consumed inodes.

```bash
rm -rf /tmp/inode_test
```

Verify that the inode utilization has returned to normal.

```bash
df -ih /tmp
```
The alert should automatically clear after the inode usage falls below the configured alert thresholds.
