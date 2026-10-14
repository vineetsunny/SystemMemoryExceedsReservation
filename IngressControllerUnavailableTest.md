# Reproduce IngressControllerUnavailable Alert

This scenario based on an invalid node selector configuration was applied to the OpenShift IngressController, preventing router pods from being scheduled onto any worker nodes. After router-default was scaled down and back up, all router pods remained in Pending state, causing external route traffic to fail and making the OpenShift Web Console inaccessible.

---

***1. Set an Invalid Node Selector***
```bash
 oc patch ingresscontroller/default -n openshift-ingress-operator --type=merge -p '{"spec":{"nodePlacement":{"nodeSelector":{"matchLabels":{"test-stuck-pod":"true"}}}}}'
 ```
 
***2. Scale the deployment to 0 replicas***
 ```bash
 oc scale deploy/router-default -n openshift-ingress --replicas=0
```


***3. All old pods will be terminated and new pods will be in pending state***
```bash
oc get pods -n openshift-ingress
```

***4. Verify the alert***
 GUI won't be available. Use CLI to verify the alert:

```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -c prometheus -- \
  curl -s 'http://localhost:9090/api/v1/alerts' |
jq -r '.data.alerts[] |
  [.labels.alertname, .state,
   (.labels.severity // "unknown"),
   (.labels.namespace // "-")] | @tsv' |
column -t
```
---

## Clean up:

***1. Remove the Node Selector Constraint from the Operator***

```bash
oc patch ingresscontroller/default -n openshift-ingress-operator --type=json -p='[{"op": "remove", "path": "/spec/nodePlacement/nodeSelector"}]'
```
After this change, pending pods will be terminated and new pods generated.

***2. Verify the pods status***

```bash
 oc get pods -n openshift-ingress
```

***3. Alert should be resolved, and now GUI must be accessible***

