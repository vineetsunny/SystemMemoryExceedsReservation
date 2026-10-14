# IngressControllerUnavailable

PrometheusRule Source: `ingress-operator` · Alert Severity: `Warning` · Pending Period: `5m`

---

## Meaning:

The alert triggers when an OpenShift IngressController remains unavailable for 5 minutes. This condition can occur when all router pods in the `openshift-ingress` namespace are not Ready or available, causing the IngressController to become unavailable.

An individual router pod being unavailable does not necessarily trigger the alert, provided the remaining router pods maintain the required availability.

The alert expression evaluates the IngressController's `Available` condition. A value of `0` indicates that the IngressController is unavailable.

***Expression***

```bash
ingress_controller_conditions{condition="Available"} == 0
```
---

## Impact

- External access to applications through OpenShift Routes may be disrupted.
- Applications hosted on the cluster may become inaccessible to external users.
- The OpenShift Web Console may become inaccessible if the default IngressController is affected.
- Application availability may be impacted until the router pods and IngressController recover.

---

## Diagnosis

| Check | Command | Expected Result |
|---|---|---|
| Check IngressController status | `oc get ingresscontroller -n openshift-ingress-operator` | IngressController should be available and healthy. |
| Check detailed conditions | `oc describe ingresscontroller default -n openshift-ingress-operator` | Identify the `Available` condition and any failure reason. |
| Check router pods | `oc get pods -n openshift-ingress` | Router pods should be `Running` and `Ready`. |
| Check router deployment | `oc get deployment router-default -n openshift-ingress` | Desired and available replicas should match. |
| Check pod scheduling issues | `oc get pods -n openshift-ingress -o wide` | Identify router pods in `Pending` state or nodes where they are scheduled. |
| Check scheduling events | `oc get events -n openshift-ingress --sort-by=.lastTimestamp` | Identify scheduling failures, node selector mismatches, or resource constraints. |
| Check IngressController configuration | `oc get ingresscontroller default -n openshift-ingress-operator -o yaml` | Review node selectors, message, replica settings, and other relevant configuration. |

---

## Mitigation

- Review the IngressController conditions, router pod status, and namespace events.

- If router pods are pending because no nodes match the configured selector, correct the selector or apply the intended label to eligible nodes, following the approved cluster configuration.

- If scheduling fails due to insufficient CPU or memory, assess node capacity and resolve the resource constraint.

- Allow the Ingress Operator to reconcile the corrected configuration and recreate the router pods. Avoid manually scaling the managed router deployment unless explicitly required by an approved recovery procedure.

--

## Validation

***Step 1 : Ensure the ingress_controller_conditions metric value is "1"***

```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -c prometheus -- \
  curl -sG 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=ingress_controller_conditions{condition="Available"}' \
  | jq .
```

***Step 2 : Ingress pods should be in Running state***

```bash
oc get pods -n openshift-ingress
```

***Step 3 : Alert should be resolved***

Check Prometheus console: `Observe > Alerting`

## Related Alerts

- IngressControllerDegraded
- ClusterOperatorDown
