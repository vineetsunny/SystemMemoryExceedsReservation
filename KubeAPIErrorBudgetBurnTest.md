# Reproduce KubeAPIErrorBudgetBurn Alert

This procedure reproduces the `KubeAPIErrorBudgetBurn` alert by intentionally
causing Kubernetes API requests to fail with HTTP 5xx errors.

The test uses a `ValidatingWebhookConfiguration` with `failurePolicy: Fail`
and points it to a non-existent webhook service. When a ConfigMap creation
request is sent, the API server attempts to call the webhook, but the webhook
service is unavailable, causing the API request to fail.

# Procedure:

## 1. Create the Demo Namespace

Create a dedicated namespace for the webhook demo:

```bash
oc create ns api-error-demo
```

## 2. Create the Validating Webhook Configuration

Create the following `ValidatingWebhookConfiguration`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: api-error-demo

webhooks:
  - name: demo.openshift.io
    admissionReviewVersions:
      - v1

    sideEffects: None
    failurePolicy: Fail
    timeoutSeconds: 2

    clientConfig:
      service:
        namespace: api-error-demo
        name: webhook-server
        path: /validate

    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        operations:
          - CREATE
        resources:
          - configmaps
```

## 3. Verify the Webhook Configuration

Confirm that the webhook configuration was created successfully:

```bash
oc get validatingwebhookconfiguration api-error-demo
```

## 4. Test the Webhook

Attempt to create a ConfigMap:

```bash
oc create configmap test --from-literal=a=b
```

Expected result:

```text
error: failed to create configmap: Internal error occurred:
failed calling webhook "demo.openshift.io":
failed to call webhook:
```

> **Note:** The webhook is configured with `failurePolicy: Fail`. Since the `webhook-server` service does not exist, the API request fails.

## 5. Start the Worker Script

Create a file named `worker.sh` with the following content:

```bash
#!/bin/bash

TOKEN=$(oc whoami -t)
API=$(oc whoami --show-server)

while true
do
    NAME=test-$RANDOM

    cat <<EOF | \
    curl -sk \
      -X POST \
      -H "Authorization: Bearer ${TOKEN}" \
      -H "Content-Type: application/json" \
      "${API}/api/v1/namespaces/default/configmaps" \
      -d @- >/dev/null
{
  "apiVersion": "v1",
  "kind": "ConfigMap",
  "metadata": {
    "name": "${NAME}"
  },
  "data": {
    "a": "b"
  }
}
EOF

    sleep 0.02
done
```

Make the script executable:

```bash
chmod +x worker.sh
```

## 6. Start with 10 Workers

Start 10 worker processes in the background:

```bash
for i in {1..10}
do
    ./worker.sh &
done
```

## 7. Monitor the Status of requests

Keep monitoring the cluster and observe the behavior of the ConfigMap creation requests and the validating webhook.

***Using GUI -*** Observe > Metrics

```bash
sum:apiserver_request:burnrate1h 
```
```bash
sum:apiserver_request:burnrate5m
```

***Or (Using CLI)***

```bash
watch -n 30 '
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
query="sum:apiserver_request:burnrate1h" \
| jq -r ".data.result[] | [.metric.__name__, .value[1]] | @tsv"
'

watch -n 30 '
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
query="sum:apiserver_request:burnrate5m" \
| jq -r ".data.result[] | [.metric.__name__, .value[1]] | @tsv"
'
```

***API returns the most 5xx errors (CLI)***

```bash
watch -n 30 '
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
query="
sort_desc(
sum by(resource,verb,code)(
rate(apiserver_request_total{job=\"apiserver\",code=~\"5..\"}[5m])
)
)
" | jq -r ".data.result[] |
[.metric.resource,.metric.verb,.metric.code,.value[1]]
| @tsv"
'
```

***Slow GET/LIST requests (>1 second)***
 
```bash
watch -n 30 '
oc -n openshift-monitoring exec prometheus-k8s-0 -- \
curl -sg http://localhost:9090/api/v1/query \
--data-urlencode \
query="
sort_desc(
(
sum by(resource,verb)(
rate(apiserver_request_duration_seconds_count{
job=\"apiserver\",
verb=~\"GET|LIST\"
}[5m]))
-
sum by(resource,verb)(
rate(apiserver_request_duration_seconds_bucket{
job=\"apiserver\",
verb=~\"GET|LIST\",
le=\"1\"
}[5m]))
)
)
" | jq -r ".data.result[] |
[.metric.resource,.metric.verb,.value[1]]
| @tsv"
'
```


---

# Cleanup

## Stop All Workers

Terminate the running worker processes:

```bash
pkill -f worker.sh
```

## Delete the Webhook Configuration

```bash
oc delete validatingwebhookconfiguration api-error-demo
```

## Delete the Namespace

```bash
oc delete namespace api-error-demo
```

After cleanup, verify that the webhook configuration and namespace have been removed:

```bash
oc get validatingwebhookconfiguration api-error-demo
oc get namespace api-error-demo
```
