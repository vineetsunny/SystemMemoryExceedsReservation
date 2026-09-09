# Reproduce AlertmanagerFailedReload Alert

This scenario is based on a misconfiguration in the main Alertmanager configuration file.


***1: Scale Down CMO***

```bash
oc -n openshift-monitoring scale deployment/cluster-monitoring-operator --replicas=0
```

***2. Back Up Platform Secret***

```bash
oc -n openshift-monitoring get secret alertmanager-main -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d > platform-alertmanager-backup.yaml
```

***3. Inject Broken Config (Missing SMTP Port)***

```bash
BAD_CONFIG=$(cat <<EOF | base64 -w0
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com'
  smtp_from: 'alert@example.com'
route:
  receiver: 'default'
receivers:
- name: 'default'
EOF
)
```

***4. Patch the secret***

```bash
oc -n openshift-monitoring patch secret alertmanager-main --type=json -p="[{\"op\": \"replace\", \"path\": \"/data/alertmanager.yaml\", \"value\":\"$BAD_CONFIG\"}]"
```

***5. Verify Immediate Parse Error in Logs***
```bash
oc -n openshift-monitoring logs -l app.kubernetes.io/name=alertmanager -c alertmanager --tail=20 | grep "Loading configuration file failed"
```

***6. Wait 10 Minutes & Check Alert***


### Restore Steps :


***1. Apply the Secret Patch***

```bash
RESTORE_CONFIG=$(base64 < platform-alertmanager-backup.yaml | tr -d '\r\n')

oc -n openshift-monitoring patch secret alertmanager-main --type=json -p="[{\"op\": \"replace\", \"path\": \"/data/alertmanager.yaml\", \"value\":\"$RESTORE_CONFIG\"}]"
```

***2. Scale the Operator Back Up (confirm this)***

```bash
oc -n openshift-monitoring scale deployment/cluster-monitoring-operator --replicas=1 
```

***3. Verify the logs and alert should be resolved***

```bash
oc -n openshift-monitoring logs -l app.kubernetes.io/name=alertmanager -c alertmanager --tail=20 | grep -i "Completed loading of configuration file"
```
Confirm that the logs contain the following message `Completed loading of configuration file`
