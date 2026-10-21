# NorthboundStale

PrometheusRule Source: `networking-rules` · Alert Severity: `Warning` · Pending Period: `10m` · [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-network-operator/NorthboundStaleAlert.md)

---

## Meaning 

This alert indicates that the OVN-Kubernetes controller has not successfully synchronized network changes with the OVN Northbound Database for more than 120 seconds. 
This may indicate an issue with the controller or database, potentially affecting the networking control plane on the affected node.

***Expression***

```bash
time() - max_over_time(ovnkube_controller_nb_e2e_timestamp[5m]) > 120
```

---

## Impact

- Existing workloads may continue to have network connectivity.
- Newly created workloads may experience networking issues or fail to start properly.
- Network policy changes may not be applied to existing workloads.
- The impact is generally limited to the affected node.

---

## Diagnosis

| Command | What to look for | Action |
|---|---|---|
| `oc get pod -n openshift-ovn-kubernetes -l app=ovnkube-node -o wide` | Identify `ovnkube-node` pods and the node on which they run | Locate the pod running on the node where the alert fired. |
| `oc describe pod/<podname> -n openshift-ovn-kubernetes` | Check pod status, container states, restarts, events, and errors. | Investigate failed, terminated, or repeatedly restarting containers. |
| `oc logs <podname> -n openshift-ovn-kubernetes -c ovnkube-controller` | Look for errors related to northbound database synchronization or connectivity. | Investigate controller errors and database connection failures. |
| `oc logs <podname> -n openshift-ovn-kubernetes -c nbdb` | Look for database errors, leader election issues, and Raft communication problems. | Investigate database health and leader election failures. |
| `oc logs <podname> -n openshift-ovn-kubernetes -c <container> --previous` | Check logs from the previous container instance for errors starting with `E`. | Identify the cause of a previous container crash or restart. |


---

## Mitigation

- As a general fix Restart the pods hosting the failing components to reset state and restore broken connections.

- If pod restarts don't work and you can't easily fix the issue from the logs, contact Red Hat Support.

---

## Validation

1) Check that all pods in the openshift-ovn-kubernetes namespace are running and ready.

```bash
oc get pods -n openshift-ovn-kubernetes -o wide
```

2) Review recent logs for Northbound database connectivity errors, synchronization failures, and recovery messages.

```bash
oc logs <podname> -n openshift-ovn-kubernetes -c <container> --since=10m --timestamps
```
Check the relevant containers, such as `ovnkube-controller` and `nbdb`, and confirm that the errors have stopped and normal operations have resumed.

3) Verify Alert Recovery from prometheus console - `Observe > Alerting`.
