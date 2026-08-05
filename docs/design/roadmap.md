# CLDK roadmap

**Pass:** 2026-08-05, amended 2026-08-05 after implementation evidence
**Planned with:** Rahul Krishna
**Status:** current  (supersede by editing, not by adding a second roadmap)

Theme: **full JavaScript and HTML support**, triggered by the finding that
`codeanalyzer-typescript` analyzed *no* JavaScript at all —
`src/syntactic_analysis/discovery.ts:5` restricted discovery to `.ts/.tsx/.mts/.cts`,
so a pure-JS project emitted an empty symbol table and exited 0. Measured on OWASP
NodeGoat (50 `.js`, 24 `.html`): 0 modules, 0 edges, an 84-byte `analysis.json`.

## What this amendment changes, and why

The first pass built its expensive half on a premise that implementing the cheap half
falsified. Recording that plainly, because the reasoning matters more than the outcome.

**The falsified premise.** `<lang>` was defined as the artifact's source language on the
grounds that the manifest language is load-bearing: a consumer must be able to tell
TypeScript-quality edges from JavaScript-quality ones. Direct measurement, same
analyzer, same application, only the file extension differing:

| | NodeGoat as `.js` | renamed `.ts` |
| --- | --- | --- |
| resolved call sites (tsc) | **28** | 20 |
| callables in symbol table | **24** | **24** |
| ES6 classes captured | yes | yes |
| `this.x = fn` captured | no | **no** |

Identical with and without `node_modules` materialized. The TypeScript compiler resolves
JavaScript *better* than the same code renamed `.ts`, because it models `module.exports`
and `require()` as module syntax in `.js` files and not in `.ts` files. Every gap found
is an **idiom** gap — `this.x = fn` inside a constructor function, object-literal methods
— missing in **both** languages. There is no JS-vs-TS quality axis to encode.

**The second correction.** The 0.x line, which is where every consumer actually sits,
emits schema v1: top-level keys `symbol_table`, `call_graph`, `external_symbols`,
`synthesized_callables`, with path-based signatures (`src/index.makeHandle`). There is
no `can://` id, no application root, and no manifest `language`. The entire identity
question is therefore a **v2 concern**, and v2 reaches no consumer until candidate 9.

**Net effect.** The schema v3 major, the cross-root edge vocabulary, and the second
analyzer repo all leave the near-term plan. What remains is a v1 delivery track with no
contract decisions in it.

## Decisions

1. **One analyzer for the JS/TS/HTML ecosystem.** `codeanalyzer-typescript` keeps its
   name and grows extractors for JavaScript and HTML. No `codeanalyzer-javascript`, no
   `codeanalyzer-html`, no `codeanalyzer-web`. A pure-TypeScript project is one where a
   single extractor fires — a configuration, not a separate product. Two backends over
   the same file types would fork ~4,200 LOC of language-neutral engine (dataflow 2,521,
   schema 784, providers 950), duplicate every idiom fix forever, and churn every id the
   day a TypeScript repo gains one `eslint.config.js`.
   *Renaming is a separate, deferrable decision from remit, and is not being taken.*
2. **Ship on 0.x.** `python-sdk` pins `codeanalyzer-typescript==0.4.3` and its model layer
   is the v1 flat `TSApplication`, so nothing released on the v2 line reaches a consumer.
   0.x is the supported line until candidate 9 concludes.
3. **HTML lands in the v1 shape.** An `.html` file is a `symbol_table` entry; its inline
   `<script>` callables are that module's `functions`; `<script src>` and handler-attribute
   edges are ordinary `call_graph` entries; library calls from inline script are
   `external_symbols`. No new node kinds and no envelope work are required on 0.x.
   `stripTsExtension` (`src/schema/schema.ts:433-435`) does not strip `.html`, so
   signatures read `views/login.html.handleSubmit` — kept deliberately, since it
   disambiguates `login.html` from `login.js` and needs no code change.

## Candidates

| # | Feature | Line | Moves schema? | Blocked by |
| - | ------- | ---- | ------------- | ---------- |
| 1 | JS file discovery (`.js/.jsx/.mjs/.cjs`) — **done**, issue #84 | 0.x | no | — |
| 4 | CommonJS module vocabulary (`require`, `module.exports`, `exports.x`) | 0.x | v1 only | — |
| 6 | HTML: inline `<script>`, `<script src>`, handler-attribute edges | 0.x | v1 only | — |
| 10 | Idiom gap: `this.x = fn` and object-literal methods not materialized as callables | 0.x | v1 only | — |
| 9 | `python-sdk` model migration (pin `0.4.3` → v2/v3; `TSApplication` → envelope) | v2 | yes — public API | own planning pass |
| 2 | Artifact-language vs family `<lang>`, per-node `language` | v2 | yes — id shape | 9 |
| 3 | Cross-artifact edge vocabulary as first-class schema | v2 | yes | 9, 2 |
| 7 | Template engines (Handlebars, EJS, Pug) | v2 | yes | 6, 9 |

Candidates 5 and 8 from the first pass are closed: there is no separate SDK facade to
design (candidate 5 folds into 9) and no second repo to create (candidate 8).

**Candidate 10 is language-neutral and does not belong to this theme.** It is listed
because it was discovered here and it dominates real-world JS recall, not because it is
a JavaScript problem — NodeGoat yields the same 24 callables whether its files are `.js`
or `.ts`, and idiomatic TypeScript (`sample-app`) resolves 81% of call sites against
NodeGoat's 11%. It should be fixed once, for both languages.

## Collision groups

- **0.x track: none.** v1 is not governed by the parity clause, has exactly one consumer
  (`python-sdk`, at a pin this org controls), and the agreed remedy for a wrong name is
  to yank and reship. Candidates 1, 4, 6 and 10 can proceed without a design session.
- **v2 track — identity and artifact vocabulary**: candidates 2, 3, 7, 9.
  One decision, taken inside candidate 9's pass, since v2 has no consumer until then.
  What it must settle: whether `<lang>` is family or artifact, where per-node `language`
  lives, and what an HTML document is as a first-class node kind.

## Dependency order

    0.x (v1, no contract gate)
      1 (discovery — done) ──▶ 4 (CommonJS)
                           └──▶ 6 (HTML inline script)
      10 (idiom gap) — independent, language-neutral, highest recall impact

    v2 (parked)
      9 (SDK model migration) ──▶ 2 (identity) ──▶ 3 (artifact edges) ──▶ 7 (templates)

## Release trains

| Train | Carries | Notes |
| ----- | ------- | ----- |
| `codeanalyzer-typescript` 0.6.0 | 1 | v1 schema. `python-sdk` pin `0.4.3` → `0.6.0`, one line. Unblocks #52 and `cocoa#1`. |
| `codeanalyzer-typescript` 0.7.0 | 4, 10 | v1 schema. |
| `codeanalyzer-typescript` 0.8.0 | 6 | v1 schema. HTML as `symbol_table` entries. |
| `main` / 1.x | forward-ports | Keeps the v2 line current. No consumer until candidate 9. |
| — parked — | 9, then 2, 3, 7 | Sequenced by candidate 9's own planning pass. |

The v3 major from the first pass is **withdrawn**. Nothing on the 0.x track needs it, and
the v2 track's shape is candidate 9's to decide.

## Not now

- **Candidate 9 — `python-sdk` model migration.** The real blocker: nothing `cants` has
  released since schema v2 reaches any consumer. Deferred to **its own planning pass**;
  it touches the java and python analyzers too. Open question it must answer: migrate
  v1→v2 now and v2→v3 later, or skip v2 and migrate once.
- **The `<lang>` identity decision (candidate 2).** Parked with 9 — v2 has no consumer,
  so deciding now buys nothing and risks coining permanent vocabulary from a position we
  have twice had to revise.
- **First-class HTML node kinds and cross-artifact edge kinds (candidates 3, 7).** The v1
  shape carries HTML adequately for the 0.x track; promote to real vocabulary only when
  v2 is consumable.
- **Renaming the repo/package.** Remit is broadening; the name is not moving. Revisit only
  if it demonstrably confuses users.
- **CSS**, **sourcemaps and minified `dist/` bundles**, **browser DOM sink taxonomy**,
  **JSX-as-HTML unification** — unchanged from the first pass.
- **The `<lang>` rule for `codeanalyzer-clang` and `codeanalyzer-java`/`-kotlin`** — still
  out of scope, and now less urgent since no `<lang>` rule is being coined this pass.

## Risks

**Span units become intra-file with HTML.** `cants` emits UTF-16 char offsets
(`src/schema/graphs.ts:37`); `codeanalyzer-python` emits utf-8 byte offsets
(`py_schema.py:198`) — both under `span.bytes`. Today that is a cross-analyzer
inconsistency. Inline `<script>` puts HTML offsets and JavaScript offsets in **one file
and one span space**, so a unit mismatch silently points every inline-script span at the
wrong text, with no error. *Mitigation, gating:* candidate 6 must fix the unit convention
for both offsets before emitting a single inline-script span, and the fixture must include
non-ASCII content so the gate can fail.

**Two live lines.** 0.x and `main` both receive work, forward-ported in one direction.
`discovery.ts` differed between `v0.5.0` and `main` by one line when this began; the cost
grows with every 1.x commit.

**v1 vocabulary coined ahead of v2.** Candidates 4, 6 and 10 name fields on the v1 schema
before candidate 2 governs the same concepts on v2. Accepted: v1 has one pinned consumer
and is not parity-clause governed, and the remedy is to yank and reship. This is
affordable *here specifically* and does not generalize — once the canonical schema carries
a name it lives in consumers' Neo4j projections and cached `analysis.json`, and yanking a
package version does not un-coin it.

**This roadmap has been revised once by implementation evidence.** Both revisions came
from measuring rather than reasoning. Treat the v2 track's shape as provisional until
something on it is actually built.

## Starting now

Candidate **1** shipped — `codeanalyzer-typescript` issue #84, branch `feat/issue-84` off
`v0.5.0`, targeting 0.6.0.

Candidate **10** (the idiom gap) is the highest-recall next step and needs no design
session. Everything else here has no issue yet, by design.
