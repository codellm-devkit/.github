# IaC Schema Contract Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish the additive schema-v2 JSON and Neo4j contracts that `codeanalyzer-iac` 0.1.0 must emit for Helm L1–L3.

**Architecture:** Add a self-contained `v2/iac` branch to `codeanalyzer-schema`; do not modify any existing language analyzer schema. One Draft 2020-12 JSON Schema describes the typed analysis tree, a separate graph-contract meta-schema describes the neutral-plus-IaC projection, and a semantic checker enforces invariants that JSON Schema cannot express.

**Tech Stack:** JSON Schema Draft 2020-12, Python 3.11+, `jsonschema`, `unittest`, canonical JSON fixtures.

**Spec:** `docs/design/specs/2026-09-02-codeanalyzer-iac-helm.md`

## Global Constraints

- Work in `codeanalyzer-schema` on a feature branch based on its current canonical-schema branch; do not edit v1 analyzer snapshots.
- The analysis envelope is exactly `schema_version:"2.0.0"`, `language:"iac"`, and `max_level:1|2|3`; the IaC Neo4j catalog is exactly `schema_version:"1.0.0"`.
- Existing language analyzers keep their current schema-v2 output. IaC is a separate additive branch composed with neutral `Artifact`, `ConfigKey`, and `Package` identities.
- `Artifact.kind` remains `"artifact"`; raw `source` occurs once on `Artifact` and is required together with its lowercase 64-character SHA-256 digest.
- `Artifact.iac` is absent or one discriminated facet. The Helm-first train never emits a generic `IaCEntity` object or collection.
- Whole-file IDs remain `can://artifact/<app>/<percent-encoded-path>`; contained semantic IDs use `can://iac/<app>/<dialect>/...`; aliases are nodes and never canonical IDs.
- Source spans use one-based `[line,column]` pairs and half-open UTF-8 byte offsets `[start,end]` into `Artifact.source`.
- Named collections are JSON objects, edges are identity-only `{src,dst}`, and absent facts are omitted rather than serialized as `null`.
- Analysis levels are additive: L1 source facts, L2 resolution facts, L3 profiles/renders/Kubernetes desired state. L4 is rejected.
- Secret-derived output records key names and SHA-256 hashes only; no rendered Secret value is permitted in a derived node, diagnostic, or fixture.

---

## File Structure

Create these focused files in `codeanalyzer-schema`:

```text
v2/iac/json/analysis.schema.json          complete IaC analysis envelope and all typed definitions
v2/iac/json/analysis.l1.sample.json       Helm source-only conformance fixture
v2/iac/json/analysis.l2.sample.json       L1 fixture plus cross-file resolution
v2/iac/json/analysis.l3.sample.json       L2 fixture plus profiles, renders, and resources
v2/iac/neo4j/contract.schema.json         meta-schema for a codeanalyzer-iac graph catalog
v2/iac/neo4j/schema.neo4j.sample.json     complete graph catalog fixture for version 1.0.0
scripts/check_iac.py                      semantic invariants beyond Draft 2020-12
tests/test_check_iac.py                   focused checker unit tests
README.md                                 commands and ownership of the v2/iac branch
```

Keep JSON analysis samples and the graph sample in separate directories: `scripts/check.py` validates every `*.sample.json` next to each schema.

---

### Task 1: Register the IaC envelope and neutral artifact contract

**Files:**
- Create: `codeanalyzer-schema/v2/iac/json/analysis.schema.json`
- Create: `codeanalyzer-schema/v2/iac/json/analysis.l1.sample.json`

**Interfaces:**
- Consumes: the approved spec's JSON envelope, Artifact, Span, ConfigKey, IdentityAlias, Diagnostic, and identity-only Edge contracts.
- Produces: schema definitions named `Analysis`, `Analyzer`, `Application`, `Artifact`, `Span`, `ConfigKey`, `IdentityAlias`, `Diagnostic`, `Package`, `Edge`, `IaCFacet`, and `HelmChart`; later tasks extend the same `$defs` object.

- [ ] **Step 1: Create the implementation branch and schema child issue**

```bash
git switch feat/v1-schemas
git switch -c feat/iac-v2-contract
gh issue create --repo codellm-devkit/codeanalyzer-schema \
  --title "Add the schema-v2 IaC contract" \
  --body "Child of codellm-devkit/.github#52. Implements the accepted IaC/Helm contract from .github/docs/design/specs/2026-09-02-codeanalyzer-iac-helm.md: JSON schema 2.0.0, Neo4j catalog 1.0.0, level fixtures, and semantic conformance checks."
```

Expected: a new issue URL is printed; add that URL to the eventual pull request body.

- [ ] **Step 2: Write the smallest valid L1 fixture before the schema exists**

Create `analysis.l1.sample.json` with one raw file and one Helm chart anchor. Use real source bytes and compute each digest with `printf %s "$source" | shasum -a 256`; do not use illustrative hashes. The stable assertions are:

```json
{
  "schema_version": "2.0.0",
  "language": "iac",
  "max_level": 1,
  "analyzer": {"name": "codeanalyzer-iac", "version": "0.1.0"},
  "application": {
    "id": "can://iac/payments",
    "kind": "application",
    "artifacts": {
      "README.md": {
        "id": "can://artifact/payments/README.md",
        "kind": "artifact",
        "path": "README.md",
        "format": "text",
        "source": "payments infrastructure\n",
        "sha256": "79accdf71a72e00a8b25fe1284094368e885e2fe55db0cf686a83c5359bf43df",
        "size_bytes": 24,
        "config_keys": {},
        "aliases": []
      },
      "charts/api/Chart.yaml": {
        "id": "can://artifact/payments/charts/api/Chart.yaml",
        "kind": "artifact",
        "path": "charts/api/Chart.yaml",
        "format": "yaml",
        "source": "apiVersion: v2\nname: api\nversion: 1.2.3\ntype: application\n",
        "sha256": "4a0bf85edced2838bb82ae1a52bf8830a718462ff54f4c6fb11ce539fabb53af",
        "size_bytes": 58,
        "config_keys": {},
        "aliases": [{
          "id": "can://iac/payments/helm/chart/charts/api",
          "kind": "helm_chart",
          "target": "can://artifact/payments/charts/api/Chart.yaml"
        }],
        "iac": {
          "dialect": "helm",
          "kind": "helm_chart",
          "status": "complete",
          "api_version": "v2",
          "name": "api",
          "version": "1.2.3",
          "chart_type": "application",
          "maintainers": [],
          "keywords": [],
          "sources": [],
          "annotations": {},
          "dependencies": {},
          "renders": {}
        }
      }
    },
    "packages": {},
    "external_chart_references": {},
    "kubernetes_resource_addresses": {},
    "diagnostics": {},
    "edges": {}
  }
}
```

- [ ] **Step 3: Verify discovery fails while the schema is absent**

Run: `python3 scripts/check.py v2/iac/json`

Expected: exit 1 with `no schemas under`.

- [ ] **Step 4: Create the closed root, neutral types, and Helm discriminator**

Create `analysis.schema.json` with Draft 2020-12, `$id` `https://codellm-devkit.github.io/schema/v2/iac/analysis.schema.json`, `additionalProperties:false` at every typed object, and this root:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://codellm-devkit.github.io/schema/v2/iac/analysis.schema.json",
  "title": "codeanalyzer-iac schema v2 analysis output",
  "x-cldk": {
    "schemaVersion": "2.0.0",
    "language": "iac",
    "source": {
      "repo": "codellm-devkit/codeanalyzer-iac",
      "release": "v0.1.0",
      "definedIn": "internal/model"
    }
  },
  "$ref": "#/$defs/Analysis",
  "$defs": {
    "Analysis": {
      "type": "object",
      "additionalProperties": false,
      "required": ["schema_version", "language", "max_level", "analyzer", "application"],
      "properties": {
        "schema_version": {"const": "2.0.0"},
        "language": {"const": "iac"},
        "max_level": {"type": "integer", "minimum": 1, "maximum": 3},
        "analyzer": {"$ref": "#/$defs/Analyzer"},
        "application": {"$ref": "#/$defs/Application"}
      }
    },
    "Analyzer": {
      "type": "object",
      "additionalProperties": false,
      "required": ["name", "version"],
      "properties": {
        "name": {"const": "codeanalyzer-iac"},
        "version": {"type": "string", "minLength": 1}
      }
    }
  }
}
```

Add exact neutral constraints:

- `Application`: required `id`, `kind:"application"`, `artifacts`, `packages`, `external_chart_references`, `kubernetes_resource_addresses`, `diagnostics`, and `edges`; `id` matches `^can://iac/[^/]+$`.
- `Artifact`: required `id`, `kind:"artifact"`, `path`, `format`, `source`, `sha256`, `size_bytes`, `config_keys`, and `aliases`; optional `iac` and `codeanalyzer_iac_config`; path is relative POSIX and rejects empty, leading `/`, `.` segments, and `..` segments.
- `Span`: required `start`, `end`, and `bytes`; each line/column tuple has exactly two integers with minimum 1; byte tuple has exactly two non-negative integers.
- `ConfigKey`: required `id`, `kind:"config_key"`, `name`, `path`, and `span`; optional `iac` references `HelmValueFacet`.
- `IdentityAlias`: required `id`, `kind`, and `target`; `id` matches `^can://iac/`, while `target` accepts `^can://(artifact|iac)/`.
- `Diagnostic`: required `id`, `kind:"diagnostic"`, `severity`, `code`, `message`; `severity` is `info|warning|error`, optional phase is `load|dependency|values|template|decode`, and optional fields are `artifact_id` and `span`.
- `Package`: required `id`, `kind:"package"`, and `purl`; both `id` and `purl` contain the same registered Package URL.
- `Edge`: exactly `{src,dst}`, both non-empty strings.
- `IaCFacet`: `oneOf` references every concrete Helm artifact facet. In this task register `HelmChart`; later tasks add the remaining facets to this same union.

Use this discriminator shape for every concrete facet:

```json
"HelmChart": {
  "type": "object",
  "additionalProperties": false,
  "required": ["dialect", "kind", "status", "api_version", "name", "version", "dependencies", "renders"],
  "properties": {
    "dialect": {"const": "helm"},
    "kind": {"const": "helm_chart"},
    "status": {"enum": ["complete", "partial", "failed"]},
    "api_version": {"enum": ["v1", "v2"]},
    "name": {"type": "string", "minLength": 1},
    "version": {"type": "string", "minLength": 1},
    "kube_version": {"type": "string"},
    "description": {"type": "string"},
    "chart_type": {"enum": ["application", "library"]},
    "keywords": {"type": "array", "items": {"type": "string"}},
    "home": {"type": "string"},
    "sources": {"type": "array", "items": {"type": "string"}},
    "maintainers": {"type": "array", "items": {"$ref": "#/$defs/HelmMaintainer"}},
    "icon": {"type": "string"},
    "app_version": {"type": "string"},
    "deprecated": {"type": "boolean"},
    "annotations": {"type": "object", "additionalProperties": {"type": "string"}},
    "dependencies": {"type": "object", "additionalProperties": {"$ref": "#/$defs/HelmDependency"}},
    "renders": {"type": "object", "additionalProperties": {"$ref": "#/$defs/HelmRender"}}
  }
}
```

Define `HelmMaintainer` with required `name` and optional `email` and `url`. Define the references to later-level types now so the schema remains one document; their definitions are added in Tasks 2 and 3.

- [ ] **Step 5: Correct the fixture digests and validate L1**

Run this command for each literal source and replace the fixture digest if its checked-in value differs:

```bash
printf %s 'apiVersion: v2
name: api
version: 1.2.3
type: application
' | shasum -a 256
```

Run: `python3 scripts/check.py v2/iac/json`

Expected: schema and `analysis.l1.sample.json` both print `ok`.

- [ ] **Step 6: Commit the neutral contract**

```bash
git add v2/iac/json/analysis.schema.json v2/iac/json/analysis.l1.sample.json
git commit -m "feat(schema): add IaC v2 artifact envelope"
```

---

### Task 2: Add complete Helm L1 source types

**Files:**
- Modify: `codeanalyzer-schema/v2/iac/json/analysis.schema.json`
- Modify: `codeanalyzer-schema/v2/iac/json/analysis.l1.sample.json`

**Interfaces:**
- Consumes: `Artifact`, `Span`, `ConfigKey`, `Diagnostic`, and the `IaCFacet` union from Task 1.
- Produces: all L1 Helm artifact facets plus `HelmDependency`, `HelmNamedTemplate`, `HelmTemplateCall`, `HelmValueReference`, `HelmResourceTemplate`, and `HelmLookupReference`.

- [ ] **Step 1: Expand the L1 fixture with a values file and template**

Add `charts/api/values.yaml` and `charts/api/templates/deployment.yaml`. Assert the values artifact has `config_keys.image.tag.iac.kind == "helm_value"`, and the template facet has one value reference, one named-template call, one resource template, and one lookup reference. Use these required shapes:

```json
"config_keys": {
  "image.tag": {
    "id": "can://artifact/payments/charts/api/values.yaml@key/image.tag",
    "kind": "config_key",
    "name": "tag",
    "path": "image.tag",
    "span": {"start": [2, 3], "end": [2, 14], "bytes": [9, 20]},
    "iac": {"kind": "helm_value"}
  }
}
```

```json
"iac": {
  "dialect": "helm",
  "kind": "helm_template",
  "status": "complete",
  "roles": ["resource"],
  "named_templates": {},
  "template_calls": {
    "6:11": {
      "id": "can://iac/payments/helm/charts/api/templates/deployment.yaml/template-call@6:11",
      "kind": "helm_template_call",
      "call_kind": "include",
      "name_expression": "api.fullname",
      "span": {"start": [6, 11], "end": [6, 39], "bytes": [73, 101]}
    }
  },
  "value_references": {
    "10:16": {
      "id": "can://iac/payments/helm/charts/api/templates/deployment.yaml/value-reference@10:16",
      "kind": "helm_value_reference",
      "path_expression": "image.tag",
      "span": {"start": [10, 16], "end": [10, 33], "bytes": [162, 179]}
    }
  },
  "resource_templates": {},
  "lookup_references": {}
}
```

- [ ] **Step 2: Run the schema check to expose missing facet definitions**

Run: `python3 scripts/check.py v2/iac/json`

Expected: FAIL with unresolved `$ref` or a `oneOf` validation error for `helm_values` or `helm_template`.

- [ ] **Step 3: Add every L1 artifact and child definition**

Extend `IaCFacet.oneOf` with exactly these concrete definitions and closed vocabularies:

| Definition | `kind` | Required named fields |
| --- | --- | --- |
| `HelmRequirements` | `helm_requirements` | `status`, `roles:[legacy_dependency_manifest]`, `dependencies{}` |
| `HelmLock` | `helm_lock` | `status`, `roles`, `dependencies{}`; roles are `dependency_lock|legacy_dependency_lock` |
| `HelmValues` | `helm_values` | `status`, `roles`; roles are `default|parent|override` |
| `HelmValuesSchema` | `helm_values_schema` | `status`, `roles:[validation_schema]` |
| `HelmTemplate` | `helm_template` | `status`, `roles`, `named_templates{}`, `template_calls{}`, `value_references{}`, `resource_templates{}`, `lookup_references{}` |
| `HelmCRD` | `helm_crd` | `status`, `roles:[custom_resource_definition]` |
| `HelmIgnore` | `helm_ignore` | `status`, `roles:[ignore_rules]` |

Every row also requires `dialect:"helm"`. Template roles are the unique subset of `resource|helper|notes|test|hook`.

Add the exact child discriminants and fields:

```json
"HelmTemplateCall": {
  "type": "object",
  "additionalProperties": false,
  "required": ["id", "kind", "call_kind", "name_expression", "span"],
  "properties": {
    "id": {"type": "string", "pattern": "^can://iac/"},
    "kind": {"const": "helm_template_call"},
    "call_kind": {"enum": ["template", "include", "block", "tpl"]},
    "name_expression": {"type": "string"},
    "target_id": {"type": "string", "pattern": "^can://iac/"},
    "span": {"$ref": "#/$defs/Span"}
  }
}
```

- `HelmDependency`: `id`, `kind:"helm_dependency"`, `name`, `version_constraint`, `span`; optional `alias`, `repository`, `condition`, `tags[]`, and typed `import_values[]` entries that are either a string or `{child,parent}`.
- `HelmNamedTemplate`: `id`, `kind:"helm_named_template"`, `name`, `span`.
- `HelmValueReference`: `id`, `kind:"helm_value_reference"`, `path_expression`, `span`; optional `target_id` permits L2 refinement.
- `HelmResourceTemplate`: `id`, `kind:"helm_resource_template"`, `document_index`, `span`.
- `HelmLookupReference`: `id`, `kind:"helm_lookup_reference"`, `group_expression`, `version_expression`, `resource_kind_expression`, `namespace_expression`, `name_expression`, `span`.
- `HelmValueFacet`: exactly `{kind:"helm_value"}`.

No L1 definition requires a resolution target or render output.

- [ ] **Step 4: Validate all concrete L1 values**

Run: `python3 scripts/check.py v2/iac/json`

Expected: PASS. Then run:

```bash
python3 - <<'PY'
import json
p = json.load(open('v2/iac/json/analysis.l1.sample.json'))
a = p['application']['artifacts']
assert a['README.md'].get('iac') is None
assert a['charts/api/values.yaml']['config_keys']['image.tag']['iac'] == {'kind': 'helm_value'}
assert a['charts/api/templates/deployment.yaml']['iac']['roles'] == ['resource']
assert p['max_level'] == 1
PY
```

Expected: exit 0.

- [ ] **Step 5: Commit Helm L1**

```bash
git add v2/iac/json/analysis.schema.json v2/iac/json/analysis.l1.sample.json
git commit -m "feat(schema): define Helm source facts"
```

---

### Task 3: Add L2 resolution and L3 evaluation contracts

**Files:**
- Modify: `codeanalyzer-schema/v2/iac/json/analysis.schema.json`
- Create: `codeanalyzer-schema/v2/iac/json/analysis.l2.sample.json`
- Create: `codeanalyzer-schema/v2/iac/json/analysis.l3.sample.json`

**Interfaces:**
- Consumes: all L1 maps and node IDs from Tasks 1–2.
- Produces: `HelmChartReference`, `CodeAnalyzerIaCConfig`, `HelmRenderProfile`, `HelmValueLayer`, `HelmRender`, `KubernetesResource`, `KubernetesResourceAddress`, and all named edge-family definitions.

- [ ] **Step 1: Create an additive L2 fixture**

Copy the L1 fixture to `analysis.l2.sample.json`, set `max_level` to 2, and only add facts:

- set the existing template call's `target_id` to a real `HelmNamedTemplate.id`;
- set the existing value reference's `target_id` to the existing `ConfigKey.id`;
- add a `HelmChartReference` in `external_chart_references`;
- retain `packages{}` and `diagnostics{}` from L1, adding entries only when a reference has a registered PURL or an application-scoped diagnostic exists;
- add edge maps named by the relationship vocabulary.

Use the edge map shape below; keys are stable caller-selected row IDs and values stay identity-only:

```json
"iac_references_value": {
  "charts/api/templates/deployment.yaml@10:16": {
    "src": "can://iac/payments/helm/charts/api/templates/deployment.yaml/value-reference@10:16",
    "dst": "can://artifact/payments/charts/api/values.yaml@key/image.tag"
  }
}
```

- [ ] **Step 2: Create an additive L3 fixture**

Copy the L2 fixture to `analysis.l3.sample.json`, set `max_level` to 3, and add:

- one `Artifact:CodeAnalyzerIaCConfig` sibling facet with a config-origin profile;
- one default profile contained by the chart;
- ordered value layers keyed `0000`, `0001`;
- one successful default render and one failed explicit render;
- one `KubernetesResource` for a Deployment and one Secret whose `secret_data` is `{key,sha256}` only;
- one stable `KubernetesResourceAddress` for each named resource;
- matching identity-only edge maps.

The resource and Secret payload contract is:

```json
"secret_data": {
  "password": {
    "key": "password",
    "sha256": "5994471abb01112afcc18159f6cc74b4f511b99806da59b3caf5a9c173cacfc5"
  }
}
```

Do not place the plaintext whose digest is shown in any derived field, message, or edge.

- [ ] **Step 3: Run validation to prove the new fixtures are not yet accepted**

Run: `python3 scripts/check.py v2/iac/json`

Expected: FAIL on missing L2/L3 definitions or disallowed properties.

- [ ] **Step 4: Add the closed L2/L3 node types**

Add these exact fields:

- `HelmChartReference`: required `id`, `kind:"helm_chart_reference"`, `name`, `version_constraint`; optional `repository`, `purl`, and `resolved_chart_id`.
- `CodeAnalyzerIaCConfig`: required `kind:"codeanalyzer_iac_config"`, `config_version:1`, `render_profiles{}`.
- `HelmRenderProfile`: required `id`, `kind:"helm_render_profile"`, `name`, `origin`, `chart_id`, `release_name`, `namespace`, `value_layers{}`, `api_versions[]`; `origin` is `default|config`; optional `kube_version`.
- `HelmValueLayer`: required `id`, `kind:"helm_value_layer"`, `ordinal`, `source_id`; ordinal is a non-negative integer.
- `HelmRender`: required `id`, `kind:"helm_render"`, `status`, `profile_id`, `renderer_name`, `renderer_version`, `value_layer_ids[]`, `effective_values_sha256`, `diagnostics{}`, `resources{}`; `status` is `succeeded|partial|failed|skipped`; optional `phase` uses the diagnostic phase vocabulary.
- `KubernetesResource`: required `id`, `kind:"kubernetes_resource"`, `api_version`, `resource_kind`, `manifest_sha256`, `render_id`, `origin_ids[]`; optional `namespace`, `name`, `generate_name`, `labels`, `annotations`, `plural`, `address_id`, and `secret_data`; explicitly disallow `status` and arbitrary manifest/spec maps.
- `KubernetesSecretDatum`: exactly `{key,sha256}` with a lowercase 64-character digest.
- `KubernetesResourceAddress`: required `id`, `kind:"kubernetes_resource_address"`, `group`, `resource_kind`, `namespace`, `name`; optional `plural`.

Define `Application.edges` as a closed object whose optional properties are maps of `Edge` for every relationship in spec section 5:

```json
"Edges": {
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "has_artifact": {"$ref": "#/$defs/EdgeMap"},
    "defines_config": {"$ref": "#/$defs/EdgeMap"},
    "iac_part_of_chart": {"$ref": "#/$defs/EdgeMap"},
    "iac_declares_dependency": {"$ref": "#/$defs/EdgeMap"},
    "iac_targets_chart_reference": {"$ref": "#/$defs/EdgeMap"},
    "iac_resolves_to_chart": {"$ref": "#/$defs/EdgeMap"},
    "iac_identified_by_package": {"$ref": "#/$defs/EdgeMap"},
    "iac_defines_template": {"$ref": "#/$defs/EdgeMap"},
    "iac_has_template_call": {"$ref": "#/$defs/EdgeMap"},
    "iac_calls_template": {"$ref": "#/$defs/EdgeMap"},
    "iac_has_value_reference": {"$ref": "#/$defs/EdgeMap"},
    "iac_references_value": {"$ref": "#/$defs/EdgeMap"},
    "iac_declares_profile": {"$ref": "#/$defs/EdgeMap"},
    "iac_renders_chart": {"$ref": "#/$defs/EdgeMap"},
    "iac_has_value_layer": {"$ref": "#/$defs/EdgeMap"},
    "iac_reads_from": {"$ref": "#/$defs/EdgeMap"},
    "iac_has_render": {"$ref": "#/$defs/EdgeMap"},
    "iac_configured_by": {"$ref": "#/$defs/EdgeMap"},
    "iac_has_diagnostic": {"$ref": "#/$defs/EdgeMap"},
    "iac_produces": {"$ref": "#/$defs/EdgeMap"},
    "iac_targets_resource": {"$ref": "#/$defs/EdgeMap"},
    "iac_derived_from": {"$ref": "#/$defs/EdgeMap"},
    "iac_alias_of": {"$ref": "#/$defs/EdgeMap"}
  }
}
```

- [ ] **Step 5: Validate all three analysis levels**

Run: `python3 scripts/check.py v2/iac/json`

Expected: all four printed paths (one schema, three fixtures) report `ok`.

- [ ] **Step 6: Commit the full JSON contract**

```bash
git add v2/iac/json
git commit -m "feat(schema): add Helm resolution and render contracts"
```

---

### Task 4: Register the neutral-plus-IaC Neo4j catalog

**Files:**
- Create: `codeanalyzer-schema/v2/iac/neo4j/contract.schema.json`
- Create: `codeanalyzer-schema/v2/iac/neo4j/schema.neo4j.sample.json`

**Interfaces:**
- Consumes: JSON node kinds and edge families from Tasks 1–3.
- Produces: a graph-catalog validator compatible with `codeanalyzer-iac --emit schema`, including neutral shared labels and reserved `IAC_*` relationship types.

- [ ] **Step 1: Write a failing graph catalog sample**

Create `schema.neo4j.sample.json` with generator `codeanalyzer-iac`, schema version `1.0.0`, and entries for every node/relationship below. Each node key is `id`; every relationship has `properties:{}`.

Required node labels:

```text
Application, IaCApplication, Artifact, IaCArtifact, HelmArtifact,
HelmChart, HelmRequirements, HelmLock, HelmValues, HelmValuesSchema,
HelmTemplate, HelmCRD, HelmIgnore, ConfigKey, IaCValue, HelmValue,
CodeAnalyzerIaCConfig, HelmDependency, HelmChartReference,
HelmNamedTemplate, HelmTemplateCall, HelmValueReference,
HelmResourceTemplate, HelmLookupReference, HelmRenderProfile,
HelmValueLayer, HelmRender, IaCDiagnostic, HelmDiagnostic, KubernetesResource,
KubernetesResourceAddress, IdentityAlias, IaCAlias, Package
```

Required relationship types are uppercase forms of every edge property in Task 3. `HAS_ARTIFACT` and `DEFINES_CONFIG` are the only neutral names; all other types begin `IAC_`.

- [ ] **Step 2: Verify the missing graph schema fails discovery**

Run: `python3 scripts/check.py v2/iac/neo4j`

Expected: exit 1 with `no schemas under`.

- [ ] **Step 3: Create the graph contract meta-schema**

Do not reuse the v1 prefix rule, because this backend intentionally composes with shared neutral nodes. Create a closed meta-schema requiring `schema_version`, `generator`, `marker_labels`, `node_labels`, `relationship_types`, `constraints`, and `indexes`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://codellm-devkit.github.io/schema/v2/iac/neo4j/contract.schema.json",
  "title": "codeanalyzer-iac Neo4j graph contract",
  "x-cldk": {
    "kind": "neo4j-graph-contract",
    "validates": ["schema.neo4j.sample.json"]
  },
  "type": "object",
  "additionalProperties": false,
  "required": ["schema_version", "generator", "marker_labels", "node_labels", "relationship_types", "constraints", "indexes"],
  "properties": {
    "schema_version": {"const": "1.0.0"},
    "generator": {"const": "codeanalyzer-iac"},
    "marker_labels": {"type": "array", "items": {"type": "string"}, "uniqueItems": true},
    "node_labels": {"type": "array", "items": {"$ref": "#/$defs/NodeLabel"}, "minItems": 1},
    "relationship_types": {"type": "array", "items": {"$ref": "#/$defs/RelationshipType"}, "minItems": 1},
    "constraints": {"type": "array", "items": {"type": "string", "pattern": "^CREATE CONSTRAINT "}},
    "indexes": {"type": "array", "items": {"type": "string", "pattern": "^CREATE (INDEX|FULLTEXT INDEX|VECTOR INDEX) "}}
  }
}
```

`NodeLabel` requires `label`, `merge_label`, `key`, and `properties`; `RelationshipType` requires `type`, `from`, `to`, and `properties`; property types are exactly `string|integer|float|boolean|string[]|integer[]|float[]|boolean[]`.

Add semantic name patterns:

- neutral labels are exactly `Application|Artifact|ConfigKey|Package|IdentityAlias`;
- owned labels match `^(IaC|Helm|Kubernetes|CodeAnalyzerIaC)`;
- relationship types are exactly `HAS_ARTIFACT|DEFINES_CONFIG` or match `^IAC_[A-Z0-9_]+$`.

- [ ] **Step 4: Validate the catalog and assert identity-only edges**

Run:

```bash
python3 scripts/check.py v2/iac/neo4j
python3 - <<'PY'
import json
p = json.load(open('v2/iac/neo4j/schema.neo4j.sample.json'))
assert p['schema_version'] == '1.0.0'
assert p['generator'] == 'codeanalyzer-iac'
assert all(r['properties'] == {} for r in p['relationship_types'])
assert {r['type'] for r in p['relationship_types']} >= {
    'HAS_ARTIFACT', 'DEFINES_CONFIG', 'IAC_ALIAS_OF', 'IAC_PRODUCES'
}
PY
```

Expected: both commands exit 0.

- [ ] **Step 5: Commit the graph contract**

```bash
git add v2/iac/neo4j
git commit -m "feat(schema): register IaC Neo4j catalog"
```

---

### Task 5: Enforce semantic invariants and document the contract

**Files:**
- Create: `codeanalyzer-schema/scripts/check_iac.py`
- Create: `codeanalyzer-schema/tests/test_check_iac.py`
- Modify: `codeanalyzer-schema/scripts/check.py`
- Modify: `codeanalyzer-schema/README.md`

**Interfaces:**
- Consumes: the three analysis fixtures and graph catalog from Tasks 1–4.
- Produces: `check_document(document: dict) -> list[str]`, `collect_nodes(application: dict) -> dict[str,dict]`, `assert_monotone(lower: dict,higher: dict) -> list[str]`, `check_catalog(catalog: dict) -> list[str]`, and a repository-level command that runs structural plus semantic validation.

- [ ] **Step 1: Write failing tests for invariants JSON Schema cannot express**

```python
from copy import deepcopy
import json
from pathlib import Path
import unittest

from scripts.check_iac import check_catalog, check_document

ROOT = Path(__file__).resolve().parents[1]


class CheckIaCTest(unittest.TestCase):
    def load(self, level: int) -> dict:
        path = ROOT / f"v2/iac/json/analysis.l{level}.sample.json"
        return json.loads(path.read_text())

    def test_rejects_dangling_edge(self):
        doc = self.load(2)
        doc["application"]["edges"]["iac_references_value"]["bad"] = {
            "src": "can://iac/payments/missing",
            "dst": "can://artifact/payments/charts/api/values.yaml@key/image.tag",
        }
        self.assertIn("dangling edge source", "\n".join(check_document(doc)))

    def test_rejects_alias_chain(self):
        doc = self.load(1)
        alias = doc["application"]["artifacts"]["charts/api/Chart.yaml"]["aliases"][0]
        alias["target"] = alias["id"]
        self.assertIn("alias target is not canonical", "\n".join(check_document(doc)))

    def test_rejects_bad_source_digest(self):
        doc = self.load(1)
        doc["application"]["artifacts"]["README.md"]["source"] += "changed"
        self.assertIn("sha256 does not match source", "\n".join(check_document(doc)))

    def test_rejects_plaintext_secret_amplification(self):
        doc = self.load(3)
        resource = next(iter(doc["application"]["artifacts"]["charts/api/Chart.yaml"]["iac"]["renders"].values()))["resources"]
        secret = next(v for v in resource.values() if v["resource_kind"] == "Secret")
        secret["secret_data"]["password"]["value"] = "hunter2"
        self.assertTrue(check_document(doc))

    def test_catalog_relationships_are_identity_only(self):
        catalog = json.loads((ROOT / "v2/iac/neo4j/schema.neo4j.sample.json").read_text())
        catalog["relationship_types"][0]["properties"]["ordinal"] = "integer"
        self.assertIn("relationship properties must be empty", "\n".join(check_catalog(catalog)))
```

- [ ] **Step 2: Run the tests to verify import failure**

Run: `python3 -m unittest tests.test_check_iac -v`

Expected: FAIL with `ModuleNotFoundError: No module named 'scripts.check_iac'`.

- [ ] **Step 3: Implement recursive collection and semantic checks**

Implement traversal over dictionaries and lists, treating every object with string `id` and string `kind` as a node. Enforce unique IDs, real edge endpoints, a canonical alias target, matching Artifact digests, relative artifact paths, span byte bounds, profile layer ordinals, and Secret hash-only fields:

```python
def collect_nodes(application: dict) -> dict[str, dict]:
    nodes: dict[str, dict] = {}

    def visit(value: object) -> None:
        if isinstance(value, dict):
            node_id = value.get("id")
            kind = value.get("kind")
            if isinstance(node_id, str) and isinstance(kind, str):
                if node_id in nodes:
                    raise ValueError(f"duplicate node id: {node_id}")
                nodes[node_id] = value
            for child in value.values():
                visit(child)
        elif isinstance(value, list):
            for child in value:
                visit(child)

    visit(application)
    return nodes
```

`check_document` catches duplicate-ID `ValueError` and returns every violation in deterministic sorted order. For each Artifact with non-empty source, compute `hashlib.sha256(source.encode("utf-8")).hexdigest()`; an empty source is an explicitly ineligible raw artifact and is never parsed. For every Package, require `id == purl`. For every alias, require `target in nodes`, reject targets whose kind is an alias kind, reject `id == target`, and require exactly one matching `iac_alias_of` edge. For every edge, require both endpoints in `nodes`. For Secret `secret_data`, require each datum's keys equal `{"key","sha256"}`.

`check_catalog` rejects duplicate labels/types, unknown endpoint labels, non-empty relationship properties, non-allowlisted neutral names, non-`IAC_*` owned relationships, and constraints/indexes that mention labels absent from `node_labels`.

- [ ] **Step 4: Add a monotonicity comparison and its test**

Implement a projection that removes only `max_level`, treats L2 `target_id` additions as allowed null-to-value refinement, and verifies every lower-level scalar/map/list member remains unchanged in the higher-level fixture:

```python
def assert_monotone(lower: dict, higher: dict) -> list[str]:
    errors: list[str] = []

    def walk(left: object, right: object, path: tuple[str, ...]) -> None:
        if isinstance(left, dict):
            if not isinstance(right, dict):
                errors.append(f"type changed at {'/'.join(path)}")
                return
            for key, value in left.items():
                if key == "max_level":
                    continue
                if key not in right:
                    errors.append(f"removed {'/'.join(path + (key,))}")
                else:
                    walk(value, right[key], path + (key,))
            return
        if isinstance(left, list):
            if not isinstance(right, list) or left != right[:len(left)]:
                errors.append(f"list changed at {'/'.join(path)}")
            return
        if left != right:
            errors.append(f"value changed at {'/'.join(path)}")

    walk(lower, higher, ())
    return sorted(errors)
```

Structure fixtures so L2 resolution adds `target_id` keys that were absent at L1; it must not replace an existing scalar. Test both `L1 ⊆ L2` and `L2 ⊆ L3`.

- [ ] **Step 5: Make the repository checker invoke IaC semantics**

After normal Draft validation succeeds, have `scripts/check.py` load every `v2/iac/json/analysis.l*.sample.json`, print each semantic violation, and call `assert_monotone` on adjacent levels. Keep this guarded by path existence so v1-only selectors still work.

Run:

```bash
python3 -m unittest tests.test_check_iac -v
python3 scripts/check.py
```

Expected: all unit tests pass and every schema/sample prints `ok` with exit 0.

- [ ] **Step 6: Document ownership and commands**

Replace the one-line README with sections for v1 snapshots and the new `v2/iac` normative contract. Include exact commands:

```bash
python3 -m pip install jsonschema
python3 scripts/check.py
python3 -m unittest discover -s tests -v
```

State that `codeanalyzer-iac` must byte-match `v2/iac/neo4j/schema.neo4j.sample.json`, while the JSON schema is normative and analyzer golden outputs validate against it.

- [ ] **Step 7: Run the full contract gate**

Run:

```bash
python3 -m unittest discover -s tests -v
python3 scripts/check.py
git diff --check
```

Expected: tests pass, every schema/sample reports `ok`, and `git diff --check` has no output.

- [ ] **Step 8: Commit the conformance gate**

```bash
git add scripts/check.py scripts/check_iac.py tests/test_check_iac.py README.md
git commit -m "test(schema): enforce IaC contract invariants"
```

---

## Plan Completion Gate

Run from `codeanalyzer-schema`:

```bash
python3 -m unittest discover -s tests -v
python3 scripts/check.py
git status --short
```

Expected: all tests and schema checks pass. Only intentional branch changes may appear in status. Record the exact `codeanalyzer-schema` commit in the `codeanalyzer-iac` module comment or README before starting the backend plan.
