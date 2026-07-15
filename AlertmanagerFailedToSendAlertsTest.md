# Steps to generate AlertmanagerFailedToSendAlerts alert

**1) Ensure that Alertmanager is integrated with a notification receiver. For testing, Slack was used as the receiver. Refer to the following guide for the integration steps:**
https://www.redhat.com/en/blog/how-to-integrate-openshift-namespace-monitoring-and-slack

**2) To perform the tests from the OpenShift side, access the Alertmanager UI:**
  a) Log in to the OpenShift cluster.
  b) Navigate to Administration → Cluster Settings → Configuration → Alertmanager.
  c) Click on Alertmanager → Under Receivers section → edit the receiver

**3) Test Cases:**

`Test Case 1: Invalid Slack Webhook URL`
Action: Configured an invalid Slack webhook API URL while keeping the hostname unchanged.
Verify the logs: oc logs -n openshift-monitoring -l app.kubernetes.io/name=alertmanager -c alertmanager --tail=50
Notification delivery failed with HTTP 404 (Not Found), indicating that the configured webhook endpoint was invalid.

`Test Case 2: Invalid Receiver Hostname`
Action: Modified the receiver endpoint hostname.
Verify the logs: oc logs -n openshift-monitoring -l app.kubernetes.io/name=alertmanager -c alertmanager --tail=50
Expected Result: Notification delivery failed due to TLS certificate verification failure (x509: certificate signed by unknown authority), indicating that the endpoint could not be authenticated.
