# Reproduce HAControlPlaneDown Alert


Note: Before starting the reproduction, ensure you have SSH access and node management/console access to the target control-plane node. These are required to recover the node if kubelet/CRI-O becomes unresponsive or the node becomes unavailable.

Here we have two methods to reproduce the alert.


## Method A — Stop the Kubelet or CRI-O Service***

This is the simplest method and directly simulates a control-plane node becoming unhealthy.

1. Access the target control-plane node

Run:
```bash
oc debug node/<nodeName>
chroot /host
```

2. Stop the kubelet or CRI-O service

```bash
systemctl stop kubelet
```

Alternatively, to simulate a CRI-O failure:

```bash
systemctl stop crio
```

After the service is stopped, monitor the node:
```bash
oc get nodes
```

The target node should eventually transition to `NotReady`, and the relevant control-plane health conditions may cause the HAControlPlaneDown alert to fire.

---

***Recovery:***

Use SSH or the node management console to access the node.

For kubelet:
```bash
systemctl start kubelet
```
For CRI-O:

```bash
systemctl start crio
```

Alternatively, reboot the node through the management console if the node cannot be recovered normally.

Then verify:
```bash
oc get nodes
```

---

## Method B — Create Severe Memory Pressure

This method simulates a control-plane node becoming unhealthy due to severe memory pressure.

Note : Run this only in a test/non-production environment. Allocate memory gradually and monitor the node continuously. Do not blindly use a large value such as 50 GiB; the required amount depends on the node's available memory.

1. Run the following script inside one of the target control plane node.

```bash
oc debug node/<nodeName>
chroot /host
```
2. Create the script:

```bash
#!/bin/bash

# Usage:
#   ./memory-pressure.sh 40
#   ./memory-pressure.sh 60

MEM_GIB=${1:-4}

echo "Starting memory pressure: ${MEM_GIB} GiB"

python3 - "$MEM_GIB" <<'PY'
import sys
import time

gib = int(sys.argv[1])
size = gib * 1024 * 1024 * 1024

print(f"Allocating {gib} GiB of memory...")

# Allocate memory
data = bytearray(size)

# Touch every page so the kernel actually backs it with RAM
for i in range(0, size, 4096):
    data[i] = 1

print(f"Successfully allocated {gib} GiB.")
print("Memory remains allocated.")
print("Press Ctrl+C to release it.")

try:
    while True:
        time.sleep(10)
except KeyboardInterrupt:
    print("\nReleasing memory...")
    del data
    print("Memory released.")
PY
```
Make it executable:
```bash
chmod +x /tmp/memory-pressure.sh
```

2. Increase the utilization until it reach 99-100%

```bash
/tmp/memory-pressure.sh 40
```
Increase the value i.e 40 based on the node's available memory and the behavior you are trying to reproduce.

3. Monitor the node how much memory is utilized:
 
oc adm top nodes
example output:
```bash
worker-cluster-l6qh2-3          5628m        36%         59634Mi         99%  
```

If utilization is less, then increase the script value and monitor again.

4. Increase the impact on the kubelet cgroup
Once severe memory pressure has been established, open another terminal and access the node:

```bash
oc debug node/<NodeName>
chroot /host
```

```bash
bash -c "echo \$\$ > /sys/fs/cgroup/system.slice/kubelet.service/cgroup.procs && python3 -c 'a = bytearray(4 * 1024 * 1024 * 1024)'"
```
---

***Recovery***

After the alert has been reproduced, stop the memory-pressure processes before attempting normal cluster recovery.

1. SSH to the affected node and find the memory-pressure process

```bash
ps -ef | grep memory-pressure
```
OR

You can also search for the Python process:
```bash
ps -ef | grep python3
```
2. Stop the memory-pressure script

Terminate the identified process:

```bash
kill -9 <PID>
```

3. Stop the additional bytearray test process

Find it:
```bash
ps -ef | grep bytearray
```
Then terminate the identified process:

```bash
kill -9 <PID>
```
4. Verify memory recovery

```bash
oc adm top nodes
```

5. Verify kubelet and node status

On the node:

```bash
systemctl status kubelet
```
If necessary:

```bash
systemctl start kubelet
```

Then from the OpenShift cluster:

```bash
oc get nodes
```
Wait until the affected control-plane node returns to `Ready` state.

Confirm that the control-plane operators recover and that the HAControlPlaneDown alert clears.

`Recommended approach`: Use Method A when you simply need to reproduce the alert. Use Method B when you specifically need to test behavior under severe memory pressure and node recovery.
