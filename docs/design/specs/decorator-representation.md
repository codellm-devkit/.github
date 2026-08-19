# Spec: structured decorator representation in canonical schema v2

Status: draft for review
Date: 2026-08-19
Scope: canonical schema v2 node fields, `codeanalyzer-python` + `python-sdk`

---

## 1. Summary

`codeanalyzer-python` emits decorators as flat source strings:

```json
"decorators": ["trace('hot')", "functools.lru_cache(maxsize=128)"]
```

The keystone (`designing-cldk-changes/references/canonical-schema.md`, `type` and `callable`
field tables) specifies the opposite, at L1, and says so explicitly:

> `decorators` | `decorator[]` | 1 | Structured `{ name, args[], span }` — **not** flat strings.

`codeanalyzer-typescript-v2` already implements the structured shape (`src/schema/schema.ts`,
`TSDecorator`) and applies it to classes, callables, parameters, and properties. Python is the
divergent analyzer, not the pioneer.

This spec makes Python conform: `decorators` becomes a list of structured `PyDecorator` objects
carrying a resolved `qualified_name`, on four node kinds instead of one.

**No `schema_version` bump is claimed.** Schema v2 is a moving target until Java, Python, and
TypeScript settle on a stable shape; it is not a released, versioned contract that individual
changes amend. This change *is* part of that convergence — Python adopting the decorator shape
TypeScript already emits — not a release against a frozen contract.

### Why now

Three separately-filed issues turned out to be one contract decision:

- **#127** — `PyClass` has no `decorators` field at all, so `@dataclass`, `@app.errorhandler`,
  and Django/pydantic class decorators are dropped silently.
- **#128** — the strings carry no identity, so `@app.route` and `@blueprint.route` are
  unequal even when they resolve to the same function.
- **#129** — decoration is not recoverable from the call graph.

All three are answered by adopting the shape the contract already specifies and the TypeScript
analyzer already ships. Designing them separately would have decided the same thing three times.

---

## 2. Contract-impact triage

**Does this change schema v2 output?** Yes — an existing L1 field changes type, and gains three
new node kinds.

| Repo | What changes | Why it is in scope |
| --- | --- | --- |
| `codeanalyzer-python` | `schema/py_schema.py` (`PyDecorator`, four carriers), `syntactic_analysis/symbol_table_builder.py`, `neo4j/project.py` + `neo4j/schema.py`, `schema_version` → 2.1.0, `.claude/SCHEMA_DECISIONS.md` | the emitter |
| `python-sdk` | `cldk/models/python/projections.py:58` mirrors `decorators: List[str]` | the consumer of the field |
| docs | README schema section, `schema.neo4j.json` | regenerated from source at release |

`codeanalyzer-typescript-v2` is **not** in scope — it already conforms. `codeanalyzer-java` is
**not** in scope: it uses `annotations: List[str]`, a different field name on a v1 analyzer, and
aligning it is its own decision.

---

## 3. The shape

```python
class PyDecorator(BaseModel):
    name: str                              # locally written name: "trace", "lru_cache"
    qualified_name: Optional[str] = None   # resolved FQN: "app.trace", "functools.lru_cache"
    positional_arguments: List[str] = []   # raw source fragments: ["'hot'"]
    keyword_arguments: Dict[str, str] = {} # key -> source fragment: {"maxsize": "128"}
    span: Optional[Span] = None
```

Mirrors `TSDecorator` field-for-field, with one deliberate divergence: **`span` instead of TS's
flat `start_line`/`end_line`/`start_column`/`end_column` quadruple.** Python's v2 models use
`Span` (with utf-8 `bytes` offsets) everywhere, the keystone specifies `span`, and TypeScript's
own canonical v2 model (`src/schema/v2/model.ts`) also uses `span?: Span` — the flat quadruple in
`schema.ts` is that repo's older internal shape, not the direction of travel.

`qualified_name` is the identity mechanism, **not** a `can://` id. TypeScript coined
`qualified_name` first; the parity clause makes coining a second mechanism for the same concept
permanently wrong. A `can://` id for decorators can be added later as an additional field if
cross-repo joins ever need one — it is not needed to answer "do these two spellings mean the
same decorator".

### Carriers

| Node | Today | After |
| --- | --- | --- |
| `PyCallable` | `List[str]` | `List[PyDecorator]` |
| `PyClass` | *absent* | `List[PyDecorator]` |
| `PyClassAttribute` | *absent* | `List[PyDecorator]` |
| `PyCallableParameter` | *absent* | `List[PyDecorator]` |

Matches TypeScript's four carriers. `PyCallableParameter` is included for parity even though
Python has no parameter-decorator syntax — the field will be empty in practice, and an empty
list on every language beats a field that exists in one analyzer and not another.

All four are **L1** data — the symbol table already visits every one of these nodes. Decorators
must not be gated behind a level.

---

## 4. Decisions

| # | Decision | Rationale |
| --- | --- | --- |
| D1 | `decorators` becomes `List[PyDecorator]` under the **same field name**, breaking existing readers. | The keystone reserves the name for the structured shape. A parallel field would leave `decorators` meaning a string list in Python and an object list in TypeScript — the exact term-coined-twice failure the parity clause exists to prevent. |
| D2 | Identity is `qualified_name: Optional[str]`, not a `can://` id. | TypeScript coined it first. Parity beats novelty. |
| D3 | Four carriers: callable, class, class attribute, callable parameter. | Matches TypeScript. The keystone names type/callable/field; parameter is TS's addition and is cheap to match. |
| D4 | `span` rather than TS `schema.ts`'s flat line/col quadruple. | Python v2 and the canonical TS v2 model both use `Span`; the flat form is legacy. |
| D5 | **No decoration edge family.** #129 closes as answered by D2. | `qualified_name` makes decoration joinable in both projections without coining a cross-language edge type no other analyzer has. Revisit only if real queries fall short. |
| D6 | **No `schema_version` bump.** Schema v2 is a moving target until Java/Python/TS converge, so there is no stable contract number for this change to amend. | The version field is not a release marker while v2 is still settling. Convergence between the three analyzers is the milestone, not a number. |
| D6a | The **cache-invalidation guard** must still reject caches built with the old shape. `core.py:728` hard-codes `!= "2.0.0"`; it needs a cache epoch that moves when the emitted shape moves. | This is a correctness requirement, not a versioning one: without it a stale cache round-trips old-shape decorators through a new analyzer. Deliberately decoupled from any contract-version claim. |
| D7 | Neo4j `SCHEMA_VERSION` → 2.1.0; `:PyDecorator` keeps merging on `name`, gains `qualified_name` as a property. | Re-keying the shared `:PyDecorator` node on `qualified_name` would change merge behaviour for existing graphs and is not required by any decided use case. |
| D8 | Release order: **analyzer first, `python-sdk` follows.** | Chosen deliberately over an SDK-tolerant-first sequence. See the caveat in § 6. |

---

## 5. What this does not do

- Does **not** touch `codeanalyzer-java`'s `annotations: List[str]`, or decide whether
  `annotations` and `decorators` should ever be one field.
- Does **not** add `modifiers` to Python nodes, and does **not** resolve how
  `@staticmethod` / `@classmethod` / `@property` should be modelled — they stay decorators,
  string-matched, as today.
- Does **not** address async representation (**#130**), which stays an independent issue.
- Does **not** fix the PyCG call-graph gap found while investigating #129 —
  `functools.lru_cache` produced no call edge while `functools.wraps` in the same file did.
  That is a real bug with an undiagnosed root cause and needs its own issue.
- Does **not** add framework knowledge. Recognising that `@app.route` means "HTTP entrypoint"
  stays the SDK's job, per the provider/client boundary.

---

## 6. Caveats and accepted risks

- **`python-sdk` breaks with no version signal.** It ships `decorators: List[str]` today
  (`cldk/models/python/projections.py:58`), and there is no contract-version bump to warn a
  consumer that the field changed shape — because v2 carries no such guarantee yet. The
  mitigation is the SDK update and the release note, and the honest statement that schema v2 is
  not stable enough to pin against.
- **The release order leaves a broken window.** Analyzer-first means a `python-sdk` release
  exists that will mis-parse the new analyzer's output until the SDK catches up. The window is
  the accepted cost of not writing tolerant dual-shape parsing.
- **Stale caches are the sharp edge, not the version number.** `core.py:728` accepts any cache
  whose `schema_version` equals the current literal. Change the shape without moving that guard
  and a cached analysis silently feeds old-shape decorators to a new analyzer. D6a covers it.
- **`qualified_name` resolution is best-effort.** Jedi will not resolve every decorator —
  dynamic attribute access, conditional imports, decorators built at runtime. `None` is the
  normal case for those, not an error, and resolution must never raise.
- **`PyCallableParameter.decorators` will always be empty in Python.** Carried for
  cross-language parity only. If that is judged dead weight at review, dropping it is a
  one-line change to this spec and to D3.

---

## 7. Definition of done

- `PyDecorator` exists with the five fields in § 3, and all four carriers populate it
- The fixture in #128 emits `qualified_name: "functools.lru_cache"` for
  `@functools.lru_cache(maxsize=128)` and `{"maxsize": "128"}` as its keyword arguments
- `@dataclass` on a class emits on the **class** node, not just its methods
- `@app.route` and `@blueprint.route` bound to the same function produce the same
  `qualified_name`
- An unresolvable decorator emits `qualified_name: null` and does not raise
- `analysis.json(-a 1) ⊆ ... ⊆ analysis.json(-a 4)` still holds
- Neo4j exposes `qualified_name` on `:PyDecorator` and `PY_DECORATED_BY` traverses from
  `:PyClass`
- A cache written before this change is **rejected and rebuilt**, not round-tripped — the
  regression test writes an old-shape cache and asserts the rebuild
- `python-sdk` parses the new shape
- `.claude/SCHEMA_DECISIONS.md` carries a one-line entry per decision D1–D8, linking here
