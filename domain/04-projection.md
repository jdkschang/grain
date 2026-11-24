# 4 · Projection (read side)

*Components:* `read-model-processor-v2` (+ `l1_cache`), `kv-store-lmdb`,
`query-processor`, `query-request-handler`, `datastar`.
*Macros:* `defreadmodel`, `defquery`.

## Types

```fsharp
/// A read model is a *fold* — the most DMMF-native concept in Grain.
type Reducer<'s> = 's -> PersistedEvent -> 's

type ReadModel<'s> = {
  Name      : BlockName
  Consumes  : NonEmptySet<EventType>        // :events
  Version   : PositiveInt                   // bump ⇒ cache invalidated, full replay
  Initial   : 's
  Reduce    : Reducer<'s>
  Layout    : CacheLayout
  L1Ttl     : Duration }                    // 0 = always check ledger

and CacheLayout =                           // chosen automatically except Partitioned
  | Monolithic                              // < 10K keys
  | Segmented of segments: int              // ≥ 10K keys, 64 hash segments
  | Partitioned of
      partitionOf : ('entity -> PartitionKey) *   // on STATE, not on events
      entityOf    : (PersistedEvent -> EntityId)

type Scope =
  | Whole
  | ByTags      of Tags
  | ByFilters   of NonEmptyList<Filter>     // overrides the Consumes-derived filter
  | OnePartition of PartitionKey

/// Cache identity — tenant isolation is part of the key, by construction.
type ProjectionKey = { Tenant: TenantId; Model: BlockName; Version: int; ScopeHash: Hash }

type Snapshot<'s> = { State: 's; Watermark: EventId option }

type Project<'s> = TenantId -> BlockName -> Scope -> GrainResult<'s>
```

## Workflow: `Project` (incremental fold with two-tier cache)

```text
key = ProjectionKey(tenant, model, version, hash scope)
L1 hit within TTL ─────────────────────────────────────────────▶ state
else snapshot ← L1 ∪ L2(LMDB, Fressian) ∪ {Initial, None}
     events   ← Read(tenant, filterOf scope, After snapshot.Watermark)
     state'   ← fold Reduce snapshot.State events          (pure)
     write L1; write-back L2 when ≥10 new events
     ─────────────────────────────────────────────────────────────▶ state'
```

Correctness rests on one law: **`project = fold reduce initial (read all)`**.
The cache is an optimisation of that equation and must never change its value.

## Queries

```fsharp
type Query = { Id: Guid; Name: QueryName; IssuedAt: OffsetDateTime; Payload: Map }

type QueryHandler = QueryContext -> GrainResult<QueryResult>   // must not append
type QueryResult  = { Result: Value }                          // :query/result (data or hiccup)

let processQuery = lookupHandler >=> validateShape >=> authorize >=> runQuery
```

## Invariants

| # | Invariant |
|---|---|
| P1 | Reducers are pure and total over their `Consumes` set (unknown type ⇒ identity). |
| P2 | Two tenants never share a cache entry. |
| P3 | A version bump discards all cached snapshots for that model. |
| P4 | Partition moves: an entity lives in exactly one partition after each event. |
| P5 | Queries cause no state change in the ledger. |
