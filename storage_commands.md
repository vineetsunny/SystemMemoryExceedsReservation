# MultipathDeviceSuspended alert

The Linux operating system has temporarily stopped all input/output (I/O) operations to a particular storage device.
multipath device called mpatha.
The MultipathDeviceSuspended alert is intended to trigger when the Device Mapper multipath device becomes suspended.



### How to test this alert in a lab
On a disposable test multipath device, with no active workloads or mounted filesystem:


- Get the mpatha name:

multipath -ll
dmsetup info -c -o name,major,minor,uuid

- Suspend the mpatha device:

dmsetup suspend <device-name>

- verify:

dmsetup info -c -o name,suspended | grep <last 4 digit>


Expected:
Name     Suspended
mpatha   1


The Prometheus expression should evaluate to true if the exporter reports 0 for suspended devices and 1 for active devices, as assumed by your alert definition.

### Restoration :

After validation, restore immediately:

dmsetup resume <device-name>

verify - dmsetup info -c -o name,suspended

Expected:
Name     Suspended
mpatha   0

---

# MultipathPathCritical

### How to test this alert in a lab

- 1. Take WWID of one of the Device mapper device:

multipath -ll 3624a9370e6fd19e77b5e4eec02b0e519

- 2. check the mapping:

ls -l /sys/block/dm-<>/slaves/


- 3. fail one path 

multipathd -k 'fail path sdcae'


- 4. check the status again

multipath -ll 3624a9370e6fd19e77b5e4eec02b0e519

Note - path should be down.

- verify metric

node_dmmultipath_device_paths{device="dm-194"}

### Restoration

- Up the path

multipathd -k 'reinstate path sdcae'

- check paths health
multipath -ll 3624a9370e6fd19e77b5e4eec02b0e519


