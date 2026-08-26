# Spec: the repository-artifact layer — artifacts, dependencies, config, config-uses

Status: draft for review
Date: 2026-08-26
Scope: canonical schema v2 evolution, cross-repo (schema + three analyzers), additive
Roadmap: `docs/design/roadmap.md` candidates 15–18 (group D). Candidate 18 (config uses)
is **included with a per-projection split** — see §7.

---

## 1. Summary

Each analyzer gains a **repository-artifact layer**: the producer-side evidence a
cross-service / whole-application analysis needs but no analyzer emits today —

- a **non-source file inventory** (`artifact` nodes) with raw text captured into the
  analysis output, so a downstream consumer can cite a file's contents without re-reading
  the working tree;
- **normalized dependency declarations** (`dependency` nodes) parsed from each ecosystem's
  build manifests, with a shared `scope` vocabulary;
- **configuration definitions** (`config_key` nodes) — normalized key/value facts from
  structured config files; and
- **configuration uses** (`config_use`, a `*_USES_CONFIG` edge) — env/config reads in code,
  linked to a definition when statically resolvable.

It is **additive** to canonical schema v2: three new node kinds contained in the tree, one
new typed edge, their Neo4j projections, and one shared enum. No existing node, edge, level,
or id shape changes. Existing source/program-graph behavior is untouched.

This spec covers the **shared vocabulary** for all three analyzers, coined once here so the
three implementations cannot drift (parity clause). It does **not** implement any analyzer —
that is the `codeanalyzer-backend` rung, per repo, after this spec and its tracking record
exist.

### Why now

The roadmap parks the microservice initiative behind schema consistency (Group A). This is
its first concrete producer-side slice. The three artifact/dependency/config-definition models
anchor on the **application node** — stable — so they carry no body-node risk.

Config *uses* (candidate 18) anchor on the body node, whose state differs by projection.
`PyBodyNode` and `TSBodyNode` are unified and stable in both `analysis.json` and Neo4j, and
both already host multiple additive edges. Java's `JBodyNode` is used by its JSON emitter, but
its Neo4j projector runs off the legacy v1 IR and has no body-node label. So config-uses is
in scope here with a per-projection split (§7): all three languages in `analysis.json`, Neo4j
for Python and TypeScript, with Java's Neo4j edge waiting on Java's v1→v2 projection migration.

---

## 2. Contract-Impact Triage

**Does this change the schema v2 output?** Yes — three new node kinds (`artifact`,
`dependency`, `config_key`), one new typed edge (`config_use` / `*_USES_CONFIG`), their Neo4j
projections, and one shared `scope` enum. Additive only; nothing renamed or repurposed.

**Repos affected** (schema v2 evolution → every affected repo):

| Repo | What changes |
| --- | --- |
| `codeanalyzer-schema` | v2 model definitions (JSON Schema) for the three node kinds + the `scope` mapping table; v2 Neo4j contract projection for the three node families under each prefix |
| `codeanalyzer-python` | `:PyArtifact` / `:PyDependency` / `:PyConfigKey` emitter; non-source discovery walk; `pyproject.toml` / `requirements*` / lockfile parsing |
| `codeanalyzer-typescript` | `:TSArtifact` / `:TSDependency` / `:TSConfigKey`; `package.json` (incl. dev/peer/optional) + lockfiles; `tsconfig.json` JSONC tolerance |
| `codeanalyzer-java` | `:JArtifact` / `:JDependency` / `:JConfigKey`; `pom.xml` / Gradle **coordinate** parsing (net-new — today deps are downloaded as jars, not modeled) |
| `python-sdk` | v2 Pydantic model mirror (co-evolution rule); facade surface exposure is later, out of scope here |
| `.github` | this spec + the coordinating epic |

---

## 3. The load-bearing decision: artifacts are first-class nodes

The source design (`veias-ukg-imputer/extending-codenalyzer.md`) proposed `artifacts` /
`dependencies` / `config_definitions` as new **top-level sibling maps** under `application`,
each with a string cross-reference back to its artifact.

**Decided (locked with the user): they are first-class nodes in the containment tree**, not
parallel top-level vocabularies. The canonical keystone states there is one structure — a
scale-free node + single-parent containment tree + typed edge overlays — and that no node may
exist that isn't reachable from `application` by containment. First-class nodes satisfy that;
sibling maps would introduce a second node vocabulary outside the tree.

This **preserves every fact the plan needs** and is a strict superset of the sibling-map shape:

- every field from the source models survives verbatim;
- the two string back-references (`dependency → its manifest`, `config_def → its file`)
  become **containment edges** instead of foreign-key strings — one hop, no join, no dangling
  ref;
- artifacts get durable `can://` ids, so raw-text citations are replayable from the analysis
  output exactly like callable-id citations — strengthening the plan's "no re-reading the
  working tree" guarantee;
- the Neo4j projection the source doc already specified
  (`(:PyArtifact)-[:PY_DECLARES_DEPENDENCY]->(:PyDependency)` etc.) **is** containment rendered
  as `HAS_*`/`DECLARES`, which is exactly how the keystone projects the tree to Neo4j. Option 1
  just makes the JSON match the graph the doc already drew.

Containment placement:

```
application
  ├── symbol_table { … }                      (unchanged)
  └── artifacts { <path> → artifact }          ← new; named map keyed by repo-relative path
        artifact
          ├── dependencies { <name> → dependency }   ← new; children of the manifest artifact
          └── config_keys  { <key>  → config_key }    ← new; children of the config artifact
```

`artifacts` is a **named-map container on `application`**, exactly like `symbol_table` — the
tree grows one contained branch, it does not gain a sibling vocabulary. Each artifact carries
its own `id`, so it is a node, not a bare map entry.

---

## 4. Node designs

Common node fields (`id`, `kind`, `span`, `parent`) apply per the keystone. snake_case keys;
absent = no fact (no `null`); language extras additive.

### 4.1 `artifact` node

Contained under `application`.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | `can://<lang>/<app>/@artifact/<repo-relative-path>` |
| `kind` | `"artifact"` | the node-kind ladder entry |
| `artifact_kind` | closed enum | `build_manifest` \| `dependency_lockfile` \| `configuration` \| `deployment_manifest` \| `container` \| `infrastructure` \| `ci` \| `script` \| `documentation` \| `data` \| `other`. **`other` is the catch-all — coverage over perfect classification; a file is never dropped for lack of a parser.** Classification is a typed field, not `node.kind` (keeps the shared node-kind ladder clean) and not `tags` (keeps it queryable/validated). |
| `path` | string | repo-relative key, matching symbol-table convention |
| `format` | string? | e.g. `toml`, `yaml`, `json`, `properties`, `xml` |
| `source` | string? | producing subsystem, optional |
| `content_hash` | string | always present |
| `size_bytes` | int | always present |
| `text` | string? | verbatim contents, subject to capture policy |
| `text_encoding` | string? | e.g. `utf-8`; absent when bytes don't decode |
| `text_truncated` | bool | `true` when size exceeded the cap and `text` holds a prefix |

**Text capture policy (emit contract, not schema-versioned — tunable without a schema bump):**
capture on by default; decodable bytes → `text` set; binary → `text` absent, still inventoried;
default cap **256 KiB**, beyond which `text` holds the leading bytes and `text_truncated=true`;
`path` + `content_hash` always present so a truncated/binary artifact dereferences to source.
Controlled by `--artifact-text` / `--no-artifact-text` (default **on**). Disabling the flag
does not change the artifact inventory, only the payload.

### 4.2 `dependency` node

Contained under its `artifact` (the manifest that declares it). The containment parent
**replaces** the source model's `artifact:` string reference.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | `can://…/@artifact/<manifest>/<ecosystem-native-name>` |
| `kind` | `"dependency"` | |
| `name` | string | **ecosystem-native identity, verbatim**: Maven `groupId:artifactId` (group never dropped); npm `@scope/pkg` (scope preserved); pypi plain name |
| `version_spec` | string? | as declared (`>=0.27`, `^1.7.0`, `6.1.4`) |
| `resolved_version` | string? | when a lockfile pins it |
| `ecosystem` | string? | `pypi` \| `maven` \| `npm` |
| `scope` | closed enum | see §5 |
| `direct` | bool | `true` for direct declarations |

### 4.3 `config_key` node

Contained under its `artifact` (the config file that defines it).

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | `can://…/@artifact/<file>/<dotted-key>` |
| `kind` | `"config_key"` | |
| `key` | string | canonical **dotted** key (`services.payments.url`); flat `.properties` and nested `.yml` share one key space |
| `namespace` | string? | the shared key-space namespace (`env`, `spring`, `vite`, …). 17 and 18 are one collision group precisely so the key space is designed once — a `config_use` (§7) resolves to a definition by shared (`namespace`, `key`). Absent for a plain in-file key with no namespace. |
| `value` | any? | the defined value when present |
| `references` | string[] | placeholder references preserved where recognizable, e.g. `["env:PAYMENT_HOST", "env:PAYMENT_PORT"]` for `http://${PAYMENT_HOST}:${PAYMENT_PORT}` |
| `span` | span? | `SourceRange` per keystone (there is no `SourceSpan`) |

Config parsing is an **overlay**: failure to understand a document's structure must not
prevent the underlying `artifact` node (with its raw `text`) from being emitted.

---

## 5. Shared `scope` vocabulary (the parity-critical decision)

Coined **once**, in `codeanalyzer-schema`, verbatim in all three analyzers. Enum:

```
runtime | development | test | build | optional | unknown
```

Ecosystem mapping — **part of the contract, encoded in the schema docs, not per-analyzer
discretion**:

| shared `scope` | Maven | npm | Python |
| --- | --- | --- | --- |
| `runtime` | `compile`, `runtime` | `dependencies` | main / `[project.dependencies]` |
| `development` | — | `devDependencies` | dev extras, `requirements-dev.txt` |
| `test` | `test` | — | test extras |
| `build` | `provided`, `system`, `import` | — | `[build-system.requires]` |
| `optional` | `<optional>true</optional>` | `optionalDependencies`, `peerDependencies` | optional extras |
| `unknown` | anything unmapped | anything unmapped | anything unmapped |

`unknown` is the catch-all — an unmapped native scope is never dropped or guessed.

The purpose is a reliable **dependency map** from which a downstream agent selects/generates
library-specific analysis (requests vs httpx, axios vs fetch, RestTemplate vs WebClient) — not
package management.

---

## 6. Neo4j projection

Containment rendered as typed edges, under each analyzer's existing prefix (per the code-labels
contract; prefixes are authoritative and unchanged):

```
(:PyApplication)-[:PY_HAS_ARTIFACT]->(:PyArtifact)
(:PyArtifact)-[:PY_DECLARES_DEPENDENCY]->(:PyDependency)
(:PyArtifact)-[:PY_DEFINES_CONFIG]->(:PyConfigKey)

(:JApplication)-[:J_HAS_ARTIFACT]->(:JArtifact)   … J_DECLARES_DEPENDENCY, J_DEFINES_CONFIG
(:TSApplication)-[:TS_HAS_ARTIFACT]->(:TSArtifact) … TS_DECLARES_DEPENDENCY, TS_DEFINES_CONFIG

(:PyBodyNode)-[:PY_USES_CONFIG]->(:PyConfigKey)     ← config use (§7); Py + TS now
(:TSBodyNode)-[:TS_USES_CONFIG]->(:TSConfigKey)     ← J_USES_CONFIG deferred to candidate 9
```

Every artifact node carries `path`, `artifact_kind`, `format`, `content_hash`, `size_bytes`,
and the `text` / `text_encoding` / `text_truncated` properties. `text` is a plain property with
**no index** — artifacts number in the dozens per repo, so it never sits on a hot traversal path.
`config_key` nodes share a key space (`namespace` + `key`) across languages, so a downstream
merge can resolve the same key emitted by two analyzers without knowing which produced it.

The Neo4j projection is always full-depth (levels gate the JSON path only), so the artifact
layer is present in the graph regardless of `-a` level.

---

## 7. Configuration uses from code (candidate 18)

A **config use** is an env/config **read in code** — `os.getenv("PAYMENT_SERVICE_URL")`,
`process.env.API_BASE`, `@Value("${payment.host}")`. It is modeled as a typed edge from the
**body node** that performs the read to the `config_key` it resolves to, named `config_use` in
`analysis.json` and `*_USES_CONFIG` in Neo4j. The edge is the seam candidate 18 defines against
the config key space §4.3/§5 coin: a use resolves to its definition by shared (`namespace`,
`key`) — no separate cross-reference vocabulary.

`config_use` payload (edge, both projections):

| Field | Type | Notes |
| --- | --- | --- |
| `namespace` | string? | the read's key space (`env`, `spring`, `vite`) — how it joins a `config_key` |
| `key` | string | the key as read (`PAYMENT_SERVICE_URL`, dotted for structured reads) |
| `resolved` | bool | `true` when a `config_key` with matching (`namespace`, `key`) exists in the analysis; the edge then terminates on that node. `false` → the edge records an **unresolved** use (the key is read but defined nowhere in the repo, which is normal and expected — the definition may only exist at deploy time) |
| `span` | span? | `SourceRange` of the read expression |

An unresolved use is still emitted — it is evidence that a key is consumed, and the fact that
its definition is *absent from the repo* is itself the signal a cross-service resolver needs.

**Per-projection split.** The anchor differs by projection, so the edge lands unevenly — by
design, not omission:

| Projection | Python | TypeScript | Java |
| --- | --- | --- | --- |
| `analysis.json` | ✅ `config_use` on `PyBodyNode` | ✅ on `TSBodyNode` | ✅ on `JBodyNode` |
| Neo4j | ✅ `PY_USES_CONFIG` | ✅ `TS_USES_CONFIG` | ⏳ `J_USES_CONFIG` deferred |

`PyBodyNode` and `TSBodyNode` are unified and stable in **both** projections and already host
multiple additive edges, so `config_use` lands in full for those two languages. Java's
`JBodyNode` is used by its JSON emitter — so the edge is in Java's `analysis.json` — but Java's
Neo4j projector still runs off the legacy v1 IR and has no body-node label to anchor on.
`J_USES_CONFIG` therefore waits on Java's v1→v2 Neo4j projection migration (roadmap candidate 9);
until then the same facts are present in Java's `analysis.json`, and the projection-parity gate
(candidate 3) will record the Java-Neo4j edge as a known, quantified gap rather than a silent one.

This split does not re-coin anything: the edge name, its payload, and the (`namespace`, `key`)
join are identical across all three languages and both projections. Only *where the edge is
projected* differs, and that difference closes when candidate 9 lands.

---

## 8. Acceptance criteria

**Shared (in `codeanalyzer-schema`, verified in each analyzer):**

- [ ] `artifact`, `dependency`, `config_key` node kinds defined in the v2 catalog with identical
      field names and enum values for python / java / typescript.
- [ ] The `config_use` edge (name, payload, `namespace`+`key` join) is defined once in the v2
      catalog, identical across all three languages.
- [ ] The `scope` enum and its ecosystem mapping table are encoded in the schema, not per-analyzer.
- [ ] JSON output round-trips through the schema for all three languages.
- [ ] Schema conformance tests cover the three node families and the `config_use` edge.

**Per analyzer:**

- [ ] `application.artifacts` is emitted as a contained named map of `artifact` nodes.
- [ ] Non-source repository files carry stable `can://…/@artifact/…` ids; unrecognized files
      are `artifact_kind:"other"`, never dropped.
- [ ] That language's common manifests are parsed into `dependency` nodes with ecosystem-correct
      identity (`groupId:artifactId` for Maven, scope-preserving for npm) and correct `scope`.
- [ ] Structured config formats expose `config_key` nodes with dotted keys and recognized
      `references[]`.
- [ ] Env/config **reads in code** emit `config_use` in `analysis.json` (all three languages),
      each carrying `namespace` / `key` / `resolved` / `span`; a read whose key is defined in the
      repo resolves to its `config_key`, an undefined key is emitted as an unresolved use.
- [ ] `PY_USES_CONFIG` and `TS_USES_CONFIG` are projected to Neo4j from the body node to the
      `config_key`. `J_USES_CONFIG` is out of scope until Java's v1→v2 Neo4j migration (candidate
      9); the projection-parity gate records it as a known Java-Neo4j gap, not a silent one.
- [ ] Text-decodable artifacts carry verbatim `text` with `text_encoding` / `text_truncated`
      per policy; binary and over-cap artifacts remain inventoried with `path` + `content_hash`.
- [ ] Artifact text is present in the Neo4j projection, not only the JSON.
- [ ] `--no-artifact-text` disables text capture without changing the inventory.
- [ ] `schema.neo4j.json` updated with the three node families and their `*_HAS_ARTIFACT` /
      `*_DECLARES_DEPENDENCY` / `*_DEFINES_CONFIG` edges under the analyzer's prefix.
- [ ] Existing source/program-graph behavior is unchanged.

---

## 9. Dependencies, sequencing, and release plan

Blocked by **Group A** (the v2 catalog these node kinds land into must exist). Downstream
consumers degrade gracefully: a missing language's artifact layer is a reported, quantified gap,
so partial rollout is useful rather than blocking.

**Release plan — schema first, analyzers in parallel, SDK last:**

1. **`codeanalyzer-schema`** lands the v2 model definitions (three node kinds, the `config_use`
   edge, the `scope` mapping table, the Neo4j contract projection). This **gates everything** —
   the analyzers implement against it.
2. **The three analyzers implement in parallel**, each on its own release train — none gates
   another. Per analyzer the units (artifact inventory → dependencies → config definitions →
   config uses) may land as separate PRs.
3. **`python-sdk`** mirrors the v2 models (co-evolution rule) after **any one** analyzer is
   conformant — it does not wait for all three.
4. **`J_USES_CONFIG`** (Java's Neo4j config-use edge) is not on this plan — it rides Java's
   v1→v2 Neo4j projection migration (roadmap candidate 9). Java's `analysis.json` carries the
   `config_use` facts in the meantime.

**Tracking:** an epic in `codellm-devkit/.github` with **one sub-issue per PR**, filed
just-in-time as each unit is picked up (not all up front). The epic links this spec rather than
restating it.
