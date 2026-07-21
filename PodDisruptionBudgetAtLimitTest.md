1. Deploy an NGINX application with 2 replicas.
```bash
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-demo-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx-demo
```
Result:
```bash
oc get pods -n default
NAME                         READY   STATUS    RESTARTS   AGE
nginx-demo-87cd4cbb7-hcqh6   1/1     Running   0          5m40s
nginx-demo-87cd4cbb7-xf8xs   1/1     Running   0          5m40s
```


2. Create the following PodDisruptionBudget (PDB):

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-demo-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx-demo

Result:
oc get pdb        
NAME             MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
nginx-demo-pdb   2               N/A               0                     16s




3. Wait for Prometheus to evaluate the alert rule, and generate the alert.

4. Resolve the alert by increasing the Deployment replicas from 2 to 3.

oc scale deployment/nginx --replicas=3 -n nginx 
oc get pods
NAME                         READY   STATUS    RESTARTS   AGE
nginx-demo-87cd4cbb7-glsbf   1/1     Running   0          3m29s
nginx-demo-87cd4cbb7-hcqh6   1/1     Running   0          10m
nginx-demo-87cd4cbb7-xf8xs   1/1     Running   0          10m

Now we have number of pods more than the desired count i.e 2, alert will be resolved.



5. Verify Alerts When the Number of Healthy Pods Falls Below the PodDisruptionBudget. 
Now Scale the deployment from 3 replicas to 1 replicas, causing the number of healthy pods to fall below the minAvailable threshold configured in the PodDisruptionBudget (PDB).
oc scale deployment/nginx-demo --replicas=1
Expected Result:
The following alerts are generated:
	1	PodDisruptionBudgetAtLimit (Critical) – Indicates that the PodDisruptionBudget has reached its disruption limit and no further voluntary pod disruptions are permitted.
	2	KubePdbNotEnoughHealthyPods (Warning) – Indicates that the number of healthy pods has dropped below the minimum required by the PodDisruptionBudget, potentially impacting application availability.
