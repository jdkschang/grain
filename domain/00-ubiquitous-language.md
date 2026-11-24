# 0 · Ubiquitous Language

Terms are used **exactly** as below across code, docs, Allium specs and the
Event Model. A term that drifts between contexts is called out.

| Term | Meaning | Context | Code anchor |
|---|---|---|---|
| **Tenant** | An isolated event stream and everything projected from it. The unit of isolation, locking, leasing and caching. | Kernel | `:tenant-id` |
| **System tenant** | The reserved tenant holding Grain's own facts (leases, heartbeats, retention activations). | Kernel | control-plane / retention |
| **Event** | An immutable, persisted fact. Has an id (UUIDv7), timestamp, type, tags and a body. | Ledger | `::event` |
| **Event candidate** | An event *before* append: type, tags, body — **no** id or timestamp. | Ledger | `::appendable-event`, `->event` |
| **Tag** | `(EntityType, EntityId)` pair that indexes an event for retrieval and for consistency checks. | Ledger | `::tag` |
| **Transaction (Tx)** | A reified `:grain/tx` event naming every event id committed atomically. | Ledger | `:grain/tx` |
| **Watermark** | The id of the last committed event in a tenant; positions are expressed relative to it. | Ledger | `:tenant/last-event-id` |
| **Consistency guard (CAS)** | A query + predicate evaluated *inside* the append transaction; the append commits only if it holds. Grain's replacement for aggregates. | Ledger | `:cas` |
| **Command** | A named request to change state. Validated, authorized, then decided. | Decision | `defcommand` |
| **Decision** | The pure outcome of a command handler: accept with events (and optional guard & result) or reject with an anomaly. | Decision | `:command-result/events` |
| **Anomaly** | A categorised failure *value* (never an exception): `incorrect`, `forbidden`, `not-found`, `conflict`, `fault`, … | Kernel | `cognitect.anomalies` |
| **Read model** | A pure left fold `(State, Event) -> State` over a filtered stream, plus versioning and caching metadata. | Projection | `defreadmodel` |
| **Projection** | The *act* (and cached result) of folding a read model for a tenant and scope. | Projection | `rmp/project` |
| **Scope** | Narrows a projection: by tags, by explicit queries, or to one partition. | Projection | `{:tags …}` |
| **Query** | A named, authorized, read-only composition of projections serving a screen or automation. | Projection | `defquery` |
| **Todo processor** | A reaction: subscribes to event types, and for each event returns a Reaction. | Automation | `defprocessor` |
| **Checkpoint** | A `:grain/todo-processor-checkpoint` event recording a processor's progress for a tenant. | Automation | |
| **Delivery guarantee** | At-most-once (checkpoint *before* effect) or at-least-once (checkpoint *after*). | Automation | `:result/checkpoint` |
| **Periodic task** | A schedule that emits trigger events per tenant, deduplicated by CAS. | Automation | `defperiodic` |
| **Event definition** | Registered metadata for an event type: schema, docs, history contract. | Retention | `defevent` |
| **History contract** | `Complete` (default) or `Windowed` (retain-at-least + keep-latest-per). | Retention | `:history` |
| **Activation** | A durable, explicit decision to enforce a windowed history for one event type. | Retention | `activate!` |
| **Compaction receipt** | Eternal evidence naming every event deleted by one atomic compaction. | Retention | |
| **Node** | One running Grain process; identified by UUIDv7; heartbeats. | Coordination | |
| **Coordinator** | The live node with the smallest UUIDv7; computes tenant assignment. | Coordination | |
| **Lease** | A node's exclusive right to run processors for a tenant. | Coordination | `lease-acquired` |
| **Service area** | The keyword namespace grouping blocks (`:example/…`). The *bounded context* of an application built on Grain. | Specification | `ns-kw` |
| **Block** | One of command / event / read-model / query / processor / periodic / screen in an Event Model. | Specification | |
| **Flow** | An ordered path of legal block → block edges. | Specification | `:flows` |

### Terms that currently drift

* **"Todo"** — means *reactive processor* in Grain, but *todo item* in the
  teaching app (`grain-todo-list`). Prefer **Reactor** / **Reaction** in prose.
* **"Result"** — `:command/result` (caller payload), `:command-result/events`
  (decision), `:result/events` (reaction), `:query/result` (query). Four
  meanings, one word. See review finding R-3.
* **"Query"** — both a `defquery` block *and* an event-store read filter
  (`:queries` in projection scope). Prefer **Filter** for the latter.
