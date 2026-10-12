# Reproduce HAProxyReloadFail Alert

This test simulates an HAProxy router reload failure by removing the HAProxy control socket from a router pod and repeatedly updating a test route to trigger configuration reload attempts. The alert is expected to fire when the router fails to reload its HAProxy configuration.

---

***Step 1: Create the Test Service and Route***

```bash
oc create service clusterip dummy-service --tcp=8080:8080 -n default
oc create route edge test-fail-route --service=dummy-service --port=8080 -n default
```

***Step 2: Break the HAProxy Control Socket in a Target Pod***

```bash
oc exec -n openshift-ingress pod/<podName> -c router -- rm -rf /var/lib/haproxy/run/haproxy.sock
```


***Step 3: Run the Route Annotation Loop***

```bash
while true; do oc annotate route test-fail-route -n default test=$(date +%s) --overwrite; sleep 2; done
```

***Step 4: Check Alert Status***

From Openshift console: `Observe > Alerting`

---

# Clean Up

***Stop the loop (Ctrl+C), then delete the pod and test resources***

```bash
oc delete pod router-default-* -n openshift-ingress
oc delete route test-fail-route -n default
oc delete service dummy-service -n default
```


