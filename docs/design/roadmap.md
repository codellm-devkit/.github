# CLDK roadmap

**Pass:** 2026-08-05  ·  **Planned with:** Rahul Krishna
**Status:** current  (supersede by editing, not by adding a second roadmap)

Theme of this pass: **full JavaScript and HTML support**, triggered by the finding
that `codeanalyzer-typescript` analyzes *no* JavaScript at all today —
`src/syntactic_analysis/discovery.ts:5` restricts discovery to
`.ts/.tsx/.mts/.cts`, so a pure-JS project emits an empty symbol table and exits 0.
Measured on OWASP NodeGoat (50 `.js`, 24 `.html`): 0 modules, 0 edges, 84-byte
`analysis.json`, no warning at default verbosity. The same gate is present verbatim
at tag `v0.5.0`, one line different from `main`.

## The two decisions this pass made

### 1. `<lang>` denotes the artifact's source language

`<lang>` in `can://<lang>/<app>/…` is the **artifact's source language**, not the
analyzer that produced it. A pure-JS application is `can://javascript/…` and its
manifest says `"language": "javascript"`.

The rejected alternative was a family-scoped literal (`can://typescript/` covering
both, with a per-module `language` attribute). It was rejected because the manifest
language is load-bearing, not cosmetic: on untyped JS the tsc resolver leg collapses
(NodeGoat, `.js` renamed to `.ts`: 20 resolved vs 174 unresolved call sites; Jelly
supplied 131 of the 170 union edges). A consumer that cannot tell TS-quality edges
from JS-quality edges cannot calibrate anything built on them.

The accepted cost: a **mixed** `.ts` + `.js` application — the common case in any
migration, or any TS project with JS tooling — spans two `can://` roots, and schema
v2 has no vocabulary for an edge crossing applications. That forces a schema major.

**Scope: web only, explicitly.** `codeanalyzer-clang` (C and C++ in one repo) and the
`codeanalyzer-java` / `codeanalyzer-kotlin` pair are **not** bound by it. See Risks.

### 2. JavaScript ships on the 0.x line, not on 1.x

Because that is where every consumer actually is:

```
python-sdk/pyproject.toml:43        codeanalyzer-typescript==0.4.3   ← exact pin
cldk/models/typescript/models.py:24 TSApplication{symbol_table, call_graph}  ← v1 flat shape
```

`cants` 1.0.0's schema-v2 output is consumed by **nothing** today. Shipping JS on 1.1.0
would have parked it behind an SDK migration with no owner (candidate 9).

This has a load-bearing side effect: **the 0.x line emits v1, which has no `can://`
ids and no manifest `language`, so there is no literal to coin.** Candidate 1 is
therefore blocked by nothing and needs no design session — it is `maintaining-cldk`
work, not a contract change.

**0.x is the supported line until v3 lands**, carrying candidate 4 as well as
candidate 1. That re-opens collision group B on the v1 schema; see Risks.

## Candidates

| # | Feature | Moves schema v2? | Collision group | Blocked by |
| - | ------- | ---------------- | --------------- | ---------- |
| 1 | JS file discovery in `cants` (`.js/.jsx/.mjs/.cjs`) | no — 0.x emits v1, which has no `can://` ids or manifest `language` | — | — |
| 2 | Artifact-language rule: `<lang>` = source language; literals coined | yes — id shape + manifest | A | — |
| 3 | Cross-root edge vocabulary (js↔ts, `<script src>`→module, handler-attr→callable) | yes — new edge kinds + cross-application endpoints | A, B | 2 |
| 4 | CommonJS module vocabulary (`require`, `module.exports`, `exports.x`) | coined on v1 first; v3 adopts verbatim | B | — |
| 5 | SDK facade + pins (`CLDK.javascript()`, html backend) | yes — public API | A | 2, 3, 9 |
| 6 | HTML artifact node kinds (document, element, inline-script, handler-attr) | yes — new node kinds | C | 3 |
| 7 | Template engines (Handlebars, EJS, Pug) | yes — reuses group C kinds | C | 6 |
| 8 | Repo topology: `codeanalyzer-html` as its own repo | no — packaging only | — | 6 |
| 9 | `python-sdk` model migration (pin `0.4.3` → v3; `TSApplication` → v3 envelope) | yes — public API | — | own planning pass |

## Collision groups

- **Group A — id and manifest vocabulary**: candidates 2, 3, 5.
  Decided once: the exact literal strings (`javascript` vs `js` vs `ecmascript`;
  `html`), whether the manifest `language` stays scalar, and what an edge endpoint
  looks like when it names a node in another application.
  Design session: **candidate 2**, carrying candidate 3.
  Candidate 1 left this group when JS moved to the 0.x line — v1 has no literal.

- **Group B — module-edge vocabulary**: candidates 3, 4.
  `require("./util")`, `import x from "./util"` and `<script src="util.js">` are the
  same relation seen three ways. **Candidate 4 now coins this vocabulary first, on
  the v1 schema**, ahead of candidate 3's design session. Decision: ship it and
  correct by yank-and-reship if v3 wants different names — see Risks for why that is
  affordable on v1 and not after.

- **Group C — artifact node kinds**: candidates 6, 7.
  Engine-agnostic at birth — Handlebars is the second consumer, and NodeGoat's actual
  XSS surface lives in its 24 `.html` views, not in its routes.
  Design session: **candidate 6**.

Candidate 3 spans groups A and B. It is the keystone of the pass.

## Dependency order

    1 (JS discovery, 0.x) ──▶ 4 (CommonJS module vocabulary, 0.x)
     └─ unblocks #52, cocoa#1                    [v1 schema — no design gate on either]

    2 (artifact-language rule + literals)
     └──▶ 3 (cross-root edge vocabulary)   ◀── keystone, spans A + B
           ├──▶ 6 (HTML node kinds) ──▶ 7 (template engines)
           │                         └──▶ 8 (codeanalyzer-html repo)
           └──▶ 5 (SDK facade) ◀── also blocked by 9

    9 (python-sdk model migration) ── deferred to its own pass; blocks 5

Candidate 4 is drawn unblocked but is **not** unconstrained: candidate 3 must adopt
its field names rather than coin its own.

**Repo topology is not a contract question.** Because `<lang>` is per-artifact, one
binary can emit several roots, so repo count and literal count are independent. What
remains is a toolchain argument: JavaScript shares essentially all of `cants`'
machinery (ts-morph already parses `.js`; `allowJs: true` is already the default at
`symbolTable.ts:115-126`), whereas HTML shares none of it. Hence JS stays in
`codeanalyzer-typescript` and HTML becomes its own repo — candidate 8. The repo name
no longer implies the emitted literal, which is the point of decoupling them.

## Release trains

| Train | Carries | Notes |
| ----- | ------- | ----- |
| `codeanalyzer-typescript` **0.6.0** (maintenance branch off `v0.5.0`) | 1 | v1 schema. `python-sdk` pin `0.4.3` → `0.6.0`, one line. Unblocks issue #52 and `cocoa#1`. |
| `codeanalyzer-typescript` **0.7.0** | 4 | v1 schema. Field names here are binding on v3. |
| `main` / 1.x | forward-ports of 1 and 4 | Keeps the v2 line current. No consumer until candidate 9. |
| **schema v3** — all analyzers + both SDKs, lockstep | 2, 3 | One migration. `codeanalyzer-java`, `-python`, `-typescript` re-cut together. |
| SDK minor | 5 | After candidate 9's pass concludes. |
| `codeanalyzer-html` 0.1.0 | 6, 8 | Additive on v3. |
| `codeanalyzer-html` 0.2.0 | 7 | Additive. |

## Not now

- **Candidate 9 — `python-sdk` model migration.** A discovered blocker with no owner:
  the SDK is pinned to `0.4.3` and its model layer is v1-shaped, so nothing `cants`
  has released since schema v2 reaches any consumer. Deferred to **its own planning
  pass** because it touches the java and python analyzers too, not just typescript.
  The open question it must answer: migrate v1→v2 now and v2→v3 later, or skip v2
  entirely and migrate once.
- **CSS** — no consumer has asked for selector or style reachability.
- **Sourcemaps and minified `dist/` bundles** — analyzing generated output rather than
  sources inverts the model. Excluded on principle, not on cost.
- **Browser DOM sink taxonomy** (`innerHTML`, `document.write`, …) — belongs to a
  security-rules layer above the schema, not to the analyzer's vocabulary.
- **JSX-as-HTML unification** — decide after group C exists and has one real consumer.
- **The `<lang>` rule for `codeanalyzer-clang` and `codeanalyzer-java` /
  `codeanalyzer-kotlin`** — explicitly out of scope, by decision. See Risks.

## Risks

**Three conventions.** Artifact-language literals are an ecosystem-wide convention,
but this pass binds only the web analyzers. `codeanalyzer-clang` already covers two
distinct languages in one repo; java/kotlin are one runtime family in two repos with
two literals. The ecosystem has no settled convention today in either direction, and
this pass deliberately does not create one. Exposure: three incompatible conventions
across web, clang and jvm — the outcome the parity clause exists to prevent. Accepted
knowingly. *Mitigation, free:* write candidate 2's rule in family-neutral terms even
though it binds only web, so later clang/jvm adoption is ratification, not
re-litigation.

**Group B coined twice — accepted, with a stated remedy.** Making 0.x the supported
line means candidate 4 coins the module-edge vocabulary on the v1 schema, months
before candidate 3 governs the same relation on v3. No up-front design session is
being spent on it: if v3 wants different names, 0.x is **yanked and reshipped**.

That remedy is affordable *here specifically*, and the reason is worth stating so it
is not generalized by mistake. The v1 line has exactly one consumer — `python-sdk`,
at an exact pin this org controls — and v1 is not governed by the parity clause. So a
bad name reaches one repo and is retracted by bumping one line.

It stops being affordable at v3. Once the canonical schema carries a name, it lives
in consumers' Neo4j projections and cached `analysis.json` files; yanking a package
version does not un-coin it. The parity clause exists for exactly that asymmetry.

**Two live lines.** 0.x and `main` both receive JS work, with forward-porting in one
direction. `discovery.ts` currently differs between `v0.5.0` and `main` by one line,
so the cost is low today and grows with every 1.x commit.

## Starting now

Candidate **1** — JS file discovery on a 0.6.0 maintenance branch off `v0.5.0`. No
design session required: the 0.x line emits v1, so no schema vocabulary is at stake.
Enters via `maintaining-cldk`, closes issue #52, unblocks `cocoa#1`.

Candidate **2** (carrying **3**) is the next design session, and the largest rock on
this roadmap. It is not started in this pass.

Everything else here has no issue yet, by design.
