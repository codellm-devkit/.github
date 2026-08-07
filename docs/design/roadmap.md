# CLDK roadmap

**Pass:** 2026-08-07
**Planned with:** Rahul Krishna
**Status:** current  (supersede by editing, not by adding a second roadmap)

Theme: **bring every `codeanalyzer-*` to canonical schema v2 consistently, then bring
`python-sdk` to v2 with strong backwards compatibility.**

This pass opened on microservice analysis and pivoted. The microservice work is
retained at the bottom as the initiative this one unblocks — the survey done for it is
what surfaced the divergence below.

## The finding

`docs/design/specs/` and the ladder's `canonical-schema.md` already state the v2
contract: one scale-free node with `id` / `kind` / `span` / children, containment as the
single-parent relation, typed overlays (`call_graph`, `cfg`, `cdg`, `ddg`, `summary`,
`param_in`, `param_out`), durable `can://<lang>/<app>/<file>/<type>/<sig>` ids above the
callable line and `@line:col` / `@tag` ordinals below it, source stored once per module.
The contract is not the gap.

The gap is that **every analyzer implements it differently, and each one's conformance
test checks it against its own declared schema rather than against the canonical one.**
That is exactly how `codeanalyzer-python` and `codeanalyzer-typescript` both read
`SCHEMA_VERSION = "2.0.0"` while emitting materially different graphs.

Measured against the emitters, not the docs:

| | `codeanalyzer-python` 2.0.0 | `codeanalyzer-typescript` 2.0.0 | `codeanalyzer-java` 1.1.0 |
| --- | --- | --- | --- |
| `can://` ids | yes | yes | no |
| Neo4j merge labels | **9** — `PyApplication`, `PyModule`, `PySymbol`, `PyPackage`, `PyDecorator`, `PyCallSite`, `PyAttribute`, `PyVariable`, `PyCFGNode` | **2** — `Application`, `CanNode` | `JApplication`, `JSymbol`, + per-kind |
| Statement/body model | split — `PyCallSite` + `PyCFGNode` | unified — `TSBodyNode` with a `kind` discriminant | **absent from Neo4j** — deferred, `SCHEMA_DECISIONS.md` #9 |
| `cfg` / `cdg` / `ddg` / `summary` / `param_in` / `param_out` | yes, all six | yes, all six | callable→callable only (`J_CONTROL_DEP` / `J_DATA_DEP` / `J_HEAP_DATA_DEP`) |
| Decorators in `analysis.json` | structured `PyDecorator` | structured `TSDecorator` (`SCHEMA_DECISIONS.md` #4) | flat `annotations: string[]` |
| Decorators in **Neo4j** | yes — `PyDecorator` + `PY_DECORATED_BY` | **no** | no |
| Entrypoints | none | `TSApplication.entrypoints` — `analysis.json` only | `JEntrypoint` marker + `is_entrypoint` — Neo4j only |
| `MARKER_LABELS` | `[]` | `[]` | `["JEntrypoint"]` |

Three structural problems fall out.

**1. The two projections disagree inside a single analyzer.** Canonical v2 states plainly
that `analysis.json` and the Neo4j graph are two projections of one structure and *they
must agree*. TypeScript emits structured decorators and an entrypoint collection to JSON
and neither reaches its graph. Nothing checks this today, in any analyzer.

**2. The body-node model is the deepest divergence.** TypeScript's unified `TSBodyNode`
is what canonical v2 describes — one node kind ladder, `kind` as the discriminant,
statements and synthetic vertices sharing an id space. Python splits the same concept
across `PyCallSite` and `PyCFGNode`. Java has neither in its graph. Every downstream
query — dataflow, taint, cross-service reachability — is written against this shape, so
it cannot be papered over at read time.

**3. Entrypoint vocabulary has already been coined three times.** `JEntrypoint` as a
Neo4j marker label, `TSEntrypoint` as a JSON collection, nothing in Python. Canonical v2
does not define entrypoints at all, which is why it drifted. Under the parity clause a
term coined twice is coined wrong permanently — this one is at three and rising.

## Candidates

| # | Feature | Moves schema v2? | Collision group | Blocked by |
| - | ------- | ---------------- | --------------- | ---------- |
| 1 | **Canonical Neo4j projection contract** — merge-label strategy, language namespacing, what "always full-depth" obliges | yes — the projection vocabulary | A | — |
| 2 | **Unified body-node model** — one body/statement node with a `kind` discriminant, replacing the split `PyCallSite` / `PyCFGNode`, and defining what Java must add | yes — node kinds and local-id grammar | A | — |
| 3 | **Projection-parity gate** — CI proof that `analysis.json` and the Neo4j graph carry the same facts for the same run | yes — makes the two-projection clause enforceable | A | 1, 2 |
| 4 | **`can://` grammar conformance** — the per-language `signatureOf()` pinned for each analyzer, including Java which has no `can://` today | yes — identity | A | — |
| 5 | **Shared conformance suite** — each analyzer checked against *canonical* v2, not its own declared schema | yes — defines "conformant" | A | 1, 2, 4 |
| 6 | **Entrypoint vocabulary** — reconcile `JEntrypoint` / `TSEntrypoint` / nothing into one definition present in both projections | yes — new shared vocabulary | B | 1 |
| 7 | `codeanalyzer-python` migration to the settled contract | yes — merge labels, body-node model | — | A |
| 8 | `codeanalyzer-typescript` migration — chiefly closing the Neo4j projection gap (decorators, entrypoints) | yes — additive to its graph | — | A |
| 9 | `codeanalyzer-java` migration — `can://` ids, statement-level CPG in Neo4j (`SCHEMA_DECISIONS.md` #9 follow-up), v1.1.0 to v2 | yes — major | — | A |
| 10 | `codeanalyzer-clang` — assess and migrate; has no `SCHEMA_DECISIONS.md` and no declared schema version | unknown until assessed | — | A |
| 11 | Greenfield analyzers born conformant — go, dotnet (epic #34), kotlin, rust, swift, abap | yes — never migrate | — | A |
| 12 | **`python-sdk` v2 model layer** — one `Node` / `Edge` / `Application` mirroring canonical v2, replacing the four per-language model packages | no — SDK model layer | C | 7, 8, or 9 (any one conformant analyzer) |
| 13 | **Backwards-compatibility policy for the SDK** — what of the existing surface is preserved, how deprecation is signalled, how long the old models keep parsing | no — SDK contract | C | — |
| 14 | SDK TypeScript Neo4j backend queries `(:Application)-[:HAS_MODULE]->(:Module)` while the analyzer emits `TSApplication` / `TS_HAS_MODULE` / `TSModule` | no — reconciliation | — | — |

## Collision groups

- **Group A — the canonical projection contract**: candidates 1, 2, 3, 4, 5.

  One design session. These are not five decisions that happen to be adjacent; they are
  one decision seen from five sides. The conformance suite (5) cannot check ids it has
  not pinned (4) or a body model it has not chosen (2); the parity gate (3) is
  meaningless without the projection contract (1); and the merge-label strategy (1) is
  decided *by* the body-node model (2), since `TSBodyNode`'s unified label is what makes
  two merge labels sufficient where Python needs nine.

  This group is the keystone of the whole pass: it is what turns "bring everything to
  v2" from a slogan into a checkable predicate.

- **Group B — entrypoint vocabulary**: candidate 6.

  Alone in its group by content, but listed because it is the one piece of *new* shared
  vocabulary this pass coins, and it has already drifted three ways. It could fold into
  Group A's session; kept separate because it is also the hand-off point to the
  microservice initiative, and because deferring it does not block candidates 7–11.

- **Group C — SDK v2 surface**: candidates 12, 13.

  The model layer and the backwards-compatibility policy are one decision. What the new
  models look like and what of the old surface survives cannot be settled separately —
  "strong backwards compatibility" is a constraint on the new model's shape, not a
  wrapper bolted on afterwards.

Candidates 7–11 and 14 consume vocabulary and coin none. They are sequencing, not
contract.

## Dependency order

    A  (projection contract + body-node model + can:// + parity gate + conformance suite)
     │
     ├─▶ 6  entrypoint vocabulary        [Group B; may run inside A's session]
     │
     ├─▶ 7  codeanalyzer-python   ─┐
     ├─▶ 8  codeanalyzer-typescript ├─ any ONE of these unblocks 12
     ├─▶ 9  codeanalyzer-java      ─┘
     ├─▶ 10 codeanalyzer-clang     (assess first — may be larger than a migration)
     └─▶ 11 greenfield: go, dotnet (#34), kotlin, rust, swift, abap

    C  (12 SDK v2 model + 13 backcompat policy)   ◀── after any one conformant analyzer
     └─▶ (microservice initiative resumes here)

    14 SDK TypeScript backend drift — independent; fix early, it is cheap and it makes
       candidate 8's result observable from the SDK

Candidate 13 has no blocker and could be settled first: deciding the backwards-compat
policy *before* the model layer is designed is what stops the policy becoming whatever
the implementation happened to make easy.

## Release trains

| Train | Carries | Notes |
| ----- | ------- | ----- |
| — (spec + test suite) | 1, 2, 3, 4, 5, 6 | no release; the deliverable is a spec and a conformance suite |
| `codeanalyzer-python` 3.0.0 | 7 | **major** — merge labels and body-node model change. The 2.0.0 label is currently inaccurate, so the bump is a correction as much as a migration |
| `codeanalyzer-java` 2.0.0 | 9 | **major** — `can://` ids, statement-level CPG |
| `codeanalyzer-typescript` 2.1.0 | 8 | additive MINOR — it is closest to canonical; the work is projecting to Neo4j what it already computes |
| `codeanalyzer-clang` TBD | 10 | scope unknown until assessed |
| greenfield initial releases | 11 | born conformant, never migrate |
| `python-sdk` | 12, 13, 14 | version depends on candidate 13's policy — see below |

**Python and Java ride one migration, together with the SDK.** Two analyzer majors
landing separately would force the SDK through two compatibility windows.

The SDK's version is candidate 13's decision, not a foregone conclusion: a v2 model
layer added *alongside* the per-language models, with the old surface delegating, is a
minor; replacing them is a major. "Strong backwards compatibility" points at the former,
but the cost is carrying two model layers until the deprecation window closes.

## Not now

**The list that makes the rest mean something.**

- **The whole microservice initiative** — multi-application read-time normalization, the
  service-boundary vocabulary, cross-service edges, cross-service dataflow, and the
  `SystemAnalysis` facade. It was this pass's original theme and is deliberately
  deferred behind schema consistency: a facade spanning five languages built on graphs
  that disagree about merge labels and body nodes would encode the divergence into its
  own surface. Candidate 6 is the piece kept in scope, because the entrypoint vocabulary
  is coined here whether or not the microservice work starts.
- **Non-HTTP service boundaries** — brokers, gRPC, GraphQL. Downstream of the deferred
  initiative.
- **`codeanalyzer-go` analysis levels L2 to L4** — Go's entry to this roadmap is
  candidate 11, being born conformant. Depth for Go is a separate initiative.
- **Unified canonical SDK model classes replacing per-language facades** — candidate 12
  builds the shared model layer; whether `PythonAnalysis` / `JavaAnalysis` /
  `TypeScriptAnalysis` eventually collapse into one facade is a later question, and
  candidate 13's backwards-compatibility policy likely forbids it for now.

## Starting now

**Group A — the canonical projection contract.** Spec only, no implementation, into
`docs/design/specs/`.

No epic is filed. Epics are filed just-in-time when implementation starts, and a
spec-only pass has no pull request to close. The first epic under this roadmap will be
whichever of candidates 7 through 11 is picked up once Group A's spec exists to point
at — with candidate 14 available as a cheap independent starter at any time.

Everything else on this roadmap has no issue yet, by design.
