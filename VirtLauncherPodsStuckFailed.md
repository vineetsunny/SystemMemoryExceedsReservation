# VirtLauncherPodsStuckFailed

PrometheusRule Source: `prometheus-kubevirt-rules` · Alert Severity: `Critical` · Pending Period: `10m` · [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/openshift-virtualization-operator/VirtLauncherPodsStuckFailed.md)

---

## Meaning

The VirtLauncherPodsStuckFailed alert indicates that a large number of virt-launcher Pods, which host KubeVirt virtual machine workloads, have entered the Failed phase.

The alert is triggered when the number of failed virt-launcher Pods reaches the configured threshold of 200 or more and remains at that level for 10 minutes.

***Expression:***

```bash
sum(kube_pod_status_phase{phase="Failed",pod=~"virt-launcher-.*"}) >= 200
```
---

## Impact

- VM availability: Affected VMs may become unavailable or fail to start/recover successfully.
- API server and etcd pressure: A large number of failed or repeatedly recreated Pods can increase API object processing, list/watch activity, and API latency.
- Controller and scheduler load: Large numbers of VM-related Pod failures and reconciliation events can increase the workload on Kubernetes and KubeVirt controllers and the scheduler.
- Monitoring load: A high volume of VM/Pod state changes can increase metric cardinality and processing load for components such as kube-state-metrics and Prometheus.
- Operational churn: Repeated VM or Pod failures may result in continuous Pod recreation, migration attempts, CNI operations, and storage attach/detach activity.
---

## Diagnosis

| Check | Command | What to look for | Action |
|---|---|---|---|
| **Determine number of failed virt-launcher Pods** | `count by (namespace) (kube_pod_status_phase{phase="Failed", pod=~"virt-launcher-.*"} == 1)` | Number of failed `virt-launcher` Pods per namespace. A sudden increase or large number of failures indicates a widespread issue. | If failures are isolated, continue with Pod/VMI-level investigation. If failures are widespread, proceed with node, KubeVirt, network, and storage checks. |
| **Identify affected nodes** | `count by (node) ((kube_pod_status_phase{phase="Failed", pod=~"virt-launcher-.*"} == 1) * on(pod) group_left(node) kube_pod_info{pod=~"virt-launcher-.*", node!=""})` | Nodes hosting the failed `virt-launcher` Pods and the number of failures on each node. | If failures are concentrated on one or a few nodes, investigate those nodes for node, network, storage, or runtime issues. If failures are distributed across many nodes, investigate cluster-wide components. |
| **List failed virt-launcher Pods** | `oc get pods -A -l kubevirt.io=virt-launcher --field-selector=status.phase=Failed --no-headers` | Namespace, Pod name, and failed Pods. Correlate failed Pods with affected VMs and nodes. | Select representative failed Pods and inspect their events/logs. Determine whether failures have a common cause. |
| **Review KubeVirt controller logs** | `NAMESPACE="$(oc get kubevirt -A -o jsonpath='{.items[0].metadata.namespace}')"`<br>`oc -n "$NAMESPACE" logs -l kubevirt.io=virt-controller --tail=200` | Errors related to VM lifecycle, scheduling, reconciliation, migrations, API communication, or failed VMI operations. | If controller errors correlate with the failures, investigate the reported resource or component. |
| **Review KubeVirt handler logs** | `oc -n "$NAMESPACE" logs -l kubevirt.io=virt-handler --tail=200` | Errors related to VMI lifecycle, Pod management, node communication, libvirt, launcher Pods, or migration. | If errors are concentrated on specific nodes, investigate the corresponding nodes and `virt-handler` instances. |
| **Check for migration storms** | `oc get vmim -A` | Large number of active, pending, or failed VM migrations, particularly around the time of the failures. | If a migration storm is present, identify the source (descheduler, node disruption, manual migration, etc.). Reduce unnecessary migrations and investigate the trigger. |
| **Check image-pull failures** | `oc get events -A --sort-by='.lastTimestamp' \| grep -Ei 'Failed to pull\|ErrImagePull\|ImagePullBackOff\|unauthorized\|pull image'` | `ImagePullBackOff`, `ErrImagePull`, authentication/authorization failures, registry connection failures, or image-not-found errors. | Verify registry reachability, image existence, pull secrets/credentials, and registry health. Correct the underlying issue and retry affected workloads if required. |
| **Check network/CNI failures** | `oc get events -A --sort-by='.lastTimestamp' \| grep -Ei 'CNI\|FailedCreatePodSandBox\|network plugin\|network.*fail\|timeout'` | `FailedCreatePodSandBox`, CNI errors, network plugin failures, or timeouts. | Identify affected nodes and investigate OVN/CNI health and node connectivity. |
| **Check OpenShift network components** | `oc get pods -n openshift-ovn-kubernetes` | OVN-Kubernetes Pods that are not `Running/Ready`, restarting frequently, or otherwise unhealthy. | Identify affected nodes and investigate the corresponding OVN-Kubernetes components. |
| **Review OVN-Kubernetes logs** | `oc logs -n openshift-ovn-kubernetes -l app=ovnkube-node --tail=200` | CNI, OVN, network programming, connectivity, timeout, or reconciliation errors. | Correlate timestamps with failed `virt-launcher` Pods. If errors are node-specific, investigate those nodes; if cluster-wide, investigate OVN control-plane/network health. |
| **Check storage/CSI events** | `oc get events -A --sort-by='.lastTimestamp' \| grep -Ei 'mount\|attach\|volume\|failed.*volume\|FailedMount\|FailedAttach'` | `FailedMount`, `FailedAttachVolume`, volume provisioning, attachment, or filesystem errors. | Identify the affected PVC/PV and node. Verify storage backend and CSI health before restarting or recreating workloads. |
| **Review CSI/storage logs** | `oc get pods -A \| grep -Ei 'csi\|storage'`<br><br>Then inspect the relevant CSI Pod:<br>`oc logs -n <csi-namespace> <csi-pod> --tail=200` | Volume attach/mount failures, backend connectivity errors, timeouts, authentication issues, or CSI driver errors. | Correlate CSI errors with the failed VM/Pod and affected node. Engage the storage/backend team if the CSI layer is healthy but the backend is failing. |


---

## Mitigation:


***1. Reduce the impact, if required***

If active VM migrations are confirmed to be contributing to the failures, identify and cancel the affected migrations:

```bash
oc get vmim -A
oc delete vmim <migration-name> -n <namespace>
```

***2. Clean up failed virt-launcher Pods***

```bash
oc get pods -A \
  -l kubevirt.io=virt-launcher \
  --field-selector=status.phase=Failed \
  -o name | xargs -r -n50 oc delete
```
Avoid deleting Pods that are still required for active troubleshooting or recovery.

***3. Resolve the underlying cause***
`Image issues`: Resolve registry connectivity, authentication, or image/tag issues, then restart or recover the affected workloads.
`Network/CNI issues`: Resolve the identified CNI or network errors and verify that newly created virt-launcher Pods start successfully.
`Storage issues`: Resolve volume attach/mount or CSI-related failures and verify the health of the affected PVCs/DataVolumes.
`Node issues`: If failures are isolated to specific nodes, investigate and remediate the node-level issue before recovering the affected VMs.
`OpenShift Virtualization issues`: If a product regression is suspected, follow the applicable Red Hat guidance to upgrade, or move to a known-good version.

***4. Verify recovery***

Confirm that the number of failed virt-launcher Pods has fallen below the alert threshold:
```bash
sum(kube_pod_status_phase{phase="Failed",pod=~"virt-launcher-.*"} == 1)
```
Verify that:
The number of failed virt-launcher Pods remains below the alert threshold.
The VirtLauncherPodsStuckFailed alert has cleared.
If the output returns `None`, it indicates that no Pods are currently in the Failed phase.

