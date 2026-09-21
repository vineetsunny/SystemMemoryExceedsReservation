# Reproduce HCOOperatorConditionsUnhealthy Alert

Procedure to test and validate the HCOOperatorConditionsUnhealthy alert by breaking a deployment's scheduling criteria, observing the operator's degraded state, and subsequently restoring cluster health.

The deployments listed below are managed by the HyperConverged Operator (HCO) and can be used to reproduce this alert. For this test, we will use `cdi-deployment`

- virt-exportproxy
- virt-controller 
- virt-api
- cdi-apiserver 
- cdi-deployment
- kubemacpool-cert-manager
- kubemacpool-mac-controller-manager
- kubevirt-ipam-controller-manager
- kubevirt-migration-controller



1. Check the health of the deployment's pod:

```bash
oc get pods -n openshift-cnv -l cdi.kubevirt.io=cdi-deployment -o wide
```
Also, check the HCO health:

```bash
oc get hyperconverged kubevirt-hyperconverged -n openshift-cnv -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}' | column -t
```

2. Apply Invalid Node Selector Patch:

```bash
oc patch deployment cdi-deployment -n openshift-cnv -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-existent-node":"true"}}}}}'
```
The deployment was patched to force an invalid node requirement, preventing pods from scheduling.

3. Validate if change is applied or not

```bash
oc get deployment cdi-deployment -n openshift-cnv -o jsonpath='{.spec.template.spec.nodeSelector}'
```
Pod's status:
```bash
oc get pods -n openshift-cnv -l cdi.kubevirt.io=cdi-deployment -o wide
```

Due to the deployment's rolling update strategy, a new pod is created while the old pod continues running. Because the new pod cannot start, the update remains incomplete, which eventually triggers the alert.


4. Check the HCO health in degraded state:

```bash
oc get hyperconverged kubevirt-hyperconverged -n openshift-cnv -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}' | column -t
```
HCO health status show- `Available: False`

5. Check if the the alert is triggered, using OCP console:
Observe > Alerting 

---

## Recovery: 

1. Remove the Node-Selector Patch
```bash
oc patch deployment cdi-deployment -n openshift-cnv --type=json -p='[{"op": "remove", "path": "/spec/template/spec/nodeSelector/non-existent-node"}]'
```

2. Validate the deployment's pod health:

```bash
oc get pods -n openshift-cnv -l cdi.kubevirt.io=cdi-deployment -o wide
```
Stuck pod will be removed.

3. Check Operator health condition:

```bash
oc get hyperconverged kubevirt-hyperconverged -n openshift-cnv -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}' | column -t
```
Make sure the HCO operator is in healthy state by checking field `Available: True`, and alert will be resolved.
