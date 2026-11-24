# 2 · Event Ledger (core domain)

*Components:* `event-store-v3` (protocol, in-memory), `event-store-sqlite-v3`,
`event-store-postgres-v3`. *Allium:* `event-store-v3.allium`.

The ledger is the only stateful thing in Grain. Every other context is either
a pure function over it or a client that appends to it.

## Types — the event lifecycle as distinct types

DMMF: *model each lifecycle stage as its own type* rather than one record with
optional fields. Grain already does this well for events.

```fsharp
type EventBody = Map<Keyword, Value>     // must NOT contain :event/id | :event/timestamp

/// Stage 1 — what a handler produces (via ->event). No identity yet.
type EventCandidate = {
  Type : EventType
  Tags : Tags
  Body : EventBody }

/// Stage 2 — what the ledger returns. Identity and time are assigned by append.
type PersistedEvent = {
  Id         : EventId
  RecordedAt : RecordedAt
  Type       : EventType
  Tags       : Tags
  Body       : EventBody }

/// The reified transaction marker, appended with every commit.
type Transaction = {
  EventIds : NonEmptySet<EventId>
  Metadata : Map option }                // :grain/tx  — protected, never compacted
```

## Reading — position and filter

```fsharp
type Position =                          // replaces the :as-of / :after exclusivity predicate
  | FromStart
  | AsOf  of EventId                     // ≤ id
  | After of EventId                     // > id

type Filter = {
  Tags  : Tags option                    // AND-of-tags (indexed)
  Types : Set<EventType> option }

type Direction = Ascending | Descending

type ReadRequest =
  | Single of TenantId * Filter * Position * Direction * limit: PositiveInt option
  | Batch  of TenantId * NonEmptyList<Filter * Position>    // merged, deduped, id-ordered
                                                            // (direction/limit not applicable)

type EventStream = Reducible<PersistedEvent>                // streamed, never eagerly realised
```

## Appending — the consistency guard

Grain has **no aggregates**. The consistency boundary of a decision is the
set of events matched by its guard — a *dynamic consistency boundary*.

```fsharp
type ConsistencyGuard = {
  Filter    : Filter
  Position  : Position
  Predicate : EventStream -> bool }      // true ⇒ commit, false ⇒ Conflict

type AppendRequest = {
  Tenant     : TenantId
  Events     : EventCandidate list
  TxMetadata : Map option
  Guard      : ConsistencyGuard option }

type Append = AppendRequest -> GrainResult<PersistedEvent list>
type Read   = ReadRequest   -> GrainResult<EventStream>
type Tenants = unit -> Map<TenantId, { LastEventId: EventId option }>
```

## Workflow: `Append`

```text
AppendRequest
  │ validate candidates (schema registry; no persistence metadata)   → Incorrect
  │ acquire per-tenant lock (advisory lock / BEGIN IMMEDIATE / mutex)
  │ evaluate Guard against committed stream                          → Conflict
  │ assign ids  > watermark, strictly increasing; one RecordedAt
  │ write events + Transaction marker; advance watermark
  ▼ PersistedEvent list    (caller MUST use these, not its candidates)
```

## Invariants (proved in `event-store-v3.allium` + backend test-kits)

| # | Invariant |
|---|---|
| L1 | Ids within a tenant are strictly increasing in commit order. |
| L2 | All events of one append, and its `Transaction`, share one `RecordedAt`. |
| L3 | An append is atomic: all events + tx marker, or none. |
| L4 | The guard is evaluated under the same lock as the write (no TOCTOU). |
| L5 | A read never returns another tenant's events. |
| L6 | Only *Retention* may delete, and only with a receipt (see §6). |

## Backends (interchangeable by `{:conn {:type …}}`)

| Backend | Isolation mechanism | Deployment |
|---|---|---|
| in-memory | atom per tenant | tests, REPL |
| SQLite v3 | WAL, single writer | embedded single process |
| Postgres v3 | RLS + per-tenant advisory lock | multi-node (+ control plane) |
