# Spec: codeanalyzer-dotnet adopts application-scoped identity

Scope: amends [`codeanalyzer-dotnet.md`](codeanalyzer-dotnet.md) — D1 and §4.4 (identity),
       §6 (release plan); canonical schema v2 spine already shipped this change for the
       other analyzers
Origin: python-sdk#440 (`feat!: scope URI resolution and deduplication by application
        identity`, shipped in cldk v2.1.0, 2026-10-31); tracking python-sdk#435;
        producer precedent codeanalyzer-go#40 (shipped via codeanalyzer-go#41)

---

## 1. Summary

The `codeanalyzer-dotnet` design spec (`codeanalyzer-dotnet.md`, 2026-08-06) was written
against the pre-`#440` identity grammar. Since then, **python-sdk#440 shipped a breaking
change to the shared spine**: `can://` roots are now **application-scoped**, qualified by two
leading segments, and every analyzer stamps an `identity_version: 2` marker. The Python,
TypeScript, Java, and Go analyzers have adopted it; a greenfield C# analyzer must be *born*
emitting it, never the old shape.

This spec amends two things in the dotnet spec and **nothing else** — every other decision
(D2–D16) stands unchanged:

1. **Identity grammar (amends D1 and §4.4)** — adopt the two-segment
   `can://<namespace>/<application>/csharp/…` root and `identity_version: 2`.
2. **Release plan (amends §6)** — retarget the `python-sdk` pins from the pre-2.0 `1.6.0` /
   `1.7.0` to the current `cldk` 2.x line.

All eight decisions locked in the 2026-11-08 design session are consistent with the committed
dotnet spec; this document records only the delta that `#440` forces.

---

## 2. Contract-Impact Triage

**Does this change schema v2 output?** Yes — but the *shape* change already landed on the
spine in `#440`. For C# specifically it changes the **id grammar** every node emits (two root
segments instead of one) and adds the `identity_version: 2` envelope marker. It is additive at
the envelope and a **grammar change** at the id — not a new node, field, kind, or edge family.
The C# language-leaf decisions (D6–D11) are untouched.

**Which repos are touched:**

| Change | Analyzers | SDKs | Docs |
| --- | --- | --- | --- |
| Two-segment id grammar + `identity_version: 2` | `codeanalyzer-dotnet` (greenfield — adopt at birth) | `python-sdk` `cldk/models/csharp/` parses the new root; `cldk/analysis/csharp/` forwards `--namespace`/`--application` | this spec + the dotnet quickstart |

No other analyzer is touched — they adopted `#440` already. The TS SDK remains out of scope
per the base spec.

---

## 3. The amendment

### 3.1 Identity grammar — amends D1 and §4.4

**Before (base spec D1 / §4.4):**

```
can://csharp/<app>/<file>/<Type>[/<Nested>]/<member-signature>
```

**After (this spec):**

```
can://<namespace>/<application>/csharp/<file>/<Type>[/<Nested>]/<member-signature>
```

The two leading root segments are **prepended before** the `csharp` language token. The entire
language-and-entity suffix from §4.4 — `csharp/<file>/<Type>/<signature>`, the one
`CanId.SignatureOf(ISymbol)` output, the `#get`/`#set` accessor suffixes, and the
`<callable-id>@<line>:<col>` ordinal ids below the callable — is **preserved unchanged** as
that suffix. Only the root grows.

**Both root segments are always present**, including the monolith case where they coincide.

#### Derivation (the Go precedent, verbatim)

The shipped rule is language-agnostic and `codeanalyzer-dotnet` adopts it **exactly** as
`codeanalyzer-go` did — no .NET-specific identity logic:

- `--application <name>` sets `<application>`; **omitted ⇒ the selected input root basename**.
- `--namespace <name>` sets `<namespace>`; **omitted ⇒ the resolved application name**.
- An **omitted** namespace is distinguished from an **explicit empty** one (the latter is
  rejected, see encoding rules).

Worked examples:

```
monolith MyApp (no flags):
  can://MyApp/MyApp/csharp/src/Api/UserController.cs/UserController/Save(System.String)System.Boolean

explicit (--namespace online-boutique --application payments):
  can://online-boutique/payments/csharp/src/.../Checkout(…)…
```

> **This supersedes the base spec's §4.4 bullet** "`<app>` — `--app-name`, else the `.sln`
> name, else the input directory basename." The resolution rule is now the two-segment
> application/namespace rule above. `--app-name` is retained as a **deprecated alias** for
> `--application` so existing invocations keep working.

#### Encoding rules (shared with `#435`, adopted unchanged)

Root segments are **case-sensitive raw strings**. Bytes outside ASCII
letters/digits/`-._~` are percent-encoded as **uppercase** escapes, **without** trimming or
Unicode normalization. Rejected: empty / whitespace-only names, control characters, unpaired
surrogates, `/` or `\`, and a whole-name `.` or `..`.

```
--application "a%b" --namespace "team space"  →  can://team%20space/a%25b/csharp/…
```

### 3.2 The `identity_version` marker — new, amends §4.6 envelope

The analysis envelope and the `application` anchor carry an integer **`identity_version: 2`**,
**alongside** the unchanged `schema_version: "2.0.0"`. The two are orthogonal: `schema_version`
tracks the node/edge model, `identity_version` tracks the id grammar. Outputs carrying an
incompatible identity marker are rejected **read-only** and regenerated from source — never
alias-translated. Integer markers are validated **before** model coercion (per `#440`).

The raw resolved values are retained on the `application` record as **`application_name`** and
**`identity_namespace`**, so a consumer recovers them without re-parsing the `can://` id — the
same anti-pattern the `application.name` spine spec (`2026-09-23-application-name-in-analysis-json.md`)
removed.

```jsonc
{
  "schema_version": "2.0.0",
  "identity_version": 2,
  "language": "csharp",
  "max_level": 2,
  "analyzer": { "name": "codeanalyzer-dotnet", "version": "0.1.0" },
  "application": {
    "id": "can://MyApp/MyApp/csharp",
    "application_name": "MyApp",
    "identity_namespace": "MyApp",
    "kind": "application",
    "symbol_table": { /* … */ }
  }
}
```

### 3.3 Cross-application isolation (new test gate, amends §5)

Because roots now carry application identity, two projects in one solution that declare a
same-named type no longer collide. Add to the base spec's `SchemaTests`:

> **Cross-application isolation** — on the `Solution.MultiProject` fixture, a type name present
> in two projects produces **two distinct `can://` ids** differing in the `<application>`
> segment, with **no cross-application `call_graph` join** between them. Neo4j canonical-id
> indexes deduplicate modules **per application**.

This repurposes the existing `Solution.MultiProject` fixture (base spec §5) — no new fixture
needed; the gate is added, and `--application` is driven per-project in that fixture's CI
invocation.

### 3.4 Release plan — amends §6

The base spec's §6 pins `python-sdk` **1.6.0** / **1.7.0**. Those are pre-2.0 numbers written
before `cldk` reached the 2.x line. Retarget:

| Release | Contents | Gates | Blocks |
| --- | --- | --- | --- |
| `cansharp` 0.1.0 | L1 + L2, JSON + Neo4j, five platform wheels, **`identity_version: 2` + two-segment roots** | L1/L2 schema gates, Neo4j conformance, **cross-application isolation (§3.3)** | — |
| `cldk` 2.2.0 | `cldk/models/csharp/` + `cldk/analysis/csharp/`, pins `cansharp==0.1.0`, forwards `--namespace`/`--application` | SDK round-trip on all six fixtures, **incompatible-marker rejection** | needs 0.1.0 on PyPI |
| `cansharp` 0.2.0 | L3 (CFG / CDG / DDG) | L3 gates + monotonicity | — |
| `cldk` 2.3.0 | L3 surface, `reaching-defs` accepted in `prov` | | needs 0.2.0 |
| `cansharp` 0.3.0 | L4, after the points-to design session (base spec §4.9) | L4 gates + monotonicity | needs its own session |
| `docs` | language matrix + C# quickstart | | tracks 0.1.0 |

**Two independent version clocks** (confirmed against the `cldk` release history): `cldk` runs
`2.x` and pins an exact `cansharp` version per release; `cansharp` runs its own `0.x`. This is
how `cldk` already pins `codeanalyzer-java` / `-python` / `-typescript` / `-go`, so the C#
analyzer's `0.1.0 → 0.2.0 → 0.3.0` clock is independent of the SDK's `2.2.0 → 2.3.0`. There is
no conflict between the two schemes; the base spec's `1.6.0`/`1.7.0` were simply stale.

---

## 4. Decisions carried, unchanged

For the record, these base-spec decisions are **not** reopened by this amendment, and the
2026-11-08 design session confirmed each:

- **D6** partial-type merge (`parts`, `declared_in`).
- **D7** properties/events → `field` node + `#get`/`#set` callables. *(Per codeanalyzer-dotnet#40
  this is recorded as a **divergence** from codeanalyzer-typescript's first-class accessor kinds
  `getter|setter`, not a precedent match — the C# shape itself is unchanged.)*
- **D8** `record` / `record_struct` / `delegate` as first-class `type` kinds;
  `positional_parameters`, `invoke_signature`.
- **D9** `ref_kind ∈ {ref,out,in,ref_readonly}` at L1; by-ref `@formal_out:<name>` vertex at L4.
- **D10** namespace as a `module`/`type` string field (a C# namespace is a solution-wide naming
  scope — distinct from the `<namespace>` **identity** segment introduced here, which is the
  `#440` application-ownership root). See §5 note.
- **D13** CFG substrate = Roslyn `ControlFlowGraph`. **C#-specific `cfg` edge kinds
  (`await_resume` / `yield_resume`) remain deferred to the L3 design session** — not fixed here.
- **D15** UTF-8 byte span offsets. **D16** per-language Neo4j prefix `:CsNode` / `CS_*`.

---

## 5. One naming hazard to not get wrong

`#440` introduces an **identity** `<namespace>` segment (the application-ownership root), and
C# *also* has a language-level **`namespace`** (D10, a `module`/`type` string field). **These
are different concepts and must not be conflated:**

- The **identity namespace** (`<namespace>` root segment) is resolved from
  `--namespace` / the default rule (§3.1). It is about *which application owns this id*.
- The **C# `namespace` field** (D10) is the declared `namespace Foo.Bar` scope of the code. It
  is about *what the code calls itself*.

A C# file in `namespace Payments.Api` analyzed in application `payments` under namespace
`online-boutique` yields a type whose **id root** is `can://online-boutique/payments/…` and
whose **`namespace` field** is `Payments.Api`. The analyzer must source each from its own input
and never let one default from the other.

---

## 6. Decomposition and release plan

Tracking: **one feature-request issue** in `codellm-devkit/.github` that links this spec and
references the existing epic (`.github`#34), the errata child (#40), and the origin
(python-sdk#440, #435). This amendment is one unit of design work, closed by the spec commit; it
does not spawn a new epic — the C# language epic (#34) already exists and its sub-issues are
filed just-in-time. The release-plan retarget in §3.4 is the authoritative plan that §6 of the
base spec now points to.
