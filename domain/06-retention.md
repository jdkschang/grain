# 6 · Retention (bounded history)

*Components:* `event-retention`, `event-store-v3/compaction`,
`event-store-v3/event_definition`. *Macro:* `defevent`.
*Normative spec:* `components/event-retention/event-retention.allium`.

The best-modelled context in Grain: it is a textbook DMMF **state machine**
in which the dangerous state (compacting) is reachable only through an
explicit, durable, matching activation.

## Types

```fsharp
type HistoryContract =
  | Complete                                           // default: omitted :history
  | Windowed of WindowPolicy

and WindowPolicy = {
  RetainAtLeast : Duration                             // "PT1H" ⇒ parsed & normalised
  KeepLatestPer : Set<EntityType> option }             // newest event per tag key kept

type EventDefinition = {
  Type    : EventType
  Doc     : string
  Schema  : MalliSchema                                // required
  History : HistoryContract
  Source  : { Ns: Symbol; File: string; Line: int } }

type Protection =                                       // never eligible, by construction
  | TransactionMarker                                  // :grain/tx
  | Receipt | PolicyLifecycle
  | Unregistered                                       // no defevent ⇒ eternal
```

## The lifecycle (per event type)

```fsharp
type RetentionState =
  | Eternal                                            // Complete, or unregistered
  | Declared   of WindowPolicy                         // in code, inert
  | Active     of activationId: EventId * WindowPolicy // durably activated, matches code
  | Mismatched of active: WindowPolicy * declared: HistoryContract  // FAIL-CLOSED

// transitions (each is an appended, eternal event in the system tenant)
// Declared  --activate!-->   Active          (:grain.retention/policy-activated)
// Active    --deactivate!--> Declared        (:grain.retention/policy-deactivated)
// Active    --code edit-->   Mismatched      (no compaction until re-activated)
```

```text
            defevent :history          activate!
  Eternal ─────────────────▶ Declared ───────────▶ Active ──┐ compact (bounded batch)
     ▲                         ▲   ◀───deactivate!──┘  │     └─▶ CompactionReceipt
     │ remove :history         │                       │ code edits policy
     └─────────────────────────┴──────── Mismatched ◀──┘  (fail closed)
```

## Workflow: `Compact`

```fsharp
type CompactionReceipt = {                             // eternal
  ActivationId : EventId
  EventType    : EventType
  Policy       : WindowPolicy
  Tenant       : TenantId
  Cutoff       : RecordedAt                            // now - RetainAtLeast
  Deleted      : NonEmptySet<EventId> }

type Compact = TenantId -> EventType -> BatchSize -> GrainResult<CompactionReceipt option>
```

1. Require `Active`, with normalized(activation) = normalized(loaded definition).
2. Select candidates older than `Cutoff`, excluding the latest per `KeepLatestPer` key.
3. Bound by `BatchSize`; delete and append the receipt **in one transaction**.

## Invariants

| # | Invariant |
|---|---|
| R1 | Nothing is deleted without a receipt naming it. |
| R2 | Only `Active` & matching ⇒ eligible. Any doubt ⇒ eternal. |
| R3 | A `Windowed` type must be modelled and have **no reactor subscribers** (boot guard). |
| R4 | Compaction never removes the newest event per keep-latest key. |
| R5 | Removing a policy cannot resurrect compacted events (documented, irreversible). |
