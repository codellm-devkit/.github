# IaC catalog: containment edges for resource templates, lookup references, and aliases

Status: approved for implementation (decisions taken 2026-09-05)
Parent epic: codellm-devkit/.github#52
Tracking: codellm-devkit/codeanalyzer-schema#2 (catalog), analyzer child filed when picked up
Extends: `docs/design/specs/2026-09-02-codeanalyzer-iac-helm.md` §5

## 1. Problem

The accepted IaC Neo4j catalog (`schema.neo4j.json`, catalog version 1.0.0, schema commit
`e127901`) gives a `HelmTemplate` artifact an edge to each kind of contained source fact except
two: `HelmResourceTemplate` and `HelmLookupReference`. In `analysis.json` those children nest
under the template facet's `resource_templates{}` and `lookup_references{}`, so containment is
implicit. In the graph the only edge that can reach a `HelmResourceTemplate` is `IAC_DERIVED_FROM`
from a rendered `KubernetesResource`, and nothing reaches a `HelmLookupReference`.

`IaCAlias` nodes have the same shape from the other side: they nest under `Artifact.aliases[]`
in JSON but the graph has no edge from the owning artifact to the alias, only the alias's own
outgoing `IAC_ALIAS_OF`.

Observed on `sample.daytrader.microservices@8a68b59`, chart `platform/helm`: the five
`*-service.yaml` templates each hold a `Service` document and a `{{ if .Values.ocCreateRoute }}`
`Route` document. `codeanalyzer-iac` attributes render provenance only for single-region template
files (per-region emission counts are not statically knowable through `if`/`range`), so those ten
`HelmResourceTemplate` nodes receive no `IAC_DERIVED_FROM` and, lacking a containment edge, are
disconnected islands in Neo4j.

## 2. Contract-impact triage

| Question | Answer |
| --- | --- |
| Changes `analysis.json` shape? | No. Nesting under `resource_templates{}`, `lookup_references{}`, `aliases[]` is unchanged. |
| Changes the Neo4j catalog? | Yes, additive: three new relationship types. No label, property, constraint, or index changes. |
| Catalog version | Stays 1.0.0 (decision, §4). |
| Repos touched | `codeanalyzer-schema` (catalog, semantic checker, fixtures); `codeanalyzer-iac` (embedded catalog copy, projector, accepted-edge tests). |
| SDKs touched | None. No SDK consumes the IaC catalog yet. |
| Docs touched | None. The docs site has no IaC pages yet; the analyzer README's graph section is updated in the analyzer child. |

## 3. The change

Three identity-only relationship types, mirroring the existing `IAC_HAS_VALUE_REFERENCE` entry
(`from`, `to`, empty `properties`):

| Type | From | To | One edge per |
| --- | --- | --- | --- |
| `IAC_HAS_RESOURCE_TEMPLATE` | `HelmTemplate` | `HelmResourceTemplate` | entry in the template facet's `resource_templates{}` |
| `IAC_HAS_LOOKUP_REFERENCE` | `HelmTemplate` | `HelmLookupReference` | entry in the template facet's `lookup_references{}` |
| `IAC_HAS_ALIAS` | `Artifact` | `IaCAlias` | entry in the owning artifact's `aliases[]` |

Semantics carried forward from §5 of the parent spec without change:

- Identity-only. No properties; ordering and provenance stay on nodes.
- Producer-owned `IAC_*` relationships: written every generation, removed by `--eager` for the
  selected application generation, never touched for foreign producers.
- Present from L1 onward and monotone through L2 and L3, because the child nodes are L1 source
  facts (`IaCAlias` from chart detection, the two template children from template parsing).
- Exactly one edge per nested child; the semantic checker enforces the counterpart both ways
  (edge without child, child without edge).
- `IAC_ALIAS_OF` is unchanged. An alias therefore has one inbound owner edge and one outbound
  target edge. Aliases owned by a `KubernetesResourceAddress` have no JSON home today; if one is
  added later, `IAC_HAS_ALIAS.from` is extended, not replaced.

## 4. Decisions

| # | Decision | Alternatives considered |
| --- | --- | --- |
| D1 | Per-kind names `IAC_HAS_RESOURCE_TEMPLATE`, `IAC_HAS_LOOKUP_REFERENCE`, `IAC_HAS_ALIAS`. | One generic `IAC_CONTAINS`: fewer types, but breaks the per-child pattern and makes counterpart checks label-dependent. |
| D2 | Include `IAC_HAS_ALIAS` so every owned contained node has an inbound edge from its owner. | Two edges only: closes the observed islands but leaves alias nodes reachable only from themselves. |
| D3 | Catalog `schema_version` stays 1.0.0. | Bump to 1.1.0 as an additive minor. Rejected: the analyzer pins by commit and byte-compares; the version field is not yet consumed anywhere. |
| D4 | Analyzer change lands before the `v0.1.0` tag. | Ship in 0.1.1: avoided because no release exists yet and the change is additive. |

Out of scope, deliberately: the single-region provenance rule in `codeanalyzer-iac`. Recovering
`IAC_DERIVED_FROM` for multi-document template files soundly means rendering per source region,
which is a separate design question. With the containment edges in place the multi-document
resource templates are reachable from their file even without provenance.

## 5. Work per repository

### `codeanalyzer-schema` (codellm-devkit/codeanalyzer-schema#2)

- `v2/iac/neo4j/schema.neo4j.json`: add the three `relationship_types` entries; `schema_version`
  unchanged.
- `scripts/check_iac.py`: extend the containment invariant that today covers
  `value_references` (`_require_edge(..., "iac_has_value_reference", ...)`) to
  `resource_templates`, `lookup_references`, and `aliases`, in both directions; extend the
  projected-label endpoint check so the new types accept only the families above.
- `tests/test_check_iac.py`: one positive fixture and one negative fixture (missing edge,
  dangling edge, wrong endpoint label) per new type.
- Conformance fixtures under `v2/iac/json` regenerated or hand-edited to carry the edges.
- Accept and record the new schema commit on the issue.

### `codeanalyzer-iac` (child filed when picked up)

- `make sync-schema` from the accepted commit; `schema.neo4j.json` and
  `internal/contract/schema.neo4j.json` byte-identical (`make schema-check`).
- `internal/model/base.go`: three `Relationship` constants and accepted-edge registration.
- `internal/dialects/helm/templates.go`: emit `IaCHasResourceTemplate` and
  `IaCHasLookupReference` edges alongside the existing `IaCHasValueReference` emission at L1.
- `internal/dialects/helm/chart.go`: emit `IaCHasAlias` from the chart artifact to its alias where
  the alias patch is built.
- `internal/model/validate.go`: counterpart invariants for the three new edges, matching the
  `IaCHasValueReference` rule.
- `internal/emit/neo4j/project.go`: nothing beyond the catalog allowlist parse if the projector
  already projects all accepted edges; add the three types to the accepted-edge test helper.
- Tests: L1 fixture assertions, monotonicity across L1–L3, graph parity, and a DayTrader live
  assertion that every `HelmResourceTemplate` node has exactly one inbound
  `IAC_HAS_RESOURCE_TEMPLATE` edge.
- README graph section: list the three edges; `.claude/SCHEMA_DECISIONS.md`: record D1–D4.

## 6. Release plan

1. Schema PR merges; commit recorded on codeanalyzer-schema#2 (replaces `e127901` as the pin).
2. Analyzer child re-pins, emits, tests; merges to `main` after codeanalyzer-iac#2.
3. The release pipeline work item (codeanalyzer-iac#3) tags `v0.1.0` from a `main` that already
   contains this change. No lockstep with any SDK is required.

## 7. Tracking record

- Epic: codellm-devkit/.github#52 (existing; both children attach as native sub-issues).
- Schema work item: codellm-devkit/codeanalyzer-schema#2 (existing; links this spec).
- Analyzer work item: filed on `codeanalyzer-iac` when picked up, after step 1 above.
