# Spec: `codeanalyzer-iac` — shared IaC backend, Helm first

Status: approved for implementation
Date: 2026-09-02
Scope: new analyzer backend, shared IaC artifact contract, Helm source analysis and rendering

---

## 1. Summary

Create one repository and one backend, `codeanalyzer-iac`, for infrastructure-as-code and
infrastructure build/configuration dialects. Helm is the first frontend; Terraform/OpenTofu,
Ansible, Dockerfile/Compose, Packer, Kubernetes/Kustomize, CloudFormation/SAM, ARM/Bicep,
Pulumi/CDKs, Chef/Puppet/Salt, Nomad, Vagrant, and later dialects join the same backend rather
than becoming separate `codeanalyzer-*` repositories.

The backend accepts either arbitrary filesystem roots or a prepopulated Neo4j graph. It never
assumes an `infra/` directory. In filesystem mode it inventories files and populates the graph;
in graph mode it reads complete source from existing `can://artifact/<app>/...` nodes and enriches
those same nodes. Unsupported files remain ordinary raw `Artifact` records and are never dropped.

IaC interpretation is progressive typing over the existing source object:

```text
Artifact
└── IaCArtifact
    └── HelmArtifact
        └── HelmChart | HelmValues | HelmTemplate | ...
```

Those names are facets/labels on one file-backed object, not separate nodes. A `Chart.yaml` node,
for example, remains canonically identified by `can://artifact/.../Chart.yaml` while gaining
`:IaCArtifact:HelmArtifact:HelmChart`. Semantic constructs inside the file are contained typed
nodes with `can://iac/...` identities and source spans.

The Helm-first implementation and common orchestration core are typed Go. The official Helm Go SDK
supplies chart loading and rendering, while a compiled dialect registry keeps each frontend isolated
behind one interface and routes typed deltas through the shared orchestration pipeline. Future
dialect frontends may be embedded Go components or versioned native-runtime workers that emit the
same typed delta contract. There is no runtime plugin ABI in the first release.

---

## 2. Contract-impact triage

**Does this change schema v2 output?** No existing language analyzer changes its schema-v2 output.
The new backend owns a separate IaC schema-v2 branch that composes with the existing neutral
`Artifact` contract. That branch adds an optional typed IaC facet, new contained node kinds, new
identity forms under `can://iac/`, and new typed edge families. It does not rename or repurpose the
code-analysis spine. `Artifact.kind` remains `"artifact"`; the concrete interpretation is carried
by the facet and, in Neo4j, by additional labels.

**Change type:** new analyzer plus additive IaC contract registration; no existing analyzer migration.

| Repo | Change | Reason |
| --- | --- | --- |
| `codeanalyzer-iac` | New Go backend, Helm frontend, CLI, JSON emitter, Neo4j projector | Primary deliverable |
| `codeanalyzer-schema` | Add the v2 IaC JSON/Neo4j contracts and conformance fixtures | IaC extends the shared neutral `Artifact` vocabulary |
| `.github` | This cross-repo spec and the coordinating epic | Design provenance and tracking |
| `codellm-devkit` docs | IaC support matrix and CLI guide | User-facing discovery |

Existing language analyzers do not gain Helm logic and do not need to emit the optional IaC
facet. They continue to merge the same language-neutral `Artifact` IDs. The user is separately
making TypeScript project complete artifact source into Neo4j; this design assumes every graph
input artifact has complete text and does not add a filesystem fallback.

The Python and TypeScript SDK facades are deliberately outside the first delivery. Their eventual
IaC facade consumes this contract after the backend has shipped and does not block the CLI.

The analyzer emits analysis schema `2.0.0`, using language token `iac`, and a new IaC Neo4j
catalog line `1.0.0`. IaC additions are language/domain leaves under the v2 parity clause rather
than a rewrite of the code node ladder.

Graph ingestion depends on the repository-wide whole-file artifact-source contract recorded in
[`2026-09-02-artifact-source-whole-file.md`](./2026-09-02-artifact-source-whole-file.md). For this
backend, a non-empty, complete `source` whose digest equals `sha256` is an eligibility condition
for semantic enrichment, not an optional filesystem recovery hint.

---

## 3. Locked decisions

These decisions were made interactively with the maintainer. After approving the detailed Helm
spine, the maintainer authorized the recommended defaults for the remaining details so the design
could be completed without another round of leaf-level confirmations.

### 3.1 Backend and ingestion

| # | Decision | Rationale |
| --- | --- | --- |
| D1 | **One `codeanalyzer-iac` backend, one compiled registry, many dialect frontends. Helm ships first.** | Cross-dialect relationships are the value of an IaC graph; separate repositories would duplicate discovery, identity, reconciliation, and projection. |
| D2 | **Two input modes:** arbitrary filesystem roots and a prepopulated Neo4j graph. | Repositories have no universal infrastructure directory, and existing language analyzers may already have captured every config file as an `Artifact`. |
| D3 | **Graph mode reads complete `Artifact.source` from Neo4j and verifies it against `sha256`.** | Source is the agreed graph contract. A filesystem/content-provider fallback would make graph-only analysis non-reproducible. |
| D4 | **Unknown files remain raw `Artifact` records. Recognition enriches; it never replaces.** | Future frontends can reinterpret already captured assets without an ID migration or a destructive re-ingest. |
| D5 | **A file has one source dialect. Cross-dialect meaning is represented by contained output/reference nodes and edges.** | A Helm template produces Kubernetes-shaped output but is not itself a Kubernetes source artifact; a Terraform `helm_release` remains Terraform source. |

D5 is scoped to the IaC facet union. It does not prevent a sibling language analyzer from
modeling the same source as code. A future TypeScript Pulumi or CDK file, for example, can retain
its TypeScript code model while its shared `Artifact` gains one Pulumi/CDK IaC facet.

### 3.2 Artifact and semantic model

| # | Decision | Rationale |
| --- | --- | --- |
| D6 | **Progressive facet hierarchy on the same file node:** `Artifact → IaCArtifact → <Dialect>Artifact → <ConcreteKind>`. | The raw file owns source and identity; a second IaC file node would duplicate it. `MATCH (a:IaCArtifact)` remains possible across dialects. |
| D7 | **Dialect-native named containment beneath each facet. No generic `entities[]` bag.** | Helm charts, Terraform blocks, Ansible plays/tasks, and Docker stages have different genuine structures. Typed named maps preserve those semantics. |
| D8 | **Concrete artifact subtype follows parser/evaluation behavior; contextual variants use typed roles.** | `HelmTemplate` helpers, notes, tests, and hooks share a parser but have different roles. This avoids a class per filename without falling back to strings everywhere. |
| D9 | **Source text exists once, on `Artifact`; contained nodes carry UTF-8 byte spans plus line/column.** | All source-derived text remains a slice of the canonical blob, matching schema v2. Generated nodes carry provenance instead of invented spans. |
| D10 | **No concrete `IaCEntity` kind.** | `IaCEntity` incorrectly suggested a universal containment tier analogous to `PyClass`. Shared behavior is an implementation interface and optional Neo4j marker, never a vague emitted object. |

### 3.3 Identity and aliases

| # | Decision | Rationale |
| --- | --- | --- |
| D11 | **Whole-file semantics keep `can://artifact/<app>/<path>` as canonical identity.** | Language analyzers and IaC analysis must merge onto the same physical file. |
| D12 | **Contained semantic nodes use source-contained `can://iac/<app>/<dialect>/...` IDs; anonymous nodes end in `@line:column`.** | This provides deterministic uniqueness for named and unnamed constructs while retaining source provenance. |
| D13 | **Native/logical addresses are aliases, not replacement canonical IDs.** | Native names can be ambiguous, computed, or unavailable, while source containment is always present. |
| D14 | **Aliases are first-class `IdentityAlias:IaCAlias` nodes with one `IAC_ALIAS_OF` target.** | An alias can be indexed and constrained without putting multiple merge keys on the target. Aliases never chain or cycle; an alias ID resolves to exactly one canonical node; one canonical node may have several aliases. |
| D15 | **Semantic edges attach to canonical targets. Cleanup may remove IaC aliases but never their underlying `Artifact`.** | Queries have one edge topology, and IaC enrichment cannot delete shared raw data. |

Examples:

```text
can://artifact/payments/charts/api/Chart.yaml
  alias: can://iac/payments/helm/chart/charts/api

can://iac/payments/helm/charts/api/templates/_helpers.tpl/named-template/api.fullname
can://iac/payments/ansible/playbooks/site.yml/task@42:3
```

Path and native-name segments are percent-encoded. IDs are opaque handles; consumers do not split
them to recover fields. Schema validation rejects any alias ID that equals another alias or any
canonical ID, in addition to enforcing exactly one `IAC_ALIAS_OF` target.

### 3.4 Helm source contract

| # | Decision | Rationale |
| --- | --- | --- |
| D16 | **Support chart API `v1` and `v2`; record renderer name/version as provenance.** | Chart format compatibility is declared by `apiVersion`; baking Helm CLI major into the schema would conflate file format and engine behavior. |
| D17 | **`HelmChart` mirrors the complete official metadata contract with typed fields.** | Consumers need more than `name` and `version`, and `Artifact.source` remains available without an untyped duplicate metadata blob. |
| D18 | **`Chart.yaml` is the aggregate anchor. Other chart artifacts use `IAC_PART_OF_CHART`; the nearest ancestor `Chart.yaml` wins.** | This represents a multi-file chart without inventing a directory artifact. Vendored subcharts retain their own anchors. |
| D19 | **Dependency declarations are first-class contained `HelmDependency` nodes.** | Alias, condition, tags, import-values, repository, and version constraint belong to the declaration; rich relationship properties would violate the identity-only edge posture. |
| D20 | **Unresolved dependency targets are `HelmChartReference` nodes with optional standard PURL enrichment.** | The same declaration can later resolve to a vendored chart or OCI package. There is no registered `pkg:helm` PURL type, so the analyzer never invents one. |
| D21 | **Existing `ConfigKey` nodes are progressively typed as `IaCValue:HelmValue`.** | A separate Helm value node would duplicate the same source declaration and break cross-analyzer joins. |
| D22 | **Every `.Values...` use is a `HelmValueReference` node with a span and optional `IAC_REFERENCES_VALUE` target.** | Dynamic and unresolved references remain visible, and repeated use sites stay independently addressable. |
| D23 | **Named-template definitions and call sites are distinct nodes.** | `define`/`block` declarations and `template`/`include`/`tpl` calls have different spans and may remain unresolved. |

The initial concrete Helm artifact kinds are:

| Kind | Files / behavior | Typed roles |
| --- | --- | --- |
| `HelmChart` | `Chart.yaml` | `application`, `library` |
| `HelmRequirements` | chart API v1 `requirements.yaml` | `legacy_dependency_manifest` |
| `HelmLock` | `Chart.lock`, legacy `requirements.lock` | `dependency_lock`, `legacy_dependency_lock` |
| `HelmValues` | `values.yaml` and explicit values inputs | `default`, `parent`, `override` |
| `HelmValuesSchema` | `values.schema.json` | `validation_schema` |
| `HelmTemplate` | files evaluated from `templates/` | `resource`, `helper`, `notes`, `test`, `hook` |
| `HelmCRD` | files under `crds/`, not template-evaluated | `custom_resource_definition` |
| `HelmIgnore` | `.helmignore` | `ignore_rules` |

README, LICENSE, and unrelated chart files remain raw `Artifact` nodes unless another frontend
claims them.

`HelmChart` fields are `api_version`, `name`, `version`, `kube_version`, `description`,
`chart_type`, `keywords`, `home`, `sources`, typed `maintainers`, `icon`, `app_version`,
`deprecated`, and `annotations`. Dependency declarations are stored in the named
`dependencies{}` collection rather than embedded as an untyped field.

`HelmTemplate` contains these named collections:

```text
named_templates{}
template_calls{}
value_references{}
resource_templates{}
lookup_references{}
```

`call_kind` is a closed initial vocabulary: `template`, `include`, `block`, `tpl`. A `tpl` call
whose template text cannot be statically recovered remains an unresolved call node.
`lookup_references{}` records group/version/kind/namespace/name expressions but never contacts a
cluster during analysis.

### 3.5 Configuration, evaluation, and diagnostics

| # | Decision | Rationale |
| --- | --- | --- |
| D24 | **Render configuration is itself `Artifact:CodeAnalyzerIaCConfig`; it is not an `IaCArtifact`.** | It controls analysis rather than declaring infrastructure, while still retaining canonical source, hash, and provenance. |
| D25 | **Graph-only mode accepts only a config artifact already in the graph, by ID or app-relative path.** | Reading an arbitrary host file would violate graph-only reproducibility. Filesystem mode requires the config to lie beneath the selected workspace root. |
| D26 | **The config artifact contains explicit named `HelmRenderProfile` nodes; each chart contains one synthetic default profile.** | Several charts and environments cannot be represented safely by process-global flags, while a useful no-config render still needs a stable, inspectable context. |
| D27 | **Values inputs are ordered `HelmValueLayer` nodes; Helm owns merge semantics.** | Later files and explicit overrides win according to the actual engine. Reimplementing precedence would drift from Helm. |
| D28 | **Do not persist a duplicate merged-values blob.** | `HelmRender` records ordered layer IDs and `effective_values_sha256`; raw inputs already exist on their artifacts. |
| D29 | **Every chart gets one deterministic default render attempt; named profiles add renders.** | A one-command analysis should produce useful resource facts, while explicit profiles preserve environment-specific variants. |
| D30 | **Every render has typed status, phase, and first-class diagnostics.** | Failure must differ from a successful chart that produces no resources, and graph/JSON diagnostics must remain queryable. |
| D31 | **Sensitive derived values are hash-only by default.** | Complete repository input remains in `Artifact.source`, but generated or externally supplied secrets are not amplified across result nodes. |

The synthetic default profile is a real `HelmRenderProfile` contained by its `HelmChart`, with
`origin:"default"` and a deterministic ID. It uses chart defaults, a deterministic generated
release name, namespace `default`, the pinned renderer's default Kubernetes capabilities, and
locally available dependencies. It performs no dependency download, registry access, plugin
execution, cluster lookup, install, or upgrade. An explicit profile has `origin:"config"`, is
contained by the config artifact, and targets its chart through `IAC_RENDERS_CHART`.

An explicit configuration has this shape:

```yaml
renders:
  - name: production
    chart: can://artifact/payments/charts/api/Chart.yaml
    release_name: api
    namespace: payments
    values:
      - can://artifact/payments/charts/api/values-prod.yaml
    set:
      image.tag: "1.2.3"
    kube_version: "1.32"
    api_versions: []
```

The config artifact owns `render_profiles{}`. Each profile owns `value_layers{}` keyed by a
zero-padded ordinal. Literal overrides in the config are also `ConfigKey` children of that config
artifact; layers reference those nodes rather than copying values.

`--config` is the only way a file is interpreted as this analyzer's configuration; it is never
auto-detected from a conventional filename. Once selected and validated, the existing file node
keeps `kind:"artifact"` and gains an optional sibling facet:

```jsonc
{
  "id": "can://artifact/payments/.codeanalyzer-iac.yaml",
  "kind": "artifact",
  "source": "...",
  "sha256": "...",
  "codeanalyzer_iac_config": {
    "kind": "codeanalyzer_iac_config",
    "config_version": 1,
    "render_profiles": {
      "production": { "id": "can://iac/payments/config/profile/production", "origin": "config" }
    }
  }
}
```

In Neo4j that same node gains `:CodeAnalyzerIaCConfig`, `iac_config_version`, and namespaced config
properties; each profile remains a contained node. Removing `--config` on a later eager run may
remove that analyzer-owned facet and its profile subgraph, but never the underlying `Artifact` or
its source. Invalid configuration produces a config diagnostic and no explicit profile nodes.

`HelmRender.status` is `succeeded`, `partial`, `failed`, or `skipped`. `phase` is absent on
success and otherwise one of `load`, `dependency`, `values`, `template`, or `decode`.
`HelmDiagnostic` fields are `id`, `severity`, `code`, `message`, optional `artifact_id`, optional
`span`, and `phase`. A failed render never removes source-derived nodes. A partial render retains
only decoded resources and their exact render provenance.

### 3.6 Kubernetes render result

| # | Decision | Rationale |
| --- | --- | --- |
| D32 | **A rendered object uses one typed `KubernetesResource` envelope.** | Kubernetes and CRDs share `apiVersion`, `kind`, and metadata but have an open-ended `spec`; a closed class per kind cannot remain current. |
| D33 | **Kubernetes's `kind` is emitted as `resource_kind`; CLDK `kind` remains the node discriminant.** | Reusing one field for two contracts would break generic consumers. |
| D34 | **Rendered resources are render-scoped and target stable `KubernetesResourceAddress` nodes.** | Two profiles may create different desired states for the same logical target and must not overwrite each other. |
| D35 | **Objects with only `generateName` have no stable address node.** | Their final server identity does not exist before admission. |
| D36 | **Secret payloads retain key names and value hashes, never rendered values by default.** | The graph remains useful for dependency queries without copying generated credentials. |

`KubernetesResource` fields are `id`, `kind:"kubernetes_resource"`, `api_version`,
`resource_kind`, optional `namespace`, optional `name`, optional `generate_name`, `labels`,
`annotations`, `manifest_sha256`, `render_id`, and typed origin IDs. `status` is never present:
the output is desired state, not a live-cluster observation.

Its ID is scoped to the render:

```text
<render-id>/kubernetes/<group>/<kind>/<namespace>/<name>
```

The stable address excludes API version and uses the source-visible group/kind identity:

```text
can://iac/<app>/kubernetes/address/<group>/<kind>/<namespace>/<name>
```

When REST resource pluralization is known from built-in discovery data or an in-repository CRD,
it is stored as a typed field; identity does not require a live API server. The address node may
be shared by several render-scoped resources. A rendered resource uses
`IAC_DERIVED_FROM` to point at its primary `HelmResourceTemplate`; additional template/value
contributors are ordinary provenance edges.

---

## 4. JSON model

The root follows the schema-v2 envelope:

```jsonc
{
  "schema_version": "2.0.0",
  "language": "iac",
  "max_level": 3,
  "analyzer": { "name": "codeanalyzer-iac", "version": "0.1.0" },
  "application": {
    "id": "can://iac/payments",
    "kind": "application",
    "artifacts": {
      "charts/api/Chart.yaml": {
        "id": "can://artifact/payments/charts/api/Chart.yaml",
        "kind": "artifact",
        "path": "charts/api/Chart.yaml",
        "format": "yaml",
        "sha256": "...",
        "source": "...",
        "iac": {
          "dialect": "helm",
          "kind": "helm_chart",
          "api_version": "v2",
          "name": "api",
          "version": "1.2.3",
          "dependencies": {},
          "renders": {}
        },
        "aliases": [
          { "id": "can://iac/payments/helm/chart/charts/api", "kind": "helm_chart" }
        ]
      }
    },
    "external_chart_references": {},
    "kubernetes_resource_addresses": {},
    "edges": {}
  }
}
```

`Artifact.iac` is absent on unrecognized artifacts and is a discriminated union keyed by
`dialect` plus concrete `kind`. It is singular (D5). `Artifact.kind` never changes from
`artifact`. Existing `config_keys` remain contained once on the artifact; a config key's own
optional IaC facet carries `kind:"helm_value"`.

Named maps use semantic names rather than one catch-all collection. Edges are identity-only
`{src,dst}` records and live at the lowest common ancestor of their endpoints. All collections
and edge rows are deterministically sorted before emission.

### Analysis levels

IaC preserves the additive `L1 ⊆ L2 ⊆ L3` rule without pretending its nodes are code callables:

| Level | Adds | Never does |
| --- | --- | --- |
| L1 | Artifact classification, typed Helm facets, chart metadata, values, template definitions/calls/references, source diagnostics | Reference resolution or rendering |
| L2 | Chart membership, dependency resolution, named-template targets, value-reference targets, unresolved-reference records | Rendering or external I/O |
| L3 | Render profiles, value layers, render attempts, diagnostics, resource templates, rendered Kubernetes resources and addresses | Cluster access, install/upgrade, dependency fetching |

L4 is not implemented in the Helm-first train. It is reserved for typed cross-resource and
cross-dialect dependency inference after the Kubernetes relation vocabulary is designed. JSON
honors `--analysis-level`; Neo4j always receives the deepest implemented graph, currently L3.

---

## 5. Neo4j projection

The graph is a near-identity projection of the JSON model. Representative labels:

```text
(:Application:IaCApplication {id: "can://iac/payments", ...})

(:Artifact:IaCArtifact:HelmArtifact:HelmChart {
  id: "can://artifact/payments/charts/api/Chart.yaml",
  source: "...",
  sha256: "...",
  iac_dialect: "helm",
  iac_kind: "helm_chart",
  helm_name: "api",
  helm_version: "1.2.3"
})

(:ConfigKey:IaCValue:HelmValue {id: "...@key/image.tag", ...})
(:Artifact:CodeAnalyzerIaCConfig {
  id: "can://artifact/payments/.codeanalyzer-iac.yaml",
  source: "...",
  sha256: "...",
  iac_config_version: 1
})
(:HelmNamedTemplate {id: "can://iac/.../named-template/api.fullname", ...})
(:HelmRender {id: "can://iac/.../render/default@<input-hash>", ...})
(:KubernetesResource {id: "<render-id>/kubernetes/apps/Deployment/default/api", ...})
(:KubernetesResourceAddress {id: "can://iac/.../kubernetes/address/apps/Deployment/default/api"})
(:IdentityAlias:IaCAlias {id: "can://iac/payments/helm/chart/charts/api"})
```

Representative relationship types:

| Relationship | From → to | Meaning |
| --- | --- | --- |
| `HAS_ARTIFACT` | `IaCApplication → Artifact` | Shared neutral application inventory |
| `DEFINES_CONFIG` | `Artifact → ConfigKey` | Shared neutral config-key containment |
| `IAC_PART_OF_CHART` | non-anchor `HelmArtifact → HelmChart` | Nearest-chart membership |
| `IAC_DECLARES_DEPENDENCY` | `HelmChart → HelmDependency` | Source containment |
| `IAC_TARGETS_CHART_REFERENCE` | `HelmDependency → HelmChartReference` | Declared external target |
| `IAC_RESOLVES_TO_CHART` | `HelmChartReference → HelmChart` | Vendored/local resolution |
| `IAC_IDENTIFIED_BY_PACKAGE` | `HelmChartReference → Package` | Valid standard PURL identity, when available |
| `IAC_DEFINES_TEMPLATE` | `HelmTemplate → HelmNamedTemplate` | Source containment |
| `IAC_HAS_TEMPLATE_CALL` | `HelmTemplate → HelmTemplateCall` | Source containment |
| `IAC_CALLS_TEMPLATE` | `HelmTemplateCall → HelmNamedTemplate` | Resolved invocation |
| `IAC_HAS_VALUE_REFERENCE` | `HelmTemplate → HelmValueReference` | Source containment |
| `IAC_REFERENCES_VALUE` | `HelmValueReference → HelmValue` | Resolved value use |
| `IAC_DECLARES_PROFILE` | `CodeAnalyzerIaCConfig/HelmChart → HelmRenderProfile` | Explicit-config or synthetic-default containment |
| `IAC_RENDERS_CHART` | `HelmRenderProfile → HelmChart` | Profile target |
| `IAC_HAS_VALUE_LAYER` | `HelmRenderProfile → HelmValueLayer` | Ordered input model |
| `IAC_READS_FROM` | `HelmValueLayer → Artifact/ConfigKey` | Layer provenance |
| `IAC_HAS_RENDER` | `HelmChart → HelmRender` | Evaluation containment |
| `IAC_CONFIGURED_BY` | `HelmRender → HelmRenderProfile` | Evaluation context |
| `IAC_HAS_DIAGNOSTIC` | render/artifact → diagnostic | Structured failure/warning |
| `IAC_PRODUCES` | `HelmRender → KubernetesResource` | Rendered desired state |
| `IAC_TARGETS_RESOURCE` | `KubernetesResource → KubernetesResourceAddress` | Stable logical target |
| `IAC_DERIVED_FROM` | generated node → source node | Source provenance |
| `IAC_ALIAS_OF` | `IaCAlias → canonical node` | Alternate identity |

All relationship rows are identity-only. Ordering, provenance details, resolution reasons, and
other attributes live on nodes. There are uniqueness constraints for every canonical `id` label
and `IaCAlias.id`.

### Ownership and reconciliation

Every IaC-created semantic node carries `producer`, `analyzer_version`, and `iac_app_id` metadata.
The `IAC_*` relationship namespace is reserved to `codeanalyzer-iac`, so relationships need no
properties and remain identity-only. Reconciliation compares the desired canonical node-ID and
endpoint-pair sets for the selected application; existing neutral nodes are never claimed
wholesale:

- Filesystem mode creates an `Artifact` only when none exists. A conflicting existing hash is a
  diagnostic and is not overwritten silently.
- Graph mode never changes `Artifact.source`, `sha256`, or another analyzer's neutral properties.
- Apart from the shared neutral upserts below, IaC adds only its labels, namespaced `iac_*` and
  dialect properties such as `helm_*`, typed children, aliases, and `IAC_*` relationships.
- `HAS_ARTIFACT` and `DEFINES_CONFIG` reuse the shared neutral contract and are upsert-only. Eager
  cleanup never removes one because another analyzer may have emitted the same neutral fact.
- IDs make repeated runs idempotent.
- Default operation is non-destructive upsert.
- `--eager` reconciles only `codeanalyzer-iac`-owned nodes and aliases, namespaced `iac_*`
  properties/facet labels, and `IAC_*` relationships for the selected application and inputs.
  It never deletes an `Artifact`,
  `ConfigKey`, `Package`, application node from another analyzer, or a relationship owned by
  another producer.

A source-hash mismatch in graph mode emits an artifact diagnostic and skips semantic enrichment
for that artifact because any span would address different text. Analysis continues for other
artifacts. There is no unsafe “accept mismatch” flag in the first release.

---

## 6. Architecture

### 6.1 Repository layout

```text
codeanalyzer-iac/
├── cmd/codeanalyzer-iac/          CLI
├── internal/
│   ├── model/                     schema-v2 envelope, artifacts, IDs, spans, edges
│   ├── ingest/
│   │   ├── filesystem/            arbitrary-root walk and canonical Artifact creation
│   │   └── neo4j/                 Artifact/source streaming by app ID
│   ├── dialect/
│   │   ├── frontend.go            compiled frontend contract
│   │   └── registry.go            deterministic detector registry
│   ├── dialects/helm/
│   │   ├── detect.go              content + chart-context classification
│   │   ├── model.go               typed Helm facet union and child records
│   │   ├── chart.go               Chart.yaml/v1 requirements/lock parsing
│   │   ├── values.go              ConfigKey reuse and value references
│   │   ├── templates.go           Go-template AST, definitions and calls
│   │   ├── resolve.go             memberships, dependencies, templates, values
│   │   └── render.go              isolated Helm SDK rendering and decoding
│   ├── reconcile/                 producer-scoped idempotent graph updates
│   └── emit/
│       ├── json/                  deterministic analysis.json
│       └── neo4j/                 catalog, rows, Cypher, and Bolt sink
├── schema.neo4j.json
├── fixtures/helm/
├── tests/
└── .claude/SCHEMA_DECISIONS.md
```

The dialect interface accepts immutable artifact records and returns typed deltas:

```text
Detect(ArtifactContext) → no-match | one concrete dialect facet
Parse(Artifact, Facet) → contained nodes + diagnostics
Resolve(ApplicationIndex) → identity-only edges + unresolved records
Evaluate(Profile, VirtualArtifactTree) → render/results + diagnostics
```

Frontends do not write Neo4j or construct JSON directly. Emitters consume the common typed model,
which is what makes projection parity enforceable. The registry is compiled and ordered; dynamic
Go plugins are excluded because their ABI and distribution model would make releases fragile.

### 6.2 Pipeline

```text
filesystem roots                     prepopulated Neo4j
      │                                      │
      ▼                                      ▼
filesystem inventory                 stream can://artifact/<app>/...
      │ create/merge Artifact                │ verify source hash
      └──────────────────┬───────────────────┘
                         ▼
                deterministic detection       L1
                         ▼
             typed per-artifact source parse  L1
                         ▼
        application index + cross-file resolve L2
                         ▼
         default and configured Helm renders  L3
                         ▼
            typed Kubernetes result decode    L3
                         ▼
                  JSON model once
                    ├──────────────► analysis.json
                    └──────────────► Neo4j projection
```

Filesystem IDs are always relative to one stable workspace root; the one or more input paths are
selection filters, not competing identity roots. The workspace root defaults to the current
directory and may be set explicitly. Every input and config file must resolve beneath it. Paths
are canonicalized and overlapping inputs are deduplicated; symlinks are not followed outside the
workspace root. This makes `charts/api/values.yaml` mean the same thing whether the caller selects
the repository, `charts/`, or that file directly, and prevents two roots containing `values.yaml`
from colliding.

Graph mode pages artifacts rather than loading the whole application into one query result.
Parsing is parallel per artifact; resolution is deterministic per application; rendering is
parallel per independent chart/profile with bounded concurrency.

For graph rendering, the analyzer reconstructs only the selected chart's files in a private
temporary directory from graph source, validates every path against traversal, invokes the pinned
Helm Go SDK without plugins/network/cluster access, and removes the directory after the profile.
Packaged `.tgz` chart dependencies are inventoried but are not reconstructed from a text source in
the first train; unpacked vendored charts are supported.

### 6.3 CLI

```text
caniac [PATH ...] --app-name NAME [--workspace-root PATH] [--config PATH_OR_ARTIFACT_ID]
```

The command defaults to `.` when no path is supplied and may receive several files or directories.
When the path is a Neo4j URI, it implicitly runs in graph mode and enriches the selected application;
otherwise it analyzes the supplied filesystem paths. `--workspace-root` defaults to the current
directory and applies only in filesystem mode. In filesystem mode, `--config` is resolved relative
to that root and is inventoried as an artifact even when it lies outside the ordinary input filters.
In graph mode, `--config` selects an existing config artifact by ID or app-relative path. The command
emits JSON and may also write Neo4j. Graph mode selects all artifact IDs under
`can://artifact/<app-name>/`. `--app-name` is mandatory in graph mode; a name that does not resolve
fails with an actionable list rather than enriching an arbitrary application.

Common output modes are compact JSON/stdout, `analysis.json`, Cypher, direct Bolt, and schema
catalog. Secrets never appear in logs or diagnostics. Connection credentials are accepted through
the same environment/secret mechanisms as sibling Neo4j emitters and are never serialized.

---

## 7. Failure behavior

Failure is isolated at the smallest truthful unit:

- A discovery/read failure creates a diagnostic but cannot create a source-bearing artifact.
- An unrecognized artifact remains raw and is not an error.
- An ambiguous dialect match is a diagnostic and receives no dialect facet; the analyzer never
  chooses based on extension order.
- A parse error retains the artifact and every confidently parsed node before the failure, marked
  `partial` at the artifact facet.
- A graph source/hash mismatch skips that artifact's semantic model.
- A missing chart member or unresolved template/value/dependency emits an unresolved node or
  diagnostic and does not abort unrelated charts.
- A render error creates a failed/partial `HelmRender`; source facts remain valid.
- A Neo4j write is transactional per application generation. Artifact batches may stream, but the
  generation is made current only after all required rows and constraints succeed.

Strict mode raises a nonzero process exit when any error-severity diagnostic exists, after writing
the inspectable partial output. Default mode returns nonzero only for analyzer-wide failures such
as an unreadable input root, invalid configuration, unavailable graph, or failed output commit.

---

## 8. Verification contract

### Schema and identity

- Golden JSON validates against the generated v2 IaC JSON Schema.
- `schema.neo4j.json` byte-matches the catalog generated by the executable.
- Every graph node and relationship has an exact JSON counterpart; parity tests compare canonical
  row sets, not counts.
- Property tests cover percent encoding, path normalization, named IDs, anonymous `@line:column`
  IDs, alias uniqueness, and no dangling edge endpoints.
- Level fixtures prove `L1 ⊆ L2 ⊆ L3` modulo only additive resolution.

### Ingestion and reconciliation

- The same fixture analyzed from filesystem and then from prepopulated Neo4j produces identical
  typed IaC facts.
- A second run is idempotent.
- A fixture with pre-existing Python/TypeScript/Java artifact labels proves IaC enrichment keeps
  every neutral property and foreign edge.
- `--eager` removes stale IaC-owned facts and no foreign facts.
- Hash mismatch, missing source, and ambiguous app selection have exact diagnostics.

### Helm

- Fixtures cover chart API v1 and v2, application and library charts, nested/vendored charts,
  aliases and conditional dependencies, locks, values schema, CRDs, helpers, notes, hooks, tests,
  duplicate named-template definitions, dynamic `tpl`, unresolved value paths, and lookup calls.
- Named-template and value-reference expected target sets are hand-computed.
- Default and two explicit profiles produce exact, different resource sets without overwriting.
- Failed and partial render fixtures retain exact source nodes and diagnostics.
- No test depends on a live cluster, registry, chart repository, plugin, or network.

### Security and determinism

- Path traversal and symlink escape fixtures cannot write outside the private render directory.
- Kubernetes Secret values and generated credentials never appear in derived node properties,
  diagnostics, logs, snapshots, or Cypher; expected key names and hashes do.
- Output ordering and IDs are byte-stable across repeated runs and worker counts.
- Fuzz tests cover YAML, JSON Schema, Go-template, config, and rendered multi-document decoders;
  parser panics are analyzer bugs and are caught by the harness.

---

## 9. Delivery and release plan

Tracking shape: **one cross-repo epic with children filed per pull request, just in time.** The
shared schema, new backend, and docs release on separate clocks, so one work item cannot truthfully
close the change. The epic links this spec; it does not duplicate it.

Release order:

1. **Shared contract:** add the v2 IaC schema/catalog fixtures to `codeanalyzer-schema`.
2. **`codeanalyzer-iac` 0.1.0:** repository foundation, both ingestion modes, identity,
   non-destructive Neo4j reconciliation, L1 Helm artifact/chart/value/template model, L2
   reference/dependency resolution, L3 Helm evaluation, diagnostics, Kubernetes
   resources/addresses, sensitive-derived-data policy, config artifacts, and default/explicit
   render profiles, with JSON/Neo4j parity.
3. **Documentation:** CLI and graph-query guide after 0.1.0 behavior is released.

There is no lockstep release with Python, TypeScript, or Java analyzers. The shared join contract
is their already-shipped `can://artifact/<app>/<path>` identity and complete `source`; they do not
need an IaC release. The future SDK facade is its own design and release train.

---

## 10. Explicitly not now

- Implementing non-Helm frontends. Their names are in the umbrella, while each gets its own
  dialect schema loop and implementation slice.
- SDK facade integration.
- Live cluster discovery, `lookup` execution, drift detection, install, upgrade, or mutation.
- Dependency downloads, OCI authentication, registry access, or Helm plugin execution.
- Expansion of packaged chart archives from binary graph artifacts.
- A generated class/Neo4j label for every Kubernetes built-in or CRD kind.
- Kubernetes OpenAPI/CRD validation beyond syntax and in-repository CRD metadata.
- L4 cross-resource/cross-dialect dependency inference.
- Secret-manager resolution or persistence of sensitive derived values.
- Runtime-loaded dialect plugins.

These exclusions do not make files disappear. Every input remains an `Artifact`; later frontends
or analysis levels enrich the same canonical nodes.

---

## 11. Design consequences

The hierarchy has one reliable rule: source owns semantics. A file is always an `Artifact` first,
then gains exactly one IaC dialect facet and a concrete artifact kind. Dialect-native constructs
are contained beneath their source facet, while cross-file, dependency, evaluation, and generated
relationships remain explicit edges. This is intentionally not an analogy to
`module → class → callable`.

The approach costs more typed models than a universal property bag, and rendering creates
context-specific nodes. In return it preserves fidelity, keeps source and semantic identities
stable, supports exact JSON/Neo4j parity, and lets future dialects join one graph without teaching
every language analyzer how to parse infrastructure.

---

## 12. Normative references

- [Helm chart format](https://helm.sh/docs/topics/charts/)
- [Helm named templates](https://helm.sh/docs/chart_template_guide/named_templates/)
- [Helm values and precedence](https://helm.sh/docs/chart_template_guide/values_files/)
- [Helm Go SDK](https://helm.sh/docs/sdk/)
- [Kubernetes API resource concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [Kubernetes object model](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
- [Package-URL registered type definitions](https://github.com/package-url/purl-spec/tree/main/types)
