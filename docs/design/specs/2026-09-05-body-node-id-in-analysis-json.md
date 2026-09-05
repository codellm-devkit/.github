# Spec: body nodes and parameters carry their `id` in `analysis.json`

Status: draft for review
Date: 2026-09-05
Scope: canonical schema v2 spine, all three analyzers, cross-repo
Origin: codeanalyzer-python#176, python-sdk#320

---

## 1. Summary

Every node in a callable's `body`, and every entry in a callable's `parameters`, gains an `id`
field in `analysis.json`. The value is the **global ordinal id** the keystone already defines and
every Neo4j projection already writes as its `*BodyNode` merge key:

```
<callable-id>@<local>            can://python/app/svc.py/Service/run(self)@51:34
<callable-id>@<tag>              can://python/app/svc.py/Service/run(self)@entry
                                 can://python/app/svc.py/Service/run(self)@formal_in:0
                                 can://python/app/svc.py/Service/run(self)@51:8/actual_in:0
```

Nothing new is minted. The id is computed today, projected to the graph, and dropped from the
JSON model. This spec stops dropping it.

### Why

The two projections are meant to be two views of one structure. Today a body node is addressable
in Neo4j and anonymous in `analysis.json`. A consumer holding a `BodyNode` from the JSON path
cannot name it; its only recourse is to recompose `f"{callable.id}@{body_key}"` itself, which
re-implements the `can://` grammar outside the analyzers. python-sdk did exactly that
(`neo4j_backend.py:1004`, `codeanalyzer.py:1049`) and got it wrong — it composed on `signature`,
not `id`, so its `LocateResult.node_id` joins to nothing in the graph (python-sdk#320). A second
implementation of the grammar drifts the day the first one changes; the fix is to have one.

The nodes affected are precisely the ones dataflow queries use as endpoints. On the Odoo graph
measured in #176:

| kind | count |
| --- | --- |
| `actual_in` | 229,035 |
| `formal_in` | 197,847 |
| `actual_out` | 133,267 |
| `formal_out` | 120,172 |
| `call` | 77,332 |
| `statement` | 55,169 |

## 2. Contract-Impact Triage

**Does this change schema v2 output?** Yes. One new field, `id`, on two node shapes: the body
node and the callable parameter. Additive; no existing field changes; no level boundary moves.

**Is it a language leaf or the spine?** The spine. None of the three analyzers emits it, all three
mint the same value for Neo4j, and the grammar is defined in the keystone (§ Ordinal ids) rather
than by any one language. A python-only field would put a spine concept under
`x-cldk.divergences`, and the SDK's local backend would be able to name a python body node and
not a java one.

**Which repos are touched:**

| Change | Analyzers | SDKs | Docs / schema |
| --- | --- | --- | --- |
| `id` on body node and parameter | `codeanalyzer-python`, `codeanalyzer-java`, `codeanalyzer-typescript` | `python-sdk` — pin bump only; the python model is re-exported from the analyzer, consumer fix is python-sdk#320 | `codeanalyzer-schema` spine + three language schemas; keystone `canonical-schema.md` |

## 3. Where the three analyzers stand today

| | JSON body node | JSON parameter | Neo4j body-node key |
| --- | --- | --- | --- |
| python | `BodyNode`: `kind, span, callee, of, parent, method_name, …` — no `id` (`schema/py_schema.py:127`) | `PyCallableParameter`: no `id` (`py_schema.py:286`) | `_global_ordinal(callable_id, local)` (`neo4j/project.py:123`), mirrors `IdentityMap.global_id` (`dataflow/identity.py:84`) |
| java | `JBodyNode`: no `id` (`schema/JBodyNode.java`) | `JParameter`: no `id` | `V2GraphProjector.java:396`: `localKey.startsWith("@") ? callableId + localKey : callableId + "@" + localKey` |
| typescript | `TSBodyNode`: no `id` (`schema/schema.ts:158`) | `TSCallableParameter`: no `id` (`schema.ts:93`) | `fq(canId, local)` (`dataflow/attach.ts`) |

Three independent implementations of one rule, all agreeing, none exposed in the wire format.
`codeanalyzer-schema/v2/json/analysis.schema.json` lists the spine `BodyNode` as
`kind, span, callee, of, parent`, and every language schema closes its nodes with
`unevaluatedProperties: false` — so the new field fails validation there until it is recorded.

## 4. The design

### 4.1 Body node `id`

Every value in `callable.body` carries `id: string`, equal to the global ordinal id of that node:

- local key starts with `@` (`@entry`, `@exit`, `@formal_in:0`, `@formal_out`) →
  `<callable-id>` + `<local>`;
- otherwise (`15:2`, `15:2/actual_in:0`, `15:2/actual_out`) → `<callable-id>` + `@` + `<local>`.

This is the rule all three projectors already implement (§3). It is stated here once so a fourth
implementation is not needed: an analyzer stamps `id` from the same function it feeds its Neo4j
merge key, and a test asserts the two agree on every body node.

`id` is present at every level the node exists — `call` nodes at L1, statements and bookends at
L3, param vertices at L4. It is never null. Monotonicity is unaffected: the field is added, never
rewritten, and the `callee: null → id` refinement stays the only sanctioned mutation.

### 4.2 Parameter `id`

Every entry in `callable.parameters` carries `id: string`, equal to the id of the `formal_in`
vertex that represents it at L4:

```
parameters[i].id == <callable-id>@formal_in:<i>
```

where `i` is the parameter's position in the `parameters` list. This is a **forward reference
below L4**: the `formal_in` node it names does not exist in `body` until `-a 4`. That is
acceptable because the id is a promise about identity, not about presence — exactly as a `call`
node's `callee` names a callable the caller may not have in scope — and it lets a consumer relate
a declared parameter to the dataflow vertex that carries it without waiting for L4 to find out
what the vertex will be called.

**Ordering invariant.** The rule is only true if the L4 formal-vertex builder indexes formals in
`parameters`-list order. It does, in all three today:

- python: `_param_names` (`dataflow/access_paths.py:203`) and the symbol table's parameter walk
  (`symbol_table_builder.py:598`) both visit `posonlyargs, args, vararg, kwonlyargs, kwarg`, and
  `build_formals` (`dataflow/sdg.py:145`) allocates declared params first, captures and globals
  after — so indices `0..len(parameters)-1` align, and the extra `formal_in` vertices for
  captures/globals sit past the end;
- java: `SdgVertices.java:66` puts `@formal_in:i` for `params.get(i)`;
- typescript: `attach.ts:140` keys `@formal_in:${n}` from the parameter's ordinal.

It is a **new contract obligation** on each analyzer, gated by a test at L4:
for every callable and every `i`, `body[parameters[i].id].of` is `parameters[i].name` (or the
analyzer's `of` spelling for it). If a language ever needs a different formal order, it changes
`parameters` order, not the index rule.

### 4.3 What does not get an id

- **`PyCallsite`** (python's legacy parallel call-site list). #120 folded its facts into the
  `call` body node, which now carries the id. Adding an id to the legacy list would give one
  fact two homes with two spellings, the thing #120 removed. Out.
- **Class attributes and local variables** (`PyClassAttribute`, `PyVariableDeclaration`,
  `PyAttribute`/`PyVariable` in Neo4j). They are not body nodes; their Neo4j ids are minted from
  the owner's *signature*, not its `can://` id, and are being redone under
  [`2026-09-02-prune-scope-on-can-id-prefix.md`](2026-09-02-prune-scope-on-can-id-prefix.md).
  Adding their JSON id here would freeze the wrong grammar. Out; revisit after #173 lands.
- **Nothing else changes.** `span`, `callee`, `of`, `parent` and the call-site fields are
  untouched. Local ids stay the `body` map keys; edge lists inside a callable keep using local
  ids as endpoints, because the callable id is implicit there. Application-scope lists
  (`param_in`, `param_out`, `config_use`) already use the global form.

### 4.4 Versioning

- **`analysis.json` `schema_version` stays `2.0.0`.** The v2 line has no released `2.*` consumer
  contract to break, and the change is additive.
- **Graph contract unchanged.** Neo4j already carries the id; no label, property or index moves.
- Each analyzer ships as a **minor** release.
- **Model layer.** Python's `BodyNode.id` and `PyCallableParameter.id` follow `PyCallable.id`'s
  shape: `id: str = ""`, so an `analysis.json` written before this change still parses. The
  analyzer's own cache is invalidated by `analyzer_version`, not by shape, so a stale cache
  cannot serve an id-less payload from a new analyzer.

## 5. Mechanics that must not be got wrong

- **Stamp from one function.** The id must come from the same function the Neo4j projector uses
  for its merge key (python: `_global_ordinal` / `IdentityMap.global_id`; java: the
  `V2GraphProjector` helper at line 396; typescript: `fq`). Do not inline the `@`-rule at each
  emit site. Python has four body-node construction sites (`schema/l1_body.py:12`,
  `dataflow/builder.py:276,279,291,538`); one stamping pass over `body` after each level's
  emission is smaller than threading the callable id through all four.
- **The `@` rule is on the local key, not the kind.** `@formal_out` concatenates,
  `15:2/actual_out` gets a separator. Keying on `kind` would get `actual_*` wrong.
- **`callee` is a callable id, `id` is a body id.** A `call` node's `callee` is
  `can://…/callee(sig)` (no `@`), its `id` is `can://…/caller(sig)@line:col`. Tests should
  assert the two are never equal, which catches a copy-paste of the wrong field.
- **Parameter index counts every declared parameter**, including `self`/`this`-style receivers
  where the language lists them in `parameters`, and variadic/keyword parameters, in list order.
- **The JSON schema in `codeanalyzer-schema` is closed.** Each language schema fails validation on
  an unrecorded field, by design. Add `id` to the spine `BodyNode` and the spine parameter node
  as required, regenerate each language's sample from the release that emits it, and run
  `scripts/check.py`.

## 6. Decomposition and release plan

Tracking: one epic in `codellm-devkit/.github`, this spec linked, children as cross-repo
sub-issues filed when each is picked up.

| Unit | Repo | Release | Gate |
| --- | --- | --- | --- |
| 1 | codeanalyzer-python (#176) | 1.5.0 | parity test: every `body[k].id` equals the Neo4j merge key for `k`; L4 formal-order test (§4.2); monotonicity CI green |
| 2 | codeanalyzer-java | 3.1.0 | same two gates |
| 3 | codeanalyzer-typescript | 1.3.0 | same two gates |
| 4 | codeanalyzer-schema | — | spine + each language schema updated as its analyzer ships; samples regenerated; `check.py` green |
| 5 | python-sdk | next minor | pin `codeanalyzer-python==1.5.0`; python-sdk#320 reads `BodyNode.id` in the local backend and `b.id` in Neo4j; parity test across backends |

Order: **python first** — it is the requesting analyzer, and unit 5 is blocked on it alone. Java
and typescript follow on their own clocks; no analyzer reads another's output, so there is no
lockstep among units 1–3. Unit 4 trails each of 1–3. Unit 5 needs only unit 1.

Keystone doc (`canonical-schema.md` in cldk-devtools) adds `id` to the body-node field catalog
and the parameter row alongside unit 1.

## 7. Open questions for review

- **Required or optional in the spine?** This spec says required for both. The alternative —
  optional, so an analyzer may lag — trades one obligation for a consumer that has to handle
  absence forever. Recommend required.
- **Java receiver.** Java does not list `this` in `parameters`; python lists `self`. The index
  rule is per-language-consistent either way, but a cross-language consumer that assumes
  `parameters[0]` is the receiver is wrong in java. Not this spec's problem, noted so nobody
  papers over it with an offset.
