# Spec: codeanalyzer-dotnet (`cansharp`) — a C# analyzer for CLDK

Status: draft for review
Date: 2026-08-06
Scope: new language pack (C#), canonical schema v2, greenfield

---

## 1. Summary

A Roslyn-based static analyzer for C# that emits the canonical CLDK analysis schema v2
(`analysis.json` + a Neo4j projection), shipped as a self-contained native binary
distributed on PyPI under the package name **`cansharp`**.

The repository is `codellm-devkit/codeanalyzer-dotnet` — named for the platform family so
VB.NET can be added later as an additive language token — but the first (and currently only)
language it analyzes is C#, and every id it emits carries the `csharp` token.

The analyzer is greenfield: there is no v1 output to migrate from, so it targets canonical
schema v2 (`schema_version: "2.0.0"`) directly.

---

## 2. Contract-impact triage

**Does this change the schema v2 output?** Yes — a new language pack. All changes are
*additive per the parity clause*: new `kind` values on `type`/`callable`/body nodes, and new
typed fields. No shared field is renamed, no shared `kind` repurposed, no new edge family.

**Repos affected:**

| Repo | What changes | Why |
| --- | --- | --- |
| `codeanalyzer-dotnet` | everything — new repo | the analyzer itself |
| `python-sdk` | new `cldk/models/csharp/` model package + `cldk/analysis/csharp/` facade; `cansharp` added to dependencies | consumes the new analyzer |
| `docs` | language-support matrix, C# quickstart | the analyzer is user-facing |

A TypeScript SDK is not currently a consumer of new analyzers and is out of scope.

### Why v2 and not v1

The initial instruction was to emit schema v1 for backwards compatibility with the shipping
Python SDK. On inspection that premise does not hold:

- `python-sdk` 1.5.0 pins `codeanalyzer-python==0.3.1` and `codeanalyzer-typescript==0.4.3`.
- Both already emit `schema_version: "2.0.0"` — `can://` ids, `body{}`, `Span` with byte
  offsets, identity-only `{src,dst}` edges
  (`codeanalyzer-python-v0.3.0/codeanalyzer/schema/py_schema.py:526`).
- `cldk/models/python/__init__.py` re-exports those v2 models directly.
- Only the Java path is still v1 (rich-edge `JGraphEdges`, `JCompilationUnit`).
- There is no dotnet model in the SDK at all, so there is no existing consumer to stay
  compatible with.

Emitting v1 would mean authoring a new v1 consumer and then migrating it. v2 confirmed with
the maintainer.

---

## 3. Locked decisions

Decided with the maintainer during the design loop. These are transcribed to
`codeanalyzer-dotnet/.claude/SCHEMA_DECISIONS.md` when the repo is bootstrapped.

### Foundational

| # | Decision | Rationale |
| --- | --- | --- |
| D1 | **Scope: C# only. Language token `csharp`.** Ids are `can://csharp/<app>/…`; `language: "csharp"`. | VB.NET and F# can be added later as additive tokens in the same repo. A `dotnet` token would lie about scope and force an id migration when VB arrives. |
| D2 | **Compilation front end: read `obj/project.assets.json` + `.csproj` directly. No MSBuild.** | `dotnet restore` already writes every resolved package dll path to `project.assets.json`. Reading it and calling `CSharpCompilation.Create` gives a full `SemanticModel` with no `MSBuildLocator`, no SDK-version coupling, and a genuinely self-contained binary. MSBuildLocator version mismatch is the most common failure mode for Roslyn-based tools. Prereq: the target repo has been restored — true of any repo that has been built. |
| D3 | **Distribution: self-contained single-file ReadyToRun binary, per-platform wheels on PyPI as `cansharp`.** | Roslyn leans on reflection, which NativeAOT trimming breaks in ways that surface at runtime on user code rather than in our tests. R2R keeps startup acceptable without that risk. Per-platform wheels mean pip downloads exactly one binary, offline-installable, no install-time network. |
| D4 | **Project identity: a `project` typed field on the `module` node.** | `symbol_table` stays a flat `{relative-path → module}` map, as every SDK model assumes. A container node between `application` and `symbol_table` would be a spine change, not a leaf addition. One analysis.json per solution keeps cross-project call edges resolvable. |
| D5 | **First release covers L1 + L2 and both projections; L3 and L4 follow as separate releases.** | Symbol table + call graph is already the whole SDK surface most consumers use. It unblocks `python-sdk` early and keeps the monotonicity gate small while the foundation settles. |

### Language modelling

| # | Concept | Java | Python / TS | **C# decision** | Rationale |
| --- | --- | --- | --- | --- | --- |
| D6 | **Partial types** | no precedent | TS declaration merging (nearest cousin) | **Merge into one `type` node homed at the primary declaring file.** `parts: [{file, span}]` lists every declaration site; members declared elsewhere carry `declared_in: "<file key>"` so `span.bytes` slices the right `module.source`. | Roslyn already merges partials for us — one `INamedTypeSymbol` with several `DeclaringSyntaxReferences` — so the merged view is the *natural* output, not extra work. Per-file fragments would force every SDK to re-merge, and any consumer that forgets sees half a class. Partials are pervasive in real C# (WinForms, EF, ASP.NET, source generators). |
| D7 | **Properties / events / indexers** | flat fields | TS accessors are first-class *kinds*: `"getter" \| "setter"` (`src/schema/v2/model.ts`) | **Both views.** `fields{}` gets a node `kind:"property"` (declared type, modifiers, `is_auto`); non-auto accessors *also* appear in `callables{}` keyed `Bar#get` / `Bar#set`, with `kind:"method"`. Events mirror it (`kind:"event"` + `#add`/`#remove`). Indexers are callables keyed `this[]`. | Idiomatic C# exposes state as properties, so a callables-only model empties `get_fields()`. But accessors hold real logic and need CFG/DDG at L3, so a fields-only model is a dataflow blind spot. The dual view is **not** inherited from TypeScript — TS distinguishes accessors by `kind`, and matching it would add two kinds to the shared callable vocabulary. Keying instead keeps that vocabulary to the four the keystone lists; the divergence is deliberate and recorded in § 8. |
| D8 | **Type kinds** | `is_interface`/`is_enum` booleans | TS sibling collections | **First-class `kind` values: `record`, `record_struct`, `delegate`** alongside `class`/`struct`/`interface`/`enum`. Records carry `positional_parameters[]` (primary constructor); delegates carry `invoke_signature`. | Additive per the expansion rubric. "Find all records" becomes a `kind` filter, consistent with how every other type distinction is expressed. A delegate modelled as a bodyless `callable` would break the L3 assumption that every callable is a CFG root. |
| D9 | **`ref` / `out` / `in` parameters** | no precedent | no precedent | **`param.ref_kind: ref\|out\|in\|ref_readonly`** (absent = by value) at L1. At L4, each `ref`/`out` param gets an `@formal_out:<name>` vertex alongside `@formal_out` for `$ret`, with a matching `actual_out` per call site. | Canonical already sanctions `formal_out` `of` being "`$ret` or a by-ref param" — C# is the first language to exercise it. Without this, every `TryParse`/`TryGetValue` result is a dataflow dead end, and those are everywhere in C#. |
| D10 | **Namespaces** | packages | TS nested `namespace` container node (V1) | **A `namespace` string field on the `module` node.** Multi-namespace files key their types `A.F` / `B.F` and each type node carries its own `namespace`. | The canonical `module` field exists for exactly this. A C# namespace is a pure naming scope spanning the whole solution, so a per-file container node would imply a containment that is not real — unlike a TS namespace, which is a value-producing per-file scope. |
| D11 | **`error_channel`** | `throws` clause | Go `(T, error)`, Rust `Result` | **Absent by default; populated from XML doc `<exception cref="…">` when present.** | C# has no checked exceptions, so nothing in the signature declares what a method throws, and "a fact is present or absent — there is no null". An authored `<exception>` tag is a real declared-intent fact and the only such signal C# offers. A `throw`-statement scan was rejected: it is a body fact, not a signature fact, and unsound in both directions. |

### Analysis substrate

| # | Concept | **C# decision** | Rationale |
| --- | --- | --- | --- |
| D12 | **L2 virtual dispatch** | Static declared-symbol edge always, `prov:["roslyn"]`. CHA expansion behind `--devirtualize`, adding edges to every override/implementation via `SymbolFinder`, `prov:["cha"]`. | The flag only ever *adds* edges, so the monotonicity gate holds either way and provenance keeps the two distinguishable. Always-on CHA fans `IEnumerable<T>.GetEnumerator()` out to hundreds of edges; never-on leaves every interface-dispatched call a dead end, which on DI-based C# is most calls that matter. |
| D13 | **L3 CFG substrate** | Roslyn's `ControlFlowGraph` (over `IOperation`), flattened from basic blocks to statement nodes. | Roslyn already models async state machines, iterator lowering, `using`, `foreach`, pattern switches, and exception regions correctly. Hand-rolling that is exactly where a bespoke C# CFG goes wrong. `IOperation.Syntax` spans give `line:col` ids directly. Basic blocks are flattened rather than kept, so statement-level identity matches every sibling analyzer. |
| D14 | **DDG provenance token** | Reuse the sanctioned additive token **`reaching-defs`**. | Roslyn's CFG is not SSA (flow captures cover temporaries, not locals) and there is no points-to oracle, so `["ssa"]` and `["points-to"]` would both overclaim. `reaching-defs` was already sanctioned as an additive token for codeanalyzer-typescript (its issue #32); reusing it beats coining a third. |
| D15 | **Span offset units: UTF-8 byte offsets**, matching codeanalyzer-python. An offset table is computed once per file alongside the `source` blob, converting Roslyn's UTF-16 `TextSpan` positions. | The canonical field is named `bytes`, and Python is the reference analyzer the schema was written against. codeanalyzer-typescript emits UTF-16 char offsets (its V5) and is the outlier to be fixed separately — with a non-ASCII identifier or string literal, char and byte offsets disagree on the same logical position. |
| D16 | **Neo4j labels are per-language prefixed**: nodes merge on `:CsNode` plus a kind-specific label; relationships are `CS_HAS_MODULE`, `CS_DECLARES`, `CS_CALLS`, `CS_CFG_NEXT`, `CS_DDG`, and so on. | Follows the maintainer's 2026-07-02 decision on codeanalyzer-python (`PyCFGNode`, `PY_CFG_NEXT`). The CPG vocabulary stays cross-language in *shape* — same suffixes, same properties, same semantics — while each language's subgraph stays separable. codeanalyzer-typescript v2 emits unprefixed labels and is the outlier here. |

### Decided in the datamodel walk

D1–D16 were confirmed node by node with the maintainer before implementation began. That walk
also settled four points the sections above left unstated. Each is additive; none renames a
shared field or repurposes a shared `kind`.

| # | Concept | Java | Python / TS | **C# decision** | Rationale |
| --- | --- | --- | --- | --- | --- |
| D17 | **Base class and interfaces** | `superclass` + `interfaces`, names | Python `base_classes` (display strings); TS `extends_ids` / `implements_ids` | **The keystone's names, `base_types[]` + `interfaces[]`, holding `can://` ids.** A supertype resolving to metadata rather than to a node — `System.Object`, a package interface — is **dropped**, not named and not homed. | The two mature v2 analyzers disagree here and neither follows the keystone, so there is no precedent to inherit — only a fork to pick. Ids are what let the Neo4j `CS_EXTENDS` / `CS_IMPLEMENTS` overlay resolve endpoints directly instead of re-matching names at projection time, which is the fuzzy lookup the one-`SignatureOf` rule exists to prevent. Homing external supertypes the way D-call-targets homes external *call* targets was declined: it would put nearly every class's `System.Object` into `external_symbols` at L1, where that map otherwise does not exist until L2. **Known loss:** `class UserService : IUserService` shows an empty `interfaces[]` when the interface comes from a package. |
| D18 | **Top-level statements** | no precedent (no such form) | Python module-level code lands in `functions{}` | **Report Roslyn's synthesized type**: `types{ "Program": { callables{ "<Main>$…" } } }`. `module.functions{}` is empty for C# without exception. | The callable's id must carry the `Program` segment to match the symbol the call graph resolves against. Hoisting `<Main>$` into `functions{}` reads closer to the source — the author wrote no class — but then the id and the tree position disagree about where the method lives, and every consumer has to know which one to trust. One rule with no exception beats a shape that flatters the source. |
| D19 | **Attributes** | annotations, flat | Python `decorators: List[str]` (flat); keystone demands structured | **Structured `decorators: [{ name, args[], span }]`** on types, callables, fields and parameters; argument values as source-text strings. | C# attributes are where routing, ORM and DI facts live, so a later entrypoint pass reads `args[0]` rather than regex-ing C# syntax out of a string. The keystone names flat strings as the shape not to use, and Python is the outlier, not the precedent. **Known loss:** C# attributes take positional *and* named arguments (`[Route("x", Name = "y")]`). A `named_args` map was declined to avoid a field no sibling has, so named arguments are flattened into `args` as source text and a consumer needing the distinction parses it back out. |
| D20 | **Call-site keys** | n/a (v1 rich call sites) | TS `line:col` with a `/k` tiebreak (V8) | **`line:col`, disambiguated `line:col/k` on collision.** | The keystone's ordinal-id grammar, and what keeps positional ids addressable so the SDK can offer `flows_to_statement("Svc.cs:42")`. Chained calls — `x.Trim().ToLower().Split(',')`, pervasive in C# — differ by column, so collisions are rare. Keying on the invocation's opening parenthesis would make collisions structurally impossible but would point the key at `(` while `span` covers the whole expression. A per-callable sequential ordinal never collides and gives up line-level addressability entirely. |

---

## 4. Architecture

### 4.1 Pipeline

```
input: <path>  (.sln | .csproj | directory)
   │
   ▼
[1] ProjectLoader        discover .sln/.csproj; read obj/project.assets.json
   │                     → per-project: source file list + MetadataReference[]
   ▼
[2] CompilationBuilder   CSharpCompilation.Create per project,
   │                     inter-project references wired in dependency order
   │                     → Compilation[] + SemanticModel per SyntaxTree
   ▼
[3] SymbolTablePass      L1 · walk trees + symbols → module/type/callable/field nodes,
   │                          call sites as `call` body nodes (callee: null)
   ▼
[4] PartialMerger        L1 · fold multi-file partial declarations into one type node
   │
   ▼
[5] CallGraphPass        L2 · resolve each call site → can:// id; backfill `callee`;
   │                          emit call_graph edges; optional CHA expansion
   ▼
[6] ControlFlowPass      L3 · Roslyn ControlFlowGraph → statement-level cfg edges
   ▼
[7] DominancePass        L3 · post-dominance frontier → cdg edges
   ▼
[8] DataFlowPass         L3 · reaching definitions over k-limited access paths → ddg edges
   ▼
[9] InterproceduralPass  L4 · formal/actual vertices, summaries, param_in / param_out
   │
   ├──► JsonEmitter      analysis.json (or compact JSON on stdout)
   └──► Neo4jEmitter     Cypher script or direct Bolt upsert
```

Passes 6–9 are staged (see § 7). Each pass only ever *adds* to the tree built by the one
before it — that is the monotonicity invariant made structural, and it is what the
`L1 ⊆ L2 ⊆ L3 ⊆ L4` gate checks.

### 4.2 Repository layout

```
codeanalyzer-dotnet/
├─ src/Cansharp/
│  ├─ Cansharp.csproj
│  ├─ Program.cs                     CLI entry (System.CommandLine)
│  ├─ Loading/
│  │  ├─ ProjectLoader.cs            .sln / .csproj / directory discovery
│  │  ├─ AssetsReader.cs             obj/project.assets.json → dll paths
│  │  └─ CompilationBuilder.cs       CSharpCompilation.Create, ref graph
│  ├─ Ids/
│  │  └─ CanId.cs                    THE one signatureOf() + can:// builder
│  ├─ Model/                         the v2 node & edge types
│  │  ├─ Analysis.cs  Application.cs  Module.cs  TypeNode.cs
│  │  ├─ Callable.cs  FieldNode.cs    BodyNode.cs
│  │  └─ Span.cs      Edges.cs
│  ├─ Passes/
│  │  ├─ SymbolTablePass.cs          L1
│  │  ├─ PartialMerger.cs            L1
│  │  ├─ CallGraphPass.cs            L2
│  │  ├─ ControlFlowPass.cs          L3
│  │  ├─ DominancePass.cs            L3
│  │  ├─ DataFlowPass.cs             L3
│  │  └─ InterproceduralPass.cs      L4
│  └─ Emit/
│     ├─ JsonEmitter.cs
│     └─ Neo4j/{Project.cs, Rows.cs, Cypher.cs, Bolt.cs, Schema.cs}
├─ test/Cansharp.Tests/              xunit
├─ fixtures/                         sample C# apps (see § 6)
├─ packaging/
│  ├─ cansharp/                      the Python shim package
│  │  ├─ __init__.py                 binary_path(), main()
│  │  └─ pyproject.toml
│  └─ build_wheels.py                per-RID publish → platform-tagged wheel
├─ .claude/SCHEMA_DECISIONS.md
└─ .github/workflows/release.yml
```

### 4.3 Project loading (D2) — the front end

The analyzer never invokes MSBuild. For each discovered `.csproj`:

1. Parse the `.csproj` XML for `<Compile>` items, the default glob (`**/*.cs` minus
   `bin`/`obj`), `<ProjectReference>`, `<TargetFramework(s)>`, `<Nullable>`, `<LangVersion>`,
   and `<DefineConstants>`.
2. Read `obj/project.assets.json`, which `dotnet restore` writes, for the **package**
   references. Its `targets.<tfm>` section lists each resolved library with the relative
   `lib/<tfm>/*.dll` paths under its `compile` key; `libraries.<name/version>.path` gives the
   package's folder and `packageFolders` gives the NuGet roots. Together these produce the
   absolute path of every package metadata reference. Entries whose only compile item is the
   `_._` placeholder contribute nothing and are skipped.
3. Locate the **framework** references separately. `project.assets.json` does *not* carry
   them: the framework appears only as a name, under
   `project.frameworks.<tfm>.frameworkReferences` (`Microsoft.NETCore.App`, and
   `Microsoft.AspNetCore.App` for web projects). Their assemblies live in the SDK's targeting
   pack at `<dotnet-root>/packs/<name>.Ref/<version>/ref/<tfm>/*.dll`. The dotnet root is
   discovered from `project.frameworks.<tfm>.runtimeIdentifierGraphPath` — the assets file
   records it as `<dotnet-root>/sdk/<sdk-version>/PortableRuntimeIdentifierGraph.json` — and
   falls back to `DOTNET_ROOT`, then to resolving `dotnet` on `PATH`. Where several pack
   versions are installed, the highest version not exceeding the target framework wins.
4. Build a `CSharpCompilation` with those `MetadataReference`s and a `CSharpParseOptions`
   carrying the language version and preprocessor symbols.
5. `<ProjectReference>`s become `CompilationReference`s, wired in topological order so a
   downstream project sees upstream symbols. Cycles are reported and broken by falling back
   to the upstream project's built output dll if one exists.

**Multi-targeting.** A project with `<TargetFrameworks>net8.0;net48</TargetFrameworks>` is
analyzed once, against the first target framework, and the chosen TFM is recorded on the
module's `project` metadata. Analyzing every TFM would duplicate the entire symbol table
under identical ids.

**Failure mode — no restore.** If `project.assets.json` is missing, the analyzer reports
`run 'dotnet restore' first` and — unless `--strict` — degrades to a syntax-only L1 pass
(no types, no callee resolution, `max_level: 1`).

**Failure mode — no targeting pack.** If step 3 cannot locate a pack, every framework type
resolves to an error type and the semantic model is worthless while still appearing to work:
callables get no return types and L2 resolves almost nothing. This is reported as a hard
error naming the roots that were searched, never degraded to silently, because a symbol table
built without `System.Object` looks superficially valid.

**Layout risk.** The `packs/` directory layout is an implementation detail of the SDK
install, not a documented contract, and step 3 depends on it. A layout change breaks reference
resolution for every analyzed project at once. Bundling reference assemblies with the analyzer
(the `Basic.Reference.Assemblies` approach) removes that coupling at the cost of one pinned
copy per supported TFM inside the binary, and is the fallback if the discovery path proves
fragile in the field.

**Known gap.** Source generators do not run, so generated partials (`[GeneratedCode]`,
records' generated members, ASP.NET minimal-API glue) are absent unless the project has
`EmitCompilerGeneratedFiles` enabled and the files are on disk. Documented, not silently
absorbed.

### 4.4 Identity — the one `CanId`

```
can://csharp/<app>/<file>/<Type>[/<Nested>]/<member-signature>
```

- `<app>` — `--app-name`, else the `.sln` name, else the input directory basename.
- `<file>` — repo-relative POSIX path *with* extension: `src/Api/Controllers/UserController.cs`.
  Never absolute, never containing `..`.
- `<member-signature>` — produced by exactly one `CanId.SignatureOf(ISymbol)` using a fixed
  `SymbolDisplayFormat`: name, parenthesized fully-qualified parameter types, return type
  appended. `Save(System.String,System.Int32)System.Boolean`. Generic methods keep their arity
  and type parameters: `Map<T>(T)T`. Explicit interface implementations keep the qualifier:
  `IFoo.Bar()`. Property accessors append `#get` / `#set`.

Ordinal ids below the callable are `<callable-id>@<line>:<col>` for statements and
`<callable-id>@<tag>` for synthetic vertices, per the canonical grammar.

One `SignatureOf()` produces every id, caller-side and callee-side. This is what makes the
`call_graph` edge join a byte-match rather than a fuzzy lookup, and it is the single most
important invariant in the codebase.

### 4.5 Partial-type merge (D6)

Roslyn does most of the work: a `partial class` is *one* `INamedTypeSymbol` carrying several
`DeclaringSyntaxReferences`. The merger therefore:

1. Picks the **primary** declaration — the syntax reference declaring the most members; ties
   broken by ordinal path sort. The type node is homed in that file's module, and its id
   carries that file segment.
2. Emits `parts: [{file, span}]` for every declaring reference, primary included.
3. Merges `callables{}` and `fields{}` across all parts. Each member declared outside the
   primary file carries `declared_in: "<its own file key>"`.

`span.bytes` always slices the `module.source` of the file named by `declared_in` (or the
node's own module when absent). Consumers that ignore `declared_in` will slice the wrong
file, so the SDK's body accessor must honour it — called out explicitly in the handoff.

Partial *methods* fold the same way: the implementing declaration wins, the defining
declaration contributes only its signature.

### 4.6 L1 — symbol table

One `module` per `.cs` file, keyed by repo-relative path, carrying `source` (the whole file
text, once), `namespace`, `project`, `imports` (from `using` directives, including
`global using` and aliases), `types{}`, `functions{}` (empty for C# — every callable has a
containing type; top-level-statement programs synthesize the compiler's `Program`/`<Main>$`),
and `content_hash`.

Every node's text is a slice of its module's `source`; no node duplicates code. Spans carry
`start`/`end` as `[line, col]` for addressing plus `bytes` as UTF-8 offsets for slicing (D15);
because Roslyn reports UTF-16 positions, each file gets one UTF-16→UTF-8 offset table built
alongside its `source` blob, and every span converts through it.

Nested and local types nest as `types{}` on their containing type or callable. Lambdas and
local functions nest as `callables{}` on their containing callable, matching TS-v2 V10.

### 4.7 L2 — call graph

Every `InvocationExpressionSyntax`, `ObjectCreationExpressionSyntax` (including implicit
`new()`), `ElementAccessExpressionSyntax` (indexers), delegate invocation, and property
access that resolves to an accessor becomes a `call` body node keyed `line:col`. Chained
calls on one line disambiguate with a `line:col/k` suffix, per TS-v2 V8.

`SemanticModel.GetSymbolInfo` resolves each to an `IMethodSymbol`. Targets inside the symbol
table become `can://` ids; everything else is homed at
`can://csharp/<app>/@external/<assembly>/<Namespace>.<Type>/<signature>` and registered in
`application.external_symbols`, matching the Python and TypeScript convention. There are no
dangling endpoints.

With `--devirtualize`, `SymbolFinder.FindImplementationsAsync` and `FindOverridesAsync` add
one edge per concrete target, tagged `prov:["cha"]`.

### 4.8 L3 — control flow, control dependence, data dependence

`ControlFlowGraph.Create(IMethodBodyOperation)` per callable. Flattening:

- Each `BasicBlock`'s `Operations` become statement nodes keyed by
  `operation.Syntax.GetLocation()`, chained `fallthrough` within the block.
- The block's `ConditionalSuccessor` / `FallThroughSuccessor` become edges out of its last
  statement, with `kind` derived from `ConditionKind` (`true` / `false`) and loop back-edges
  detected by target ordinal.
- `ControlFlowRegion` gives try / catch / filter / finally structure; any operation that can
  throw gets an `exception` edge to the nearest enclosing handler entry, else to the finally
  entry, else to `@exit`. Region splicing, not finally-duplication — same posture as TS-v2 L3.
- `cfg.Blocks[0]` → `@entry`, the exit block → `@exit`.

`switch` expressions and pattern matching arrive already lowered by Roslyn into ordinary
conditional blocks, which is the main reason for D13.

CDG comes from the post-dominance frontier over the flattened statement CFG. DDG comes from
forward may-reaching-definitions over k-limited access paths (`--graph-field-depth`,
default 3), every edge tagged `prov:["reaching-defs"]` per D14.

### 4.9 L4 — interprocedural

Synthetic `formal_in` / `formal_out` / `actual_in` / `actual_out` vertices, `summary` edges
within a callable, `param_in` / `param_out` at the application scope. By-ref parameters get
their own `formal_out` per D9.

**Open substrate problem.** There is no off-the-shelf points-to oracle for .NET the way
WALA serves Java. The `["points-to"]` DDG refinement therefore has no backing yet, and L4
would ship first with a flow-insensitive union-find alias stub (the TS-v2 L9 posture) rather
than a real oracle. This is the main reason L4 is last in the release plan, and the decision
of what oracle to build is deferred to its own design session.

### 4.10 Neo4j projection

Co-primary, always full-depth regardless of `-a`. Per D16 the projection is per-language
prefixed, following codeanalyzer-python: every node merges on its `can://` id under a
`:CsNode` label plus a kind-specific label; containment becomes `CS_HAS_MODULE` /
`CS_DECLARES` / `CS_HAS_METHOD` / `CS_HAS_FIELD` / `CS_HAS_BODY_NODE`; overlays become
`CS_CALLS` / `CS_CFG_NEXT` / `CS_CDG` / `CS_DDG` / `CS_SUMMARY` / `CS_PARAM_IN` /
`CS_PARAM_OUT`; inheritance becomes `CS_EXTENDS` / `CS_IMPLEMENTS` relationships (JSON keeps
it as node properties). The suffixes, properties, and semantics are identical to every
sibling — only the prefix is C#-specific.

The `_k` relationship-identity discriminant is adopted from day one — `CS_DDG` keyed on
`"{var}|{prov}"`, `CS_CFG_NEXT` keyed on `kind` — so parallel dependence edges are not
silently collapsed by `MERGE`. Both Python and TypeScript had to retrofit this; there is no
reason to repeat the bug.

`schema.neo4j.json` is generated from the emitter and conformance-tested.

### 4.11 CLI surface

```
cansharp -i <path> [-o <dir>] [-a 1|2|3|4] [--app-name <name>]
         [--emit json|neo4j] [--devirtualize] [--graph-field-depth <k>]
         [--project <csproj>]... [--strict]
```

`-a` defaults to 1. With no `-o`, compact JSON goes to stdout. `--emit neo4j` forces full
depth and errors if combined with `-a`.

### 4.12 Packaging and release pipeline

```
GitHub Actions (matrix: osx-arm64, osx-x64, linux-x64, linux-arm64, win-x64)
   │
   ├─ dotnet publish -c Release -r <rid> --self-contained
   │      -p:PublishSingleFile=true -p:PublishReadyToRun=true
   │      -p:EnableCompressionInSingleFile=true
   │
   ├─ packaging/build_wheels.py  →  cansharp-<ver>-py3-none-<platform-tag>.whl
   │      binary copied into the package as data; wheel forced to the RID's platform tag
   │
   └─ twine upload  →  PyPI  ·  binaries also attached to the GitHub release
```

The Python side is a thin shim: `cansharp/__init__.py` exposes `binary_path()` (used by
`python-sdk` to shell out) and `main()` (the `cansharp` console script), and nothing else.
No Python-side analysis logic.

**Size.** ~40–60MB per wheel after R2R and single-file compression, one per platform, so a
user downloads only their own. PyPI's default 100MB per-file limit accommodates this; if
compressed size creeps past it, a limit increase must be requested before the first release —
tracked as a release-plan risk, not discovered on release day.

### 4.13 python-sdk integration

- `cldk/models/csharp/` — Pydantic models mirroring the emitted v2 shape. Because
  `codeanalyzer-python`'s `py_schema.py` is *already* v2, this is largely a rename-and-extend
  of that module rather than new modelling: the `Span`, `BodyNode`, `CfgEdge`, `CdgEdge`,
  `DdgEdge`, `SummaryEdge`, `ParamEdge`, and `Analysis` envelope types carry over unchanged.
  C#-specific additions are the D6–D11 fields: `parts`, `declared_in`, `ref_kind`,
  `positional_parameters`, `invoke_signature`, `is_auto`, and the new `kind` values.
- `cldk/analysis/csharp/` — the facade, shelling out via `cansharp.binary_path()`.
- `cansharp==<version>` added to `pyproject.toml` dependencies and `[tool.backend-versions]`.
- The `prov` literal must accept `reaching-defs` alongside `ssa` / `points-to` — the same
  lockstep requirement codeanalyzer-typescript's issue #32 imposed.
- The body accessor must honour `declared_in` when slicing partial-type members (§ 4.5).

---

## 5. Test gates

| Gate | Checks |
| --- | --- |
| `SchemaTests` (L1) | canonical envelope; every `can://` id well-formed and unique; `symbol_table` keys relative and POSIX; byte-slice check — `source[span.bytes]` equals the declaration text for every node, **including a non-ASCII fixture** that would pass under UTF-16 offsets and fail under UTF-8 (the D15 guard); partial merge produces one type with correct `parts` and `declared_in`; property/event/indexer shape; `body` call nodes keyed `line:col` with `callee: null` |
| `SchemaTests` (L2) | `call_graph` shape `{src,dst,prov,weight}`; no dangling endpoints; `callee` backfilled on every resolvable call node; externals homed and registered; `--devirtualize` only ever adds edges |
| `SchemaTests` (L3/L4) | CFG has a unique entry and exit, every node reachable from entry and reaching exit; no dangling endpoints in any edge list; `prov` tokens exactly as specified |
| `MonotonicityTests` | `L1 ⊆ L2 ⊆ L3 ⊆ L4` — the superset gate, modulo the sanctioned `callee: null → id` refinement |
| `Neo4jSchemaTests` | conformance against generated `schema.neo4j.json`; `_k` discriminant present on `CS_DDG` and `CS_CFG_NEXT`; keyed `MERGE` materializes exactly row-count relationships |

**Fixtures** (each a real restorable project, committed with its `project.assets.json` path
reproducible via `dotnet restore` in CI):

| Fixture | Exercises |
| --- | --- |
| `Console.Basic` | the happy path; top-level statements |
| `Api.Di` | interface dispatch through DI — the `--devirtualize` case |
| `Forms.Partial` | multi-file partial classes with a `.Designer.cs` |
| `Modern.Records` | records, record structs, generics with constraints, delegates |
| `Async.Iterators` | `async`/`await`, `yield return`, `using`, pattern switches — the L3 lowering cases |
| `Solution.MultiProject` | cross-project call edges, project references |

---

## 6. Release plan

| Release | Contents | Gates | Blocks |
| --- | --- | --- | --- |
| `cansharp` 0.1.0 | L1 + L2, JSON + Neo4j, all five platform wheels | L1/L2 schema gates, Neo4j conformance | — |
| `python-sdk` 1.6.0 | `cldk/models/csharp/` + `cldk/analysis/csharp/`, pins `cansharp==0.1.0` | SDK round-trip on all six fixtures | needs 0.1.0 on PyPI |
| `cansharp` 0.2.0 | L3 (CFG / CDG / DDG) | L3 gates + monotonicity | — |
| `python-sdk` 1.7.0 | L3 surface, `reaching-defs` accepted in `prov` | | needs 0.2.0 |
| `cansharp` 0.3.0 | L4, after the points-to substrate decision (§ 4.9) | L4 gates + monotonicity | needs its own design session |
| `docs` | language matrix + C# quickstart | | tracks 0.1.0 |

**Version lockstep:** each `python-sdk` release pins an exact `cansharp` version in both
`dependencies` and `[tool.backend-versions]`, as it already does for the Python, Java, and
TypeScript analyzers.

---

## 7. Known unsoundness and deferrals

Documented rather than silently absorbed:

- **Source generators do not run** (§ 4.3) — generated partials are invisible unless emitted
  to disk.
- **Only the first target framework** of a multi-targeted project is analyzed (§ 4.3).
- **Reflection, `dynamic`, and expression trees** are unmodelled — call edges through them do
  not exist.
- **No points-to oracle** (§ 4.9) — L4 alias precision is a flow-insensitive stub, and no DDG
  edge may claim `prov:["points-to"]` until a real oracle exists.
- **`unsafe` pointer arithmetic** is not tracked in the DDG.
- **`ref struct` / `Span<T>` lifetime semantics** are not modelled beyond ordinary by-ref
  parameter handling.
- **Interface default implementations** resolve to the declaring interface member; CHA
  expansion covers the implementing types.
- **VB.NET and F#** are out of scope (D1).

---

## 8. Sibling divergences this spec records

Points where the mature reference analyzers already disagree — with each other, with the
keystone, or both. None is a C#-specific question and none is fixed by this work; they are
recorded so a later cross-analyzer reconciliation has a written starting point rather than
three analyzers each having quietly picked a side.

The first two were decided in favour of `codeanalyzer-python` (D15, D16), which makes
`codeanalyzer-typescript` the outlier in each case. The last three are places where **neither**
sibling follows the keystone, so C# is the first analyzer to implement what the keystone
actually says.

1. **Span offset units** — Python documents UTF-8 bytes, TypeScript (its V5) emits UTF-16
   chars, the canonical field is named `bytes`. C# follows Python (D15).
2. **Neo4j label prefixing** — Python prefixes per language, TypeScript v2 emits unprefixed
   under a shared `:CanNode` label. C# follows Python (D16).
3. **Heritage field shape** — the keystone specifies `base_types[]` + `interfaces[]` holding
   durable ids. Python emits a single `base_classes` list of display strings; TypeScript emits
   `extends_ids` / `implements_ids`. Three names for one concept, and neither sibling uses the
   keystone's. C# implements the keystone (D17).
4. **Decorator structure** — the keystone specifies structured `{ name, args[], span }` and
   explicitly names flat strings as the shape not to use. Python emits
   `decorators: List[str]`. C# implements the keystone (D19).
5. **Accessor modelling** — TypeScript makes accessors first-class callable kinds
   (`"getter"` / `"setter"`); C# keys them `#get` / `#set` with `kind:"method"`, keeping the
   shared callable vocabulary to the four kinds the keystone lists (D7). Both are defensible;
   they are not compatible, and an SDK filter for "all accessors" cannot be written once
   across both languages until one moves.

Reconciling TypeScript or Python to any of these answers is out of scope here and would be its
own design session under the schema-evolution path. Items 3–5 are the more urgent of the five:
a divergence *from the keystone* means the SDK cannot model the field once, which is the
premise the whole single-model design rests on.
