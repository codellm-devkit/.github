# Spec: a `<service>` segment for the canonical `can://` identity grammar

Status: draft for review
Date: 2026-08-07
Scope: canonical schema v2 identity grammar, cross-repo

---

## 1. Summary

The canonical `can://` id grammar gains a **service segment as its outermost element**, and the
existing `<app>` segment collapses into it:

```
before   can://<lang>/<app>/<file>/<type>/<callable-signature>
after    can://<service>/<lang>/<file>/<type>/<callable-signature>
```

This makes a deployment unit a first-class part of node identity, so polyglot microservice
systems are addressable without a consumer inventing a second key on top of the canonical one.

The change ships as `schema_version` `2.0.0` → **`2.1.0`**, amending v2 in place rather than
opening a v3.

### Why now

The change is not motivated by a hypothetical. `cocoa` — the polyglot, k8s-native system-graph
consumer — already needs service-scoped identity and has built it *outside* the schema by
prefixing the canonical id with an ad-hoc key:

```python
# cocoa/system/facts.py:39
def _fid(service: str, qualified: str) -> str:
    return f"fn:{service}/{qualified}"
# yields: fn:emailservice/can://python/app/email_server.py/SendOrderConfirmation(req)
```

A downstream consumer wrapping a URI in a second identity scheme is the maintenance cost this
spec removes. Under the new grammar that key is simply the id:

```
can://emailservice/python/email_server.py/SendOrderConfirmation(req)
```

Same information, inside the grammar instead of wrapped around it.

The timing is also the cheapest it will ever be. Both SDKs address nodes by **signature** and
hold **zero** `can://` references (`python-sdk` 1.5.0 pins `codeanalyzer-python==0.3.1`;
`typescript-sdk` has no v2 model). `codeanalyzer-java` is still v1. `codeanalyzer-dotnet` is
unbuilt. The blast radius today is two emitters, one consumer, and the keystone docs. Once
either SDK reaches v2, it also becomes both SDKs and every persisted `analysis.json` cache.

---

## 2. Contract-impact triage

**Does this change the schema v2 output?** Yes — the **id shape** on every durable node
(≥ callable), and therefore every edge endpoint that references one: `call_graph`,
`param_in` / `param_out`, the backfilled `callee` on `call` nodes, and
`extends_ids` / `implements_ids`. It also changes the Neo4j `CanNode` uniqueness key, which
*is* the id.

This is **not additive** under the cross-language parity clause. It reshapes a shared element
of the grammar rather than adding at the leaves, which is why it is designed here rather than
absorbed as a per-analyzer extension.

**Repos affected** — change type: *schema v2 evolution*.

| Repo | What changes | Why |
| --- | --- | --- |
| `codeanalyzer-python` | `codeanalyzer/schema/ids.py`, `--service` CLI flag, Neo4j projection, v2 tests, `schema_version` → 2.1.0, `.claude/SCHEMA_DECISIONS.md` | live v2 emitter (83 `can://` references) |
| `codeanalyzer-typescript` | `src/schema/v2/emit.ts`, `src/schema/v2/model.ts`, `src/build/neo4j/schema.ts`, `--service` flag, `SCHEMA_VERSION` → 2.1.0 | live v2 emitter (34 `can://` references) |
| `cldk-devtools` | `skills/designing-cldk-changes/references/canonical-schema.md` § Identity, `skills/cldk-sdk-frontend/references/schema-contract.md` | the grammar is *defined* here |
| `cldk-skillset` | `skills/codeanalyzer-backend/references/canonical-schema.md` | second copy of the keystone |
| `cocoa` | `cocoa/system/facts.py` — drop the `fn:<service>/` wrapper for v2 languages | the driving consumer |
| `codeanalyzer-dotnet` | spec § Identity (D1) | greenfield; absorbs at zero cost before it is built |
| `docs` | grammar / identity page | user-facing |
| `python-sdk` | **nothing** | pre-v2; zero `can://` references, addresses by signature |
| `typescript-sdk` | **nothing** | pre-v2; zero `can://` references |
| `codeanalyzer-java` | **nothing** | still v1; emits no `can://` ids at all |

---

## 3. Locked decisions

| # | Decision | Rationale |
| --- | --- | --- |
| D1 | **`<service>` is the outermost segment, above `<lang>`.** | It is the only position that lets one service span several languages under a single id root. With `<lang>` outermost, a service written in Python and TypeScript fragments into `can://python/…` and `can://typescript/…` with nothing joining them. It also occupies the URI authority position, which is what a deployment boundary is. |
| D2 | **`<app>` collapses into `<service>`; `--app-name` becomes `--service`, same default (input directory name).** | The two are the same slot at different scale, and keeping both duplicates in the common 1-service-1-app case (`can://python-sdk/python/python-sdk/…`). The default is not invented here — `cocoa/system/detect.py:71-74` already falls back to "no services detected → the root is one service named after the root dir", which is byte-for-byte today's `--app-name` default. |
| D3 | **Ships as `schema_version` 2.1.0, amending v2 in place.** | A MINOR number for a breaking id change is deliberate — see § 7, first caveat. Justified because no consumer holds a `can://` id today: both SDKs are pre-v2, `codeanalyzer-java` is v1, `codeanalyzer-dotnet` is unbuilt, and `cocoa` is changing in the same pass. v2 shipped 2026-07-15 and never reached an SDK, so a v3 would be a major nobody migrates across. |
| D4 | **Cross-service edges (RPC/HTTP/gRPC) are out of scope.** | This spec makes services *addressable*; it does not model calls between them. That is a new edge family plus per-framework detection — several contract decisions, not one — and belongs in planning mode. `python-sdk-f7wire/.claude/FACADE_DECISIONS.md` already parks it as Epic E. The design loop also warns against letting framework/domain concerns reshape structural nodes. |
| D5 | **Shared code imported by several services gets one id per service (duplicated).** | The id answers *"where does this run?"*, not *"what code is this?"*. This falls out of one-run-per-service with no new mechanism, and matches `cocoa`'s model where `ServiceFacts` is per-service. A taint path through `money.py` inside `checkout` is genuinely a different path than the same function inside `email`. Cross-service dedup is the consumer's job. |

---

## 4. The grammar

### 4.1 Durable ids (≥ callable)

```
can://<service>/<lang>/<file>/<type>/<callable-signature>

can://emailservice/python/email_server.py/SendOrderConfirmation(req)
can://checkout/typescript/src/cart.ts/CartService/checkout(userId)
can://python-sdk/python/cldk/core.py/CLDK/_normalize_project_path(path)
```

`<file>` is relative to the analyzed root and may itself contain `/` — ids are opaque handles,
not parsed by splitting on the separator.

### 4.2 Ordinal ids (< callable) — unchanged

The two-tier identity rule is untouched. Statements and synthetic vertices remain addressed
within their callable:

```
<callable-id>@<line>:<col>       e.g. …/SendOrderConfirmation(req)@15:2
<callable-id>@<tag>              e.g. …@entry, …@formal_in:0, …@16:2/actual_in:0
```

### 4.3 Reserved pseudo-segments — unchanged, now service-scoped

`codeanalyzer-typescript` homes external library targets under an `@external` pseudo-segment.
That convention survives and simply inherits the new outermost segment:

```
before   can://typescript/<app>/@external/<module>/<name>
after    can://<service>/typescript/@external/<module>/<name>
```

### 4.4 Invocation model

One analyzer run per service. This is not a new operating model — `cocoa` already does exactly
this:

```python
# cocoa/system/detect.py — services are top-level dirs with a detected language
for d in candidates:                       # root/* and root/src/*
    lang = _language_of(d)
    if lang:
        out.services.append(DetectedService(name=d.name, path=str(d), language=lang))

# cocoa/system/driver.py:104 — one analyzer invocation per service
for svc in system.services:
    facts[svc.name] = runner(svc, cache_dir)
```

| Situation | Invocation |
| --- | --- |
| Single repo, no service concept | `canpy -i ./python-sdk` → `can://python-sdk/python/…` (default = root dir name) |
| Monorepo, N services | N runs: `canpy -i services/checkout --service checkout`, once per service |
| One polyglot service | One run per language, same `--service`; ids join under one authority |

The monorepo case is handled at **invocation**, not in the grammar. The grammar stays one
service per run.

---

## 5. Implementation notes per repo

### `codeanalyzer-python`

`codeanalyzer/schema/ids.py` currently hard-codes the language into the scheme constant:

```python
_SCHEME = "can://python"
def application_id(app_name: str) -> str:
    return f"{_SCHEME}/{app_name}"
```

The service must come *above* the language, so the constant cannot stay a prefix — it becomes
`can://{service}/python`. Everything below `application_id` composes from it unchanged
(`module_id`, `child_id`, `callable_sig_segment`, `ordinal_id` are all pure and parent-relative).

The cache freshness gate in `codeanalyzer/core.py:728` reads:

```python
if getattr(cached, "schema_version", None) != "2.0.0":
    logger.info("stale/incompatible analysis cache (schema_version) — rebuilding")
```

Retargeting it to `"2.1.0"` invalidates every 2.0.0 cache automatically. No separate cache
migration is needed.

### `codeanalyzer-typescript`

`src/schema/v2/emit.ts:350` builds the app root as `can://${LANGUAGE}/${appName}`; the same
inversion applies. `src/schema/v2/model.ts:39` documents the id shape in a comment and must be
updated with it. `homeExternals` (emit.ts:301) needs no logic change — it composes from `appId`.

### `cocoa`

`_fid()` drops the `fn:<service>/` wrapper for services analyzed by a v2 emitter (Python,
TypeScript), since the canonical id already carries the service. The `from_java` path
(`facts.py:63`) consumes **v1** Java output, which has no `can://` ids at all and synthesizes
`fn:<service>/<fqcn>.<sig>` — that path keeps its synthesized key until `codeanalyzer-java`
reaches v2. The wrapper removal is therefore **partial, not total**.

---

## 6. Release plan

Keystone first, analyzers in parallel, consumer last.

1. **Keystone grammar** — `cldk-devtools` and `cldk-skillset`. These *define* what the emitters
   implement, so they land first and become the authority both emitter PRs are reviewed
   against. Landing them second would make the keystone a transcript of whatever shipped.
2. **`codeanalyzer-python` 2.1.0** and **`codeanalyzer-typescript` 2.1.0** — independent of each
   other, ship in parallel.
3. **`cocoa`** — gated on *both* analyzer releases, since it consumes both.
4. **`codeanalyzer-dotnet` spec § Identity** — any time before the analyzer is built.
   **`docs`** — any time after step 1.

There is no single lockstep train. `codeanalyzer-java` stays v1 regardless, so `cocoa` is a
mixed-version consumer no matter how this ships; holding every repo to cut together would buy
a consistency that is unreachable anyway.

---

## 7. Caveats and known gaps

- **The version number understates the break.** 2.1.0 is a MINOR bump for a change that
  rewrites every durable id and the Neo4j MERGE key — `codeanalyzer-typescript`'s own rule
  (`src/build/neo4j/schema.ts`) says MAJOR on a renamed or removed key. This is a deliberate
  erratum, valid *only* because the consumer count is currently zero (D3). It must not be
  cited as precedent once an SDK consumes v2.
- **Persisted Neo4j graphs need a full re-ingest**, not an incremental merge. Every node key
  changes, so a `MERGE` against an existing 2.0.0 database creates a parallel graph rather than
  updating one.
- **Shared libraries are analyzed once per service** (D5). A `libs/` directory imported by six
  services is walked six times and produces six id sets for the same source. Accepted cost;
  the mitigation, if it ever bites, is consumer-side grouping on the `<lang>/<file>/…` suffix.
- **`cocoa`'s Java path keeps its `fn:` wrapper** until `codeanalyzer-java` reaches v2 (§ 5).
  Until then `cocoa` holds two id conventions simultaneously.
- **A deployment bundling two distinct applications loses the app boundary** (D2). The two are
  then distinguished only by file path within the service. No current consumer needs that
  boundary; if one appears, the fix is a new segment, not a revival of `<app>`.
- **Cross-service call edges remain unaddressable** (D4). This spec makes both endpoints
  *nameable*; nothing yet emits an edge between them.

---

## 8. Tracking

Decomposition: **one work item with a per-repo checklist**, filed in `codellm-devkit/.github`,
linking this spec. Chosen over an epic with per-PR sub-issues.

Recorded caveat: the work lands as roughly six pull requests across six repositories that
release on their own clocks, so no single PR closes the issue and the checklist is what carries
per-repo progress.
