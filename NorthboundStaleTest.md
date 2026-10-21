# NorthboundStale Alert – Reproduction Procedure

The alert typically remains in the Pending state for a few minutes and may not transition to Firing consistently. This behavior is expected because the reproduction method involves multiple changes and conditional dependencies, making the simulated failure temporary rather than persistent.

---

## Procedure

***Step 1: Block Northbound Database Traffic***

Access the node host environment and block TCP traffic on port 6641 to simulate a communication failure with the OVN Northbound database.

```bash
chroot /host
iptables -I INPUT -p tcp --dport 6641 -j DROP
iptables -I OUTPUT -p tcp --dport 6641 -j DROP
```

***Step 2: Suspend the OVN Controller to Trigger the Alert***

Loop repeatedly sends SIGSTOP to the ovnkube-controller process:

```bash
while true; do
  pkill -STOP -f ovnkube-controller
  sleep 1
done
```

Allow couple of minutes for the timestamp to become stale and for Prometheus to evaluate the alert.

***Step 3: Verify the Alert***

From prometheus console: Observe > Alerting

***Step 4: Verify logs***

```bash
 oc logs <podName> -n openshift-ovn-kubernetes -c nbdb -f
 ```
 Look for traces - `reconnect|ERR`

---

    
## Clean Up and Restore Normal Operation

***1. Remove the Firewall Rules***

Remove the rules created during the test.

```baah
iptables -D INPUT -p tcp --dport 6641 -j DROP
iptables -D OUTPUT -p tcp --dport 6641 -j DROP
```

***2. Stop the test loop in step 2 and resume the exact process***

Find the process id & kill.

```bash
ps -eo pid,stat,args | grep '[o]vnkube-controller'
kill -CONT <PID>
```

To retrigger the alert again, need to kill the PID and again start the loop i.e step 2.
