# GarbageCollectorSyncFailed

PrometheusRule Source: `kube-controller-manager-operator` · Alert Severity: `Warning` · Pending Period: `1h` · [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-kube-controller-manager-operator/GarbageCollectorSyncFailed.md)

---

## Meaning

The alert indicates that the Kubernetes Garbage Collector is experiencing errors while synchronizing and monitoring API resources. As a result, garbage collection may not function correctly, potentially leaving dependent or orphaned resources undeleted.

**Expression**

```promql
rate(garbagecollector_controller_resources_sync_error_total[5m]) > 0
```

This expression checks the rate of increase in Garbage Collector resource synchronization errors over the last 5 minutes. The alert triggers when the error counter increases at a rate greater than zero, indicating that synchronization errors are occurring.

## Impact:

- Kubernetes garbage collection may stop working correctly, while direct resource deletion can still succeed.

- Resources managed through owner references may not be automatically deleted, affecting cascading and orphan deletion.

- Orphaned or stale resources may accumulate, increasing resource consumption and etcd storage. Example: Deleting a ReplicaSet may leave its Pods running.

- If the issue persists, resource exhaustion, increased etcd usage, degraded performance, and potential cluster instability may occur.

---

## Diagnosis

| What to check | Command  | What to look for |
|---|---|---|
| Check `kube-controller-manager` pods | `oc get pods -n openshift-kube-controller-manager` | Verify that the `kube-controller-manager` pods are running. |
| Identify Garbage Collector synchronization errors | `POD=$(oc get pods -n openshift-kube-controller-manager -l app=kube-controller-manager -o name \| head -n 1)`<br><br>`oc logs -n openshift-kube-controller-manager "$POD" -c kube-controller-manager --since=30m \| grep -iE 'garbage controller monitor not yet synced\|syncing garbage collector\|conversion webhook\|failed to list\|failed to watch\|discovery.*error'` | Look for messages such as `unable to sync caches for garbage collector`, `timed out waiting for dependency graph builder sync during GC sync`, or `syncing garbage collector with updated resources from discovery`. |
| Increase KCM log level if the failing resource cannot be identified | `oc patch kubecontrollermanagers.operator/cluster --type=json -p='[{"op":"replace","path":"/spec/logLevel","value":"Debug"}]'` | Kube-controller-manager pods restart with increased logging. |


---

## Mitigation

- Correct or restore the conversion webhook configuration if it is unavailable or misconfigured.

- If the CRD is no longer required, follow the application cleanup procedure and remove the unused CRD.

- If network or connectivity errors are reported in the `kube-controller-manager` logs, investigate the affected Service, DNS, NetworkPolicy, TLS certificates, and underlying network infrastructure.


## Validation:

1. Verifies that the controller manager is no longer reporting Garbage Collector synchronization or conversion-related errors after recovery.

```bash
oc logs -n openshift-kube-controller-manager "$POD" -c kube-controller-manager --since=30m | grep -iE 'garbage controller monitor not yet synced|syncing garbage collector|conversion webhook|failed to list|failed to watch|discovery.*error'
```

2. Confirm that Garbage Collector synchronization has recovered by checking the prometheusmetric:

   `rate(garbagecollector_controller_resources_sync_error_total[5m])`returns `0`.
  
3. Verify the alert has been resolved from openshift console: `Observing > Alerting`

