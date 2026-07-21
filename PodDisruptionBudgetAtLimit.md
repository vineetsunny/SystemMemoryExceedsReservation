# PodDisruptionBudgetAtLimit

PrometheusRule
[OpenShift Runbook](https://github.com/openshift/runbooks/blob/master/alerts/cluster-kube-controller-manager-operator/PodDisruptionBudgetAtLimit.md)
for: 60m

## Meaning
This alert indicates that the application has reached the minimum number of healthy pods required by its PodDisruptionBudget (PDB),so Kubernetes will not allow any additional voluntary pod evictions (for example, during a node drain or cluster upgrade) until another healthy pod becomes available.


## Impact

- OpenShift cannot safely drain the node hosting the pod. This stalls cluster updates, manual node drains, and autoscaling scale-downs.
- The application is running at its absolute minimum requirement. Any unexpected failure (like a node crash or OOM kill) will instantly cause application downtime.
- The application remains fully operational and continues to serve traffic normally for now, as the alert is a warning of reduced fault tolerance rather than an active crash.

## Diagnosis

| Action | Diagnosis | What to look for | 
|--------|-----------|------------------|
| Validate the affected PodDisruptionBudget | oc get pdb -n <ns>	| Find the PDB where ALLOWED DISRUPTIONS = 0 | 
| View detailed PDB information |	oc describe pdb <pdb-name> -n <namespace>	| Check Current Healthy, Desired Healthy, and Allowed Disruptions.|
| Verify the application pods protected by the PDB | oc get pods -n <namespace> -o wide | Confirm whether all pods are Running and Ready | 
| Check the Deployment replica count | oc get deployment <deployment-name> -n <namespace> | Compare the configured replicas with the PDB requirements (minAvailable or maxUnavailable).|
| Determine whether the alert is expected | Compare the Deployment and PDB configuration | If Allowed Disruptions = 0 because the application intentionally requires all replicas to remain available, no immediate action is required. Otherwise, review scaling or PDB configuration before maintenance activities.|


## Remediation

| Scenario | How to Identify | Remediation | Action |
|----------|-----------------|-------------|------------|
| Single replica deployment	| Deployment has replicas=1 and the PDB has minAvailable=1 or maxUnavailable=0. AllowedDisruptions=0.| Increase the number of replicas so Kubernetes can safely evict one pod during maintenance oc scale deployment <deployment-name> --replicas=2 -n <namespace>	 | Requires application owner approval| 
| Pods are not Ready | Pods are in CrashLoopBackOff, Pending, ImagePullBackOff, or NotReady. | Restore pod health by investigating logs, events, scheduling issues, image pulls, or resource constraints. Once the pods become Ready, the PDB status will recover automatically.| Investigate pod issue|
| PDB configuration is too restrictive | minAvailable equals the number of replicas, or maxUnavailable=0, preventing voluntary disruptions.
Review the PDB with the application owner. If the application can tolerate fewer healthy replicas, decrease minAvailable or increase maxUnavailable.<br>oc patch pdb <pdb-name> -p '{"spec":{"minAvailable":<count>}}' | Manual approval required |
| Alert occurs during cluster upgrade or node maintenance | Alert appears while upgrading the cluster or draining nodes.| If multiple nodes are being updated simultaneously, reduce the MachineConfigPool maxUnavailable to decrease the number of nodes being drained at the same time.<br>oc patch mcp <mcp-name> --type merge --patch '{"spec":{"maxUnavailable":<count>}}' | Cluster administrator approval required | 



