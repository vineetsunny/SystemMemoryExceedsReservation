SystemMemoryExceedsReservation

Runbook · Tier 0 · warning · System memory usage ≥ 95% of reserved memory

Meaning

The node's system processes (such as kubelet, CRI-O, systemd, NetworkManager, sshd, etc.) are consuming more than 95% of the reserved system memory.

This does not necessarily mean the node is out of memory. It indicates that the reserved memory for critical node services is nearly exhausted, increasing the risk that these services may compete with application pods for memory.

Impact
Increased risk of node instability.
Critical system services (kubelet, CRI-O) may become memory constrained.
Increased probability of Out-Of-Memory (OOM) events affecting node services.
Node may eventually become NotReady.
Scheduler may overestimate available resources if reservation is too small for the workload.


Diagnosis
Variables (from alert)
NODE=<labels.node>



Diagnosis	What to look for	Cause	Action
oc debug node/$NODE -- cat /host/etc/kubernetes/kubelet.conf | grep -A5 systemReserved	memory reservation too small (for example 1Gi on a busy node)	LOW_RESERVATION	Review reservation size
oc get --raw /api/v1/nodes/$NODE/proxy/stats/summary	kubelet/runtime memory close to reservation	EXPECTED_SYSTEM_USAGE	Increase reservation if workload is expected
oc adm top node $NODE	Node memory utilization >90%	NODE_MEMORY_PRESSURE	Investigate overall node memory usage
oc describe node $NODE	MemoryPressure=True	NODE_MEMORY_PRESSURE	Platform SRE investigation
oc get pods -A --field-selector spec.nodeName=$NODE --no-headers | wc -l	Very high pod count	HIGH_POD_DENSITY	Increase reservation or reduce pod density
oc get events --field-selector involvedObject.kind=Node,involvedObject.name=$NODE --sort-by='.lastTimestamp' | tail -20	OOMKilled, Eviction events	NODE_RESOURCE_PRESSURE	Platform SRE investigation
oc get --raw /api/v1/nodes/$NODE/proxy/stats/summary	kubelet memory unusually high compared to normal baseline	KUBELET_HIGH_USAGE	Investigate kubelet activity (pod churn, image pulls, etc.)
oc get --raw /api/v1/nodes/$NODE/proxy/stats/summary	runtime (CRI-O) memory unusually high	CRIO_HIGH_USAGE	Investigate container runtime
—	No obvious issue found	UNKNOWN	Platform SRE investigation
Useful Commands
Check configured reservation
oc debug node/$NODE -- cat /host/etc/kubernetes/kubelet.conf

Look for

systemReserved:
  cpu: 500m
  memory: 1Gi
Check actual system memory usage
oc get --raw /api/v1/nodes/$NODE/proxy/stats/summary

Review

kubelet
runtime (CRI-O)
other system containers
Check node conditions
oc describe node $NODE

Look for

MemoryPressure=True
Check pod density
oc get pods -A --field-selector spec.nodeName=$NODE --no-headers | wc -l
Check node utilization
oc adm top node $NODE
Check recent node events
oc get events \
--field-selector involvedObject.kind=Node,involvedObject.name=$NODE \
--sort-by='.lastTimestamp'
Mitigation
LOW_RESERVATION

If the node is consistently using more than 95% of the reserved memory:

Increase systemReserved.memory using a KubeletConfig.
Follow the Red Hat sizing guidance or enable automatic reservation (OCP 4.8+).
HIGH_POD_DENSITY

If the node hosts a very large number of pods:

Reduce pod density (if possible), or
Increase systemReserved.memory.
NODE_MEMORY_PRESSURE

If the node itself is under memory pressure:

Investigate workloads consuming excessive memory.
Review recent deployments.
Check for memory leaks.
Consider redistributing workloads.
EXPECTED_SYSTEM_USAGE

If kubelet/CRI-O legitimately require more memory (large clusters, frequent pod churn, heavy image pulls):

Increase the reservation.
Monitor after the change.
UNKNOWN

Escalate to Platform SRE for further investigation.

Cause → Notify
Cause	Notify
LOW_RESERVATION	Platform SRE
HIGH_POD_DENSITY	Platform SRE
NODE_MEMORY_PRESSURE	Platform SRE
KUBELET_HIGH_USAGE	Platform SRE
CRIO_HIGH_USAGE	Platform SRE
EXPECTED_SYSTEM_USAGE	Platform SRE
NODE_RESOURCE_PRESSURE	Platform SRE
UNKNOWN	Platform SRE
Notes
This alert is an early warning, not necessarily an outage.
It monitors system.slice memory usage (system processes), not pod memory usage.
Increasing systemReserved.memory does not add more RAM to the node; it reserves a larger portion of existing memory for critical node services, reducing the memory available for scheduling application pods. This helps prevent kubelet and other system services from being starved under heavy load.
