# Reproduce VirtAPIDown Alert

This procedure reproduces the VirtAPIDown alert by simulating a scenario in which the virt-api pods are unable to pull their container image because the configured image is unavailable.
Because a registry outage is not available in the test environment, an invalid image tag is manually configured for the virt-api Deployment to simulate an image-pull failure.

This scenario is `representative` of a production situation where the container registry is unavailable or inaccessible during a node reboot, for operator upgrade/OCP upgrade. After the nodes restart, the required virt-api image cannot be pulled, causing the virt-api pods to enter an ImagePullBackOff state and eventually resulting in the VirtAPIDown alert.



1. Temporarily scale down the HCO and KubeVirt operators to prevent them from reconciling the changes made during the test:

```bash
export NAMESPACE="$(oc get kubevirt -A -o custom-columns="":.metadata.namespace)"
oc -n $NAMESPACE scale deploy hco-operator --replicas=0;oc -n $NAMESPACE scale deploy virt-operator --replicas=0
```

2. Configure an Invalid virt-api Image:
```bash
oc -n $NAMESPACE patch deploy virt-api --type='json' -p='[                                                      
  {"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "internal-registry.local/kubevirt/virt-api:invalid-tag"}
```

3. Recreate the virt-api Pods
```bash
oc -n $NAMESPACE scale deploy virt-api --replicas=0;oc -n $NAMESPACE scale deploy virt-api --replicas=2 
```

4. Verify the virt-api Pod Status:

```bash
oc -n $NAMESPACE get pods -l kubevirt.io=virt-api -o wide
```

Sample example:
```bash
virt-api-86b988b6cd-4db6n                              0/1     ImagePullBackOff   0              4m46s
virt-api-86b988b6cd-c77bf                              0/1     ImagePullBackOff   0              4m46s

```

5. Validate the VirtAPIDown Alert
Login to openshift console > Observe > Alerting

6. Restore the HCO and KubeVirt Operators:

```bash 
oc -n $NAMESPACE scale deploy hco-operator --replicas=1;oc -n $NAMESPACE scale deploy virt-operator --replicas=1
```

7. Verify virt-api Recovery. ( Wait for few min.)
 
```bash 
oc -n $NAMESPACE get pods -l kubevirt.io=virt-api -o wide
```
