# Reproduce `NoOvnClusterManagerLeader` Alert

## Overview
The runbook provides step-by-step instructions to simulate an OVN-Kubernetes control plane leader election failure in OpenShift, triggering the **NoOvnClusterManagerLeader** alert for testing and monitoring verification.

---

## Prerequisites

- `oc` CLI tool installed and logged in as a **cluster-admin**.
- Target Namespace: `openshift-ovn-kubernetes`

---

## Workflow

***Step 1: Record the Active Leader Pod***

Identify and record the current leader pod holding the lease before making any changes.

```bash
oc get lease ovn-kubernetes-master -n openshift-ovn-kubernetes
```


Sample Output:
```bash
NAME                    HOLDER                                   AGE
ovn-kubernetes-master   ovnkube-control-plane-dffd9c777-xlsjk   18h
```

***Step 2: Patch the Lease Object***

Overwrite the active lease holder with a non-existent pod name and extend the lease duration to prevent fast recovery.

```bash
oc patch lease ovn-kubernetes-master -n openshift-ovn-kubernetes --type=json -p '[
  {"op": "replace", "path": "/spec/holderIdentity", "value": "invalid-nonexistent-pod"},
  {"op": "replace", "path": "/spec/leaseDurationSeconds", "value": 3600}
]'
```

***Step 3: Freeze the Leader Process***
Send a SIGSTOP signal to the active leader pod container to prevent it from automatically reclaiming leadership.

Note: Replace ovnkube-control-plane-dffd9c777-xlsjk with your actual pod name from Step 1

```bash
oc exec -n openshift-ovn-kubernetes ovnkube-control-plane-dffd9c777-xlsjk -c ovnkube-cluster-manager -- kill -STOP 1
```

***Step 4: Verify Lease Manipulation***

Confirm that the lease holder is now set to the invalid identity.

```bash
oc get lease ovn-kubernetes-master -n openshift-ovn-kubernetes
```

Sample:
```bash
NAME                    HOLDER                    AGE
ovn-kubernetes-master   invalid-nonexistent-pod   18h
```

***Step 5: Track Alert Status - after 5 Min***

GUI: Observe > Alerts
![](assets/17876559441250.jpg)

OR

CLI: 
```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -- curl -s 'http://localhost:9090/api/v1/alerts' | jq '.data.alerts[] | select(.labels.alertname=="NoOvnClusterManagerLeader")'
```

Check state field in output, it should be "state": "firing",


---

## Cleanup:
Once testing is complete, restore normal cluster manager operations:

***Step 1. Unfreeze the leader process:***

```Bash
oc exec -n openshift-ovn-kubernetes <ovnkube-control-plane-*> -c ovnkube-cluster-manager -- kill -CONT 1
```

<ovnkube-control-plane-*> replace with podName.

***Step 2. Force lease re-election:***

```Bash
oc delete lease ovn-kubernetes-master -n openshift-ovn-kubernetes
```

***Step 3. Verify leader recovery:***

```Bash
oc get lease ovn-kubernetes-master -n openshift-ovn-kubernetes
```
