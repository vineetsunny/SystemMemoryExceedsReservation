# Reproduce IngressControllerDegraded Alert

The alert is triggered when the IngressController enters a degraded state due to insufficient availability of the router deployment.

The production condition represented by this test can occur naturally when router pods become unavailable and replacement pods cannot be scheduled. The `ingress-repro=true:NoSchedule` taint used in this procedure is a controlled lab mechanism to simulate a scheduling constraint.

> **Note:** Perform this procedure only in a test or non-production environment. The procedure intentionally reduces router availability and may temporarily affect ingress-dependent applications.

---

### 1. Identify the router pods

List the router pods and the nodes on which they are running:

```bash
oc get pods -n openshift-ingress -o wide
```

Identify the nodes hosting the `router-default` pods.

Sample Output:

```text
NAME                            READY   STATUS    NODE
router-default-xxxxx            1/1     Running   worker-cluster-s4zmk-1
router-default-yyyyy            1/1     Running   worker-cluster-s4zmk-2
router-default-zzzzz            1/1     Running   worker-cluster-s4zmk-3
router-default-aaaaa            1/1     Running   worker-cluster-s4zmk-4
```

---

### 2. Review the PodDisruptionBudget

Before removing router pods, check the `router-default` PDB:

```bash
oc get pdb -n openshift-ingress
```

Sample Output:

```text
NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
router-default  N/A             25%               1
```

The `ALLOWED DISRUPTIONS` value indicates how many voluntary disruptions are currently permitted by the PDB.

For example, with four router replicas and:

```text
MAX UNAVAILABLE = 25%
ALLOWED DISRUPTIONS = 1
```

removing a single router pod may not cause the IngressController to become degraded.

Therefore, additional router availability may need to be reduced to reproduce the alert. The number of pods to remove should be determined based on the actual router replica count and IngressController availability requirements in the environment.

---

### 3. Apply a temporary scheduling taint

Apply a temporary `NoSchedule` taint to the node hosting a router pod:

```bash
oc adm taint nodes <nodeName> ingress-repro=true:NoSchedule
```

The `NoSchedule` taint prevents newly created pods that do not have a matching toleration from being scheduled on the node.

Existing router pods on the node are not immediately evicted.

---

### 4. Delete the router pod

Delete the router pod running on the tainted node:

```bash
oc delete pod <router-default-pod> -n openshift-ingress
```

The router Deployment will create a replacement pod to maintain the desired replica count.

Because the selected node is now tainted, the replacement pod cannot be scheduled there unless it tolerates the taint.

Check the resulting pod state:

```bash
oc get pods -n openshift-ingress -o wide
```

If the replacement cannot be scheduled on the available nodes, it may remain in `Pending` state.

To determine why a replacement pod is Pending:

```bash
oc describe pod <pending-router-pod> -n openshift-ingress
```

---

### 5. Repeat for additional router pods if required

If removing one router pod does not cause the IngressController to become degraded, repeat the process for additional router pods as required by the environment.

The objective is to reduce the number of **available router replicas below the minimum required availability**, while retaining sufficient healthy routers to avoid unnecessary complete ingress disruption.

After each disruption, verify:

```bash
oc get pods -n openshift-ingress -o wide
```

and:

```bash
oc get deployment router-default -n openshift-ingress
```

---

## Verify the Alert Condition

Check the IngressController conditions:

```bash
oc get ingresscontroller default \
  -n openshift-ingress-operator \
  -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" | "}{.reason}{" | "}{.message}{"\n"}{end}'
```

The expected degraded state is:

```text
Degraded=True
```

You may also see conditions such as:

```text
Available=False
DeploymentAvailable=False
DeploymentReplicasMinAvailable=False
DeploymentReplicasAllAvailable=False
```

The Prometheus alert is based on:

```promql
ingress_controller_conditions{condition="Degraded"} == 1
```

Once the condition remains active for the configured **5-minute** duration, the `IngressControllerDegraded` alert should fire.

---

# Recovery

### 1. Remove the temporary taint

Remove the taint from each affected node:

```bash
oc adm taint nodes <nodeName> ingress-repro=true:NoSchedule-
```

Repeat for any other nodes that were tainted during the test.

Verify the taint has been removed:

```bash
oc describe node <nodeName> | grep -A2 Taints
```

---

### 2. Verify router pod recovery

Check the router pods:

```bash
oc get pods -n openshift-ingress -o wide
```

The Pending router pods should become schedulable and return to `Running`/`Ready`.

Check the Deployment:

```bash
oc get deployment router-default -n openshift-ingress
```

Expected state:

```text
READY   UP-TO-DATE   AVAILABLE
4/4     4            4
```

The exact replica count depends on the environment.

---

### 3. Verify IngressController recovery

Check the IngressController conditions again:

```bash
oc get ingresscontroller default \
  -n openshift-ingress-operator \
  -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" | "}{.reason}{" | "}{.message}{"\n"}{end}'
```

Expected:

```text
Available=True
Degraded=False
```

The `IngressControllerDegraded` alert should subsequently resolve once the Prometheus expression evaluates to `0` and the alert's recovery evaluation occurs.



