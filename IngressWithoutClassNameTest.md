# Reproduce `IngressWithoutClassName` Alert

This alert is triggered when an Ingress resource does not have an ingressClassName configured for the configured alert duration.

An IngressClass determines which Ingress Controller should process the Ingress resource. If no class is specified, the Ingress may not be handled by the intended Ingress Controller, which can result in the application being inaccessible through the expected ingress endpoint.

For this test, we create a simple Ingress resource without specifying ingressClassName. A Deployment or backend Service is not required because the alert condition is based on the Ingress configuration itself.


## 1. Create a test project

```bash
oc new-project ingress-alert-test
```

## 2. Create an Ingress without `ingressClassName`

A Deployment or backend Service is **not required to reproduce the `IngressWithoutClassName` alert**, because the alert is based on the Ingress resource having no `ingressClassName` configured.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-without-class
  namespace: ingress-alert-test
spec:
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
EOF
```

## 3. Verify that `ingressClassName` is not configured

Check the Ingress:

```bash
oc get ingress -n ingress-alert-test ingress-without-class
```

Expected output:

```bash
NAME                    CLASS    HOSTS              ADDRESS   PORTS   AGE
ingress-without-class   <none>   test.example.com             80      16m
```

The `CLASS` column shows `<none>`, indicating that the Ingress does not have an `ingressClassName` configured.

## 4. Verify the alert

Check the alert status in Prometheus/Alertmanager.

> **Important:** The `IngressWithoutClassName` alert is configured to fire only when an Ingress remains without an `ingressClassName` for longer than the alert's configured duration (typically 1 day). Therefore, creating the Ingress will not cause the alert to fire immediately. In a lab, the alert condition can be verified by confirming that the Ingress is missing `ingressClassName` and, if required, waiting for the configured alert duration or using an appropriate test configuration.

---

# Recovery

## 1. Check the available IngressClasses

List the available IngressClasses:

```bash
oc get ingressclass
```

Example:

```text
NAME                CONTROLLER
openshift-default   ...
```

Use the appropriate IngressClass configured for the cluster. In this example, we use:

```text
openshift-default
```

## 2. Patch the Ingress with the appropriate IngressClass

Set `ingressClassName` to `openshift-default`:

```bash
oc patch ingress ingress-without-class \
  -n ingress-alert-test \
  --type=merge \
  -p '{"spec":{"ingressClassName":"openshift-default"}}'
```

## 3. Verify the IngressClass

```bash
oc get ingress -n ingress-alert-test ingress-without-class
```

Expected output:

```text
NAME                    CLASS               HOSTS              ADDRESS   PORTS   AGE
ingress-without-class   openshift-default   test.example.com             80      19m
```

The `CLASS` column should now show:

```text
openshift-default
```

You can also verify the field directly:

```bash
oc get ingress ingress-without-class \
  -n ingress-alert-test \
  -o jsonpath='{.spec.ingressClassName}{"\n"}'
```

Expected output:

```text
openshift-default
```

## 4. Verify alert recovery

Verify the `IngressWithoutClassName` alert in Prometheus/Alertmanager.

Once the Ingress has a valid `ingressClassName`, the alert condition is no longer true. After the monitoring system evaluates the updated condition, the alert should transition to **Resolved/Inactive**.

> **Note:** Alert recovery is also subject to the alert evaluation and notification intervals configured in the monitoring stack.
