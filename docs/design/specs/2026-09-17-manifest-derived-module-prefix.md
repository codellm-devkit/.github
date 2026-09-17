# Spec: manifest-derived module prefix on `can://` ids

Status: draft for review
Date: 2026-09-17
Scope: `can://` id construction across all analyzers, cross-repo

---

## 1. Summary

A module's `can://` id gains a segment naming **the module the file belongs to, as that module
declares itself** — Maven/Gradle `artifactId`, `[project] name`, `package.json` `name` — placed
between the language segment and the intra-module path:

```
# before, with -i /path/to/daytrader-web-service
can://daytrader/java/src/main/java/org/apache/.../RunStatsDataBean.java

# after
can://daytrader/java/daytrader-web-service/src/main/java/org/apache/.../RunStatsDataBean.java
```

The motivating requirement: a caller pointing `--input` at one service repo wants that service
named in the id, without staging directories and without pointing `--input` at a parent that
sweeps in sibling repos.

**The `symbol_table` key does not change.** It stays the true `--input`-relative path. Only the id
gains the segment. §4 is why.

### Why the prefix is manifest-derived and not path-derived

The obvious alternative is to relativize file keys against the *parent* of `--input`, so the input
directory's own name lands in the path. Rejected: it makes every id a function of where the
checkout happens to sit on disk, so `mv` or a differently-laid-out CI workspace rewrites every id
in the graph. `artifactId` is what the module calls itself and survives both.

## 2. Contract-Impact Triage

**Does this change schema v2 output?** Yes — the durable `can://` id of every `module`, `type` and
`callable`, and therefore both endpoints of every `call_graph` edge. The `symbol_table` key,
`analysis.json`'s shape, and every field are untouched.

**Which repos are touched:**

| Change | Analyzers | SDKs | Docs |
| --- | --- | --- | --- |
| Manifest prefix on module ids | `codeanalyzer-java`, `codeanalyzer-python`, `codeanalyzer-typescript`; `codeanalyzer-go` **to verify** | `python-sdk`; `typescript-sdk` **to verify** | `docs` (both fronts), keystone reference in `cldk-devtools` |

The parity clause applies: all analyzers move together or the language-neutral `artifact` and
`@external` merge targets stop lining up with the code nodes beside them.

## 3. Decisions locked

| Decision | Choice | Rationale |
| --- | --- | --- |
| Prefix source | module manifest identity | path-independent, survives relocation and rename |
| Prefix scope | **id only**; `symbol_table` key stays the `--input`-relative path | §4 |
| Manifest search | walk up from each file, **bounded at `--input`** | never reads outside what the caller named; cannot be fooled by an unrelated manifest above the checkout |
| No manifest found | fall back to today's bare relative path, no prefix | a loose source tree stays analysable |
| App-name redundancy | accept and document | §5 |
| Prefix collision | fall back to the path segment for the colliding modules, and warn | §6 |

## 4. Why the `symbol_table` key keeps the real path

`symbol_table` is a **named map**. Its key's job is uniqueness; readability is the id's job.

A path-derived key is unique *by construction* — two directories cannot share a path. A
manifest-derived key is not: nothing stops two modules in one tree declaring the same `artifactId`,
and vendored copies are exactly where it is likely. Five services each vendoring one shared
internal library plausibly each build it under the same module name, at which point five modules
collapse onto one key and four are silently dropped from the map — a worse failure than any wrong
edge, because the nodes are simply absent.

A second reason: the module object carries **no path field** (`id`, `kind`, `span`, `package`,
`source`, `comments`, `imports`, `types`, `content_hash`). The key and the id are the only things
that locate a file on disk. Keeping the key a real path means nothing has to be added to preserve
that, and consumers reading from disk rather than the embedded `source` blob keep working.

The cost, stated plainly: the key is no longer the tail of the id, so the two cannot be derived
from each other by string surgery. Consumers that assume they can must join through the module
object instead.

## 5. App-name redundancy

`--app-name` still defaults to the basename of `--input`, so a caller who omits it and points at a
service gets that name in both slots:

```
can://daytrader-web-service/java/daytrader-web-service/src/main/...
```

Accepted rather than special-cased. Suppressing the prefix when it equals the app name would make
ids take two shapes depending on a name coincidence, which costs every consumer the ability to
parse positionally — a worse outcome than a redundant-looking default. Callers who mind pass
`--app-name`.

## 6. Prefix collision and the uniqueness guarantee

`artifactId` is not unique within a tree, and the id is the join key for `call_graph` `src`/`dst`.
A non-unique prefix therefore merges two distinct callables onto one graph node — the same class of
silent loss §4 rejects for the map key, and a **regression against today**, where path-derived ids
are distinct by construction.

So the prefix is applied only when it is unique across the analysis:

1. After discovery, group modules by their resolved manifest prefix.
2. A prefix claimed by exactly one manifest directory is applied.
3. A prefix claimed by two or more **distinct** manifest directories is applied to none of them;
   those modules keep the bare relative path, and a warning names the prefix and the competing
   directories.

This keeps "ids are distinct by construction" true, which is the property the design was chosen
for in the first place. The alternative — refuse the analysis on collision — is louder but makes a
vendored-duplicate monorepo unanalysable for a cosmetic reason; the warning plus fallback degrades
one prefix rather than the whole run. Revisit if the warning proves ignorable.

Note this is orthogonal to codeanalyzer-java#269: that is about *resolution* binding callers to the
wrong copy, and no id spelling fixes it. Both can be true at once and they need separate fixes.

## 7. Migration

Ids are not stable across this change, and there is no mechanical rewrite: the new segment is
derived from a manifest, not from anything present in an old id.

- **Neo4j stores** — reproject. A prefix-scoped purge (`id STARTS WITH $appPrefix + '/'`, per the
  2026-09-02 prune spec) still selects the whole application correctly, because the app segment is
  unchanged; only segments below it move.
- **L1 caches** — no action. The cache envelope keys on app name and analyzer version, so a version
  bump invalidates it.
- **Saved `analysis.json`** — regenerate. Old and new files must not be joined.
- **SDK joins** — `python-sdk`'s Java index keys on `qualified_name` for types and on `can://` id
  for callables (`_add_type` / `_callables`). The id-keyed half changes; the name-keyed half does
  not.

Schema version: the id grammar is contract, so this is a minor bump on the analysis schema, with
the analyzers refusing to read a cache or graph written by the older grammar.

## 8. Release plan

Lockstep matters here: an SDK pinned to a new analyzer while its own id handling is unchanged will
join old ids against new ones.

1. Keystone reference (`cldk-devtools`) records the new grammar. No release.
2. `codeanalyzer-java` implements and releases first — it is the reference implementation and the
   one with a pinned SDK consumer.
3. `python-sdk` bumps its `codeanalyzer-java==` pin (`pyproject.toml:46`) and releases. This is the
   gate between "released" and "consumable"; see python-sdk#418.
4. `codeanalyzer-python` and `codeanalyzer-typescript` follow, each with its own SDK pin bump.
5. `docs` updates both fronts once the analyzers agree; see docs#11.

Until step 4 completes, analyzers in one polyglot graph disagree about whether a module prefix is
present. That window is the cost of not doing all three at once, and it should be kept short rather
than designed around.

## 9. Open questions

- **`codeanalyzer-go`** — does it mint module keys the same way? Unverified; it has source but was
  not read for this spec.
- **`typescript-sdk`** — pin chain to `codeanalyzer-typescript` unverified, no local checkout.
- **Multi-module reactors** — a child module's prefix is its own `artifactId`, not the aggregator's,
  which follows from the nearest-enclosing-manifest rule. Worth confirming against a real reactor
  before release.
- **Keystone drift** — the keystone documents `can://<lang>/<app>/<file>` while every live analyzer
  emits `can://<app>/<lang>/<file>`. Pre-existing, unrelated to this change, and tracked separately
  rather than corrected here.

## 10. Relationship to Epic #39

This spec **supersedes the August 2026 won't-fix on `codellm-devkit/.github#39`** ("can:// identity
grammar gains a `<service>` segment"), which was closed NOT_PLANNED on 2026-08-20 with the
statement that "the `can://` identity grammar is settled". That settlement is deliberately
re-opened here. Recording it so a future reader does not mistake this for an oversight.

What differs from #39, and why the narrower change is worth making where the larger one was not:

| | #39 (rejected) | this spec |
| --- | --- | --- |
| Segment | `<service>` **replaces** `<app>`, outermost | module segment **added** below `<lang>` |
| CLI | `--app-name` renamed to `--service` | `--app-name` unchanged |
| Shared code | one id per service, duplicated | one id per module, deduplicated by §6 |
| Schema | 2.0.0 → 2.1.0 breaking erratum | minor bump, id grammar only |
| Monorepos | "handled at invocation, not in the grammar" | handled in the id, so one run can cover a tree |

Two consequences to carry forward rather than rediscover:

- **#36 (extract and freeze the canonical schema v2 contract) regains a prerequisite.** Its August
  unblocking was a direct consequence of the grammar being declared settled. Sequence this spec
  ahead of the freeze, or the freeze captures a grammar that is about to move.
- **#39's D5 rationale still stands on its own terms.** "The id answers *where does this run?*, not
  *what code is this?*" is an argument about semantics, not about implementation cost, and this
  spec does not refute it. It chooses differently: the module segment answers *what module is
  this?*, which is a third question, and the §6 uniqueness rule is what keeps it answerable.

### The invocation caveat this does not remove

`codeanalyzer-java#269` establishes that a single analysis spanning several source roots that
declare the same fully-qualified type binds every caller to the **last** root, silently. This spec
makes a bundled tree *addressable*; it does not make it *correctly resolved*. Anyone relying on a
one-run-per-tree invocation because ids now distinguish the modules must still either deduplicate
the copies in source or land #269's caller-local resolution. The two changes are independent and
neither substitutes for the other.
