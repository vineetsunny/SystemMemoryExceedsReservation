# Reproduce CoreDNSPanicking Alert


To reproduce this alert, a temporary sidecar was added to the dns-default DaemonSet to expose a dynamically increasing coredns_panics_total metric. This simulated the metric behavior that Prometheus observes when CoreDNS panics occur, allowing the alerting and recovery workflow to be validated safely.

`Note`: Try this test in lab cluster.

---

***Step 1: Set DNS Operator to Unmanaged***
Prevent the DNS Operator from automatically undoing temporary modifications to the dns-default DaemonSet:


```bash
oc patch dns.operator.openshift.io default \
  --type=merge \
  -p '{"spec":{"managementState":"Unmanaged"}}'
```

***Step 2: Inject the Dynamic Panic Exporter Sidecar***

Patch the dns-default DaemonSet to add a Python sidecar listening on port 9155. This sidecar increments coredns_panics_total every 10 seconds:

```bash
oc patch daemonset dns-default -n openshift-dns --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/-",
    "value": {
      "name": "panic-exporter",
      "image": "registry.access.redhat.com/ubi8/python-39:latest",
      "command": ["python3", "-c"],
      "args": [
        "import time\nfrom http.server import HTTPServer, BaseHTTPRequestHandler\nstart_time = time.time()\nclass H(BaseHTTPRequestHandler):\n def do_GET(self):\n  self.send_response(200)\n  self.send_header(\"Content-Type\", \"text/plain; version=0.0.4\")\n  self.end_headers()\n  panics = int(25 + (time.time() - start_time) / 10)\n  self.wfile.write(f\"# HELP coredns_panics_total Panics\\n# TYPE coredns_panics_total counter\\ncoredns_panics_total{{zone=\\\".\\\"}} {panics}\\n\".encode())\nHTTPServer((\"0.0.0.0\", 9155), H).serve_forever()"
      ]
    }
  }
]'
```

***Step 3: Redirect kube-rbac-proxy to the Sidecar***
Update kube-rbac-proxy (container index 1) to proxy upstream requests from port 9154 to the local sidecar on port 9155, while retaining cluster TLS certificates:


```bash
oc patch daemonset dns-default -n openshift-dns --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/1/args",
    "value": [
      "--secure-listen-address=0.0.0.0:9154",
      "--upstream=http://127.0.0.1:9155/",
      "--tls-cert-file=/etc/tls/private/tls.crt",
      "--tls-private-key-file=/etc/tls/private/tls.key",
      "--logtostderr=true",
      "--v=2"
    ]
  }
]'
```

***Step 4: Verify Health and Metrics Ingestion***

- Verify Pod Readiness:

```bash
oc get pods -n openshift-dns -l dns.operator.openshift.io/owning-dns=default
```

- Verify Dynamic Counter Increments:

```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -c prometheus -- \
  curl -s "http://localhost:9090/api/v1/query?query=coredns_panics_total"
```

- Verify Active Alerting State:

```bash
oc exec -n openshift-monitoring prometheus-k8s-0 -c prometheus -- \
  curl -s "http://localhost:9090/api/v1/query?query=ALERTS%7Balertname%3D%22CoreDNSPanicking%22%7D"
```


---

# Cleanup & Restore

***Step 1: Re-Enable Operator Management***

```bash
oc patch dns.operator.openshift.io default \
  --type=merge \
  -p '{"spec":{"managementState":"Managed"}}'
```

***Step 2: Delete Custom ConfigMap***

```bash
oc delete configmap/dns-default -n openshift-dns --ignore-not-found
```

***Step 3: Delete Standalone Test Deployments / Services***

```bash
oc delete deployment/coredns-panic-producer deployment/coredns-panic-injector service/coredns-panic-producer -n openshift-dns --ignore-not-found
```

***Step 4: Restart the CoreDNS DaemonSet***

```bash
oc rollout restart daemonset/dns-default -n openshift-dns
oc rollout status daemonset/dns-default -n openshift-dns
```

***Step 5: Final Cleanup Verification***

```bash
oc get pods -n openshift-dns -l dns.operator.openshift.io/owning-dns=default
```
