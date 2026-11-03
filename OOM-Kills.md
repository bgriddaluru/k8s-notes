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
