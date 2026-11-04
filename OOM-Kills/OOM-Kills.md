┌───────────────────┬────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Mechanism         │ Who handles it                                 │ What happens                                                                                    │
├───────────────────┼────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Container memory  │ Linux kernel, through cgroups configured by    │ If memory cannot be reclaimed sufficiently within the limit, a cgroup OOM can kill a process in │
│ limit             │ kubelet/runtime                                │ the container.                                                                                  │
├───────────────────┼────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Node memory       │ Kubelet’s eviction manager                     │ Monitors available memory and can evict Pods to protect the node—even if their individual       │
│ pressure          │                                                │ limits have not been reached.                                                                   │
├───────────────────┼────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Usage reporting   │ Kubelet’s stats pipeline                       │ Collects resource usage for metrics and management decisions.                                   │
└───────────────────┴────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────┘

For container setup, kubelet prepares resource settings; the runtime creates/configures the container’s cgroup and places its processes there. Kubelet also manages the Pod-level cgroup hierarchy.

An OOM kill does not necessarily terminate the whole container: the kernel may kill a process other than its main process. If the main process dies, the runtime observes the container exit, PLEG detects it, and kubelet reconciles it according to restart policy. Status commonly reports  OOMKilled .


# Reservation for core components


What each setting protects

┌────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Setting                            │ Intended budget                                                                                                                 │
├────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ kubeReserved                       │ Kubernetes node daemons, such as kubelet and containerd/CRI-O.                                                                  │
├────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ systemReserved                     │ OS daemons such as systemd, journald, sshd, plus headroom for kernel memory.                                                    │
├────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ evictionHard                       │ Emergency headroom: triggers eviction when available memory or storage falls below a threshold.                                 │
├────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ enforceNodeAllocatable             │ Selects which resource budgets kubelet enforces through cgroups.                                                                │
├────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ kubeReservedCgroup /               │ Identify existing cgroups containing the corresponding daemons, if enforcing those budgets.                                     │
│ systemReservedCgroup               │                                                                                                                                 │
├────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ reservedSystemCPUs                 │ Specifies CPU IDs reserved for host/Kubernetes threads, rather than just reserving a CPU quantity. Process affinity needs       │
│                                    │ corresponding configuration.                                                                                                    │
└────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘



kubeReserved:
  cpu: "500m"
  memory: "1Gi"
systemReserved:
  cpu: "500m"
  memory: "1Gi"
evictionHard:
  memory.available: "1Gi"
mergeDefaultEvictionSettings: true
enforceNodeAllocatable:
  - pods

For memory, the advertised scheduling budget is roughly:

16 GiB capacity
 − 1 GiB kubeReserved
 − 1 GiB systemReserved
 − 1 GiB hard-eviction headroom
 = 13 GiB Node Allocatable

The scheduler fits Pod requests within that 13 GiB—not their actual usage or total limits.

Does reservation guarantee protection?

Reservation values alone primarily reduce advertised capacity. They do not allocate private RAM to daemons, move processes into cgroups, or guarantee those processes cannot be OOM-killed.

With  enforceNodeAllocatable: [pods] , kubelet also applies aggregate resource controls to the Pod cgroup hierarchy. In this implementation, that cgroup budget subtracts the two reservations; the eviction headroom is handled separately.

Adding  kube-reserved  or  system-reserved  enables enforcement on the specified daemon cgroups. Those cgroups must already exist and contain the right processes. This also constrains the daemons themselves: setting their memory budget too low can cause their own OOMs.

The useful mental model is: reservations budget normal operation; eviction thresholds provide a safety margin; cgroups provide enforcement.

  
