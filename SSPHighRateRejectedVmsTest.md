## Reproduce SSPHighRateRejectedVms Alert Procedure

This test validates that the SSPHighRateRejectedVms alert is triggered when multiple VirtualMachine creation requests are rejected by the SSP admission webhook due to validation failures within a short period.

---

### 1. Create a test project

```bash
oc new-project ssp-alert-test
```

### 2. Prepare a VirtualMachine manifest that intentionally violates its validation annotation

- invalid-vm.yaml
```bash
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: test-rejected-vm
  namespace: ssp-alert-test
  annotations:
    vm.kubevirt.io/validations: |
      [
        {
          "name": "minimal-required-memory",
          "path": "jsonpath::.spec.domain.resources.requests.memory",
          "rule": "integer",
          "message": "Memory size is less than minimum required 2Gi",
          "min": 2147483648
        }
      ]
spec:
  runStrategy: Halted
  template:
    spec:
      domain:
        devices: {}
        resources:
          requests:
            memory: 512Mi
```            
            
### 3. Generate a sustained burst of rejected requests         
            
```bash
for i in $(seq 1 6); do
  oc apply -f invalid-vm.yaml 2>/dev/null || true
  sleep 2
done
```

### 4. Failure logs :

***i) Get the latest audit log files***

```bash
oc adm node-logs --role=master --path=kube-apiserver/
```
***ii) Search the affected vm name i.e test-rejected-vm in the latest log file***
```bash
oc adm node-logs <control-plane-node> --path=kube-apiserver/<audit-log-file> | \
  grep "test-rejected-vm" | tail -n 5 | \
  jq -c '{time: .requestReceivedTimestamp, user: .user.username, sourceIP: .sourceIPs[0], verb: .verb, code: .responseStatus.code, reason: .responseStatus.message}'
```
***iii) You will see following traces***

```bash
oc adm node-logs control-plane-cluster-nxkgm-2 --path=kube-apiserver/audit-2026-11-24T06-25-35.007.log | \
  grep "test-rejected-vm" | tail -n 5 | \
  jq -c '{time: .requestReceivedTimestamp, user: .user.username, sourceIP: .sourceIPs[0], verb: .verb, code: .responseStatus.code, reason: .responseStatus.message}'
{"time":"2026-11-24T05:46:19.820176Z","user":"admin","sourceIP":"100.64.0.49","verb":"create","code":422,"reason":"admission webhook \"virtualmachine-admission.ssp.kubevirt.io\" denied the request: Memory size is less than minimum required 2Gi: value 536870912 is lower than minimum [2147483648]"}
{"time":"2026-11-24T05:46:38.423364Z","user":"admin","sourceIP":"100.64.0.49","verb":"get","code":404,"reason":"virtualmachines.kubevirt.io \"test-rejected-vm\" not found"}
{"time":"2026-11-24T05:46:38.939397Z","user":"admin","sourceIP":"100.64.0.49","verb":"create","code":422,"reason":"admission webhook \"virtualmachine-admission.ssp.kubevirt.io\" denied the request: Memory size is less than minimum required 2Gi: value 536870912 is lower than minimum [2147483648]"}
{"time":"2026-11-24T05:46:43.007828Z","user":"admin","sourceIP":"100.64.0.49","verb":"get","code":404,"reason":"virtualmachines.kubevirt.io \"test-rejected-vm\" not found"}
```

### 5. Verify the alert in Prometheus / OpenShift Console

---

## Cleanup 

***Delete the testing project***

```bash
oc delete project ssp-alert-test
```

