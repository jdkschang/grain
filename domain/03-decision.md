# 3 · Decision (write side)

*Components:* `command-processor-v2`, `command-request-handler-v2`.
*Macro:* `defcommand`. *Allium:* `command-processor-v2.allium`.

## Types

```fsharp
type CommandName = CommandName of BlockName

type UnvalidatedCommand = Map<Keyword, Value>             // from HTTP / REPL / Datastar

type Command = {                                          // ::command + per-name schema
  Id        : Guid
  Name      : CommandName
  IssuedAt  : OffsetDateTime
  Payload   : Map }                                       // validated by the schema of Name

type CommandContext = {
  Tenant  : TenantId
  Command : Command
  Ledger  : LedgerReader                                  // read + project, no append
  Extra   : Map }                                         // app deps (auth claims, clock…)

/// The pure heart.  DMMF: "decide" is a function, not a method on an aggregate.
type Decision =
  | Accepted of AcceptedDecision
  | Rejected of Anomaly                                   // Forbidden | Conflict | NotFound | …

and AcceptedDecision = {
  Events : EventCandidate list                            // may be empty
  Guard  : ConsistencyGuard option
  Reply  : Map option }                                   // :command/result

type CommandHandler = CommandContext -> Decision

type Access =                                             // see review R-5
  | Guarded of (CommandContext -> bool)                   // :authorized?
  | Internal                                              // no adapter exposure
```

## Workflow: `ProcessCommand`

```fsharp
type ProcessCommand =
  UnvalidatedCommand -> TenantId -> GrainResult<CommandReply>

// as a railway (each step can only short-circuit with an Anomaly)
let processCommand =
       lookupHandler      // NotFound   — deny-by-default registry
  >=>  validateShape      // Incorrect  — ::command ∧ schema(Name)
  >=>  authorize          // Forbidden  — adapter: (authorized? ctx) must be literally true
  >=>  decide             // pure       — handler; nil/throw ⇒ Fault
  >=>  commit             // impure     — Append with Guard; guard failure ⇒ Conflict
  >=>  reply              // PersistedEvents + Reply
```

```text
 HTTP ─▶ [lookup] ─▶ [validate] ─▶ [authorize] ─▶ ( decide ) ─▶ [commit] ─▶ reply
           │NotFound    │Incorrect     │Forbidden     │Rejected     │Conflict/Fault
           └────────────┴──────────────┴──────────────┴─────────────┴──▶ Anomaly
```

## Composition

A parent handler may call a child command with storage suppressed
(`:command-processor/skip-event-storage`), collect its events, and emit them
in its own Decision so they commit **once**, atomically. In DMMF terms this is
simply *calling `decide` without `commit`* — see review R-2.

## Invariants

| # | Invariant |
|---|---|
| D1 | An unregistered command produces no events (deny by default). |
| D2 | No handler runs on a shape-invalid command. |
| D3 | A `Rejected` decision stores nothing. |
| D4 | All events of a decision land in the command's tenant only. |
| D5 | Guard failure surfaces as `Conflict`, never `Fault`. |
