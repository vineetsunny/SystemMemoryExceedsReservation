# Reproduce VirtLauncherPodsStuckFailed Alert

This alert fires when at least 200 virt-launcher Pods remain in the Failed phase for 10 minutes.
The alert is designed to detect a situation where 200 or more virt-launcher Pods remain in the Failed state for at least 10 minutes. This can indicate a large-scale VM workload failure or an underlying infrastructure/KubeVirt issue that is preventing VM workloads from recovering.

---

***1. Create a project for testing:***

```bash
oc new-project virt-launcher-vm-test
```
***2. Provision 200 VMs with minimal configuration:***

Provision the VMs based on the resources and storage available in the environment.

Two methods are provided below:

Method A: Use a containerDisk image.
Method B: Use a DataVolume with RWX storage.

Important: In a disconnected environment, replace the external quay.io image with a VM disk image available in the disconnected/internal registry.

***Method A)*** VM with containerDisk image.

```bash
for start in $(seq 1 50 200); do
  end=$((start + 49))
  echo "Creating VMs $start to $end..."
  for i in $(seq "$start" "$end"); do
    cat <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: migration-test-$i
  namespace: virt-launcher-vm-test
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/domain: migration-test-$i
    spec:
      domain:
        resources:
          requests:
            memory: 64Mi
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo:latest
---
EOF
  done | oc create -f -

  if [ "$end" -lt 200 ]; then
    echo "Waiting 3 minutes before next batch..."
    sleep 180
  fi
done
```

OR 

***Method B)*** VM With RWX storage mode:

```bash
for start in $(seq 1 10 200); do
  end=$((start + 9))
  echo "Creating VMs $start to $end..."
  for i in $(seq "$start" "$end"); do
    cat <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: migration-test-$i
  namespace: virt-launcher-vm-test
spec:
  runStrategy: Always
  dataVolumeTemplates:
  - metadata:
      name: datavolume-migration-test-$i
    spec:
      storage:
        accessModes:
        - ReadWriteMany
        volumeMode: Block
        resources:
          requests:
            storage: 1Gi
        storageClassName: ocs-external-storagecluster-ceph-rbd
      source:
        registry:
          url: docker://quay.io/kubevirt/cirros-container-disk-demo:latest
  template:
    metadata:
      labels:
        kubevirt.io/domain: migration-test-$i
    spec:
      domain:
        resources:
          requests:
            memory: 64Mi
        devices:
          disks:
          - name: disk0
            disk:
              bus: virtio
      volumes:
      - name: disk0
        dataVolume:
          name: datavolume-migration-test-$i
EOF
  done | oc create -f -

  if [ "$end" -lt 200 ]; then
    echo "Waiting 1 minute before next batch..."
    sleep 60
  fi
done
```

`Note`: Change the image and storageClassName according to the environment. 

***3. Prevent KubeVirt components from immediately recreating failed workloads***

Before scaling, record the original replica counts and then scale to 0.

Verify:
```bash
oc get pods -n openshift-cnv
```

Scale KubeVirt deployment replicas to 0: 

```bash
oc scale deployment/hco-operator -n openshift-cnv --replicas=0
oc scale deployment/virt-operator -n openshift-cnv --replicas=0
oc scale deployment/virt-controller -n openshift-cnv --replicas=0
```



***4. Simulate the failure**

Two failure methods can be used. Choose one.

***Method A)*** Killing the QEMU process inside every virt-launcher Pod: 

```bash
for POD in $(oc get pods -n virt-launcher-vm-test \
  -l kubevirt.io=virt-launcher \
  --field-selector=status.phase=Running \
  -o jsonpath='{.items[*].metadata.name}'); do

  PID=$(oc exec -n virt-launcher-vm-test "$POD" -c compute -- \
    pgrep -x virt-launcher 2>/dev/null)

  echo "Killing virt-launcher PID $PID in $POD"

  if [ -n "$PID" ]; then
    oc exec -n virt-launcher-vm-test "$POD" -c compute -- \
      kill -9 "$PID"
  fi

done
```

***OR***

***Method B)*** Rebooting the nodes (worker) where VMs are running. Make sure VMs are not running on Master nodes.

```bash
oc debug node/<nodeName> -- chroot /host reboot
```

Change the <nodeName> with worker's node name.

***5. Validate if all VMIs in the failed state:***

Using CLI:

```bash
oc get vmi -A | grep -i failed | wc
```

Using GUI:
```bash
count by (namespace) (kube_pod_status_phase{phase="Failed", pod=~"virt-launcher-.*"} == 1)
```

***6. Wait for 10 min, and check the alert***

login to Openshift Console > Observe > Alerting

---

***To clean the setup:***

a.) Restore KubeVirt components
```bash
oc scale deployment/hco-operator -n openshift-cnv --replicas=1
oc scale deployment/virt-operator -n openshift-cnv --replicas=1
oc scale deployment/virt-controller -n openshift-cnv --replicas=1

```

b.) Delete the test VMs

```bash
oc delete vm --all -n virt-launcher-vm-test
```

c.) Verify the test namespace is clean
oc get vm,vmi,pods,dv,pvc -n virt-launcher-vm-test

Finally, verify that KubeVirt is healthy:
```bash
oc get co
oc get pods -n openshift-cnv
```

