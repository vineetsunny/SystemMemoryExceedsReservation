# Steps to generate the SystemMemoryExceedsReservation alert.
 
**1. Determine the remaining RSS memory at the node level :** 
oc exec -n openshift-monitoring prometheus-k8s-0 -c prometheus -- curl -s http://localhost:9090/api/v1/query --data-urlencode \
  'query=(sum(kube_node_status_capacity{resource="memory"} - kube_node_status_allocatable{resource="memory"}) by (node) * 0.95) - sum(container_memory_rss{id="/system.slice"}) by (node)' | jq '.data.result[] | {node: .metric.node, megabytes_left_before_alert: ((.value[1] | tonumber) / 1024 / 1024)}'

**2. Take any one of the node where you want to increase the system's memory:**
a) oc debug node/<node-name>
b) chroot /host
b) systemd-run --slice=system.slice python3 -c 'import time; x = b"x" * 810 * 1024 * 1024; time.sleep(1400)'

`Note`: The memory allocation value i.e 810  will be varied based on the available size on RSS memory (in step 1, you identified), you can increase the size up to that level so that it should be almost 95% full.

**3. How to identify how much memory consumed by each process in RSS:** `(Run below command inside the target node)`
ps -eo pid,user,rss,cmd --sort=-rss | grep -E "PID|crio|kubelet|systemd|NetworkManager|python" | head -n 15 | awk '
NR==1 {print $1, $2, "RSS_MEMORY", $4; next} 
{
  rss=$3; 
  if(rss > 1024*1024) {
    printf "%-6s %-8s %-10.2f GB %s\n", $1, $2, rss/(1024*1024), $4
  } else {
    printf "%-6s %-8s %-10.2f MB %s\n", $1, $2, rss/1024, $4
  }
}'

