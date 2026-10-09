# Spec: heap-mediated dataflow — the `heap_flow` edge family

Status: accepted
Date: 2026-10-09
Scope: schema v2 evolution; three analyzers, the SDK dataflow walk, the keystone — cross-repo

---

## 1. Summary

The canonical schema gains an eighth edge family:

| Edge list | Level | Endpoints | Lives on | Notes |
| --- | --- | --- | --- | --- |
| `heap_flow` | 4 | statement → statement | application | `var` (k-limited access path), `prov: ["points-to"]` |

One statement writes a heap location, another reads it, and no call connects them. The edge
records that flow. It is placed on the **application** because its endpoints sit in different
callables and the keystone places an edge at the lowest common ancestor of its endpoints.

### Why

Cross-callable dataflow today exists **only where a call carries it**. Measured on a TypeScript
graph (codeanalyzer-typescript 1.6.3, 76,352 nodes), joining both endpoints of every dataflow
edge to their owning callable through `TS_HAS_BODY_NODE` — 67,129 body nodes, 67,129 edges, so a
total relation rather than a sample:

| edge | total | intra-callable | cross-callable |
| --- | --- | --- | --- |
| `TS_DDG` | 47,281 | 47,281 | **0** |
| `TS_CDG` | 24,952 | 24,952 | **0** |
| `TS_SUMMARY` | 11,697 | 11,697 | **0** |
| `TS_CFG_NEXT` | 36,677 | 36,677 | **0** |
| `TS_PARAM_IN` | 2,673 | 18 | **2,655** |
| `TS_PARAM_OUT` | 1,645 | 9 | **1,636** |

Every inter-procedural edge is call-mediated. `TS_CALLS` is call-mediated by definition. So the
SDK's SDG walk — which traverses exactly `DDG`, `CDG`, `PARAM_IN`, `PARAM_OUT`, `SUMMARY`,
`CALLS` — cannot see a write-then-read pair with no call between them. `taint()`,
`paths_between`, `flows_to_call` and `flows_to_argument` are all blind to it, identically.

No analyzer offers a substitute. `TS_HAS_FIELD` (1,561), `PY_HAS_ATTRIBUTE` and `J_HAS_FIELD`
declare that a type *has* a field and stop; nothing records which callable reads or writes one.
Searching for the pair a consumer would need: `PY_READS`/`PY_WRITES` appear **zero** times across
`codeanalyzer-python`, `codeanalyzer-python-v2-tip` and `codeanalyzer-python-v0.3.0`, and are
absent from the 25-type python schema and from a live graph.

The pattern this misses is ordinary, not exotic: a stateful service written by one request
handler and read by another, with no call edge between the two handlers. On an Angular or Express
application that is a routine shape, and it is invisible to every verb on the dataflow surface.

## 2. Contract-Impact Triage

**Does this change schema v2 output?** **Yes.** A new edge family is new shared cross-language
vocabulary in the keystone's edge-families table, which had seven entries.

**Which repos are touched:**

| Change | Analyzers | SDKs | Docs |
| --- | --- | --- | --- |
| Schema v2 evolution: `heap_flow` | `codeanalyzer-typescript`, `codeanalyzer-python`, `codeanalyzer-java` | `python-sdk` (SDG walk + models), TS SDK | the keystone; each analyzer's schema docs |

All three analyzers already reach L4 (each emits `PARAM_IN`) and each already carries a field
node, so no analyzer needs new node kinds. `codeanalyzer-iac` is L2 and unaffected.

## 3. The design

`heap_flow` is additive and self-contained:

- **Endpoints**: `statement → statement`, write to read, in dataflow direction. Both endpoints
  already exist at L3; no new node kinds.
- **Placement**: the application, per the lowest-common-ancestor rule.
- **`var`**: the same k-limited access path `ddg.var` already carries (`this.state.pending`). The
  vocabulary for naming a heap location is not new — only the cross-callable placement is.
- **`prov`**: `["points-to"]`, necessarily. Heap resolution is approximate, and `points-to` is
  already the keystone's least-certain tier, so a consumer ranking witnesses by `weakest` places
  a heap hop below an `ssa` one without any new ranking rule.
- **Level**: 4, and only 4. L4 is defined as the level that adds the points-to oracle, which is
  exactly what deciding whether two access paths alias requires.

### Decisions

- **A new family, not a cross-callable `ddg`.** Letting a `ddg` edge be placed on the application
  when its endpoints differ would add no vocabulary and need no SDK walk change at all, since
  `taint()` already traverses `ddg`. It was rejected because consumers rely on the current
  guarantee: the measurement above establishes that `ddg` is 100% intra-callable today, and code
  that joins a `ddg` edge to "the" owning callable is correct now and would silently break. A new
  family keeps "intra-callable edges hang on the callable" true and makes the new capability
  opt-in at every consumer.
- **One direct edge, not a pair through the field node.** `field_write: statement → field` plus
  `field_read: field → statement` would make the field first-class and joinable — "every writer
  of this field" becomes one query — and it is what the consumer-side implementations assumed.
  It was rejected for the walk: a heap step would be two hops through a non-statement node,
  breaking the SDG walk's uniform statement-to-statement shape and costing a pair-traversal
  rewrite rather than adding a relationship type to a list. The join it enables is recoverable
  from `var` plus the containment tree.
- **L4 only; no syntactic L3 variant.** Mirroring how `ddg` spans L3→L4 was tempting: at L3, join
  writes and reads carrying an identical literal access path on an identically declared type,
  cheaply and without aliasing. Rejected because L3 is specified as AST-only and per-callable
  parallel, and any cross-callable join needs a global pass. Honouring L3's cost model matters
  more than covering the easy cases early.
- **No analyzer version floor bump.** The SDK change adds `{TS,PY,J}_HEAP_FLOW` to the walk's
  relationship list. On a graph without the edge the walk matches nothing and every answer is
  what it is today. So the SDK and the analyzers do **not** need version lockstep, which is
  unusual for a schema evolution and is what makes the staged release plan below possible.

### Not decided here

- **The k-limit.** `ddg.var` is k-limited and this reuses that; whether heap paths want the same k
  is an analyzer-side question for the first implementation to measure and report.
- **Whether consumers re-enable heap imputation.** `osprey` carried a `HeapShape` over
  `PY_READS`/`PY_WRITES` and removed it on 2026-10-09 once those edges were shown never to have
  existed. Re-enabling is a consumer decision, out of scope here.
- **Precision reporting.** Whether `heap_flow` should carry a confidence or an aliasing-strength
  attribute beyond `prov`. Defer until one analyzer has emitted it against a real application.

## 4. Decomposition and release plan

An epic in `codellm-devkit/.github` coordinating five work items, each closed by one PR against a
different repository. Children are filed as each is picked up, not up front.

1. **Keystone** — `heap_flow` in the edge-families table and the levels table.
2. **`codeanalyzer-typescript`** — emit it, with measurements against a real application.
3. **`python-sdk`** — the walk token, the models, and the `taint()` docs.
4. **`codeanalyzer-java`** — emit it.
5. **`codeanalyzer-python`** — emit it.

**Order: TypeScript first as proof, then the SDK, then Java and Python.** TypeScript is where the
gap was measured and where a graph exists to validate against, so the contract is proven
end-to-end on one language before two more analyzers commit to it. The SDK lands second so
`taint()` picks the edge up as soon as there is one to pick up. Java and Python follow in any
order and gate nothing.

No lockstep anywhere, because the SDK change is inert on a graph that lacks the edge. A consumer
on an old analyzer sees exactly today's answers.

## 5. Definition of done

- The keystone's edge-families and levels tables carry `heap_flow`.
- All three analyzers emit it at `-a 4`, with `var` and `prov: ["points-to"]`, and no dangling
  endpoints.
- `python-sdk`'s SDG walk traverses it, and `taint()` returns a witness whose `weakest.prov` is
  `points-to` for a write-then-read pair with no call path between the two callables.
- `taint()` answers unchanged on a graph without the edge.
- Each analyzer reports, against a real application, how many `heap_flow` edges it emitted and how
  many sinks gained an entry that no call path reached — the number this family exists to move.
