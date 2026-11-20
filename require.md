
1. Dump the libvirt domain XML

oc exec -n namespace virt-launcher-pod-name -c compute -- virsh dumpxml namespace_vm-name | xmllint --xpath '//devices/disk' -

```bash
    <disk type='block' device='disk' model='virtio-non-transitional'>
      <driver name='qemu' type='raw' cache='none' error_policy='stop' io='native' discard='unmap'/>
      <source dev='/dev/rootdisk' index='2'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <alias name='ua-rootdisk'/>
      <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
    </disk>
    <disk type='file' device='disk' model='virtio-non-transitional'>
      <driver name='qemu' type='raw' cache='none' error_policy='stop' discard='unmap'/>
      <source file='/var/run/kubevirt-ephemeral-disks/cloud-init-data/default/linux-test/noCloud.iso' index='1'/>
      <backingStore/>
      <target dev='vdb' bus='virtio'/>
      <alias name='ua-cloudinitdisk'/>
      <address type='pci' domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
    </disk>
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' error_policy='stop' io='native' discard='unmap'/>
      <source dev='/var/run/kubevirt/hotplug-disks/opt-disk' index='3'/>
      <backingStore/>
      <target dev='sda' bus='scsi'/>
      <alias name='ua-opt-disk'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```

2. Get the minor and major number from the disks

oc exec -n namespace virt-launcher-pod -c compute -- stat -c "%n -> Major:Minor = %t:%T (hex)" disk-path-1 disk-path-2

```bash
oc exec -n default virt-launcher-linux-test-grktk -c compute -- stat -c "%n -> Major:Minor = %t:%T (hex) | %Hr:%Lr (dec)" /dev/rootdisk /var/run/kubevirt/hotplug-disks/opt-disk
/dev/rootdisk -> Major:Minor = fb:50 (hex) | ?r:?r (dec)
/var/run/kubevirt/hotplug-disks/opt-disk -> Major:Minor = fb:70 (hex) | ?r:?r (dec)
```

convert hex to dec

3. get the values.
oc debug node/<nodeName> -- chroot /host grep -H . /sys/dev/block/251:80/queue/max_hw_sectors_kb /sys/dev/block/251:80/queue/max_sectors_kb /sys/dev/block/251:112/queue/max_hw_sectors_kb /sys/dev/block/251:112/queue/max_sectors_kb

```bash
oc debug node/worker-cluster-nxkgm-1  -- chroot /host grep -H . /sys/dev/block/251:80/queue/max_hw_sectors_kb /sys/dev/block/251:80/queue/max_sectors_kb /sys/dev/block/251:112/queue/max_hw_sectors_kb /sys/dev/block/251:112/queue/max_sectors_kb
Starting pod/worker-cluster-nxkgm-1-debug-gnsq5 ...
To use host binaries, run `chroot /host`. Instead, if you need to access host namespaces, run `nsenter -a -t 1`.
/sys/dev/block/251:80/queue/max_hw_sectors_kb:4096
/sys/dev/block/251:80/queue/max_sectors_kb:4096
/sys/dev/block/251:112/queue/max_hw_sectors_kb:4096
/sys/dev/block/251:112/queue/max_sectors_kb:4096
```
