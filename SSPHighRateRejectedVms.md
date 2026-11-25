## SSPHighRateRejectedVms

PrometheusRule Source: `prometheus-k8s-rules-cnv` · Alert Severity: `Warning` · Pending Period: `5m` · [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/openshift-virtualization-operator/SSPHighRateRejectedVms.md)

---

### Meaning

This alert is triggered when a user or automation script attempts to create or update a large number of virtual machines (VMs), but the VM configurations do not meet the validation requirements.

The virt-template-validator component validates VM configurations before they are created or updated. If the configurations fail validation, the corresponding VM operations are rejected.

***Expression:***

```bash
kubevirt_ssp_template_validator_rejected_increase > 5
```

`kubevirt_ssp_template_validator_rejected_increase` represents the increase in VM requests rejected by the SSP template validator and alert condition evaluates whether the increase exceeds 5.


---

### Impact

- The affected VMs are not created or updated because their configurations fail validation.
- Users may be unable to provision new VMs or modify existing VMs as expected.
- Automated VM provisioning or update workflows may fail, potentially affecting the overall environment.

---

### Diagnosis

Follow the steps below to identify rejected VM requests, determine the admission validation failure, and investigate the corresponding validator logs.

| Step | Diagnostic Action | Command / Procedure | Expected Result |
| --- | --- | --- | --- |
| 1 | Identify the available API server audit log files | `oc adm node-logs --role=master --path=kube-apiserver/` | Lists the available audit log files on control-plane nodes. Identify the log file covering the time when the alert was triggered. |
| 2 | Search for the affected VM in the audit logs | `oc adm node-logs <CONTROL_PLANE_NODE> --path=kube-apiserver/<AUDIT_LOG_FILE> \| grep "<VM_NAME>" \| tail -n 5 \| jq -c '{time: .requestReceivedTimestamp, user: .user.username, sourceIP: .sourceIPs[0], verb: .verb, code: .responseStatus.code, reason: .responseStatus.message}'` | Displays recent API requests associated with the affected VM, including the timestamp, user, operation, response code, and rejection reason. |
| 3 | Identify the SSP operator namespace | `export NAMESPACE="$(oc get deployment -A \| grep ssp-operator \| awk '{print $1}')"` | Identifies the namespace where the SSP operator is deployed. |
| 4 | Check virt-template-validator logs | `oc -n $NAMESPACE logs --tail=-1 -l name=virt-template-validator` | Identifies additional VM configuration validation errors, if logged by the validator. |
| 5 | Check validator pod status and restarts | `oc -n $NAMESPACE get pods -l name=virt-template-validator` | Identifies pod restarts or availability issue |

---

### Mitigation

1. Identify the VM configurations that failed validation by reviewing the virt-template-validator and audit logs.
2. Determine which validation rules were violated, such as minimum memory requirements or other configuration constraints.
3. Correct the invalid VM configurations to meet the required validation rules.
4. Retry the failed VM creation or update operations and verify that the VM operations complete successfully and that the alert is cleared.

