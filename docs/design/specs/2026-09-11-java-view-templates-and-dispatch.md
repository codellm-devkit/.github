# Spec: view templates as artifacts, and the dispatch edge that reaches them

Status: draft for review
Date: 2026-09-11
Scope: codeanalyzer-java (analyzer), python-sdk (Java facade, deferred), cross-repo vocabulary
Origin: roadmap pass 2026-09-11, Group D (candidates 16, 17); closes the vocabulary item of #48

---

## 1. Summary

A Java web application's view layer is in the graph as bytes and absent as structure. The
repository-artifact layer already inventories every JSP, Facelet and Thymeleaf page with its full
`source`, but classifies them `roles: ["unknown"]`, and nothing connects a servlet or controller
to the page it renders: the SDG stops at the string literal inside
`getRequestDispatcher("/x.jsp")`. This spec adds two things, both additive:

1. **A `view-template` artifact role**, with the classification rules that assign it, so a view
   is findable at all.
2. **`J_DISPATCHES_TO`**, an edge from the body node that dispatches (a `forward` / `include` /
   `sendRedirect` call, or a `return` in a controller) to the `Artifact` it dispatches to, with
   `application.view_dispatches[]` / `view_dispatches_unresolved[]` in `analysis.json` — the same
   level-graded, never-guessing shape `config_uses[]` already has.

It also **declares the shipped artifact-layer vocabulary canonical**, which is what #48 asked for
(§ 6). A role coined on a contested vocabulary is the one mistake the parity clause makes
permanent, so the two decisions are one spec.

### Measured on DayTrader (codeanalyzer-java 3.2.0, `--emit neo4j`)

| | |
| --- | --- |
| view files inventoried, all `roles: ["unknown"]` | 23 `.jsp`, 15 `.xhtml`, 20 `.html` (against 141 `.java`) |
| descriptors inventoried, never parsed | `web.xml` (19 `url-pattern`s, 3 welcome files, 2 error pages), `faces-config.xml` |
| dispatch call sites | 22 `getRequestDispatcher`, 20 `include`, 2 `forward`, 1 `sendRedirect` |
| of which the target is a **string literal** | **5** — 3 JSPs, 2 servlet URLs (`/servlet/PingServlet2ServletRcv`, …), 1 `welcome.faces` |
| of which the target is a **variable** | **17** |
| `.jsp` strings anywhere in the graph, none as an edge | 33 distinct |
| `<%= %>` / in-JSP `request.getParameter` with no node of any kind | 348 / 14 |

Two consequences shape the design. The literal tier alone reaches 3 of 23 JSPs, so the
dataflow tier is not optional for real code — exactly as it was not for config reads. And two of
the five literals are servlet URLs, not files: the edge must say plainly "this is not an artifact"
and hand those to the route work (roadmap Group E) rather than guess.

### What this spec does not do

- **No template-internal code.** `<%= %>`, `${…}`, `th:*`, `#{…}`, `<jsp:include>` inside a
  template — roadmap Group F. The role coined here is what Group F hangs its nodes under.
- **No routes.** `web.xml` servlet mappings, welcome files, `*.faces`, `@RequestMapping` paths —
  Group E. A dispatch whose target is a URL rather than a file is recorded unresolved with reason
  `no-such-artifact` so Group E can close it later.
- **No JSF outcome navigation.** `via: "navigation"` is reserved and its rule stated (§ 4.4), but
  it activates only once a JSF entrypoint finder exists (roadmap candidate 22, rides with Group E);
  without one, a `return "success"` in an arbitrary method cannot be told from a JSF action.
- **No new node kinds, no `schema_version` bump.** Everything is a role value, an edge kind, and a
  populated-but-existing field.

---

## 2. Contract-Impact Triage

| Question | Answer |
| --- | --- |
| Does this change the schema v2 output? | **Yes, additively.** One `roles` value; three `format` values (`jsp`, `xhtml`, `html`); one edge kind `J_DISPATCHES_TO`; two lists on `application`; `argument_expr` populated on `return` body nodes (the field already exists on every body node, empty on returns today). |
| Repos touched | `codeanalyzer-java` (emits); `python-sdk` (models + Java facade + Neo4j reconstruct); `codeanalyzer-python` / `codeanalyzer-typescript` (**vocabulary only** — they adopt `view-template` and `PY_DISPATCHES_TO` / `TS_DISPATCHES_TO` when they grow templates; no work scheduled). |
| Change type | Schema v2 evolution, additive at the leaves. |

---

## 3. Where the analyzers stand today

| | codeanalyzer-python 1.5.1 | codeanalyzer-typescript 1.5.3 | codeanalyzer-java 3.2.0 |
| --- | --- | --- | --- |
| `roles` vocabulary | `dependency-manifest service-topology container-image ci env tool-config packaging script docs legal iac unknown` | same set | same set (`artifacts/ArtifactDiscovery.java:48-90`) |
| a template role | none (Jinja/Django → `unknown`) | none (`.vue`/`.svelte` → `unknown`) | none (`.jsp`/`.xhtml`/`.html` → `unknown`) |
| code → artifact edge precedent | `PY_USES_CONFIG` from `PyBodyNode` to `ConfigKey`, `prov: string[]` | `TS_USES_CONFIG`, same shape | `J_USES_CONFIG` from `JBodyNode ∪ JCallable ∪ JField ∪ JType` (annotations have no body node) |
| unresolved reads | `config_reads_unresolved[]` `{site, callee, key, reason, prov}` | same | same |
| `return` body node payload | — | — | span only; `argument_expr: []` |

The vocabulary is identical across the three, which is the fact #48 needed established before
anything was added to it.

---

## 4. Design decisions

### D1. The role is `view-template`; `format` names the syntax

Hyphenated like every multi-word role already shipped. `template` alone was rejected because
codeanalyzer-iac (#52) will need the word for Helm templates; `view` alone loses the fact that the
file carries embedded code, which is what Group F depends on.

Classification rules, first-match-wins, inserted **before** the generic `*.xml` row:

| Pattern | `format` | `roles` |
| --- | --- | --- |
| `*.jsp`, `*.jspx`, `*.jspf`, `*.tag`, `*.tagx` | `jsp` | `view-template` |
| `*.xhtml` | `xhtml` | `view-template` |
| `*/templates/*.html`, `*/WEB-INF/*.html` | `html` | `view-template` |
| `faces-config.xml` | `xml` | `tool-config` |

`*` crosses `/` in this matcher (`ArtifactDiscovery.globMatches`), so `*/templates/*.html` covers
`src/main/resources/templates/admin/users.html`. A bare `*.html` anywhere else stays `unknown`: a
static page and a Thymeleaf template are not distinguishable by name (DayTrader's 20 `.html` are
WebSocket test pages), and misclassifying a static page as a template would later hand Group F a
file with no expressions to extract. `faces-config.xml` is added to the descriptor rows for the
same reason `web.xml` is there — it is read by § 4.4, and a consumer should find it as
`tool-config` rather than `unknown`.

The three `format` values are new but the field is open (python already has nine, TS ten, no two
lists equal), so this coins nothing that needs a sibling to agree.

### D2. One edge, `J_DISPATCHES_TO`, with `via` carrying the mechanism

```
(:JBodyNode)-[:J_DISPATCHES_TO {via: string, prov: string[]}]->(:Artifact)
```

- **`from` is `JBodyNode` only.** Every dispatch site is a body node: the `forward` / `include` /
  `sendRedirect` / `setViewName` / `new ModelAndView(…)` call node, or the `return` node itself
  once it carries its expression (D3). Unlike `J_USES_CONFIG`, there is no annotation-shaped
  source to force a union.
- **`to` is `Artifact`**, un-prefixed, the cross-language merge target. The edge is `J_`-prefixed
  because the source is Java code — same reasoning as `J_USES_CONFIG`; python coins
  `PY_DISPATCHES_TO` for `render_template` / `redirect` when it gets there.
- **`via`** ∈ `forward | include | redirect | view-name | navigation`. One edge with a discriminant
  rather than `J_RENDERS` + `J_REDIRECTS_TO`: the consumer question is "what pages can this
  code reach", and the mechanism is an attribute of the answer, not a different answer.
- **`prov`** ∈ `literal | table | dataflow`, the tier that resolved it (§ 4.5 for `table`), monotone with the analysis level
  exactly as on `J_USES_CONFIG`: `["literal"]` at L1, `"dataflow"` when the target reached the
  site over the L3 DDG (17 of DayTrader's 22 dispatches need it).
- **MERGE on the endpoint pair.** A body node dispatches to one target; a `return` on two paths of
  one method is two body nodes. No `_k` discriminant is needed.

Which Java calls count, and what the literal is:

| Site | `via` | target expression |
| --- | --- | --- |
| `RequestDispatcher.forward(req, res)` | `forward` | the argument of the `getRequestDispatcher(…)` / `getNamedDispatcher(…)` call that produced the receiver |
| `RequestDispatcher.include(req, res)` | `include` | same |
| `HttpServletResponse.sendRedirect(x)` | `redirect` | `x` |
| `return x` in a Spring controller method (D3) | `view-name` | `x` |
| `new ModelAndView(x, …)`, `ModelAndView.setViewName(x)` | `view-name` | `x` |
| Spring view name with `redirect:` / `forward:` prefix | `redirect` / `forward` | the remainder |

The `forward` / `include` rows anchor on the **dispatching** call (that is the node whose
execution reaches the page), and read the target off the `getRequestDispatcher` call node one
receiver up — `receiver_expr` already carries the full chain
(`ctx.getRequestDispatcher("/quoteDataPrimitive.jsp")`), and the dispatcher's own body node
carries `argument_expr: ['"/quoteDataPrimitive.jsp"']`.

### D3. `return` body nodes carry their expression

`argument_expr` is set to `[<expression source>]` on every `return` node with a value, and stays
`[]` for a bare `return`. The field already exists on every `JBodyNode` (`string[]`), so this is
a population change, not a schema addition; it is also the smallest fact that lets the literal
tier see `return "home"` at all, and it benefits any later dataflow over returns. The
alternatives — a callable-level edge fed only by the dataflow tier, or deferring Spring view
names to Group F — both left Thymeleaf with a role and no edges, and Thymeleaf's only dispatch
mechanism *is* the Spring view name.

### D4. Resolution never guesses

A target expression resolves to an artifact, or is recorded unresolved with a reason. There is no
"closest match".

**Path targets** (`forward`, `include`, `redirect`, and view names after prefix stripping): the
literal is a servlet-context-relative path. It resolves iff exactly one artifact's repo-relative
path ends with the webapp-relative form of it (`/marketSummary.jsp` →
`src/main/webapp/marketSummary.jsp`); a leading context path (`/daytrader/…`) is not stripped —
that is a route fact, Group E's.

**View names** (`via: "view-name"`), gated to callables whose type or callable carries `spring`
in `entrypoint_frameworks` and whose return type is `String` or `ModelAndView`, so a `return
"foo"` in ordinary code never matches a template named `foo.html`:

1. `redirect:` / `forward:` prefixes re-dispatch as path targets with the corresponding `via`.
2. Otherwise the candidate path is `<prefix><name><suffix>`, where prefix/suffix come from the
   application's own `ConfigKey`s when present as literals — `spring.mvc.view.prefix` / `.suffix`
   for JSP, `spring.thymeleaf.prefix` / `.suffix` for Thymeleaf — and from Thymeleaf's defaults
   (`classpath:/templates/`, `.html`) when absent. `classpath:` and `/WEB-INF/` are matched
   against any resource or webapp root, as the runtime would.
3. It resolves iff exactly one `view-template` artifact matches; two or more is `ambiguous`.

**JSF** (`via: "navigation"`, reserved): explicit `faces-config.xml` `navigation-rule`
`from-outcome → to-view-id`, then implicit `<outcome>.xhtml`, gated to callables in a JSF managed
bean — which needs the finder that roadmap candidate 22 adds. Until then nothing is emitted with
this `via`, and the rule is here so it is coined once.

### 4.5. May-dispatch over a static string table — `prov: ["table"]` (added 2026-09-11, codeanalyzer-java#261)

Running 3.3.0 on DayTrader resolved 3 of 23 dispatches; the rest go through
`TradeConfig.getPage(N)` = `return webUI[webInterface][pageNumber]`, a static `String[][]` of page
paths indexed by a runtime-selected interface. No single target exists on any path, so the tiers
above refuse correctly — and the 20 remaining JSPs stay unreachable although the set of pages the
code can reach is fully static. The **table tier** closes that shape deliberately as a
may-dispatch: a target expression that is a call `T.m(…)` whose every `return` is an array access
rooted at a static `String[]` / `String[][]` field of `T` with an array-initializer of string
literals resolves to **every literal in that initializer**, one `J_DISPATCHES_TO` per matching
artifact, `prov: ["table"]`. The same closure applies to a caller's argument when the
interprocedural tier binds a parameter; the union of all callers is the candidate set and `prov`
is `["table"]` when any table contributed.

This is the first over-approximation in the pass, and `prov` is what keeps it honest: `literal` and
`dataflow` edges still mean exactly one target; a `table` edge means "one of these". Table entries
naming no artifact (a servlet URL) contribute nothing; a table matching none is
`no-such-artifact`. Scope is the table shape only — not constant propagation, maps, enums, or
`switch`-returned literals. The callee is found by declaring-type simple name plus method name
within the tree, so the tier runs at `-a 1`; two same-named types both declaring the method make
it give up rather than pick one.

**Unresolved reasons**, mirroring `config_reads_unresolved`:

| `reason` | when |
| --- | --- |
| `non-literal` | the target never closed on a string at any attempted tier (`prov` lists the tiers tried) |
| `no-such-artifact` | a literal that maps to no inventoried file — servlet URLs (`/servlet/PingServlet2ServletRcv`), `*.faces`, absolute URLs, a template outside the repo |
| `ambiguous` | a view name matching more than one `view-template` artifact |

---

## 5. Wire format

### `analysis.json`

```jsonc
"application": {
  "view_dispatches": [
    { "src": "can://…/PingServlet2Jsp.java/PingServlet2Jsp/doGet(…)@69:13",
      "dst": "can://daytrader8/artifact/src/main/webapp/PingServlet2Jsp.jsp",
      "via": "forward", "prov": ["literal"] }
  ],
  "view_dispatches_unresolved": [
    { "site": "can://…/PingServlet2Servlet.java/PingServlet2Servlet/doGet(…)@73:13",
      "callee": "forward(javax.servlet.ServletRequest, javax.servlet.ServletResponse)",
      "target": "/servlet/PingServlet2ServletRcv", "via": "forward",
      "reason": "no-such-artifact", "prov": ["literal"] },
    { "site": "…@88:9", "callee": "include(…)", "target": null, "via": "include",
      "reason": "non-literal", "prov": ["literal"] }
  ]
}
```

Both lists are absent (not `[]`) when the layer produced nothing, matching `config_uses`. `target`
is the literal when there was one and `null` for `non-literal`. Every `return` body node gains
`argument_expr: ["<expr>"]` where a value is returned.

### Neo4j (`V2SchemaCatalog`)

```
J_DISPATCHES_TO   from [JBodyNode]   to [Artifact]   { via: string, prov: string[] }
```

Unresolved dispatches are not projected as relationships — there is no target node — and stay in
`analysis.json`, as `config_reads_unresolved` does except for its `JExternal` form; whether an
`Artifact`-less form is worth a relationship is an open question (§ 9). `Artifact` gains no
property; `roles` is already `string[]`. `schema_version` stays 2.0.0 (#50).

---

## 6. What this closes on #48

#48 lists three vocabularies the artifact-layer spec mandated and no analyzer implemented, and
recommends option 1, "amend the spec to what shipped". That spec exists only on the unmerged
branch `feat/configuration-files`; on `main` this document is the first to state the artifact
vocabulary, and it states the shipped one as the contract:

- **`format: string` + `roles: string[]`**, open, replacing the closed `artifact_kind` enum.
  `roles` is many-valued on purpose (`build.gradle` is both `dependency-manifest` and
  `tool-config`), which a single enum cannot express.
- **Dependency `kind` ∈ `runtime | dev | optional | build`**, replacing `scope`.
- **`ecosystem` is a per-analyzer literal** (`pypi`, `npm`, `maven`), not a shared table.

Epic #45's "Contract as shipped" section already says the same; this makes it a committed spec
rather than an issue comment, and `view-template` is the first addition made under it. Errata
items #48 later accumulated — the `@key:env/` id grammar and the orphan sweep on re-push — are
**not** closed here; they are cross-analyzer contract questions of their own and #48 stays open
for them.

---

## 7. SDK surface (python-sdk, deferred to its own PR)

Mirrors the config-use surface exactly, so nothing new has to be learned:

- models `JViewDispatch {src, dst, via, prov}` and `JViewDispatchUnresolved {site, callee,
  target, via, reason, prov}`; `JApplication.view_dispatches` / `.view_dispatches_unresolved`,
  both `Optional[List[…]] = None`.
- `JavaAnalysis.get_view_dispatches(view: str | None = None)` and
  `get_view_dispatchers(path: str) -> List[JCallableOverview]`, alongside `get_config_uses` /
  `get_config_readers`.
- the Neo4j reconstructor reads `J_DISPATCHES_TO` back into `view_dispatches`; unresolved
  entries are JSON-only, as `config_reads_unresolved` is today.

`JArtifact` is unchanged — `roles` was already an open list.

---

## 8. Decomposition and release plan

Decided 2026-09-11: **a single work item on `codeanalyzer-java`**, with the SDK follow-up as a
definition-of-done line rather than a child issue. No epic: the cross-repo part of this change is
vocabulary, recorded here, and only one repo has a PR to ship.

| Train | Carries | Notes |
| --- | --- | --- |
| `codeanalyzer-java` 3.3.0 | D1–D4, § 5 | additive MINOR; `schema.neo4j.json` regenerated; parity gate covers the new list |
| `python-sdk` next rc | § 7 | after 3.3.0 is on PyPI; pins move with it |
| `codeanalyzer-python`, `codeanalyzer-typescript` | — | no train; adopt the names when they grow templates |

---

## 9. Open questions for review

1. **Unresolved dispatches in the graph.** `config_reads_unresolved` projects to
   `J_READS_CONFIG_UNRESOLVED` on a `JExternal`; a `no-such-artifact` dispatch has an equally
   real target (a servlet URL). Projecting it now would pre-empt Group E's route node; leaving it
   JSON-only means a Cypher consumer cannot see it. This spec leaves it JSON-only.
2. **`faces-config.xml` `from-view-id → to-view-id`** is an `Artifact → Artifact` edge with no
   code in between. It is template-to-template navigation, which is Group F's shape, and is not
   emitted here.
3. **`*.html` under the webapp root.** Left `unknown` by design (D1). If a project keeps Thymeleaf
   templates outside `templates/` and `WEB-INF/`, the fix is a configuration knob on the rule, not
   a broader default.
