# Spec: `<app>` becomes the outermost `can://` segment, and the application root carries an id

Status: draft for review
Date: 2026-09-07
Scope: canonical schema v2 identity grammar, cross-repo
Supersedes: `can-uri-service-segment.md` (2026-08-07) — see §3
Origin: codeanalyzer-schema#5; codellm-devkit/.github#50

---

## 1. Summary

Two changes to identity, designed together because the second is only half-useful without the
first.

**A. The deployment unit moves to the outermost position.**

```
before   can://<lang>/<app>/<file>/<type>/<callable-signature>
after    can://<app>/<lang>/<file>/<type>/<callable-signature>
```

**B. The application root gets its `can://` id in the graph, and merges on it.**

`analysis.json` already emits `application.id` (`can://java/daytrader8`). The Neo4j projection
drops it: `:JApplication` merges on `name` — the raw `--app-name` string — and carries no `id`
property at all. Same in codeanalyzer-python's `:PyApplication`. This spec closes that.

### Why they belong in one change

Both are the same defect seen from two ends: **the deployment unit is not part of identity.**
A is about node ids not naming their unit; B is about the node that *represents* the unit not
having an id. Fixing B alone leaves the root addressable by an id whose shape still buries the
app inside it; fixing A alone leaves the one node that names the app un-addressable. And they
share a migration — every `can://` string moves — so splitting them means paying that cost twice.

---

## 2. Contract-impact triage

**Does this change the schema v2 output?** Yes, and **not additively**. It reshapes the id of
every durable node (≥ callable) and therefore every edge endpoint referencing one: `call_graph`,
`param_in`/`param_out`, the backfilled `callee`, `base_types`/`interfaces`, `config_uses.dst`,
and every `<callable-id>@<local>` ordinal id derived from a callable id. It also changes the
Neo4j uniqueness key of the application root.

This is a reshaping of a shared element of the grammar, not an addition at the leaves — which is
exactly why it is designed here rather than absorbed per-analyzer.

**Change type:** schema v2 evolution. **Repos affected:**

| Repo | What changes | `can://` refs today |
| --- | --- | --- |
| `codeanalyzer-java` | `schema/CanId.java`, `V2GraphProjector` root node, `V2SchemaCatalog`, `BoltWriter` prefix scoping, `L1Cache` invalidation, fixtures | 180 |
| `codeanalyzer-python` | `codeanalyzer/schema/ids.py` (`_SCHEME`), Neo4j projection root, fixtures | 343 |
| `codeanalyzer-typescript` | `src/schema/ids.ts`, `src/build/neo4j/schema.ts`, fixtures | 344 |
| `codeanalyzer-typescript-v2` | same, on the v2 line | 46 |
| `python-sdk` | v2 models, Neo4j backends, prefix-scoped queries, expected-JSON fixtures | 192 (372 on `release/2.0`) |
| `codeanalyzer-schema` | the shared schema definition | — |
| `cldk-devtools` | `designing-cldk-changes/references/canonical-schema.md` § Identity; `cldk-sdk-frontend/references/schema-contract.md` | the grammar is *defined* here |
| `cldk-skillset` | `codeanalyzer-backend/references/canonical-schema.md` | second copy of the keystone |
| `docs` | identity/grammar page | user-facing |
| `codeanalyzer-dotnet` | absorbs before it is built | greenfield |

**Mitigating fact:** id construction is centralised in exactly one module per analyzer —
`CanId.java`, `ids.py`, `ids.ts`. The large counts above are dominated by expected-output
fixtures, not call sites. The code change is small; the fixture churn is the work.

---

## 3. Re-triage: what changed since `can-uri-service-segment.md`

That spec (2026-08-07, still draft) proposed the same *positional* change under a different name,
`can://<service>/<lang>/…`. Its reasoning has expired in two ways that change the answer.

**Its cost table is stale by two orders of magnitude.** It recorded:

| Repo | Aug 7 spec | Today |
| --- | --- | --- |
| `codeanalyzer-java` | *"nothing — still v1, emits no `can://` ids at all"* | v2 emitter, 180 refs, released 3.1.1 |
| `python-sdk` | *"nothing — pre-v2, zero `can://` references"* | 372 refs on `release/2.0`, v2 models, prefix-scoped queries |

**Its version decision no longer stands.** D3 shipped a breaking id change as MINOR `2.1.0`,
justified by "no consumer holds a `can://` id today: both SDKs are pre-v2, `codeanalyzer-java` is
v1, `codeanalyzer-dotnet` is unbuilt, and `cocoa` is changing in the same pass." Every clause of
that is now false except the last. A breaking change to the id shape can no longer be a MINOR.

**And a hazard it could not have known about.** Since `2026-09-02-prune-scope-on-can-id-prefix.md`,
every destructive Neo4j statement is scoped on the `can://` id *prefix*
(`WHERE x.id STARTS WITH $appPrefix`). The outermost segment is now load-bearing for a **safety**
mechanism, not only for identity. `BoltWriter.java:79` already documents how delicate that
matching is.

**What survives.** The positional argument, unchanged and still correct: with `<lang>` outermost,
one deployment unit written in two languages fragments into `can://python/…` and
`can://typescript/…` with nothing joining them. So do D4 (cross-service RPC edges out of scope)
and D5 (shared code duplicated per unit; the id answers *where does this run*).

---

## 4. Design decisions

### D1 — The segment is `<app>`, not `<service>`

The Aug 7 spec called it `<service>` and renamed `--app-name` to `--service` (its D2). Rejected.

**A monolith has no service.** `can://daytrader/java/…` is a correct id for a monolith;
`can://<service>=daytrader` is a category error the grammar would force on every non-microservice
project — which is most of them. The concept the positional argument actually needs is *the unit
that is built and deployed together*: one for a monolith, one per service in a microservice
system. CLDK already has that word, and it is `app`.

Keeping `<app>` also makes this change strictly cheaper than the superseded one: **the Aug 7 D2
disappears entirely** — no CLI flag rename, no new term coined in a grammar the parity clause
governs, no migration note for `--app-name` users. The default (input directory name) is already
right for both shapes.

**Cost, stated:** for a genuine microservice system the operator must pass a distinct
`--app-name` per service, exactly as the superseded spec required a distinct `--service`. Nothing
in the grammar enforces distinctness; see D3.

### D2 — Position: `<app>` outermost, above `<lang>`

```
can://<app>/<lang>/<file>/<type>/<callable-signature>

can://daytrader8/java/src/main/java/.../TradeAction/doGet(HttpServletRequest)
can://checkout/python/app/cart.py/Cart/checkout(user_id)
can://checkout/typescript/src/cart.ts/CartService/checkout(userId)
```

It is the only position that lets one deployment unit span several languages under a single id
root, and it occupies the URI authority slot, which is what a deployment boundary is.

Two consequences beyond addressability, both improvements:

- **The reserved pseudo-segments regularise.** `can://java/<app>/@external/<type>/<sig>` becomes
  `can://<app>/java/@external/<type>/<sig>`, and the artifact scheme —
  today `can://artifact/<app>/<path>`, a *third* outermost shape that is neither lang nor app —
  becomes `can://<app>/artifact/<path>`. One rule instead of three.
- **The destructive-delete scope becomes expressible.** Today an app analyzed by two analyzers has
  two unrelated prefixes (`can://java/daytrader`, `can://python/daytrader`), so a cross-language
  delete scope cannot be written. After this, `can://<app>/` is one prefix covering every node of
  that app in every language — which is what the scoping spec wanted and could not have.

Ordinal ids (`<callable-id>@<line>:<col>`, `@<tag>`) are untouched; they inherit the new callable
id and the two-tier rule is unchanged.

### D3 — The application root carries its id and merges on it

`:JApplication` / `:PyApplication` gain an `id` property (`can://<app>`), and the uniqueness
constraint moves from `name` to `id`. `name` is retained as a display property.

Three defects this closes, all live today:

1. **Silent collision.** The merge key is free-text `--app-name`. Two services analyzed with the
   same name unify into one node with no diagnostic — the multi-service failure mode that
   motivated this spec.
2. **The root is unaddressable.** Every other v2 node is keyed on its `can://` id; the root alone
   is keyed on a display string, so a consumer holding `can://<app>` cannot reach the node that
   represents it.
3. **The root is invisible to the safety mechanism.** `RowBuilder.node` attaches the `:JCanNode`
   index anchor only when the merge value starts with `can://` (`RowBuilder.java:96,132`). The app
   node's merge value is the bare name, so it never gets the anchor — meaning the prefix-scoped
   destructive statements structurally cannot reach the application root. It is the one node the
   delete scope cannot see.

Under D2 this is more than a fix: `can://<app>` is exactly the prefix every node in the app
shares, so the root becomes the natural anchor for scoped deletes rather than their one exception.

### D4 — This change IS the coordinated re-baseline (#50)

The graph contract is deliberately held at `2.0.0` while `codeanalyzer-java` undoes a unilateral
drift, with `codellm-devkit/.github#50` open for "a coordinated re-baseline across all three
analyzers, not a unilateral bump." That re-baseline has had no forcing function. This change is
one: it is breaking, it touches every analyzer, and it must land in lockstep or the grammar
fragments.

So this spec proposes the re-baseline happen here: **`analysis.json` `schema_version` and the
Neo4j graph contract both move to `3.0.0`**, together, across every analyzer. MAJOR is the honest
number — Aug 7's MINOR argument is dead (§3), and #50 already records a second breaking change
(the `_module` removal) that the held version has been hiding.

### D5 — Out of scope

- **Cross-app edges** (RPC/HTTP/gRPC between deployment units). This spec makes units
  *addressable*; it does not model calls between them. That is a new edge family plus per-framework
  detection — several contract decisions, so it belongs in planning mode. Carried over unchanged
  from the superseded spec's D4.
- **Enforcing app-name distinctness.** The grammar makes collisions *visible* (D3 makes them a
  constraint violation rather than a silent merge); it does not prevent an operator passing the
  same `--app-name` twice.
- **Shared code dedup.** One id per app, duplicated across apps that import the same library. The
  id answers *where does this run*, not *what code is this*. Carried over from the superseded D5.

---

## 5. Migration

**Every persisted artifact is invalidated.** Not just graphs:

- **L1 caches.** `codeanalyzer-java`'s `L1Cache` stores built `JModule` trees keyed by content
  hash, and those trees contain ids. A cache written before this change deserialises into ids of
  the old shape. The cache must be version-stamped and rejected on mismatch, not silently reused.
- **Persisted `analysis.json`.** Any stored payload carries old ids. `schema_version` `3.0.0` is
  what lets a consumer detect this; a reader must gate on it.
- **Neo4j graphs.** Old and new ids do not collide (the prefix differs), so a re-push produces a
  *second* disconnected copy rather than an update. The prefix-scoped delete cannot remove the old
  copy either, because it scopes on the new prefix. Databases must be wiped, or migrated by an
  explicit one-off statement, before the first `3.0.0` push.

**No compatibility shim.** Emitting both shapes, or accepting either on read, doubles the surface
the parity clause governs and leaves consumers guessing which they hold. `schema_version` is the
mechanism; a hard cut with a detectable version is cheaper than a shim nobody removes.

---

## 6. Decomposition and release plan

**Proposed, pending sign-off** (§ decomposition is the user's call, not the author's):

Epic in `codellm-devkit/.github`, children filed just-in-time, one PR each:

| # | Repo | Unit |
| --- | --- | --- |
| 1 | `cldk-devtools` + `cldk-skillset` | the grammar in both keystone copies — the definition moves first, so implementations have something to conform to |
| 2 | `codeanalyzer-java` | `CanId`, root id + merge key, prefix scoping, cache stamp, fixtures |
| 3 | `codeanalyzer-python` | `ids.py`, root id + merge key, fixtures |
| 4 | `codeanalyzer-typescript` (+ `-v2`) | `ids.ts`, graph schema, fixtures |
| 5 | `python-sdk` | v2 models, Neo4j backends, prefix-scoped queries, fixtures |
| 6 | `docs` | identity page |

**Release plan.** All analyzers cut `schema_version` `3.0.0` **in lockstep** — a partial rollout
is the fragmentation the parity clause forbids, so no analyzer releases until every analyzer's
child is merged. `python-sdk` follows, pinning the three released analyzer versions together.

**This does not ride codeanalyzer-java 3.1.1.** That is a patch release carrying an opt-in CLI
flag; this is a breaking cross-repo change to the identity grammar. Conflating them would ship a
MAJOR contract break under a PATCH version number.

---

## 7. Verification

- Every analyzer's conformance gate green at `schema_version` `3.0.0`.
- No id in any emitted payload matches `^can://(java|python|typescript|artifact)/` — the old shape
  is gone, not merely supplemented.
- `can://<app>/` is a prefix of every node id the app emits, in every language — checked by query,
  since this is what makes the delete scope correct.
- A projection of one app by two analyzers into one database shares exactly one `:*Application`
  node reachable by `can://<app>`, and the scoped delete removes both languages' nodes.
- Two apps analyzed under the same `--app-name` produce a uniqueness-constraint violation rather
  than a silent merge.
- A cache written before the change is rejected, not reused.
