# CLDK roadmap

**Pass:** 2026-08-07 (schema v2 consistency) · amended 2026-09-11 (Java web layer, below)
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

**No consumer sees v2 yet, and that is the opportunity.** The v2 work is unreleased on
both analyzers. `python-sdk/pyproject.toml` pins `codeanalyzer-python==0.3.1` and
`codeanalyzer-typescript==0.4.3`, which emit `SCHEMA_VERSION` **1.2.0** and **1.0.0**
respectively — the v1 vocabulary (`PyApplication` / `PY_HAS_MODULE`; bare `Application` /
`Module` / `Symbol` / `HAS_MODULE`). Both SDK Neo4j backends match their pins exactly.
`codeanalyzer-typescript`'s tags stop at `v1.0.0`; the `2.0.0` schema is a development
tip. So the divergence below is between two **unreleased** lines, and fixing it before
either ships is enormously cheaper than reconciling shipped contracts afterwards.

Measured against the emitters at their development tips, not the docs and not the
released lines:

| | `codeanalyzer-python` 2.0.0 | `codeanalyzer-typescript` 2.0.0 | `codeanalyzer-java` 1.1.0 |
| --- | --- | --- | --- |
| `can://` ids | yes | yes | no |
| Neo4j merge labels | **9** — `PyApplication`, `PyModule`, `PySymbol`, `PyPackage`, `PyDecorator`, `PyCallSite`, `PyAttribute`, `PyVariable`, `PyCFGNode` | **2** — `Application`, `CanNode` | `JApplication`, `JSymbol`, + per-kind |
| Statement/body model | split — `PyCallSite` + `PyCFGNode` | unified — `TSBodyNode` with a `kind` discriminant | **absent from Neo4j** — deferred, `SCHEMA_DECISIONS.md` #9 |
| `cfg` / `cdg` / `ddg` / `summary` / `param_in` / `param_out` | yes, all six | yes, all six | callable→callable only (`J_CONTROL_DEP` / `J_DATA_DEP` / `J_HEAP_DATA_DEP`) |
| Decorators in `analysis.json` | **flat `decorators: List[str]`** — `ast.unparse` output; there is no `PyDecorator` model (`schema/py_schema.py:274`) | structured `TSDecorator` (`SCHEMA_DECISIONS.md` #4) | flat `annotations: string[]` |
| Decorators in **Neo4j** | yes — `:PyDecorator` + `PY_DECORATED_BY`, but the node carries only the raw string (`neo4j/project.py:443`) | **no** | no |
| Entrypoints | none | `TSApplication.entrypoints` — `analysis.json` only | `JEntrypoint` marker + `is_entrypoint` — Neo4j only |
| `MARKER_LABELS` | `[]` | `[]` | `["JEntrypoint"]` |

Three structural problems fall out.

**0. Python and TypeScript have mirror-image decorator gaps.** Python has the graph
projection and flat data; TypeScript has the structured data and no graph projection.
Neither is closer to canonical here — they fail the same contract from opposite sides, and
closing one gap does not inform the other. (Corrected 2026-08-19: this table previously
credited Python with a structured `PyDecorator`. No such model exists — the name is a Neo4j
label whose only property is the unparsed source string. Tracked as
codellm-devkit/codeanalyzer-python#128.)

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
| 14 | ~~SDK TypeScript Neo4j backend vocabulary drift~~ — **withdrawn, not a defect.** See Not now | — | — | — |
| 15 | **Move the SDK's analyzer pins from the 0.x line to the v2 line** — `codeanalyzer-python==0.3.1` and `codeanalyzer-typescript==0.4.3` in `pyproject.toml`, both schema v1 | no — dependency pins, but it is what makes v2 reach a consumer | C | 7, 8 |

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

    C  (12 SDK v2 model + 13 backcompat policy + 15 pin move)  ◀── after any one conformant analyzer
     └─▶ (microservice initiative resumes here)

Candidate 15 is the step that makes any of this visible to a user: until the SDK's pins
move off the 0.x line, a conformant analyzer changes nothing a consumer can observe.

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
| `python-sdk` | 12, 13, 15 | version depends on candidate 13's policy — see below. Candidate 15 moves the pins, so this train is what actually delivers v2 to consumers |

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
- **Candidate 14, withdrawn — it was not a defect.** The first pass recorded the SDK's
  TypeScript Neo4j backend as drifting from its analyzer, because it queries bare
  `Application` / `Module` / `HAS_MODULE` while `codeanalyzer-typescript`'s schema at the
  development tip emits `TSApplication` / `TSModule` / `TS_HAS_MODULE`. That compared a
  released consumer against an unreleased producer. `pyproject.toml` pins
  `codeanalyzer-typescript==0.4.3`, whose `SCHEMA_VERSION` is `1.0.0` and whose labels are
  exactly the bare ones the SDK queries; the same holds for `codeanalyzer-python==0.3.1`
  at `1.2.0`. Both backends are correctly matched to their pins. Recorded rather than
  deleted, because the mistake is instructive: **compare a consumer against the version it
  pins, not against the working tree.** The real work it pointed at is candidate 15.
- **Unified canonical SDK model classes replacing per-language facades** — candidate 12
  builds the shared model layer; whether `PythonAnalysis` / `JavaAnalysis` /
  `TypeScriptAnalysis` eventually collapse into one facade is a later question, and
  candidate 13's backwards-compatibility policy likely forbids it for now.

## Starting now

**Group A — the canonical projection contract.** Spec only, no implementation, into
`docs/design/specs/`.

No epic is filed. Epics are filed just-in-time when implementation starts, and a
spec-only pass has no pull request to close. The first epic under this roadmap will be
whichever of candidates 7 through 11 is picked up once Group A's spec exists to point at.

Everything else on this roadmap has no issue yet, by design.

---

# Pass 2026-09-11 — the Java web layer

**Planned with:** Rahul Krishna

Theme: **make the view layer of a Java web application reachable in the graph** — JSP,
servlets and their descriptors, Thymeleaf, JSF/Facelets. Prompted by an exploitability run
over DayTrader (`codeanalyzer-java` 3.2.0) whose findings skewed to build and deployment
files because the analyzer "read `src/main/java` only — JSP was never parsed."

## The finding

The claim is half right. The symbol table walks `src/*/java` roots only, but the
repository-artifact layer (#45) already inventories every JSP with its **full source**
(`format: text`, `roles: ["unknown"]`) and `web.xml` too (`roles: ["tool-config"]`, never
parsed). The bytes are in the graph; the structure is not. Measured on DayTrader:

- 23 `.jsp` files against 141 `.java`; 348 `<%= %>` expressions (XSS sinks) and 14
  `request.getParameter` reads inside JSPs (taint sources) with no node of any kind;
- 33 distinct `.jsp` strings in the graph, all as **text** — servlets reach their views
  through `getRequestDispatcher("/x.jsp").forward(...)` and the SDG stops at the string
  literal, never at the artifact;
- no template is an entrypoint and no entrypoint has a URL: `web.xml` servlet-mappings and
  welcome-files are unread, and Spring `@RequestMapping` arguments sit raw in decorators.

The servlet side needs no new contract: `javaee/` already ships Jakarta, Spring, JAX-RS,
Struts and Camel entrypoint finders, and DayTrader's 151 entrypoints come from them. The
gap is entirely the view side — and it is not Java-specific. `codeanalyzer-python` has no
template role either (Jinja/Django) and `codeanalyzer-typescript` none for Vue/Svelte, so
whatever Java coins here is the vocabulary the other two inherit.

## Candidates

Numbering continues from the first pass.

| # | Feature | Moves schema v2? | Collision group | Blocked by |
| - | ------- | ---------------- | --------------- | ---------- |
| 16 | **View-template artifact role** — `.jsp`, `.xhtml`, Thymeleaf `.html`, and the like classified instead of `unknown` | yes — the shared open `roles` vocabulary, which #48 says is itself unreconciled | D | #48 (folded in) |
| 17 | **View-dispatch edges** — `forward` / `include` / `sendRedirect` / `ModelAndView` / returned view name / JSF navigation outcome, from a callable or body node to the `Artifact`; view-name resolution needs the Spring/Thymeleaf prefix and suffix from config keys | yes — a new edge kind; `J_USES_CONFIG` is the precedent (language-prefixed source, un-prefixed target, `prov`) | D | 16 |
| 18 | **Descriptor-derived routes** — `web.xml` servlet/filter/welcome-file/error-page mappings and `faces-config.xml` navigation as a URL on the entrypoint, unified with annotation-derived routes (`@RequestMapping`, `@Path`) | yes — a route field on entrypoints; no analyzer has one | E | 6 (first pass) |
| 19 | **Template-internal code model** — how `<%= %>`, `${...}`, `th:*`, `#{...}` appear as nodes with spans beneath a non-code file: the node kind and the id grammar under an artifact | yes — a new kind in the shared ladder and a new id shape | F | 16 |
| 20 | **Template ↔ code dataflow** — `request.setAttribute` / `model.addAttribute` / managed-bean properties bound to template reads, and in-template `getParameter` as an SDG source | yes — `ddg` endpoints on template nodes, new `prov` values | F | 17, 19 |
| 21 | Bundler output inventory — `static/`, `dist/` JS and CSS as artifacts with a role | yes — same `roles` vocabulary as 16 | D | — |
| 22 | JSF and Thymeleaf entrypoint finders — `@Named` / `@ManagedBean`; `@Controller` already matches | **no** — an additive `entrypoint_frameworks` value | — | — |

Candidate 22 is not a contract decision; it is a maintenance-rung item recorded so it is not
mistaken for one. It ships with 18.

## Collision groups

- **Group D — template-as-artifact vocabulary**: candidates 16, 17, 21.

  One role name and one edge name, coined once for three analyzers: python's
  `render_template("x.html")` and TypeScript's SFC imports are the same edge from the same
  role. The session **also settles #48** — the `roles` vocabulary is still contested between
  the spec's closed `artifact_kind` and the shipped open `roles[]`, and a role coined on a
  contested vocabulary is the one planning mistake the parity clause makes permanent. So the
  spec amendment that resolves #48 and the spec that coins the template role are one PR.

- **Group E — entrypoint route vocabulary**: candidate 18.

  The first pass's candidate 6 (entrypoint vocabulary, Group B) settled `is_entrypoint`,
  `entrypoint_frameworks` and the entrypoint report; it did not give an entrypoint a URL.
  Routes are where the microservice initiative's service boundaries begin, so this is the
  hand-off point to that deferred work, decided once across descriptor-derived and
  annotation-derived sources. One twist reaches back into Group D: a welcome-file JSP is
  itself URL-addressable, so an `Artifact` can be an entrypoint, and E's field has to be
  legal on D's node.

- **Group F — template code model**: candidates 19, 20.

  One decision for JSP, Thymeleaf and Facelets; per-engine extractors are additive under
  it. Jasper/JspC translation to a generated servlet is one option *inside* this session
  (real callables, but spans that point at generated lines, not the JSP), not a candidate of
  its own.

## Dependency order

    #48 ──┐
          ├─▶ 16 role ──▶ 17 dispatch edges ──┐
                    │                         ├─▶ 20 template ↔ code dataflow
                    └─▶ 19 template code model ┘

    6 (first pass, Group B) ──▶ 18 routes  [Group E; bridges to the microservice initiative]

    21 bundler inventory — not now (below)
    22 finders — rides with 18, no contract

Group D unblocks everything else on this pass and needs nothing new: every fact it
projects — the JSP text, the `web.xml` text, the string literal at the dispatch call site —
is already in the graph. Group E is independent of D and can run in parallel; Group F
cannot start until the role exists to hang nodes under.

## Release trains

| Train | Carries | Notes |
| ----- | ------- | ----- |
| `codellm-devkit/.github` spec | #48 resolution + 16, 17 | one spec PR; the vocabulary is what the other analyzers inherit |
| `codeanalyzer-java` 3.3.0 | 16, 17 | additive MINOR — a role value, an edge kind; `schema_version` stays 2.0.0 pending the coordinated re-baseline (#50) |
| `codeanalyzer-java` 3.4.0 | 18, 22 | additive MINOR |
| `codeanalyzer-java` next | 19, 20 | additive — new kind under an artifact; whether it is a MINOR depends on the id grammar Group F chooses |
| `python-sdk` | 16–20 as they land | the reconstructor and facade grow with each; a template node kind (19) is the first that needs a new model |
| `codeanalyzer-python`, `codeanalyzer-typescript` | 16, 17 vocabulary only | no work scheduled; they adopt the names when they grow templates (Not now) |

## Not now

- **Candidate 21, bundler output inventory** — inventory-only value; built JS is
  `codeanalyzer-typescript`'s turf, and a Java repository that also carries a frontend is the
  multi-language case the deferred microservice initiative owns. The role, if one is ever
  coined, belongs to Group D's vocabulary and is recorded here so it is coined there.
- **Python and TypeScript templates** — Jinja/Django, Vue/Svelte SFCs. Same vocabulary,
  different repos; they adopt Group D's names when they start, and nothing here schedules
  them.
- **Freemarker / Velocity / Struts Tiles** — not in this pass's scope. Additive engines
  under Group F's model if wanted later.
- **Frontend bundler analysis** (as opposed to inventory) — never Java's; TypeScript's.

## Starting now

**Group D — candidates 16 and 17, with #48 folded in.** Enters `designing-cldk-changes`;
the epic is filed when implementation starts, not before.

Everything else on this pass has no issue yet, by design.
