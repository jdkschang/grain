# 8 · Specification (the meta-domain)

*Components:* `event-model`, `event-model-validator`, `code-agent-tools`.
*Macro:* `defeventmodel`. Grain's thesis — *eliminate architectural variance* —
lives here: the topology of an application is itself a typed value that is
checked against the running registries.

## Types

```fsharp
type BlockKind =
  | Command | Event | ReadModel | Query | Reactor | Periodic | Screen

type BlockRef = BlockKind * BlockName               // identity is (kind, name)

type Block =
  | CommandBlock   of schema: Schema * reads: Set<ReadModelRef> * produces: Set<EventRef>
  | EventBlock     of schema: Schema
  | ReadModelBlock of consumes: Set<EventRef> * version: int option
  | QueryBlock     of schema: Schema * reads: Set<ReadModelRef>
  | ReactorBlock   of subscribes: Set<EventRef> * reads: Set<ReadModelRef> * produces: Set<CommandRef>
  | PeriodicBlock  of Schedule * produces: Set<EventRef | CommandRef>
  | ScreenBlock    of queries: Set<QueryRef> * commands: Set<CommandRef>   // design-only

type AlliumRef = { Spec: RepoPath; Kind: AlliumKind; Name: string }
and AlliumKind = Actor | Contract | Entity | Enum | Invariant | Rule | Surface | Value | Variant

type ServiceArea = {
  Name   : AreaName
  Blocks : Map<BlockRef, Block * AlliumRef list>
  Flows  : Map<FlowName, Edge list> }

and Edge = { From: BlockRef option; To: BlockRef option }   // None = entry / terminus
```

## The legal grammar (the only edges that typecheck)

```text
command        → event
event          → read-model
read-model     → command | query
query          → screen  | reactor
screen         → command
reactor        → command
periodic       → command | event
```

This is a **closed state machine of information flow**. It is the strongest
architectural constraint in Grain: e.g. a screen can never read a read model
directly, a reactor can never append without a command (in the model).

## Workflows

```fsharp
type ValidateTopology  = ServiceArea list -> Registries -> Verdict    // runtime, pure
type ValidateComposition = ServiceArea list -> AlliumSpecs -> Verdict // CI only
type VerifyOrThrow     = unit -> unit                                 // boot guard

and Verdict = { Fatal: Finding list; Warnings: Finding list; Coverage: Coverage }
```

**Composition gate:** every Command ⇢ ≥1 Allium `Rule`; every Screen ⇢ ≥1
Allium `Surface`; every reference resolves.

**Strict boot:** full coverage of live blocks; declared `produces`/`reads` on
macros agree with the model; windowed events are modelled and unsubscribed.
