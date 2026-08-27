# Reproduce VirtHandlerRESTErrorsBurst Alert

By temporarily revoking the API authorization of the virt-handler ServiceAccount cluster-wide, simulate a severe control-plane communication breakdown (HTTP 403 Forbidden).

***Step 1: Back up the original binding***

```Bash
oc get clusterrolebinding kubevirt-handler -o yaml > kubevirt-handler-crb-backup.yaml
```

***Step 2: Clear the subjects array (Triggers 403 Errors)***
Strip the ServiceAccount authorization from the binding by replacing the subjects array with an empty list ([]).

```Bash
oc patch clusterrolebinding kubevirt-handler --type='json' \
  -p='[{"op": "replace", "path": "/subjects", "value": []}]'
```

***Step 3: Monitor the metric ratio***
Keep running your query command. Note that null was expected initially because there were 0 errors before the test (0 in numerator = no matrix result):

```Bash
oc exec -n openshift-monitoring prometheus-k8s-0 -c prometheus -- \
  curl -s -G 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=sum(rate(kubevirt_rest_client_requests_total{code=~"(4|5)[0-9][0-9]",namespace="openshift-cnv",pod=~"virt-handler-.*"}[5m])) / sum(rate(kubevirt_rest_client_requests_total{namespace="openshift-cnv",pod=~"virt-handler-.*"}[5m]))' \
  | jq '.data.result[0].value[1]'
```
Sample output once the fault accumulates (Targeting > 0.80 ):

```Bash
oc exec -n openshift-monitoring prometheus-k8s-0 -c prometheus -- \
  curl -s -G 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=sum(rate(kubevirt_rest_client_requests_total{code=~"(4|5)[0-9][0-9]",namespace="openshift-cnv",pod=~"virt-handler-.*"}[5m])) / sum(rate(kubevirt_rest_client_requests_total{namespace="openshift-cnv",pod=~"virt-handler-.*"}[5m]))' \
  | jq '.data.result[0].value[1]'
"0.8430893723473105"
```

***Verify Alert:***

---
```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -c prometheus -- \
  curl -s 'http://localhost:9090/api/v1/alerts' \
  | jq '.data.alerts[] | select(.labels.alertname | contains("VirtHandlerRESTErrorsBurst"))'
```


![](assets/17877368935639.jpg)


***Step 5: Restore permissions when done***

a. Restore authorization via patch:

```Bash
oc patch clusterrolebinding kubevirt-handler --type='json' \
  -p='[{"op": "add", "path": "/subjects", "value": [{"kind": "ServiceAccount", "name": "kubevirt-handler", "namespace": "openshift-cnv"}]}]'
```

b. Validate log recovery: (no entry for forbidden messages):

```bash
oc logs -n openshift-cnv -l app.kubernetes.io/component=compute --since=5m -f | grep -i "forbidden"
```
