# IngressWithoutClassName

PrometheusRule Source: `ingress-operator` · Alert Severity: `Warning` · Pending Period: `1d` 

---

## Meaning

The alert indicates that an Ingress resource has been created without an ingressClassName configured and has remained in this state for longer than one day.

```bash
openshift_ingress_to_route_controller_ingress_without_class_name == 1
```
An Ingress without a class may not be processed by the intended Ingress Controller, depending on the cluster's IngressClass configuration.

---

## Impact

- The affected Ingress may not be processed by the expected Ingress Controller.

- External access to the application through the Ingress may fail or may not behave as expected.

- Traffic may not be routed to the intended backend application.

- Other Ingress resources are not necessarily affected; the impact is generally limited to the Ingress resource without a valid class.:

---

## Diagnosis: 
<br/>

| Check | Command | What to look for | Action |
|---|---|---|---|
| Identify Ingress resources without a class | `oc get ingress -A` | The `CLASS` column shows `<none>` for the affected Ingress | Note the namespace and Ingress name |
| Check the affected Ingress configuration | `oc get ingress <ingress-name> -n <namespace> -o yaml` | `spec.ingressClassName` is missing or empty | Confirm that the Ingress does not specify an IngressClass |
| Check available IngressClasses | `oc get ingressclass` | Identify the IngressClass associated with the intended Ingress Controller | Determine the appropriate IngressClass to use |
| Check Ingress details and events | `oc describe ingress <ingress-name> -n <namespace>` | Review the IngressClass, status, address, and events | Investigate any errors or warnings related to the Ingress |
| Check the Ingress Controller | `oc get ingresscontroller -n openshift-ingress-operator` | Verify that the expected Ingress Controller is available and not degraded | If the controller is degraded, investigate the Ingress Controller separately |
| Verify Ingress status | `oc get ingress <ingress-name> -n <namespace> -o wide` | Check whether an address has been assigned and review the current status | If the Ingress is not being processed, proceed with mitigation |
| Verify application accessibility | `curl -Ik https://<ingress-hostname>` | Confirm whether the application is accessible through the Ingress | If access fails, investigate the Ingress, IngressClass, and backend configuration |


## Mitigation 

1. List available IngressClasses

```bash
  oc get ingressclass
```

2. Describe the class to confirm its underlying controller implementation:

```bash
oc describe ingressclass <ingress-class-name>
```
Look for .spec.controller in the output. For example:

- `openshift.io/ingress-to-route`: The default OpenShift Router (HAProxy) controller, associated with the `openshift-default` IngressClass.

- `k8s.io/ingress-nginx or nginx.org/ingress-controller`: Custom NGINX Ingress Controller implementations, typically associated with the `nginx` IngressClass

3. Patch the affected Ingress with appropriate ingressClassName:

```bash
oc patch ingress <ingress-name> \
  -n <namespace> \
  --type=merge \
  -p '{"spec":{"ingressClassName":"<ingressclass>"}}'
```

Note: Use ingressclass as `openshift-default` if your application is a standard HTTP/HTTPS web service that needs to leverage native OpenShift routing features (like automatically generated OpenShift Route CRDs, standard TLS edge/passthrough, or integration with OpenShift Service Mesh)

---

## Verification:

1. Verify the Ingress references the IngressClass
   
   ```bash
   oc get ingress <ingress-name> -n <namespace> \
     -o jsonpath='{.spec.ingressClassName}{"\n"}'
   ```
3. Confirm the IngressWithoutClassName alert is no longer firing.
