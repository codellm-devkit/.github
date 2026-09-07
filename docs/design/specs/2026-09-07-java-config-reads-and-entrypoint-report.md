# Spec: Java config-read edges and the entrypoint report

Status: draft for review
Date: 2026-09-07
Scope: codeanalyzer-java (analyzer), python-sdk (Java facade, deferred), cross-repo
Origin: codeanalyzer-java#231

---

## 1. Summary

codeanalyzer-java gains two things codeanalyzer-python already emits:

1. **Config-read facts.** Java projects 9,688 `:ConfigKey` nodes but no relationship saying which
   code reads one, so `DEFINES_CONFIG` is the only config relationship in the graph. This spec adds
   `application.config_uses[]` and `application.config_reads_unresolved[]` to `analysis.json`, and
   `J_USES_CONFIG` / `J_READS_CONFIG_UNRESOLVED` to the Neo4j projection.
2. **The entrypoint report.** `:JApplication` carries exactly `name`, `schema_version`,
   `analyzer_name`, `analyzer_version`. Java marks 2,604 `:JEntrypoint` nodes but says nothing about
   how the pass that found them behaved, so a consumer cannot tell "this application has no
   entrypoints" from "the detection pass found nothing". This spec adds
   `application.entrypoint_report{}` to `analysis.json` and the two properties python already
   projects onto its application root.

### What this spec does not do

Issue #231 listed a third goal — projecting comment nodes — on the stated grounds that
codeanalyzer-python projects them. **It does not.** There is no `:PyComment` node type in
`codeanalyzer/neo4j/schema.py`; `project.py:843` collapses comments to a single `docstring`
property, which is exactly what `V2GraphProjector.docstringOf` does in Java. The python-sdk Neo4j
backend documents this as a deliberate ceiling:

> Projection-lossy fields (inherent to what the graph stores): comments collapse to a single
> docstring (module-level comments dropped)
> — `python-sdk/cldk/analysis/python/neo4j/neo4j_backend.py:73`

So the goal is closed by fact: there is no parity gap and no python term to match. Projecting
comment nodes remains available as a future change, but it would be a **new** capability with no
precedent, not the parity fix #231 described. Out of scope here.

Also out of scope: CRUD (codeanalyzer-java#187), the `can://` id missing from the `:JApplication`
root (codeanalyzer-schema#5), and codeanalyzer-typescript's identical config gap.

---

## 2. Contract-impact triage

**Does this change the schema v2 output?** Yes, additively, in both projections. The spine is
untouched: no shared field renamed, no shared `kind` repurposed, no rich-edge variant, and every new
node reachable from `application` by containment.

| Addition | Projection | Rubric slot |
| --- | --- | --- |
| `application.config_uses[]` | `analysis.json` | application-scope edge list, the slot `param_in`/`param_out` occupy |
| `application.config_reads_unresolved[]` | `analysis.json` | application-scope list |
| `application.entrypoint_report{}` | `analysis.json` | typed field on the root |
| `J_USES_CONFIG` | Neo4j | new relationship type |
| `J_READS_CONFIG_UNRESOLVED` | Neo4j | new relationship type |
| `entrypoint_frameworks` on `type` and `callable` | `analysis.json` | typed field (see D5a) |
| `entrypoint_frameworks`, `entrypoint_report_json` | Neo4j `:JApplication` | new node properties |
| `entrypoint_frameworks` | Neo4j `:JType`, `:JCallable` | new node property |

**Repos affected.**

| Repo | Deliverable | This cycle? |
| --- | --- | --- |
| `codeanalyzer-java` | detection pass, entrypoint report, both projections | yes |
| `python-sdk` | Java facade config + entrypoint-coverage accessors | **no — deferred** |
| docs | contract note | with the analyzer |

### The graph contract version does not move

`V2SchemaCatalog.SCHEMA_VERSION` is held at `2.0.0`. The class javadoc's "graph contract 2.2.0" is
stale text; the constant's own comment records that this analyzer previously drifted to 2.1.0 /
2.2.0 / 3.0.0 alone while codeanalyzer-python stayed at 2.0.0, and that the drift was undone. Moving
the number is a coordinated re-baseline across all three analyzers, tracked at
codellm-devkit/.github#50 — not a local bump, and not this spec's to make.

**Recorded consequence.** These additions are additive over labels the 2.0.0 baseline already
reserves, so they ride the held version the same way the repository-artifact layer does. The cost is
that a consumer **cannot detect the new relationships from `schema_version` alone**: a 2.0.0 graph
may or may not carry `J_USES_CONFIG`. Detection is by presence, not by version, until #50 lands.
This is the same gap the `_module` removal already lives with, and it is written down here rather
than papered over.

---

## 3. The python anchor

What is being matched, so the parity claim in this spec is checkable.

| Concern | codeanalyzer-python |
| --- | --- |
| Resolved read | `PY_USES_CONFIG` — `PyBodyNode → ConfigKey`, props `{prov}` (`schema.py:296`) |
| Unresolved read | `PY_READS_CONFIG_UNRESOLVED` — `PyApplication → PyExternal`, props `{key, reason, prov, _k}` (`schema.py:305`) |
| JSON model | `PyApplication.config_uses: List[PyConfigUseEdge]`, `.config_reads_unresolved: List[PyConfigRead]` (`py_schema.py:631`) |
| Edge src | GLOBAL ordinal id `<callable-id>@<local-id>` of a `call` node |
| Tiers | literal from `-a 2`; dataflow widens at L3/L4; `prov ∈ {literal, dataflow}` |
| Report model | `PyEntrypointReport{frameworks_detected, rulesets, unresolved, errors}` (`py_schema.py:274`) |
| Report projection | `entrypoint_frameworks: string[]` + `entrypoint_report_json: string` on `:PyApplication` (`project.py:78`) |

---

## 4. Design decisions

Each was put to the user; the divergence and its cost are recorded, not just the answer.

### D1 — `src` widens to the annotated element

**The divergence.** Python's model anchors a config read on a `call` body node, because Python's
config reads *are* calls (`os.getenv("X")`, `settings.get("X")`). Java's dominant idiom is an
annotation — `@Value("${db.url}")` on a field or constructor parameter, `@ConfigurationProperties`
on a class — which has no call node to anchor on. Call-site reads (`System.getenv`,
`Environment.getProperty`, `Properties.getProperty`) do have one.

**Decision.** One edge type, with a **union `src` endpoint set**:

| Read idiom | `src` | Neo4j endpoint |
| --- | --- | --- |
| `System.getenv("X")`, `env.getProperty("X")`, `props.getProperty("X")` | call body-node id | `:JBodyNode` |
| `@Value("${x}")` on a field | field id | `:JField` |
| `@Value("${x}")` on a parameter | the owning callable's id | `:JCallable` |
| `@ConfigurationProperties("x")` on a class | type id | `:JType` |

`J_USES_CONFIG` is therefore declared `from: [JBodyNode, JCallable, JField, JType] → to: [ConfigKey]`.

**Cost, stated.** Java's `src` endpoint set is wider than python's. A cross-language consumer cannot
assume the source of a config-read edge is a body node. The alternative — call sites only — is
literal parity that reports almost nothing on a Spring application, which is most of the daytrader8
corpus the issue measured on. Widening the endpoint set is additive and breaks no shared name; a
`src` that silently omits Spring would be a correct-looking empty, the exact failure mode the
unresolved-read list exists to prevent.

**Rejected:** minting a synthetic body node per annotation read. It preserves python's endpoint
shape but invents a body-node `kind` with no AST region behind it, which is a new entry in the
shared node-kind vocabulary — a higher parity cost than widening one endpoint list.

**Two shapes this decision left open, settled during implementation.**

*An unresolved annotation read has no callee to ghost.* `@Value("${undeclared.key}")` is routine —
the value comes from the environment rather than a checked-in file — but `J_READS_CONFIG_UNRESOLVED`
runs to a `:JExternal` ghost of the callee, and an annotation is not a call. Decided: **ghost the
annotation type** (`can://java/<app>/@external/org.springframework.beans.factory.annotation.Value/value()`).
An annotation is not literally a callee, but `@Value` injection *is* a read and the ghost names what
performed it, so every unresolved read projects one edge shape rather than some carrying no endpoint
at all. Rejected: using the existing `:JAnnotation` node, which would widen `dst` to a union too; and
projecting no edge, which would leave the read visible in `analysis.json` and invisible in the graph
— the exact backend split #231 complained about.

*`@ConfigurationProperties("datasource")` binds a prefix, not a key.* Python has no prefix-binding
idiom, so there is no precedent to match. Decided: **one `J_USES_CONFIG` per declared key beneath the
prefix**, because `get_config_readers("datasource.url")` finding the binding class is the accessor's
actual question. Fan-out is bounded by the declared key set. Rejected: a single edge naming the
prefix, since no `:ConfigKey` node exists for a bare prefix and the `dst` would dangle.

### D2 — Naming: `J_` prefix

`J_USES_CONFIG` and `J_READS_CONFIG_UNRESOLVED`, matching python's `PY_`-prefixed pair and Java's
existing convention for every relationship with a language-specific endpoint.

The counter-argument was real and is recorded: the `dst` is `:ConfigKey`, an **unprefixed
cross-language merge target**, and `DEFINES_CONFIG` is unprefixed for exactly that reason. An
unprefixed `USES_CONFIG` would let one `MATCH` answer "which code reads this key" across every
analyzer over the same repository. It was rejected because python has already shipped
`PY_USES_CONFIG`: adopting the unprefixed name means either renaming there or living with two
spellings of one concept, and a term spelled twice is the failure the parity clause names. The
consequence — a cross-language config-read query needs one `MATCH` per language — is accepted.

### D3 — Tiers: literal at L1, dataflow at L3/L4

Python gates its literal tier at `-a 2`. Java's `call` body nodes already carry `argument_expr`,
`receiver_type` and `method_name` **at L1** (`JBodyNode`), and annotation reads are pure L1 data, so
nothing in the literal tier needs a call graph.

- **L1** — literal tier. A detector-matched read whose key argument is a string literal (or an
  annotation whose value is a literal `${...}` placeholder), resolved against the declared
  `:ConfigKey` set. `prov: ["literal"]`.
- **L3** — intra tier. A bare-name key closes when every DDG-reaching definition at the call site is
  the same single string literal. `prov: ["dataflow"]`.
- **L4** — interprocedural tier. A key naming a parameter closes when the callable never rebinds it
  and *every* call site targeting it supplies the same literal — directly, or through one
  caller-side hop of the intra tier. `prov: ["dataflow"]`.

All three ship in this change. Monotonicity holds by construction: a read closed at a lower tier is
never recomputed, and a tier only converts an unresolved read into an edge. So
`config_uses(-a 1) ⊆ config_uses(-a 3) ⊆ config_uses(-a 4)`, the same additive contract as the DDG's
`ssa` → `points-to` widening.

**Three refusals that make the tiers safe**, each as load-bearing as the closures — a tier that
closes a read it should not have emits an edge claiming code reads a key it never reads, which is
strictly worse than the unresolved record it replaced:

1. *Any non-closing reaching def kills the resolution*, rather than being skipped. Two paths
   assigning different literals means the read is genuinely ambiguous, and picking one would be a
   confident wrong answer.
2. *The intra tier consults only `prov` containing `ssa`, and this is what keeps `-a 3 ⊆ -a 4` true.*
   At `-a 4` the DDG also carries `points-to` edges that may-alias the variable's use to an unrelated
   write, which is not a `name = "literal"` shape. Because a non-closing def kills resolution,
   admitting an alias edge would *remove* an edge the ssa-only L3 set resolved cleanly — a widening
   that breaks the additive contract. codeanalyzer-python documents hitting this first.
3. *An incomplete call-site set does not close.* A callee whose callers the analyzer could not fully
   see may be handed a different key elsewhere. Known ceiling, stated rather than papered over: a
   `public` method can be called from outside the analyzed project, which no in-project call graph
   rules out — the same whole-application assumption python's tier makes.

The DDG use site is matched by span **containment**, not id equality: the CFG/DDG is statement-level
while a `call` body node is keyed by its own narrower span, so a def's recorded use site is the
enclosing statement. Containment covers both the bare-expression-statement case and the common
`return env.getProperty(key);` nesting without special-casing either.

**Divergence from python, stated:** the same flag yields different availability on the two
analyzers (`-a 1` answers on Java, returns empty on python). Java has the data a level earlier and
withholding it would be an artificial gate.

### D4 — Unresolved reads mint their ghost inline

Not a divergence; python's precedent is followed directly. `J_READS_CONFIG_UNRESOLVED` runs
`:JApplication → :JExternal`, mirroring `PY_READS_CONFIG_UNRESOLVED`'s
`PyApplication → PyExternal` shape and `J_IMPORTS`'s treatment of an unresolved target.

The `:JExternal` ghost for the callee is **minted at detection time** (python's `_call_endpoint`
shape), not looked up in the L2 external-symbol set. Without this, an unresolved read would be
invisible at L1 and absent whenever `--external-calls` is off — precisely the runs where the
literal tier is the only tier available.

Edge properties mirror python: `{key, reason, prov, _k}`, `reason ∈ {non-literal, undefined-key}`,
`_k = "<key>|<reason>"`. The discriminant is load-bearing: one callee (`System.getenv`) legitimately
reads many undeclared keys across a codebase, and a plain endpoint-pair MERGE would collapse them
onto one relationship and keep only the last `SET`.

### D5 — Entrypoint report: python's four keys, honestly populated

`JEntrypointReport{frameworks_detected, rulesets, unresolved, errors}`, identical to
`PyEntrypointReport` so one SDK model parses `entrypoint_report_json` from either analyzer.

| Key | Java meaning |
| --- | --- |
| `frameworks_detected` | which of the five finders actually matched something |
| `rulesets` | the five finder names (Spring, JaxRs, Jakarta, Struts, Camel) — that **is** Java's ruleset vocabulary, hardcoded rather than data-driven |
| `unresolved` | count, by finder, of declarations the pass could not decide because that finder threw |
| `errors` | one deduplicated message per finder that failed |

Projected as python does: `entrypoint_frameworks: string[]` plus
`entrypoint_report_json: string` (sorted-key JSON) on `:JApplication`. **Always present, even when
empty** — an absent report and an empty one must not be the same observation, which is the whole
point of the record.

**Known ceiling on `unresolved`/`errors`, recorded rather than left implicit.** They capture a finder
that *throws*. The five finders currently swallow their own resolution failures internally
(`StrutsEntrypointFinder.java:41` logs and returns `false`), so those near-misses are not counted
today, and both fields will usually be empty. Making them count is a change to finder behaviour, and
finder accuracy is out of scope for this spec — a separate change, not a silent gap.

### D5a — Attribution lives on the node, not in a run-scoped tally

Settled during implementation, and it is the load-bearing decision behind D5's
`frameworks_detected`. The finders return booleans and `TypeBuilder`/`CallableBuilder` collapsed them
with `anyMatch`, so *which* finder matched was discarded. Recovering it needs somewhere to put it.

**Decision.** `entrypoint_frameworks: string[]` on `type` and `callable`, with `frameworks_detected`
computed as a union over the built tree at assembly time.

**Why not a run-scoped collector threaded through the builders.** `L1Extractor` reuses a cached
module byte-for-byte and skips the build entirely, so the finders never run for that file. A tally
kept during the walk would under-report `frameworks_detected` on a warm cache — which is precisely
the ambiguous empty this whole record exists to prevent. The tree carries the attribution whether the
module was rebuilt or reused.

**Why not deriving it from the emitted decorators in a later pass.** Cache-immune and schema-free,
but it reimplements the five finders' matching tables in a second place, which drifts the day either
side changes.

This adds two node-level fields the triage table above did not name, and it is *more* python parity
rather than less: `PySymbol` and `PyCallable` already carry `is_entrypoint` alongside
`entrypoint_frameworks` (`schema.py:110-111,133-134`). `is_entrypoint` is retained and is exactly
`entrypoint_frameworks` being non-empty.

### D6 — One edge per matching key, not "exactly one match or nothing"

A read that closes on a literal is resolved against the declared `:ConfigKey` set by namespace
preference order — the first namespace with at least one match wins — and then emits **one
`J_USES_CONFIG` edge per matching key**. A read becomes an `undefined-key` record only when *zero*
keys match.

This follows codeanalyzer-python's resolver directly (`config_use.py:304`, `for key in matched`), and
it is the only rule that survives the real shapes: one key legitimately appears in both
`application.properties` and `application-dev.properties`, and D1's `@ConfigurationProperties` prefix
claims every key beneath it by design. A rule demanding a unique match would drop both cases on the
floor as unresolved, reporting nothing where the honest answer is "several".

Stated because the tracking issue's first draft said "exactly one", which is wrong and would have
been implemented as written.

---

## 5. Decomposition

Epic in `codellm-devkit/.github`; children filed on the repo they change, **just-in-time**, as each
unit is picked up. Each child is one PR.

| # | Repo | Unit | Tracked |
| --- | --- | --- | --- |
| 1 | codeanalyzer-java | literal tier: detectors, resolver, `config_uses` / `config_reads_unresolved` in `analysis.json`, both new relationship types | #232 → PR #233 |
| 2 | codeanalyzer-java | dataflow tiers over the L3 DDG and L4 call graph, `prov: ["dataflow"]` | #236 → PR #237, stacked on #233 |
| 3 | codeanalyzer-java | entrypoint report: model, finder plumbing, node attribution, both projections | #234 → PR #235 |
| 4 | python-sdk | Java facade accessors — **deferred, not this cycle** | not filed |

Children are filed just-in-time as each unit is picked up; a plan mirrored into the backlog ahead of
the work is inventory. Child 4 stays unfiled by the release-plan decision below.

**Ordering note.** Child 2 is stacked on child 1's branch rather than `main`: the tiers extend the
same resolver. Child 3 is independent and branches from `main`. Each branch regenerates
`schema.neo4j.json` against itself, so one regeneration is owed once children 1 and 3 have both
landed — the checked-in artifact is a whole-catalog snapshot, not a per-change diff.

## 6. Release plan

**codeanalyzer-java 3.0.3 → 3.1.0** (MINOR — additive JSON fields, additive graph relationships).
Children 1–3 land on that train. Nothing gates the analyzer.

Graph contract stays `2.0.0` (§2).

**python-sdk is deferred to a later cycle.** Consequence, stated plainly: the analyzer will emit
config-read edges and the entrypoint report that nothing consumes, and #231's actual complaint —
`get_config_readers()` returning an unconditional `[]`, `get_entrypoint_coverage()` reporting
`entrypoint_report_unavailable` — stays unfixed until child 4 ships. The Java facade
(`cldk/analysis/java/`) has **no** config accessor and **no** entrypoint-coverage accessor today, so
that child is new SDK surface, not a read-the-new-field change.

## 7. Verification

- Conformance: `analysis.v2.schema.json` admits the three new application-level members; the L1
  conformance gate stays green.
- Monotonicity: `config_uses(-a 1) ⊆ config_uses(-a 3) ⊆ config_uses(-a 4)` on a fixture with both
  a literal and a dataflow-only read.
- Corpus: on daytrader8, `J_USES_CONFIG` is non-empty and every `dst` resolves to a projected
  `:ConfigKey`; `MATCH (a:JApplication) RETURN any(k IN keys(a) WHERE k CONTAINS 'entrypoint')`
  returns `TRUE`.
- No dangling endpoints: every `src` resolves to a projected `:JBodyNode` / `:JCallable` /
  `:JField` / `:JType`, every unresolved-read `dst` to a `:JExternal`.
- Cache: a warm `-c <dir>` run produces a report byte-identical to the cold one (the D5a invariant).

**What the corpus can and cannot prove, recorded so a reviewer does not over-read the runs.**
daytrader8's config reads are all either literals or undeclared keys — it contains no non-literal
read for a dataflow tier to close — and the other checked-in fixtures have no config reads or no
checked-out sources. So corpus runs demonstrate *monotonicity and the absence of false positives*;
the tier closures themselves are demonstrated by fixtures only. Separately, `--l3-engine wala` needs
a built project, so the configuration where D3's `ssa` filter actually earns its keep is worth one
run against a real build before the train ships.
