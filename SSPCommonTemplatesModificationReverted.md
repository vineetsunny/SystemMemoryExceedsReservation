## SSPCommonTemplatesModificationReverted

PrometheusRule Source: `prometheus-k8s-rules-cnv` · Alert Severity: `Warning` · [Runbook](https://github.com/openshift/runbooks/blob/master/alerts/openshift-virtualization-operator/SSPCommonTemplatesModificationReverted.md)

---

### Meaning

This alert fires when the Scheduling, Scale, and Performance (SSP) Operator detects modifications to its managed common templates and restores them to their expected state during reconciliation.

The SSP Operator manages common templates and the Template Validator resources. These resources are continuously reconciled to ensure that their configuration matches the desired state maintained by the operator.

If a user, automation, or another controller modifies a managed common template, the SSP Operator may revert those changes during its reconciliation cycle.

***Expression:***

```bash
kubevirt_ssp_common_templates_restored_increase > 0
```
This metric indicates that one or more common templates have been restored by the SSP Operator.

---

### Impact

- Any modifications made directly to SSP-managed common templates are automatically reverted by the SSP Operator.

---

### Diagnosis

***1. Identify the SSP Operator namespace***

```bash
export NAMESPACE="$(oc get deployment -A | grep ssp-operator | awk '{print $1}')"
```

***2. Inspect SSP Operator logs***

```bash
oc -n $NAMESPACE logs --tail=-1 -l control-plane=ssp-operator | grep 'common template' -C 3
```

---

### Mitigation

1. Avoid modifying common templates managed by the SSP Operator.<br/>
Changes are reverted during reconciliation.

2. Resolve conflicting automation
- Identify the automation modifying the managed templates.
- Remove conflicting changes from its source configuration.
- Update automation to exclude SSP-managed templates.
- Prevent automation from reapplying the changes.

3. Use custom template copies
- Create a copy of the common template for customization.
- Apply changes to the copied template.
- Use a unique name to avoid conflicts with SSP-managed templates.
