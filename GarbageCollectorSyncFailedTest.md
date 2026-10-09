# Reproduce GarbageCollectorSyncFailed Alert

This scenario is based on a CRD is registered in the cluster, but its API conversion mechanism is incorrectly configured or becomes unavailable. When the Garbage Collector tries to discover or monitor that resource, the API server cannot successfully provide the required resource information. The Garbage Collector is then unable to synchronize its monitor/cache, which can result in `GarbageCollectorSyncFailed` alert.

## Procedure: 

***Step 1: Apply the CRD with the Required Approval Annotation***

```bash
oc apply -f - <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: testcsinodes.storage.k8s.io
  annotations:
    api-approved.kubernetes.io: "https://github.com/kubernetes/enhancements/pull/1111"
spec:
  group: storage.k8s.io
  names:
    kind: TestCSINode
    listKind: TestCSINodeList
    plural: testcsinodes
    singular: testcsinode
  scope: Cluster
  versions:
    - name: v1beta1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
EOF
```

***Step 2: Break Conversion Routing for the CRD***

```bash
oc patch crd testcsinodes.storage.k8s.io --type=merge -p '{
  "spec": {
    "conversion": {
      "strategy": "Webhook",
      "webhook": {
        "conversionReviewVersions": ["v1beta1"],
        "clientConfig": {
          "service": {
            "name": "non-existent-webhook",
            "namespace": "default",
            "path": "/convert",
            "port": 443
          }
        }
      }
    }
  }
}'
```
***Step 3: Verify the logs***

```bash
for pod in $(oc get pods -n openshift-kube-controller-manager \
  -l app=kube-controller-manager -o name); do
  echo "===== $pod ====="
  oc logs -n openshift-kube-controller-manager "$pod" \
    -c kube-controller-manager --since=30m |
    grep -iE 'garbage controller monitor not yet synced|syncing garbage collector|conversion webhook|failed to list|failed to watch|discovery.*error'
done
```

***Step 4: Verify the alert in openshift console*** - `Observing > Alerting`


## Cleanup: 

***1. Delete the CRD:***

```bash
oc delete crd testcsinodes.storage.k8s.io --ignore-not-found
```
