# Grain — Domain Specification

> A type-first description of Grain's own domain, written in the style of
> Scott Wlaschin's *Domain Modeling Made Functional* (DMMF).
> Notation is F#-flavoured pseudocode — it is a **design language**, not
> code to compile. Each type maps onto an existing Malli schema, protocol,
> or macro in `components/`; the mapping is given beside each section.

Grain's *problem domain* is not counters or todo lists — it is **building
long-lived information systems with zero architectural variance**. Its
users are application authors (human or AI). Its nouns are Commands,
Events, Read Models, Queries, Processors and Periodic Tasks; its verbs are
*decide*, *append*, *project*, *react*, *schedule*, *compact*, *coordinate*.

## Reading order

| # | File | Bounded context | Owned by (components) |
|---|------|-----------------|-----------------------|
| 0 | [`00-ubiquitous-language.md`](00-ubiquitous-language.md) | — | glossary |
| 1 | [`01-shared-kernel.md`](01-shared-kernel.md) | Shared Kernel | `event-store-v3/schemas`, `anomalies`, `schema-util` |
| 2 | [`02-event-ledger.md`](02-event-ledger.md) | **Event Ledger** (core) | `event-store-v3`, `-sqlite-v3`, `-postgres-v3` |
| 3 | [`03-decision.md`](03-decision.md) | **Decision** (write side) | `command-processor-v2`, `command-request-handler-v2` |
| 4 | [`04-projection.md`](04-projection.md) | **Projection** (read side) | `read-model-processor-v2`, `query-processor`, `query-request-handler`, `kv-store-lmdb` |
| 5 | [`05-automation.md`](05-automation.md) | **Automation** | `todo-processor-v2`, `periodic-task`, `event-tailer`, `pubsub` |
| 6 | [`06-retention.md`](06-retention.md) | **Retention** | `event-retention`, `event-store-v3/compaction` |
| 7 | [`07-coordination.md`](07-coordination.md) | **Coordination** | `control-plane` |
| 8 | [`08-specification.md`](08-specification.md) | **Specification** (meta) | `event-model`, `event-model-validator`, `code-agent-tools` |
| 9 | [`09-architecture-review.md`](09-architecture-review.md) | — | findings & recommendations |

Visual summary: [`grain-domain-map.png`](grain-domain-map.png) /
[`grain-domain-map.pdf`](grain-domain-map.pdf).

## Context map

```text
                         ┌───────────────────────────┐
                         │   SPECIFICATION (meta)    │  conformist to all
                         │  EventModel · Allium link │  runtime registries
                         └─────────────┬─────────────┘
                                       │ validates (strict boot guard)
        ┌──────────────┐   decide     ▼        project    ┌──────────────┐
 HTTP ─▶│   DECISION   │──────────────────────────────────▶│  PROJECTION  │─▶ Screen
        │ Command→Evts │      ┌──────────────────┐         │ Fold · Query │
        └──────┬───────┘      │   EVENT LEDGER   │◀────────┴──────────────┘
               │ append(CAS)  │  (Shared Kernel: │  read(Position, Filter)
               └─────────────▶│  Event, Tag,     │
        ┌──────────────┐      │  TenantId, Tx)   │      ┌──────────────┐
        │  AUTOMATION  │─────▶│                  │◀─────│  RETENTION   │
        │ React·Tick   │◀─────│                  │      │ Policy·Receipt│
        └──────┬───────┘ tail └────────▲─────────┘      └──────────────┘
               │ lease-check           │ heartbeats / leases (system tenant)
               ▼                       │
        ┌──────────────────────────────┴──┐
        │          COORDINATION           │  Customer/Supplier → Automation
        │ Node · Coordinator · Lease      │
        └─────────────────────────────────┘
```

* **Event Ledger** is the *core domain* and the shared kernel: every other
  context depends on its `Event`, `Tag`, `TenantId` and `EventId` types and
  on nothing else of each other's.
* **Coordination** and **Retention** *eat their own dog food*: their state is
  itself event-sourced (system tenant), so they are clients of the Ledger,
  not privileged infrastructure.
* **Specification** is a *conformist* context: it does not own runtime
  behaviour, it reconciles a declared topology against the runtime registries.

## The one-line architecture (DMMF "pipeline" view)

```fsharp
type GrainSystem =
  UnvalidatedCommand
    -> Result<ValidatedCommand, Anomaly>      // parse, don't validate
    -> Decision                                // pure: Context -> Decision
    -> Result<PersistedEvent list, Anomaly>    // impure shell: append w/ CAS
    -> Projection                              // pure fold over the ledger
    -> QueryResult                             // pure read
    // ...and Reactions close the loop:  PersistedEvent -> Reaction -> Command/Event
```
