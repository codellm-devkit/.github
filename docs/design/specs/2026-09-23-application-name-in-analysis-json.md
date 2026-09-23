# Spec: the application root carries its `name` in `analysis.json`

Status: accepted
Date: 2026-09-23
Scope: canonical schema v2 spine, all three analyzers, cross-repo
Origin: codeanalyzer-typescript#213 (shipped in v1.6.3), codeanalyzer-typescript#186

---

## 1. Summary

The `application` root in `analysis.json` gains `name: string`: the resolved application name,
the same string every analyzer already builds the root's `can://` id from and already writes as
the `name` property of its Neo4j Application node.

```
"application": { "id": "can://sample-app", "name": "sample-app", "kind": "application", … }
```

Nothing new is computed. Each analyzer resolves the name today, uses it for the id and the graph
anchor, and drops it from the JSON model. This spec stops dropping it, in all three.

### Why

codeanalyzer-typescript v1.6.3 (#213) started emitting `application.name`, describing it as
parity with Java and Python. The parity holds in Neo4j and not in `analysis.json`: the Java v3.3.3
and Python v1.5.4 payloads have no `name` on the root. The closed TypeScript schema in
codeanalyzer-schema rejects the field, which blocks recording v1.6.x there.

Leaving it TypeScript-only would put a spine concept (the root's name) under
`x-cldk.divergences`, and a consumer could name a TypeScript application from its JSON but not a
Java or Python one. Dropping it would make consumers recover the name by parsing `can://` ids,
the grammar-outside-the-analyzers failure that
[`2026-09-05-body-node-id-in-analysis-json.md`](2026-09-05-body-node-id-in-analysis-json.md)
removed for body nodes. So `name` goes in the spine, for every analyzer.

## 2. Contract-Impact Triage

**Does this change schema v2 output?** Yes. One new field, `name`, on the `application` root.
It is additive: no existing field changes and no level boundary moves.

**Is it a language leaf or the spine?** It is the spine. All three analyzers resolve the value
by the same rule (§3), and the graph contract already carries it on every language's Application
label.

**Which repos are touched:**

| Change | Analyzers | SDKs | Docs / schema |
| --- | --- | --- | --- |
| `name` on `application` | `codeanalyzer-java`, `codeanalyzer-python` (typescript already ships it) | `python-sdk`: `JApplication` / `TSApplication` models gain `name`; `PyApplication` comes re-exported from the analyzer | `codeanalyzer-schema` spine; keystone `canonical-schema.md` in cldk-devtools |

## 3. Where the three analyzers stand today

| | Resolution | JSON root | Neo4j Application |
| --- | --- | --- | --- |
| java v3.3.3 | `--app-name` if non-blank, else the input directory's basename (`CodeAnalyzer.java:460`) | `JApplication`: `id, kind, symbol_table, …`, no `name` | `JApplication.name` (`V2GraphProjector.java:86`) |
| python v1.5.4 | `options.app_name or project_dir.name` (`core.py:666`, `neo4j/emit.py:56`) | `PyApplication`: `id, kind, symbol_table, …`, no `name` (`schema/py_schema.py:613`) | `PyApplication.name` |
| typescript v1.6.3 | `--app-name`, else the input basename, trimmed; empty falls back to `app` (`schema/emit.ts:97`) | `TSApplication.name` (#213) | `TSApplication.name` |

The resolution rule is the same in all three. The id is `can://` followed by that name
(`applicationIdOf`, and the equivalent in the other two).

## 4. The design

### 4.1 The field

`application.name: string`, present at every level (L1 up). It holds the resolved application
name described in §3: the value the root id is built from, before any `can://` encoding. It is
never empty. An analyzer that would resolve an empty name uses `app`, as TypeScript already does.

### 4.2 Versioning

- **`analysis.json` `schema_version` stays `2.0.0`.** The change is additive, and TypeScript
  v1.6.3 already ships it under 2.0.0.
- **Graph contract unchanged.** Every Application label already carries `name`.
- The spine lists `name` as **optional** until Java and Python both release it. It then becomes
  **required** in a follow-up schema change (unit 4).
- **Model layer.** New model fields default to `""` (`name: str = ""`), the same shape as
  `PyApplication.id`, so an `analysis.json` written before this change still parses.

## 5. Mechanics that must not be got wrong

- **Emit the resolved name, not the raw flag.** Java already warns about this at
  `CodeAnalyzer.java:655`: the payload and the id are both built from the resolved `application`,
  so the name must be that value too. A raw `--app-name` of `"  "` must not reach the JSON.
- **One source for id and name.** Stamp `name` from the same variable the id is minted from. The
  test to add: `application.id == "can://" + application.name` on every fixture that has no
  characters `can://` would encode.
- **The schema repo is closed per language.** Adding `name` to the spine `Application` makes it an
  evaluated property for all three language schemas at once. The language schemas must not also
  declare it.

## 6. Decomposition and release plan

Tracking: one epic in `codellm-devkit/.github` that links this spec. Children are filed as
cross-repo sub-issues when each unit is picked up.

| Unit | Repo | Release | Gate |
| --- | --- | --- | --- |
| 1 | codeanalyzer-schema | — | spine `Application` gains optional `name`; samples recorded from java v3.3.3, python v1.5.4, typescript v1.6.3; `check.py` green. This also closes open uptake PRs #7, #8 and #9 |
| 2 | codeanalyzer-java | next minor | `JApplication.name` emitted; id/name parity test; release uptake PR green against the spine |
| 3 | codeanalyzer-python | next minor | `PyApplication.name: str = ""` emitted; the same parity test |
| 4 | codeanalyzer-schema | — | `name` becomes required in the spine; samples recorded from the unit 2 and 3 releases; `check.py` green |
| 5 | python-sdk | next minor | `JApplication.name` / `TSApplication.name` (`str = ""`); pin bump to the unit 3 release |

Order: unit 1 first. It unblocks recording the current releases and is safe because `name` is
optional. Units 2 and 3 ship on their own clocks. No analyzer reads another's output, so they do
not need lockstep. Unit 4 waits for both 2 and 3. Unit 5 needs unit 3 for the pin, and its model
fields can land any time after unit 1.

The keystone doc (`canonical-schema.md` in cldk-devtools) adds a `name` row to the `application`
field table alongside unit 1.
