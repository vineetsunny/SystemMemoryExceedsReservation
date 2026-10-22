1. PodDisruptionBudget (PDB) blocks node drain
Cause: During an MCP update, OpenShift needs to drain a node before applying certain configuration changes or rebooting it. A PDB can prevent pod eviction when the disruption would violate the application's availability requirements.
- Symptoms: MCP remains UPDATING=True; node drain or eviction retries fail.
- What to check: PDB disruptionsAllowed, healthy replica count, pod events, and Machine Config Operator logs.
- Resolution: Restore sufficient healthy replicas, resolve unavailable workloads, or make an approved PDB adjustment after assessing service availability.
2. Upgrade stuck while fetching the OS image or applying MachineConfig
Cause: The node cannot retrieve required image content, or the update process is blocked while applying the desired configuration. Potential causes include registry connectivity, DNS or proxy issues, authentication failures, unavailable image content, and MCD errors.
- Symptoms: MCP remains UPDATING=True; the affected node stays on its previous configuration or fails to complete the update.
- What to check: MCP conditions, Cluster Operator status, MCD logs, node events, registry connectivity, and image-pull errors.
- Resolution: Fix the underlying registry, network, authentication, or configuration issue, then allow the managed rollout to resume.
3. Node hardware or operating-system failure
Cause: A node experiences a hardware, disk, memory, kernel, filesystem, or boot issue that prevents it from operating normally or completing an upgrade.
- Symptoms: Node becomes NotReady, stops responding, fails to return after reboot, or reports hardware and I/O errors.
- What to check: Node conditions and events, kubelet and kernel logs, disk capacity, memory pressure, boot console, and infrastructure hardware health.
- Resolution: Address the underlying OS or hardware fault. Involve the infrastructure team if physical hardware failure is suspected, and verify node readiness before resuming normal operations.
