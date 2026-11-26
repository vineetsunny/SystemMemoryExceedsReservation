## Reproduce SSPCommonTemplatesModificationReverted Alert Procedure


This procedure validates that the SSPCommonTemplatesModificationReverted alert fires when an operator-managed OpenShift Virtualization common template in the openshift namespace is modified and automatically reverted by the SSP Operator. Patching an existing managed label or spec field on a common template triggers reconciliation, increments kubevirt_ssp_common_templates_restored_total (driving kubevirt_ssp_common_templates_restored_increase > 0), and fires the alert. Because the SSP Operator immediately restores the template to its default state, the alert auto-resolves once the Prometheus recording rule lookback window expires without requiring manual cleanup.

---


### 1. Testing purpose here we used fedora small template: 

```bash
oc get templates -n openshift -l template.kubevirt.io/type=base --show-labels | grep -i fedora-server-small
```

### 2. Overwrite a managed label on fedora-server-small

```bash
oc patch template fedora-server-small -n openshift --type=json -p='[{"op": "replace", "path": "/metadata/labels/flavor.template.kubevirt.io~1small", "value": "false"}]'
```

### 3. Verification: Within 2–3 seconds, check the label to confirm the SSP Operator reverted it back to "true":

```bash
oc get template fedora-server-small -n openshift -o jsonpath='{.metadata.labels.flavor\.template\.kubevirt\.io/small}{"\n"}'
```

The output should return true.

### 4. Verify the restoration log and counter increment

```bash
oc -n openshift-cnv logs -l control-plane=ssp-operator --tail=30 | grep -iE "restor|common template|fedora-server-small"
```
log messages confirming openshift/fedora-server-small was restored/updated.


### 5. Wait 30–60 seconds for the recording rule and alert to fire

Re-run `kubevirt_ssp_common_templates_restored_increase` in Observe → Metrics after ~30–60 seconds.Verification: The recording rule will evaluate to 1 (> 0).


---

## Restore:

- The alert should automatically resolve within an hour.
- Avoid making further changes to the common templates and allow the recording rule's lookback window to expire.
- No manual cleanup is required. The SSP Operator has already restored fedora-server-small to its original state.
