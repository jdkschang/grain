# 7 · Coordination (control plane)

*Component:* `control-plane`. Event-sourced on the system tenant of the shared
Postgres ledger — no external coordination service.

## Types

```fsharp
type NodeId = NodeId of UuidV7                      // ordering ⇒ election

type NodeEvent =                                    // :grain.control/*
  | Heartbeat     of NodeId * metadata: Map         // windowed: PT1H, keep-latest per node
  | NodeDeparted  of NodeId
  | LeaseAcquired of NodeId * TenantId
  | LeaseReleased of NodeId * TenantId

type NodeStatus = Active | Stale                    // derived: now - lastHeartbeat > threshold

// read models (pure folds over NodeEvent)
type ActiveNodes    = Map<NodeId, { LastSeen: RecordedAt; Metadata: Map }>
type LeaseOwnership = Map<TenantId, NodeId>

type Strategy =
  | RoundRobin
  | Custom of (ActiveNodes -> Set<TenantId> -> LeaseOwnership -> Assignment)

type Assignment = Map<NodeId, Set<TenantId>>        // a partition: each tenant exactly once
```

## Pure core (`assignment.clj` — "no IO, data in, data out")

```fsharp
let coordinator : ActiveNodes -> NodeId option      // min NodeId
let assign      : ActiveNodes -> Set<TenantId> -> LeaseOwnership -> Strategy -> Assignment
let leaseDiff   : LeaseOwnership -> Assignment -> NodeEvent list    // acquires + releases
let routeFor    : LeaseOwnership -> ActiveNodes -> NodeId -> TenantId -> Route

and Route =
  | Local  of reason: Owner | NoOwner               // NoOwner = graceful degradation
  | Remote of owner: NodeId * address: string       // ⇒ 503 + Retry-After, sticky cookie
```

## Imperative shell (loops)

```text
every heartbeat-interval:   append Heartbeat(self)
if self = coordinator:      diff ← leaseDiff(current, assign(...)); append diff
every reconcile:            my ← LeaseOwnership⁻¹(self); start/stop tenant pollers to match
on stop:                    drain; append NodeDeparted(self)
```

## Invariants (property test-kit `cp1…cp8`)

| # | Invariant |
|---|---|
| C1 | Lease exclusivity: at most one owner per tenant at any watermark. |
| C3 | Coordinator convergence: all nodes agree once heartbeats settle. |
| C5 | Failover liveness: a stale node's tenants are reassigned. |
| C6 | Lease fencing: a node that lost a lease stops processing (lease-check in A4). |
| C7 | Catch-up completeness: the new owner resumes from the checkpoint. |
| C8 | Tenant isolation is preserved across reassignment. |
