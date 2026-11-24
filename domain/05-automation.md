# 5 · Automation (reactions and schedules)

*Components:* `todo-processor-v2`, `periodic-task`, `event-tailer`, `pubsub`.
*Macros:* `defprocessor`, `defperiodic`. Event Modeling calls this the
**automation** / "todo list" pattern: *event → reactor → command/event*.

## Reactor types

```fsharp
type ReactorName = ReactorName of BlockName

type Reactor = {
  Name      : ReactorName
  Topics    : NonEmptySet<EventType>        // subscriptions (trigger wiring)
  React     : ReactionContext -> Reaction
  OnError   : ErrorPolicy }

and ErrorPolicy = SkipAndCheckpoint | RetryWithoutCheckpoint   // retry-on-error?

/// The return of a reactor *as it should be typed*: four mutually exclusive cases.
and Reaction =
  | Emit        of EventCandidate list * ConsistencyGuard option   // pure; batch checkpoint
  | AtMostOnce  of Effect * onSuccess: EventCandidate list          // checkpoint → effect
  | AtLeastOnce of Effect * onSuccess: EventCandidate list
                         * onFailure: EventCandidate list           // effect → checkpoint
  | Acknowledge                                                     // {}
  | Failed      of Anomaly

and Effect = unit -> Async<unit>             // the ONLY place side-effects live

type Checkpoint = {                          // :grain/todo-processor-checkpoint
  Processor   : ReactorName
  TriggeredBy : EventId                      // high-watermark per (tenant, processor)
  Kind        : Single | BatchRange of from: EventId }
```

### Workflow: `ProcessEvent`

```text
event ─▶ lease-check(tenant) ─no─▶ skip
          │yes
          ▼
       React ctx ─▶ match Reaction
          Emit         → Append(events + Checkpoint, CAS: newest checkpoint < event)
          AtMostOnce   → Append(onSuccess + Checkpoint) ; if not Conflict → run Effect
          AtLeastOnce  → if already checkpointed → skip
                         else run Effect → Append(onSuccess|onFailure + Checkpoint)
          Acknowledge  → Append(Checkpoint)
          Failed/throw → OnError
```

**Catch-up**: on start (or lease gain) read `After lastCheckpoint` for the
reactor's topics and replay; `:grain/tx` markers are skipped.

## Periodic types

```fsharp
type Schedule =
  | Cron     of expr: CronExpr * zone: ZoneId option     // UNIX cron
  | Interval of every: PositiveInt * unit: Seconds | Minutes | Hours

type Periodic = {
  Name     : BlockName
  Schedule : Schedule
  Tick     : TenantId -> Instant -> Emit }   // always Emit with a CAS guard
```

Every node runs every schedule; the **CAS guard makes the tick idempotent
across the cluster** — the first node to append wins, others get a silent
`Conflict`. A periodic never does work itself: it is always paired with a
Reactor subscribed to its trigger event.

## Live delivery

```fsharp
type Tailer  = TenantId -> Set<EventType> -> Observable<PersistedEvent>   // polls ledger
type PubSub  = Topic -> Observable<PersistedEvent>                        // node-local
```

The tailer bridges the shared ledger to each node's local pub/sub so that an
SSE stream (Datastar) on *any* node sees events appended on *any* node.

## Invariants

| # | Invariant |
|---|---|
| A1 | For a (tenant, reactor), checkpoints advance monotonically. |
| A2 | Emit + Checkpoint commit atomically: no duplicate emission across lease moves. |
| A3 | AtMostOnce never runs an effect twice; AtLeastOnce never skips one. |
| A4 | A reactor runs only on the lease holder for that tenant. |
| A5 | One periodic tick ⇒ at most one trigger event per tenant per period. |
