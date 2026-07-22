# PodDisruptionBudgetAtLimit

PrometheusRule
[OpenShift Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-kube-controller-manager-operator/PodDisruptionBudgetAtLimit.md)
for: 1h

## Meaning
This alert indicates that the application has reached the minimum number of healthy pods required by its PodDisruptionBudget (PDB),so Kubernetes will not allow any additional voluntary pod evictions (for example, during a node drain or cluster upgrade) until another healthy pod becomes available.

Alert condition:
`max by (namespace, poddisruptionbudget) (kube_poddisruptionbudget_status_current_healthy == kube_poddisruptionbudget_status_desired_healthy and on (namespace, poddisruptionbudget) kube_poddisruptionbudget_status_expected_pods > 0)`

The alert is generated when the number of healthy pods equals the minimum number of healthy pods required by the PodDisruptionBudget (Current Healthy = Desired Healthy), leaving no additional pods available for voluntary disruption. The alert is evaluated only for PDBs that protect one or more pods (Expected Pods > 0).

## Impact

- OpenShift cannot safely drain the node hosting the pod. This stalls cluster updates, manual node drains, and autoscaling scale-downs.
- The application is running at its absolute minimum requirement. Any unexpected failure (like a node crash or OOM kill) will instantly cause application downtime.
- The application remains fully operational and continues to serve traffic normally for now, as the alert is a warning of reduced fault tolerance rather than an active crash.

## Diagnosis

**Step 1: Identify the affected PodDisruptionBudget**
List all PodDisruptionBudgets in the affected namespace.

`oc get pdb -n <namespace>`

Verify:
Identify the PDB associated with the alert.
Check whether ALLOWED DISRUPTIONS is 0.

**Step 2: Review the PodDisruptionBudget details**

Display detailed information about the PDB.

`oc describe pdb <pdb-name> -n <namespace>`

Verify:<br>
`Current`  --- Number of healthy pods that are currently Running and Ready. <br>
`Desired`  --- Minimum number of healthy pods that must remain available <br>
`Allowed Disruptions` --- Number of pods that can be voluntarily disrupted <br>

This helps determine whether the PDB has exhausted its disruption budget.

**Step 3: Verify the health of the application pods**

Check the status of the pods protected by the PDB.

`oc get pods -n <namespace> -o wide`

Verify:

All pods are in the Running state.
All pods are Ready (READY = 1/1 or the expected value).

If any pod is in Pending, CrashLoopBackOff, ImagePullBackOff, or NotReady, investigate and resolve the pod issue before proceeding.

**Step 4: Verify the Deployment configuration**

Check the Deployment managing the application.

`oc get deployment <deployment-name> -n <namespace>`

Verify:

The configured number of replicas.
Compare the replica count with the PDB configuration (minAvailable or maxUnavailable).


## Remediation

| Scenario | How to Identify | Remediation | Action |
|----------|-----------------|-------------|------------|
| Single replica deployment	| Deployment has replicas=1 and the PDB has minAvailable=1 or maxUnavailable=0. AllowedDisruptions=0.| Increase the number of replicas so Kubernetes can safely evict one pod during maintenance oc scale deployment <deployment-name> --replicas=2 -n <namespace>	 | Requires application owner approval| 
| Pods are not Ready | Pods are in CrashLoopBackOff, Pending, ImagePullBackOff, or NotReady. | Restore pod health by investigating logs, events, scheduling issues, image pulls, or resource constraints. Once the pods become Ready, the PDB status will recover automatically.| Investigate pod issue|
| PDB configuration is too restrictive | minAvailable equals the number of replicas, or maxUnavailable=0, preventing voluntary disruptions| Review the PDB with the application owner. If the application can tolerate fewer healthy replicas, decrease minAvailable or increase maxUnavailable.<br>oc patch pdb <pdb-name> -p '{"spec":{"minAvailable":<count>}}' |Manual approval required|
| Alert occurs during cluster upgrade or node maintenance | Alert appears while upgrading the cluster or draining nodes.| If multiple nodes are being updated simultaneously, reduce the MachineConfigPool maxUnavailable to decrease the number of nodes being drained at the same time.<br>oc patch mcp <mcp-name> --type merge --patch '{"spec":{"maxUnavailable":<count>}}' | Cluster administrator approval required | 



