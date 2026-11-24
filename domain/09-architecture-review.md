# 9 · Architecture Review — a DMMF lens

Reviewed at `dbf5b52`. Findings are ordered by impact. Each cites the code it
is grounded in. "✅" marks things Grain already does exemplary well.

## What Grain gets right

| ✅ | Observation |
|---|---|
| **Lifecycle as types** | `::appendable-event` vs `::event`: ids/timestamps are *assigned by append*, and candidates that carry them are rejected (`event_store_v3/interface/schemas.clj`). This is DMMF ch. 6 ("model each state as a type") done properly. |
| **Failures as values** | Every workflow returns a `cognitect.anomalies` map; handler exceptions and `nil` are converted to `::fault` at the shell (`command_processor_v2/core.clj:11-38`). Railway-oriented programming without the monad ceremony. |
| **No aggregates; DCB instead** | Consistency is expressed as a CAS query + predicate evaluated under the tenant lock. Boundaries are per-decision rather than per-entity class — more flexible than aggregates and naturally functional. |
| **Pure core / imperative shell** | `control_plane/assignment.clj` is explicitly "no IO, data in, data out"; read models are pure folds; effects in reactors are thunks. |
| **Fail-closed retention** | Bounded history is inert until a durable activation whose normalized policy equals the loaded one (§6). Dangerous states are unreachable by accident. |
| **Topology as a value** | `defeventmodel` + a closed edge grammar + a strict boot guard turns "architecture" into something a test can fail. This is the project's real moat. |

## Findings

### R-1 · Reactor return is an *implicit* union — an effect with no checkpoint is silently dropped  `important`
`todo_processor_v2/core.clj:226-229`:
```clojure
(if (:result/effect result)
  (case (:result/checkpoint result)
    :after  (process-effect-after context result)
    :before (process-effect-before context result))   ; no default clause
  (process-pure-result context result))
```
`{:result/effect f}` without `:result/checkpoint` (or with a typo like `:afer`)
throws `IllegalArgumentException`, is caught by the `Throwable` handler, and —
unless `retry-on-error?` — **checkpoints past the event with no effect run**.
The illegal state is representable *and* silently destructive.
*Recommendation:* make `Reaction` a closed union (§5) — e.g. validate the
return against a Malli `:multi` keyed on a discriminator, or split into
`:result/at-most-once` / `:result/at-least-once` keys so the delivery
guarantee is part of the case name, not an optional field.

### R-2 · `decide` and `commit` are fused; composition needs a mode flag  `design`
`execute-command` both runs the handler and appends; composition therefore
needs `:command-processor/skip-event-storage` threaded through context.
DMMF would expose two functions: `decide : Context -> Decision` (pure) and
`commit : Decision -> Result<PersistedEvent list>`. Composition becomes plain
function composition of `decide`s, and the boolean disappears.

### R-3 · Four "result" vocabularies for one concept  `language`
`:command-result/events` + `:command/result`, `:result/events` +
`:result/effect` + `:result/on-success`, `:query/result`. Each block returns
"events to append, optionally guarded, plus a reply". A single
`Decision`/`Reaction` vocabulary would let one validator, one doc, and one
mental model cover all of them — directly serving the "zero variance" goal.

### R-4 · The teaching example races on its own invariant  `important (docs)`
`example_service/core/commands.clj` `create-counter` checks name uniqueness
against a projection but appends **without a CAS guard**. Two concurrent
creates with the same name both succeed. Since this file is what humans and
agents copy, it should model the canonical DCB pattern:
```clojure
:command-result/cas {:types #{:example/counter-created}
                     :predicate-fn (fn [evs] (not-any? #(= name (:name %)) evs))}
```
(Tagging the event with `[:counter-name name-hash]` would let the guard be an
indexed tag read instead of a type scan.)

### R-5 · `:authorized?` is optional at definition but mandatory at use  `design`
`defcommand`/`defquery` accept a missing `:authorized?`; adapters then deny
(`command_request_handler_v2/core.clj:85-86`). Deny-by-default is correct, but
the *type* lets you write an unreachable command and find out at runtime. Make
access explicit: `:access :internal` or `:authorized? f` — require one at macro
expansion (`defevent` already rejects a missing `:schema`, so there is
precedent).

### R-6 · Mutually exclusive read options encoded as predicates  `nit`
`:as-of` / `:after` exclusivity is an `[:fn as-of-or-after]` check; `:reverse?`
and `:limit` are *ignored* for batch reads (protocol docstring). Both are
"make illegal states unrepresentable" opportunities: a `Position` union and
distinct `Single` / `Batch` request types (§2). At minimum, reject rather than
ignore `:limit` on a batch.

### R-7 · Flat event map merges envelope and payload  `trade-off`
Body keys are merged into the event beside `:event/*` keys. It's ergonomic in
reducers, but collision is only prevented by a validation rule, and schema
evolution can't distinguish envelope from payload. Keep the flat *read*
shape, but consider documenting `:event/*` (and `:grain/*`) as a reserved
namespace enforced on every `defevent` schema.

### R-8 · Smaller items  `nit`
* `todo_processor_v2/core.clj:220` — `"Todo Processor returned nil: %s"` is
  never `format`ted.
* `control_plane/assignment.clj:26` — an unknown strategy keyword silently
  falls back to round-robin; prefer `Incorrect` at `start`.
* Retention durations are ISO strings (`"PT1H"`) in definitions; parse once
  into `Duration` at `defevent` expansion so downstream code never sees text.
* Versioned component names (`-v2`, `-v3`) coexist in one workspace. They are
  *implementation generations*, not bounded contexts; the `domain/` names
  (Ledger, Decision, Projection, …) are the stable vocabulary.

## Suggested next steps

1. Fix R-1 (smallest change, highest blast radius) with a return-shape schema.
2. Fix R-4 so the canonical example teaches DCB.
3. Introduce `Decision` / `Reaction` Malli schemas (R-2, R-3) as *aliases*
   over today's keys — non-breaking — then validate handler returns in dev.
4. Require explicit access at macro time (R-5).
