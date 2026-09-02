# Spec: scope every destructive statement on the `can://` id prefix

Status: draft for review
Date: 2026-09-02
Scope: Neo4j projection across all analyzers, cross-repo

---

## 1. Summary

Every statement that deletes goes from matching on labels and an internal `_module` property to
matching on the **`can://` id prefix** of the application being pushed:

```cypher
-- before (java, after codeanalyzer-java#213)
MATCH (x:JCompilationUnit|JType|JCallable|...|JBodyNode) WHERE x._module = $m ...

-- after
MATCH (x:<index-anchor>) WHERE x.id STARTS WITH $appPrefix + '/' ...
```

The `can://` id is already a hierarchical path — `can://<lang>/<app>/<file>/<type>/<signature>` —
so a prefix match *is* containment. Scoping by prefix therefore gets language, application and
module scoping from the identity that already exists, instead of from a denormalized property and a
hand-maintained label list that have to be kept correct by hand.

Two things retire as a consequence:

- **`_module`** — the file key duplicated onto every node. Already the `<file>` segment of the id.
- **the label list as a safety device** — labels stay only where an index requires one (§6).

### Why now

Three defects in six weeks, all the same shape: a delete whose scope was expressed by something
other than identity.

| | what happened |
| --- | --- |
| codeanalyzer-typescript#116 | `MATCH (n) WHERE n._module IS NOT NULL AND NOT n:CanNode DETACH DELETE n` deleted the java and python graphs outright. Surfaced only as an out-of-memory error, because the delete was large enough to exhaust `dbms.memory.transaction.total.max` and roll back. A smaller foreign graph would have gone silently. |
| codeanalyzer-java#213 | `MATCH (x {_module: $m}) DETACH DELETE x` — unlabelled — deleted a sibling analyzer's nodes wherever a file key collided across languages. |
| unfiled, found while writing this | **all three** analyzers' per-module purges are application-blind. `_module` is a bare project-relative path, so two applications *in the same language* that both contain `src/main/java/Foo.java` (or its equivalent) delete each other's nodes for that file. Typescript's own comment — *"a sibling analyzer's nodes sharing this `_module` key are never in scope"* — is true across languages and false across applications. |

The first two are fixed — typescript in `c1c27f3`, java in #213 — and both were fixed by adding
labels to the match. That treats the symptom: it makes the statement match fewer wrong things, while
leaving scope expressed by a property that carries no language, no application, and no guarantee.
The third is what remains in all three analyzers when you fix it that way, and it is the one a label
cannot fix at all, because both applications' nodes carry the same labels.

## 2. Contract-Impact Triage

**Does this change schema v2 output?** The Neo4j projection, yes — `_module` disappears from every
node's properties and from the emitted graph catalog. `analysis.json` is untouched: `_module` is a
projection-level property that never appears there.

**Which repos are touched:**

| Change | Analyzers | SDKs | Docs |
| --- | --- | --- | --- |
| Prune scoping + `_module` removal | `codeanalyzer-java`, `codeanalyzer-python`, `codeanalyzer-typescript` | any SDK whose Neo4j backend reads `_module` — **to verify, not assumed** | graph-contract docs |

Each analyzer's graph contract takes a **MAJOR** bump: a property is removed, which is breaking by
every analyzer's own stated bump policy, even though the property is internal by convention.

## 3. Where the three analyzers stand today

| | java | python | typescript |
| --- | --- | --- | --- |
| `_module` on nodes | yes | yes | yes |
| purge anchored on | 15-label disjunction | 6-label disjunction | `:CanNode` |
| app-scoped purge | **no** | **no** | yes (`id STARTS WITH`) |
| `_module` indexes | **none** | one per label | one (`:CanNode`) |
| deletion gated on `--eager` | **no** | **no** | yes, on every destructive path |
| shared merge label | `JSymbol` (3 of 11 labels) | `PySymbol` (3 of 10) | `CanNode` (all) |

Java and python arrived at the same design independently — a *semantic* shared label covering
call/resolve targets only, and per-label everything else. TypeScript generalised its label to
"anything with a `can://` id", which is why it alone could anchor on one label and index one
property.

## 4. The design

**One rule.** A destructive statement is scoped by the id prefix of the thing it is scoped to:

| scope | prefix |
| --- | --- |
| one application | `can://<lang>/<app>` |
| one module within it | `can://<lang>/<app>/<file>` |

Match the node itself by equality and its descendants by `prefix + '/'` (§6).

**What that buys, structurally rather than by discipline:**

- other languages are out of scope — different `<lang>` segment
- other applications are out of scope — different `<app>` segment, which is what closes the third
  defect above
- shared cross-language nodes are out of scope **by construction**: artifacts are
  `can://artifact/<app>/<path>`, deliberately outside every language namespace, and packages are
  `pkg:<type>/...`, not `can://` at all. Nodes keyed on `name` rather than `id` — packages,
  annotations, decorators — cannot match a prefix predicate on `id` at all.

That last point is the one to weigh. `codeanalyzer-java`'s `CypherWriter.DESCENDANTS` currently
spends thirteen lines of prose defending exactly this invariant, enforced by a hand-maintained list
of relationship types that a future edge kind can silently break. Under this design the id grammar
enforces it and the prose becomes a description rather than a guard.

## 5. What retires

- **`_module`**, and every index on it. `codeanalyzer-java#217` — "add the missing `_module`
  indexes" — is resolved by deleting the property rather than by indexing it.
- **The label list as a correctness device.** Java's `MODULE_OWNED` and python's
  `MODULE_OWNED_PATTERN` exist to stop a match reaching a sibling's graph. The prefix does that.
- **The `NOT n:CanNode` legacy-wipe predicate** in typescript, which describes foreign nodes exactly
  as precisely as it describes stale local ones.

## 6. Mechanics that must not be got wrong

1. **Prefix boundary.** `can://java/app/src/Foo.java` is also a prefix of
   `can://java/app/src/Foo.javaX`. Match descendants on `prefix + '/'` and the node itself by
   equality. A bare `STARTS WITH prefix` is wrong.
2. **Empty prefix.** `STARTS WITH ''` matches every node in the database. Every call site must
   refuse a null or empty application id — typescript already guards this and says why.
3. **An index still needs a label, and it should be language-specific.** Neo4j property indexes are
   label-scoped; there is no label-agnostic property index, so `MATCH (n) WHERE n.id STARTS WITH $p`
   scans the whole store. `STARTS WITH` *is* index-backed on a range index (a prefix seek), unlike
   `CONTAINS` or `ENDS WITH` — so the operator choice here is deliberate, not incidental.

   **Each analyzer carries its own marker label** — `JCanNode`, `PyCanNode`, `TSCanNode` — on every
   node keyed by a `can://` id, and anchors its prunes on that. Naming says the mechanical truth:
   keyed by a `can://` id, and mine.

   Language-specific rather than one shared `CanNode`, for three reasons. The index holds only one
   analyzer's nodes, so it is smaller and three analyzers pushing concurrently do not contend on
   one index. The label re-adds a language guard for free: get the prefix wrong and the blast radius
   is still one language, which makes "an anchor error costs a slow query, not someone's data"
   true rather than merely intended. And it sidesteps a planner question — the alternative of
   anchoring on a disjunction of existing labels, using the `id` indexes the per-label uniqueness
   constraints already create, depends on whether the planner uses indexes for a label disjunction,
   which varies by Neo4j version.

   The cost is the one-label cross-language query: a polyglot consumer asking for "all code nodes
   regardless of language" needs a three-way disjunction instead of `MATCH (n:CanNode)`. If that
   query is wanted, carry both labels (`:CanNode:JCanNode`) — prunes anchor on the narrow one,
   consumers match the wide one. Add the shared label when a consumer needs it, not before.

   One marker per language namespace, not per repository: `JCanNode` (java), `PyCanNode` (python),
   `TSCanNode` (typescript), `JSCanNode` (javascript). The typescript analyzer emits two of these,
   because it covers two language namespaces — the marker follows the `<lang>` segment of the id it
   anchors, so that a prefix predicate and its anchor can never disagree about which language a node
   belongs to.
4. **Batching.** Deleting an application in one transaction exhausts
   `dbms.memory.transaction.total.max` — measured at 2.7 GiB in typescript#116. Use
   `CALL { ... } IN TRANSACTIONS OF N ROWS`.

## 7. Interaction with the `<service>` segment spec

`can-uri-service-segment.md` (draft, 2026-08-07) reorders the grammar to
`can://<service>/<lang>/<file>/...`, collapsing `<app>` into an outermost `<service>`.

**The two are compatible and independent.** This spec depends only on the id being a hierarchical
path whose language and application scope appear as a prefix — true before and after that change.
What changes is the prefix string an analyzer computes, from `can://<lang>/<app>` to
`can://<service>/<lang>`, and both analyzers already derive it from one function.

Worth noting they reinforce each other: under the service grammar, `can://<service>/` scopes a whole
polyglot deployment unit and `can://<service>/<lang>/` scopes one analyzer's share of it — exactly
the two scopes a prune needs. Neither spec should block on the other, and whichever lands second
should confirm the prefix helper is the only place that needs to change.

## 8. Decomposition and release plan

**Deliberately not decided here** — see §9. The shape below is a proposal for that conversation, not
a settled plan.

Per-analyzer work is independent: each repo changes its own writers, drops `_module` from its own
catalog, and cuts its own MAJOR. No lockstep is needed, because no analyzer reads another's nodes —
that is the entire point of the change.

Suggested order, cheapest verification first:

1. **typescript** — already prefix-scopes its prune and gates on `--eager`; the work is retiring
   `_module` and fixing the legacy-wipe predicate, which is the live data-loss bug of the three.
2. **java** — largest gap: no app scoping, no `--eager` gate, no `_module` indexes to remove because
   it never had any.
3. **python** — prefix-scope the purge, then delete the per-label `_module` indexes it added.

Docs follow the last analyzer.

## 9. Open questions for review

1. ~~Does any SDK read `_module`?~~ **Decided: `_module` is dropped.** Each analyzer should still
   grep its SDK's Neo4j backend while implementing, so a consumer that reads it is updated in the
   same train rather than discovered afterwards — but the removal is not conditional on the answer.
2. **Should `--eager` become the universal gate?** TypeScript already refuses to delete without it:
   *"managing the database's lifetime is the operator's call, not the analyzer's."* Java and python
   delete unconditionally. Adopting it is a behaviour change beyond scoping, and it is the other
   half of `codeanalyzer-java#213`'s title. Recommended, but it is a separate decision from where
   the scope comes from.
3. **Does any polyglot consumer need "all code nodes regardless of language"?** §6.3 recommends
   per-language marker labels and adding a shared `CanNode` alongside only if such a consumer
   exists. `cocoa` is the candidate; confirm before deciding.
4. **Tracking shape** — one epic with a child per analyzer, or one issue per repo linking this spec.
