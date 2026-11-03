## Overview
1. Generic PLEG globally queries the runtime roughly every 1 second
2. Each relist lists Pods/containers; detailed status calls are made for changed Pods or those needing reinspection—not necessarily every container.

List runtime Pods/containers
          ↓
Compare previous and current states
          ↓
Fetch detailed status for changed Pods
          ↓
Update kubelet’s runtime-status cache
          ↓
Send lifecycle events to kubelet

1. Periodically inspect the runtime

 Start  launches a worker loop. The configured global relist period is 1 second ( kubelet.go:215 ).

 Relist  calls:

podList, err := g.runtime.GetPods(ctx, true)

 true  includes exited containers, not just running ones. The timer resets after relisting finishes, so this is not a guaranteed observation every wall-clock second.

2. Compare snapshots

For each Pod UID,  podRecords  holds:

•  old : the previously processed runtime snapshot.
•  current : the newly listed snapshot.

 reconcilePodRecord  compares container IDs from both snapshots, including Pod sandboxes. Including the old snapshot lets it detect containers that disappeared.

For example:

Previous: container A = running
Current:  container A = exited
Result:   ContainerDied

 generateEvents  maps changes to events:

┌───────────────────────────────────────────────────────┬──────────────────────────────────┐
│ Observed change                                       │ Event                            │
├───────────────────────────────────────────────────────┼──────────────────────────────────┤
│ Becomes running                                       │ ContainerStarted                 │
├───────────────────────────────────────────────────────┼──────────────────────────────────┤
│ Becomes exited                                        │ ContainerDied                    │
├───────────────────────────────────────────────────────┼──────────────────────────────────┤
│ Previously exited container disappears                │ ContainerRemoved                 │
├───────────────────────────────────────────────────────┼──────────────────────────────────┤
│ Running container disappears without an observed exit │ ContainerDied + ContainerRemoved │
├───────────────────────────────────────────────────────┼──────────────────────────────────┤
│ No state change                                       │ No event                         │
└───────────────────────────────────────────────────────┴──────────────────────────────────┘

Unknown-state changes generate  ContainerChanged  internally, but this implementation filters that event out before delivery.

3. Refresh status before notifying kubelet

For a changed Pod,  updateCache  retrieves detailed runtime status and stores it in kubelet’s Pod runtime-status cache.

The ordering matters: when kubelet receives the event and schedules reconciliation, updated status is already available.

If status retrieval fails, PLEG holds back the events and retries inspection later rather than advancing the old snapshot.

4. Deliver the event—not perform the recovery

PLEG sends events through  eventChannel , exposed by  Watch() .

Kubelet’s  syncLoopIteration  consumes them and schedules the affected Pod for synchronization. The Pod worker and runtime manager then handle restart policy, backoff, and container operations.


## PLEG Unhealthy when Container runtime doesn't return


The sequence in this checkout is:

1. PLEG’s health check fails.  runtimeErrors()  includes an error such as  PLEG is not healthy: ... .
2. The main sync loop pauses event processing, logging  Skipping pod synchronization . It retries the health check with exponential backoff, capped at 5 seconds. This does not cancel Pod workers already executing.
3. The separate Node-status loop reports  Ready=False , reason  KubeletNotReady , with the PLEG error in the message—assuming the API server is reachable.
4. PLEG continues relisting. Once it becomes healthy again, the main loop resumes. The Node can return to  Ready  if its other readiness checks also pass.

Existing containers are not automatically killed, and kubelet does not automatically restart itself. They may keep running, but kubelet’s ability to detect and correct workload changes is impaired.

Cluster-side handling of  NotReady  can subsequently restrict scheduling and trigger eviction, depending on taints, tolerations, and timing—not an immediate PLEG-driven deletion.
