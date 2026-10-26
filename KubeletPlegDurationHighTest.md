# Reproduce KubeletPlegDurationHigh Alert

Reproduce the KubeletPlegDurationHigh alert by temporarily suspending the CRI-O process and automatically resuming it after 20 minutes. This test demonstrates how an unresponsive container runtime can affect kubelet's Pod Lifecycle Event Generator (PLEG) and potentially trigger the alert.

---

## Procedure

***Step 1: Access the Target Node***

```bash
oc debug node/<NODENAME>
```

***Step 2: Suspend CRI-O and Schedule Automatic Recovery***

Run the following command to suspend the CRI-O process for 20 minutes and automatically resume it afterward:

```bash
nohup bash -c 'echo "$(date): Stopping CRI-O"; killall -STOP crio; sleep 1200; echo "$(date): Resuming CRI-O"; killall -CONT crio; echo "$(date): Test complete"' >/tmp/crio-test.log 2>&1 &
```

***Step 3: Monitor the Alert and Node Status***

```bash
oc get nodes
```

Sample output:

```bash
oc get nodes
NAME                            STATUS     ROLES                         AGE     VERSION
worker-cluster-9pfh5-1          NotReady   worker                        6d      v1.34.9
```

When CRI-O is unresponsive, kubelet may experience delays while communicating with the container runtime and relisting pods. This can increase PLEG relisting latency and potentially cause the KubeletPlegDurationHigh alert to fire.

***Step 4: Verify the alert in the OpenShift monitoring console***
`Observe > Alerting`

---

## Recovery

- Allow the 20-minute suspension period to complete. The background process should automatically resume CRI-O and nodes status would be `Ready` state.

```bash
oc get nodes
```
Sample output:

```bash
worker-cluster-9pfh5-1          Ready    worker                        6d      v1.34.9
```

The alert will recover once the node returns to the Ready state.
