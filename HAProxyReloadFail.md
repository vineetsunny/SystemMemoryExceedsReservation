# HAProxyReloadFail

PrometheusRule Source: `ingress-operator` · Alert Severity: `Warning` · Pending Period: `5m`

---

## Meaning

The alert indicates that the OpenShift Ingress Controller has failed to reload its HAProxy configuration, which will result in the router not picking up recently created or modified routes.It may occur when the router encounter HAProxy process, socket, or configuration-related problems.

***Expression***

```bash
template_router_reload_failure == 1
```

The expression evaluates to true when `template_router_reload_failure` equals `1`. If the condition remains true continuously for 5 minutes, the alert transitions from Pending to Firing. 

---

## Impact

- Newly created or modified routes may not be applied to the running HAProxy configuration.

- Applications accessed through affected routes may experience connectivity issues or continue using outdated routing configuration.

- Existing routes may continue working if their previously loaded configuration remains valid.

- If the issue persists, application availability may be affected, depending on the route changes and the scope of the failure.

---

## Diagnosis

| Check | Command | Action |
|---|---|---|
| Check router pods | `oc get pods -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default` | Verify that router pods are Running and Ready. Investigate pods that are restarting or not ready. |
| Check router logs | `oc logs -n openshift-ingress <router-pod-name> -c router --since=30m` | Identify HAProxy reload failures, configuration errors, socket errors, or other relevant errors. |
| Check recent route changes | `oc get routes -A --sort-by=.metadata.creationTimestamp -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CREATED:.metadata.creationTimestamp,HOST:.spec.host'` | Identify recently created or modified routes that may correlate with the failure. |
| Verify route accessibility | `oc get route <route-name> -n <namespace>` | Identify the route host and verify that the application is accessible. Investigate connection failures. |

---

## Mitigation

- If the logs identify an invalid or problematic route configuration, correct the affected route or related configuration using the appropriate supported procedure.

- If the router remains unhealthy after the underlying issue is corrected, follow the approved OpenShift router recovery procedure. Restart or replace a router pod only when justified by the diagnosis and operational policy.

- Avoid repeatedly modifying routes or restarting router pods without first identifying the cause, as this may increase disruption or obscure the original failure.

---

## Validation

1. Confirm that the affected router pods are Running and Ready:

```bash
oc get pods -n openshift-ingress
```

1. Verify that the metric `template_router_reload_failure` returns to 0 for the affected router.

2. Confirm that the HAProxyReloadFail alert transitions from Firing to Resolved in ***Observe → Alerting***.

3. Verify that the affected route is accepted and that the application is reachable through the route

---

## Related Alerts

- HAProxyDown
