## Reproduce KubeletHealthState Alert

The test is performed by dropping incoming traffic to port 10248 using iptables inside the target node's host to make the local MCD health check fail consecutively.

---

1. Access the node via an active oc debug session

```bash
oc debug node/<target-node-name>
chroot /host
```


2. Block local access to port 10248 using iptables

```bash
iptables -A INPUT -p tcp --dport 10248 -j DROP
```

Sample Output:

```bash
iptables -L -n | grep 10248
DROP       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:10248
```

3. Observe the MCD health check failures

```bash
oc logs -n openshift-machine-config-operator <mcd-pod-name> -f | grep -i "health"
```
Sample Output:

```bash
W1014 16:18:40.772973    3117 daemon.go:1412] Got an error from auxiliary tools: kubelet health check has failed 1 times: Get "http://localhost:10248/healthz": context deadline exceeded
W1014 16:19:40.774561    3117 daemon.go:1412] Got an error from auxiliary tools: kubelet health check has failed 2 times: Get "http://localhost:10248/healthz": context deadline exceeded
W1014 16:20:40.775187    3117 daemon.go:1412] Got an error from auxiliary tools: kubelet health check has failed 3 times: Get "http://localhost:10248/healthz": context deadline exceeded
```


4. Verify the alert from prometheus console: 
Observe > Alerting

---

## Cleanup

To clear the rule and restore the health check endpoint

```bash
iptables -D INPUT -p tcp --dport 10248 -j DROP
```
