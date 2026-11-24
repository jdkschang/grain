# 1 · Shared Kernel

Types every context depends on. *Constrained simple types* — DMMF ch. 5 —
are created only through smart constructors; you cannot hold an invalid one.

```fsharp
// ── identity ───────────────────────────────────────────────────────────
type TenantId   = private TenantId of Guid
type EventId    = private EventId of UuidV7       // time-ordered; total order per tenant
type EntityId   = EntityId of Guid
type EntityType = EntityType of Keyword
type EventType  = EventType of QualifiedKeyword   // :area/name
type AreaName   = AreaName of Keyword             // service area (= bounded context)
type BlockName  = BlockName of AreaName * Name    // :area/name

module EventId =
  let create (u: Guid) : Result<EventId, Anomaly> =
    if uuidVersion u = 7 then Ok (EventId u) else Error (incorrect "Must be UUID v7")

// ── time ───────────────────────────────────────────────────────────────
type RecordedAt = RecordedAt of OffsetDateTime    // UTC, µs precision, assigned by append

// ── indexing ───────────────────────────────────────────────────────────
type Tag  = { EntityType: EntityType; EntityId: EntityId }
type Tags = Set<Tag>

// ── failure as data ────────────────────────────────────────────────────
type AnomalyCategory =
  | Incorrect    // schema / shape invalid
  | Forbidden    // authorization denied
  | NotFound     // unknown command/query/name
  | Conflict     // CAS guard failed / invariant violated
  | Fault        // handler threw, returned nil, store failed
  | Unavailable | Interrupted | Busy | Unsupported

type Anomaly = { Category: AnomalyCategory; Message: string; Explain: Map option }

type GrainResult<'a> = Result<'a, Anomaly>
```

| Type | Malli / code |
|---|---|
| `TenantId` | `::tenant-id :uuid` |
| `EventId` | `::id ::uuid-v7` (`[:fn uuid-v7?]`) |
| `Tag` | `::tag [:tuple ::entity-type ::entity-id]` |
| `RecordedAt` | `::timestamp [:time/offset-date-time]` |
| `Anomaly` | `cognitect.anomalies` map + `:error/explain` |

**Kernel invariants**

1. *No cross-tenant operation exists.* Every function that touches the ledger
   takes exactly one `TenantId`; a batch read of N filters shares one.
2. *Failures are values.* Every workflow returns `GrainResult`; exceptions are
   caught at the shell and converted to `Fault`.
3. *Ids are the clock.* Ordering, "as-of", "after" and watermarks are all
   expressed in `EventId`, never wall time.
