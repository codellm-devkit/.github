# Helm-First IaC Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `codeanalyzer-iac` 0.1.0 as one typed IaC backend that ingests arbitrary filesystem selections or existing Neo4j artifacts and emits Helm L1–L3 analysis with exact JSON/Neo4j parity.

**Architecture:** A typed Go orchestrator inventories immutable source artifacts, runs an ordered compiled dialect registry, applies typed deltas to one application model, and projects that model to JSON or Neo4j. Helm is the first in-process frontend; the frontend boundary can later be backed by versioned native-runtime workers without changing artifact identity or introducing a runtime plugin ABI now.

**Tech Stack:** Go 1.26.0, Cobra 1.10.2, Helm Go SDK 4.2.4, Neo4j Go Driver 5.28.4, goccy/go-yaml 1.19.2, santhosh-tekuri/jsonschema/v6 6.0.2, Kubernetes apimachinery 0.36.1, Go `text/template/parse`, Go fuzzing.

**Spec:** `../.github/docs/design/specs/2026-09-02-codeanalyzer-iac-helm.md`

## Global Constraints

- Create one repository, `codellm-devkit/codeanalyzer-iac`; do not create per-dialect repositories.
- The Helm-first orchestrator is Go 1.26.0. Future Terraform/OpenTofu, Ansible, Docker/Compose, Packer, Kubernetes/Kustomize, CloudFormation/SAM, ARM/Bicep, Pulumi/CDKs, Chef/Puppet/Salt, Nomad, and Vagrant frontends may be Go components or native-runtime worker adapters behind the same typed delta boundary.
- Pin `helm.sh/helm/v4` to 4.2.4. Do not shell out to `helm`, contact a cluster, enable DNS, load plugins, fetch dependencies, access registries, or perform install/upgrade.
- The executable is `caniac`; `codeanalyzer-iac` is an accepted Cobra alias and the repository/analyzer name.
- Positional filesystem inputs default to `.` and are selection filters beneath one `--workspace-root`; a single Neo4j/Bolt URI positional switches to graph input. Never assume an `infra/` directory.
- `--app-name` is mandatory in graph mode. Filesystem mode derives it from the workspace-root directory only when the flag is absent.
- Graph mode reads every `can://artifact/<app>/...` node page by page, requires complete string `source`, and verifies lowercase SHA-256 before semantic analysis. It never falls back to the host filesystem.
- Unknown files stay raw `Artifact` objects. Detection adds at most one IaC dialect facet to a file and never replaces `can://artifact/...` identity.
- Whole-file semantics are labels/facets on the Artifact. Contained source nodes use `can://iac/...` IDs and spans; there is no emitted `IaCEntity`.
- JSON honors `--analysis-level 1|2|3`; Neo4j and Cypher always analyze/project L3. L4 is an explicit error.
- All collection keys, IDs, rows, and diagnostics are deterministic. Output for `--jobs 1` and `--jobs N` must be byte-identical.
- `Artifact.source` is the only raw source copy. Generated Secret data stores key names and value hashes only.
- The default graph operation is non-destructive upsert. `--eager` removes only producer-owned semantic nodes, IaC aliases, IaC labels/properties, and `IAC_*` relationships for the selected application generation.
- `HAS_ARTIFACT` and `DEFINES_CONFIG` are shared, upsert-only relationships. Never delete an Artifact, ConfigKey, Package, foreign label/property, or foreign relationship.
- All relationships are identity-only. Relationship ordering, resolution reason, layer ordinal, and provenance live on nodes.
- Use test-first cycles and do not advance from L1 to L2 or L2 to L3 while the current conformance gate is red.
- Treat the DayTrader and Quarkus Coffee Shop repositories as mandatory live acceptance corpora. Tests clone the pinned commits over the network, analyze each repository root, validate JSON against the accepted schema and semantic checker, compare rendered resource identities with Helm 4.2.4, and prove filesystem/Neo4j graph-input parity. The production analyzer still never shells out to Helm or accesses the network.
- Pinned live refs are `sample-daytrader/sample.daytrader.microservices@8a68b59430a94a242c54384763da9eb7682728b4` and `quarkuscoffeeshop/quarkuscoffeeshop-helm@aa3c842658e0fc7e44fa25132d8b817eab225cbe`. A scheduled `head` lane resolves each default branch and intentionally fails on semantic drift until expectations are reviewed; it never rewrites expectations automatically.

---

## File Structure

Create this repository layout; every file has one responsibility:

```text
codeanalyzer-iac/
├── .claude/SCHEMA_DECISIONS.md
├── .github/workflows/ci.yml
├── .github/workflows/live.yml
├── .gitignore
├── cmd/codeanalyzer-iac/main.go
├── internal/cli/root.go
├── internal/cli/root_test.go
├── internal/options/options.go
├── internal/model/base.go
├── internal/model/helm.go
├── internal/model/kubernetes.go
├── internal/model/delta.go
├── internal/model/identity.go
├── internal/model/identity_test.go
├── internal/model/validate.go
├── internal/model/validate_test.go
├── internal/contract/embed.go
├── internal/contract/embed_test.go
├── internal/contract/schema.json
├── internal/contract/schema.neo4j.json
├── internal/ingest/source.go
├── internal/ingest/filesystem/source.go
├── internal/ingest/filesystem/source_test.go
├── internal/ingest/neo4j/source.go
├── internal/ingest/neo4j/source_test.go
├── internal/dialect/frontend.go
├── internal/dialect/registry.go
├── internal/dialect/registry_test.go
├── internal/dialects/helm/detect.go
├── internal/dialects/helm/detect_test.go
├── internal/dialects/helm/chart.go
├── internal/dialects/helm/chart_test.go
├── internal/dialects/helm/values.go
├── internal/dialects/helm/values_test.go
├── internal/dialects/helm/templates.go
├── internal/dialects/helm/templates_test.go
├── internal/dialects/helm/resolve.go
├── internal/dialects/helm/resolve_test.go
├── internal/dialects/helm/config.go
├── internal/dialects/helm/config_test.go
├── internal/dialects/helm/render.go
├── internal/dialects/helm/render_test.go
├── internal/dialects/helm/kubernetes.go
├── internal/dialects/helm/kubernetes_test.go
├── internal/core/analyzer.go
├── internal/core/analyzer_test.go
├── internal/emit/json/write.go
├── internal/emit/json/write_test.go
├── internal/emit/neo4j/rows.go
├── internal/emit/neo4j/project.go
├── internal/emit/neo4j/project_test.go
├── internal/emit/neo4j/catalog.go
├── internal/emit/neo4j/catalog_test.go
├── internal/emit/neo4j/cypher.go
├── internal/emit/neo4j/cypher_test.go
├── internal/reconcile/reconcile.go
├── internal/reconcile/reconcile_test.go
├── internal/reconcile/bolt.go
├── internal/reconcile/bolt_integration_test.go
├── testdata/helm/l1-v2/
├── testdata/helm/l1-v1/
├── testdata/helm/nested/
├── testdata/helm/profiles/
├── testdata/helm/render-failures/
├── testdata/helm/security/
├── tests/live/acceptance_test.go
├── tests/live/graph_test.go
├── tests/live/harness_test.go
├── tests/live/repositories.json
├── schema.json
├── schema.neo4j.json
├── LICENSE
├── Makefile
├── README.md
├── go.mod
└── go.sum
```

`internal/model` owns wire names and contracts. Dialects return `model.Delta`; they never marshal JSON, generate Cypher, or call Neo4j. Emitters consume only a fully validated `model.Analysis`.

---

### Task 1: Create the repository, pin the contract, and expose the CLI shell

**Files:**
- Create: `codeanalyzer-iac/go.mod`
- Create: `codeanalyzer-iac/cmd/codeanalyzer-iac/main.go`
- Create: `codeanalyzer-iac/internal/options/options.go`
- Create: `codeanalyzer-iac/internal/cli/root.go`
- Create: `codeanalyzer-iac/internal/cli/root_test.go`
- Create: `codeanalyzer-iac/internal/contract/embed.go`
- Create: `codeanalyzer-iac/internal/contract/embed_test.go`
- Create: `codeanalyzer-iac/schema.json`
- Create: `codeanalyzer-iac/schema.neo4j.json`
- Create: `codeanalyzer-iac/.claude/SCHEMA_DECISIONS.md`
- Create: `codeanalyzer-iac/README.md`
- Create: `codeanalyzer-iac/Makefile`

**Interfaces:**
- Consumes: the accepted `codeanalyzer-schema` commit and its `v2/iac/json/analysis.schema.json` and `v2/iac/neo4j/schema.neo4j.sample.json`.
- Produces: `options.Options`, `options.InputMode`, `cli.Runner`, `cli.New(version string, run cli.Runner) *cobra.Command`, embedded `contract.AnalysisSchema` and `contract.Neo4jSchema`.

- [ ] **Step 1: Create and clone the empty repository**

From `/Users/rkrsn/workspace/codellm-devkit`, run:

```bash
gh repo create codellm-devkit/codeanalyzer-iac --public --clone --description "Unified infrastructure-as-code analyzer backend for CLDK"
cd codeanalyzer-iac
git switch -c feat/helm-first-backend
gh issue create --repo codellm-devkit/codeanalyzer-iac \
  --title "Ship the Helm-first unified IaC backend" \
  --body "Child of codellm-devkit/.github#52. Implements the accepted .github/docs/design/specs/2026-09-02-codeanalyzer-iac-helm.md contract through Helm L3, with filesystem and Neo4j inputs plus JSON/Neo4j parity."
```

Expected: `git remote -v` names `codellm-devkit/codeanalyzer-iac`, the current branch is `feat/helm-first-backend`, and a child issue URL is printed for the eventual pull request body.

- [ ] **Step 2: Initialize the pinned Go module and copy the accepted schemas**

```bash
go mod init github.com/codellm-devkit/codeanalyzer-iac
go mod edit -go=1.26.0
go mod edit -require=github.com/spf13/cobra@v1.10.2
go mod edit -require=github.com/neo4j/neo4j-go-driver/v5@v5.28.4
go mod edit -require=github.com/goccy/go-yaml@v1.19.2
go mod edit -require=github.com/google/go-cmp@v0.7.0
go mod edit -require=github.com/Masterminds/sprig/v3@v3.3.0
go mod edit -require=github.com/santhosh-tekuri/jsonschema/v6@v6.0.2
go mod edit -require=helm.sh/helm/v4@v4.2.4
go mod edit -require=k8s.io/apimachinery@v0.36.1
go mod edit -require=golang.org/x/sync@v0.22.0
cp ../codeanalyzer-schema/v2/iac/json/analysis.schema.json schema.json
cp ../codeanalyzer-schema/v2/iac/neo4j/schema.neo4j.sample.json schema.neo4j.json
cp ../codeanalyzer-go/LICENSE LICENSE
go mod download
```

Expected: `go.mod` says `go 1.26.0` and resolves Helm exactly at `v4.2.4`.

Create `.gitignore` with `/caniac`, `/analysis.json`, `/graph.cypher`, `/coverage.out`, and `.DS_Store`.

- [ ] **Step 3: Write failing CLI and embed tests**

```go
func TestGraphURIRequiresAppName(t *testing.T) {
	cmd := New("0.1.0", runnerFunc(func(context.Context, options.Options) error { return nil }))
	cmd.SetArgs([]string{"neo4j://localhost:7687"})
	err := cmd.Execute()
	if err == nil || !strings.Contains(err.Error(), "--app-name is required in graph mode") {
		t.Fatalf("got %v", err)
	}
}

func TestFilesystemDefaultsToDot(t *testing.T) {
	var got options.Options
	cmd := New("0.1.0", runnerFunc(func(_ context.Context, opts options.Options) error { got = opts; return nil }))
	cmd.SetArgs([]string{"--app-name", "payments"})
	if err := cmd.Execute(); err != nil { t.Fatal(err) }
	if diff := cmp.Diff([]string{"."}, got.Inputs); diff != "" { t.Fatal(diff) }
}

func TestEmbeddedSchemasAreValidJSON(t *testing.T) {
	for name, data := range map[string][]byte{"analysis": AnalysisSchema, "neo4j": Neo4jSchema} {
		if !json.Valid(data) { t.Fatalf("%s schema is invalid", name) }
	}
}

type runnerFunc func(context.Context, options.Options) error
func (f runnerFunc) Run(ctx context.Context, opts options.Options) error { return f(ctx, opts) }
```

Production and tests share this boundary:

```go
type Runner interface {
	Run(context.Context, options.Options) error
}
```

- [ ] **Step 4: Run tests to verify packages are missing**

Run: `go test ./internal/cli ./internal/contract`

Expected: FAIL because the packages and exported functions do not exist.

- [ ] **Step 5: Define options and mode validation**

```go
type InputMode string
const (
	FilesystemMode InputMode = "filesystem"
	GraphMode InputMode = "neo4j"
)

type EmitMode string
const (
	EmitAuto EmitMode = "auto"
	EmitJSON EmitMode = "json"
	EmitCypher EmitMode = "cypher"
	EmitNeo4j EmitMode = "neo4j"
	EmitSchema EmitMode = "schema"
)

type Options struct {
	Inputs []string
	Mode InputMode
	WorkspaceRoot string
	AppName string
	Config string
	AnalysisLevel int
	AnalysisLevelSet bool
	Jobs int
	Format string
	Emit EmitMode
	OutputDir string
	Eager bool
	Strict bool
	Neo4jURI string
	Neo4jUser string
	Neo4jPassword string
	Neo4jDatabase string
}
```

Implement `func (o Options) Validate() error` with these exact rules:

- no args becomes filesystem input `.`, except `--emit schema` and `--version`, which short-circuit without an input;
- exactly one URI with scheme `neo4j|neo4j+s|neo4j+ssc|bolt|bolt+s|bolt+ssc` selects graph mode;
- URI and filesystem paths cannot be mixed;
- graph mode requires app name and forbids workspace root; filesystem mode derives a missing app name from the canonical workspace-root basename before Source construction;
- level accepts 1–3 only; Neo4j/Cypher force level 3 and reject an explicitly supplied lower level, using `AnalysisLevelSet` populated from `cmd.Flags().Changed("analysis-level")`;
- format accepts `json`; `msgpack` returns `msgpack output is not yet implemented; use --format json`;
- auto emit resolves to JSON for filesystem and direct Neo4j for graph input;
- graph-mode `--config` accepts an Artifact ID or app-relative path; filesystem config validation is deferred until the workspace root is canonicalized.

- [ ] **Step 6: Implement the Cobra shell and embedded contracts**

`cli.New` uses `Use:"caniac [PATH ...]"`, `Aliases:[]string{"codeanalyzer-iac"}`, `Args:cobra.ArbitraryArgs`, `SilenceUsage:true`, and binds `--workspace-root`, `--app-name`, `--config`, `-a/--analysis-level`, `-j/--jobs`, `-o/--output`, `-f/--format`, `--emit`, `--eager`, `--strict`, and Neo4j credential flags. Precedence is explicit flag, then `NEO4J_URI|NEO4J_USERNAME|NEO4J_PASSWORD|NEO4J_DATABASE`, then defaults.

Use `//go:embed` without duplicating schema bytes:

```go
//go:embed schema.json
var AnalysisSchema []byte

//go:embed schema.neo4j.json
var Neo4jSchema []byte
```

Place `embed.go` and generated copies under `internal/contract`, because Go embed cannot use `..`. Keep the authoritative files at repository root and have `Makefile sync-schema` copy them to `internal/contract/schema.json` and `internal/contract/schema.neo4j.json`; test byte equality between root and embedded copies.

- [ ] **Step 7: Add a main that keeps stdout data-only**

```go
var version = "0.1.0-dev"

type bootstrapRunner struct{}
func (bootstrapRunner) Run(context.Context, options.Options) error {
	return errors.New("analysis pipeline is not wired")
}

func main() {
	if err := cli.New(version, bootstrapRunner{}).Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

Tests inject `runnerFunc` and do not call `bootstrapRunner`. Task 11 deletes `bootstrapRunner` and passes the production runner before any end-to-end gate.

- [ ] **Step 8: Pass tests and record schema decisions**

Run:

```bash
make sync-schema
go test ./internal/cli ./internal/contract
go vet ./...
```

Expected: PASS. In `.claude/SCHEMA_DECISIONS.md`, record schema 2.0.0, graph schema 1.0.0, Artifact facet identity, absence of `IaCEntity`, one-source-dialect rule, and accepted `codeanalyzer-schema` commit.

- [ ] **Step 9: Commit the repository foundation**

```bash
git add .
git commit -m "feat: bootstrap unified IaC analyzer"
```

---

### Task 2: Implement typed base models, identities, deltas, and validation

**Files:**
- Create: `codeanalyzer-iac/internal/model/base.go`
- Create: `codeanalyzer-iac/internal/model/helm.go`
- Create: `codeanalyzer-iac/internal/model/kubernetes.go`
- Create: `codeanalyzer-iac/internal/model/delta.go`
- Create: `codeanalyzer-iac/internal/model/identity.go`
- Create: `codeanalyzer-iac/internal/model/identity_test.go`
- Create: `codeanalyzer-iac/internal/model/validate.go`
- Create: `codeanalyzer-iac/internal/model/validate_test.go`

**Interfaces:**
- Consumes: embedded analysis schema from Task 1.
- Produces: all wire model structs, `ArtifactFacet`, `ValueFacet`, `Delta`, `NewApplication(appName string,artifacts map[string]*Artifact) *Application`, `NewAnalysis(level int,app *Application) *Analysis`, `Apply(*Application,Delta) error`, `ArtifactID(app,path string) (string,error)`, `SemanticID(app,dialect string,segments ...string) string`, `AnonymousID(parent,kind string,line,column int) string`, `ConfigKeyID(artifactID,path string) string`, `Validate(*Application) error`, and `AllNodes(*Application) map[string]Node`.

- [ ] **Step 1: Write identity property tests first**

```go
func TestArtifactIDEncodesSegmentsAndNormalizesSeparators(t *testing.T) {
	got, err := ArtifactID("pay ments", `charts\api values\Chart.yaml`)
	if err != nil { t.Fatal(err) }
	want := "can://artifact/pay%20ments/charts/api%20values/Chart.yaml"
	if got != want { t.Fatalf("got %q want %q", got, want) }
}

func TestArtifactIDRejectsTraversal(t *testing.T) {
	for _, path := range []string{"../Chart.yaml", "/tmp/Chart.yaml", "charts/../Chart.yaml"} {
		if _, err := ArtifactID("payments", path); err == nil { t.Fatalf("accepted %q", path) }
	}
}

func TestAnonymousIDIncludesLineAndColumn(t *testing.T) {
	got := AnonymousID("can://iac/payments/helm/chart", "task", 42, 3)
	if got != "can://iac/payments/helm/chart/task@42:3" { t.Fatal(got) }
}
```

- [ ] **Step 2: Run the identity tests to verify failure**

Run: `go test ./internal/model -run 'TestArtifactID|TestAnonymousID'`

Expected: FAIL with undefined functions.

- [ ] **Step 3: Implement canonical IDs and byte spans**

Use `url.PathEscape` once per already-normalized segment, never over the complete path. `normalizeRelativePath` converts backslashes to `/`, rejects absolute paths, empty/`.`/`..` segments, NUL, and cleaned path changes.

```go
type Span struct {
	Start [2]int `json:"start"`
	End [2]int `json:"end"`
	Bytes [2]int `json:"bytes"`
}

func SemanticID(app, dialect string, segments ...string) string {
	parts := []string{"can://iac", encodeSegment(app), encodeSegment(dialect)}
	for _, segment := range segments { parts = append(parts, encodeSegment(segment)) }
	return strings.Join(parts, "/")
}
```

- [ ] **Step 4: Define the typed model without an `IaCEntity`**

```go
type Node interface { NodeID() string; NodeKind() string }
type ArtifactFacet interface {
	NodeKind() string
	DialectName() string
	isArtifactFacet()
}
type ValueFacet interface { NodeKind() string; isValueFacet() }

type Analysis struct {
	SchemaVersion string `json:"schema_version"`
	Language string `json:"language"`
	MaxLevel int `json:"max_level"`
	Analyzer Analyzer `json:"analyzer"`
	Application *Application `json:"application"`
}

type Application struct {
	ID string `json:"id"`
	Kind string `json:"kind"`
	Artifacts map[string]*Artifact `json:"artifacts"`
	Packages map[string]*Package `json:"packages"`
	ExternalChartReferences map[string]*HelmChartReference `json:"external_chart_references"`
	KubernetesResourceAddresses map[string]*KubernetesResourceAddress `json:"kubernetes_resource_addresses"`
	Diagnostics map[string]*Diagnostic `json:"diagnostics"`
	Edges map[Relationship]map[string]Edge `json:"edges"`
}

type Artifact struct {
	ID string `json:"id"`
	Kind string `json:"kind"`
	Path string `json:"path"`
	Format string `json:"format"`
	SHA256 string `json:"sha256"`
	Source string `json:"source"`
	SizeBytes int64 `json:"size_bytes"`
	ConfigKeys map[string]*ConfigKey `json:"config_keys"`
	Aliases []IdentityAlias `json:"aliases"`
	IaC ArtifactFacet `json:"iac,omitempty"`
	CodeAnalyzerIaCConfig *CodeAnalyzerIaCConfig `json:"codeanalyzer_iac_config,omitempty"`
}
```

Define each Helm and Kubernetes struct to match the accepted `schema.json` exactly. Named child collections are explicit struct fields (`Dependencies`, `NamedTemplates`, `TemplateCalls`, `ValueReferences`, `ResourceTemplates`, `LookupReferences`, `Renders`, `RenderProfiles`, `ValueLayers`, `Diagnostics`, `Resources`), not `map[string]Node` or `entities`.

- [ ] **Step 5: Write failing delta and validation tests**

Test that `Apply` rejects a patch for a missing artifact, a second dialect facet, a duplicate canonical ID, a dangling edge, an alias chain, an invalid span, and a bad Artifact digest. Test that applying the same delta twice is idempotent.

```go
func TestApplyRejectsSecondDialect(t *testing.T) {
	app := fixtureApplication()
	app.Artifacts["Chart.yaml"].IaC = &HelmChart{Kind:"helm_chart", Dialect:"helm"}
	err := Apply(app, Delta{ArtifactPatches: map[string]ArtifactPatch{
		app.Artifacts["Chart.yaml"].ID: {Facet: fakeTerraformFacet{}},
	}})
	if !errors.Is(err, ErrDialectConflict) { t.Fatalf("got %v", err) }
}

type fakeTerraformFacet struct{}
func (fakeTerraformFacet) NodeKind() string { return "terraform_module" }
func (fakeTerraformFacet) DialectName() string { return "terraform" }
func (fakeTerraformFacet) isArtifactFacet() {}
```

- [ ] **Step 6: Implement delta application and whole-model validation**

```go
type ArtifactPatch struct {
	Facet ArtifactFacet
	ConfigFacet *CodeAnalyzerIaCConfig
	ConfigKeys map[string]*ConfigKey
	Aliases []IdentityAlias
}

type Delta struct {
	ArtifactPatches map[string]ArtifactPatch
	Packages map[string]*Package
	ExternalChartReferences map[string]*HelmChartReference
	KubernetesResourceAddresses map[string]*KubernetesResourceAddress
	Edges map[Relationship]map[string]Edge
	Diagnostics map[string]*Diagnostic
}
```

`NewApplication(appName,artifacts)` initializes every named root map and creates one deterministic `has_artifact` edge per Artifact. Parsers create the remaining containment edges, including `defines_config` for ConfigKeys and `iac_alias_of` for each alias, at the same time as their nodes.

`Apply` merges only absent/equal entries, sorts aliases, and returns a typed conflict for different facts at one key. `Validate` checks every non-empty source digest, forbids semantic facets on empty-source Artifacts, checks ID uniqueness, spans against UTF-8 byte length, every edge endpoint, alias single-target/canonical rules, one source dialect, render/profile references, and Secret hash-only data. `AllNodes` recursively walks every explicit named collection.

- [ ] **Step 7: Validate Go JSON against the embedded schema**

Compile `schema.json` once with `jsonschema.NewCompiler()`. Add a test that marshals a complete typed fixture, unmarshals into `any`, and calls the compiled schema's `Validate` method. Assert `IaC == nil` omits the key and no JSON key contains uppercase letters.

Run: `go test ./internal/model ./internal/contract`

Expected: PASS.

- [ ] **Step 8: Commit typed model and invariants**

```bash
git add internal/model internal/contract
git commit -m "feat: add typed IaC analysis model"
```

---

### Task 3: Inventory arbitrary filesystem selections safely

**Files:**
- Create: `codeanalyzer-iac/internal/ingest/source.go`
- Create: `codeanalyzer-iac/internal/ingest/filesystem/source.go`
- Create: `codeanalyzer-iac/internal/ingest/filesystem/source_test.go`
- Create: `codeanalyzer-iac/testdata/ingest/workspace/README.md`
- Create: `codeanalyzer-iac/testdata/ingest/workspace/charts/api/Chart.yaml`
- Create: `codeanalyzer-iac/testdata/ingest/workspace/charts/api/values.yaml`

**Interfaces:**
- Consumes: `model.Artifact`, `model.Diagnostic`, `model.ArtifactID`, and validated filesystem `options.Options`.
- Produces: `ingest.Source`, `ingest.Result`, `filesystem.New(appName,root string,inputs []string,config string) (*Source,error)`, and `(*Source).Load(context.Context) (ingest.Result,error)`.

- [ ] **Step 1: Write selection, deduplication, and escape tests**

```go
func TestOverlappingInputsProduceOneArtifactPerPath(t *testing.T) {
	s, err := New("payments", fixtureRoot(t), []string{"charts", "charts/api/Chart.yaml"}, "")
	if err != nil { t.Fatal(err) }
	got, err := s.Load(context.Background())
	if err != nil { t.Fatal(err) }
	if _, ok := got.Artifacts["charts/api/Chart.yaml"]; !ok { t.Fatal("missing chart") }
	if len(got.Artifacts) != 2 { t.Fatalf("got %d artifacts", len(got.Artifacts)) }
}

func TestConfigOutsideSelectionIsInventoried(t *testing.T) {
	s, err := New("payments", fixtureRoot(t), []string{"charts/api"}, ".codeanalyzer-iac.yaml")
	if err != nil { t.Fatal(err) }
	got, err := s.Load(context.Background())
	if err != nil { t.Fatal(err) }
	if _, ok := got.Artifacts[".codeanalyzer-iac.yaml"]; !ok { t.Fatal("config not inventoried") }
}
```

Also create table tests for absolute input outside root, `..`, symlink escape, unreadable file, CRLF preservation, invalid UTF-8, empty text, selecting one file directly, and excluding root VCS administration directories such as `.git` from a whole-repository selection.

- [ ] **Step 2: Run tests to verify the filesystem source is absent**

Run: `go test ./internal/ingest/filesystem`

Expected: FAIL with undefined `New`.

- [ ] **Step 3: Define the source boundary**

```go
type Result struct {
	Artifacts map[string]*model.Artifact
	Diagnostics map[string]*model.Diagnostic
}

type Source interface {
	Load(context.Context) (Result, error)
}
```

- [ ] **Step 4: Implement workspace-root identity and safe walking**

Resolve root and each selection with `filepath.Abs` plus `filepath.EvalSymlinks`, then require `filepath.Rel(root,resolved)` to be neither `..` nor `../...`. Walk directories without following symlinked directories and skip VCS administration directories (`.git`, `.hg`, `.svn`) rather than ingesting checkout internals. Dedupe on normalized app-relative POSIX path, sort paths, read raw bytes once, reject non-UTF-8 source with a diagnostic, and construct:

```go
artifact := &model.Artifact{
	ID: id,
	Kind: "artifact",
	Path: rel,
	Format: detectFormat(rel, source),
	SHA256: fmt.Sprintf("%x", sha256.Sum256(raw)),
	Source: string(raw),
	SizeBytes: int64(len(raw)),
	ConfigKeys: map[string]*model.ConfigKey{},
	Aliases: []model.IdentityAlias{},
}
```

Unsupported text formats are retained. Binary/non-UTF-8 files, including packaged `.tgz` charts, are retained as raw Artifacts with complete `sha256`/`size_bytes`, `source:""`, and `IAC_SOURCE_NOT_TEXT`; they are ineligible for semantic parsing or reconstruction. Text files always carry complete source, never a truncated prefix.

- [ ] **Step 5: Pass focused and race tests**

Run:

```bash
go test ./internal/ingest/filesystem
go test -race ./internal/ingest/filesystem
```

Expected: PASS.

- [ ] **Step 6: Commit filesystem ingestion**

```bash
git add internal/ingest testdata/ingest
git commit -m "feat: inventory arbitrary filesystem inputs"
```

---

### Task 4: Stream and verify prepopulated Neo4j artifacts

**Files:**
- Create: `codeanalyzer-iac/internal/ingest/neo4j/source.go`
- Create: `codeanalyzer-iac/internal/ingest/neo4j/source_test.go`

**Interfaces:**
- Consumes: `ingest.Result`, `model.Artifact`, graph-mode options.
- Produces: `neo4jingest.Queryer`, `neo4jingest.New(queryer Queryer, appName string, pageSize int) *Source`, and `(*Source).Load(context.Context) (ingest.Result,error)`.

- [ ] **Step 1: Write fake-query tests for paging and source eligibility**

```go
type Queryer interface {
	ReadArtifacts(ctx context.Context, prefix, afterID string, limit int) ([]ArtifactRow, error)
	ListApplications(ctx context.Context) ([]string, error)
}

type ArtifactRow struct {
	ID string
	Path string
	Format string
	Source any
	SHA256 string
	SizeBytes int64
	Foreign map[string]any
}

func TestLoadPagesByArtifactID(t *testing.T) {
	q := &fakeQueryer{pages: [][]ArtifactRow{{row("a")}, {row("b")}, {}}}
	got, err := New(q, "payments", 1).Load(context.Background())
	if err != nil { t.Fatal(err) }
	if len(got.Artifacts) != 2 { t.Fatalf("got %d", len(got.Artifacts)) }
	if diff := cmp.Diff([]string{"", q.rows[0].ID, q.rows[1].ID}, q.after); diff != "" { t.Fatal(diff) }
}
```

Add exact tests for missing `source`, non-string source, bad hash, duplicate path/ID, app prefix escape, zero matching artifacts, preserving foreign properties in `ArtifactRow.Foreign`, and config selection by full Artifact ID or app-relative path.

- [ ] **Step 2: Run tests to verify failure**

Run: `go test ./internal/ingest/neo4j`

Expected: FAIL with undefined `New` or `ArtifactRow`.

- [ ] **Step 3: Implement cursor paging and hash verification**

Use this query contract in the real adapter:

```cypher
MATCH (a:Artifact)
WHERE a.id STARTS WITH $prefix AND a.id > $after_id
RETURN a.id AS id, a.path AS path, a.format AS format,
       a.source AS source, a.sha256 AS sha256, a.size_bytes AS size_bytes,
       properties(a) AS properties
ORDER BY a.id
LIMIT $limit
```

The prefix is `can://artifact/<encoded-app>/`. Verify the ID/path relation and `sha256(source)` before adding an eligible Artifact. On missing/mismatched source, add `IAC_GRAPH_SOURCE_MISSING` or `IAC_GRAPH_SOURCE_HASH_MISMATCH`, represent the input as an ineligible raw Artifact with `source:""`, leave its existing graph properties untouched, and skip dialect parsing later. Never read a filesystem path.

When the prefix has no rows, call `ListApplications` and return an error containing the requested application and the sorted available application names; never select one implicitly.

- [ ] **Step 4: Implement exact graph config selection**

Resolve a full `can://artifact/...` selector directly; otherwise normalize it as an app-relative path and construct the Artifact ID. Require the selected artifact in the loaded page set. An absent selection is fatal `IAC_CONFIG_ARTIFACT_NOT_FOUND`; invalid content is handled by Task 9.

- [ ] **Step 5: Pass graph-source tests**

Run: `go test -race ./internal/ingest/neo4j`

Expected: PASS with no live database.

- [ ] **Step 6: Commit graph ingestion**

```bash
git add internal/ingest/neo4j
git commit -m "feat: ingest verified artifacts from Neo4j"
```

---

### Task 5: Add the compiled frontend registry and Helm detection

**Files:**
- Create: `codeanalyzer-iac/internal/dialect/frontend.go`
- Create: `codeanalyzer-iac/internal/dialect/registry.go`
- Create: `codeanalyzer-iac/internal/dialect/registry_test.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/detect.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/detect_test.go`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/Chart.yaml`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/values.yaml`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/templates/deployment.yaml`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/templates/_helpers.tpl`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/templates/NOTES.txt`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/templates/tests/smoke.yaml`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/crds/widgets.yaml`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/.helmignore`

**Interfaces:**
- Consumes: immutable `model.Artifact` inventory and `model.Delta`.
- Produces: `dialect.Frontend`, `dialect.Detection`, `dialect.Registry`, `Registry.Detect(ArtifactContext) (Detection,error)`, `Registry.DetectAll(*model.Application) (map[string]Detection,model.Delta)`, `Registry.ResolveAll(context.Context,*model.Application) model.Delta`, `helm.New() dialect.Frontend`, and Helm content/context classification.

- [ ] **Step 1: Write registry conflict and deterministic-order tests**

```go
type Frontend interface {
	Name() string
	Detect(ArtifactContext) (Detection, bool, error)
	Parse(context.Context, *model.Artifact, Detection) (model.Delta, error)
	Resolve(context.Context, *model.Application) (model.Delta, error)
	Evaluate(context.Context, *model.Application, EvaluationInput) (model.Delta, error)
}

type ArtifactContext struct {
	Artifact *model.Artifact
	Artifacts map[string]*model.Artifact
}

type Detection struct {
	Dialect string
	Kind string
	Roles []string
	ChartArtifactID string
}

func TestRegistryRejectsAmbiguousDetection(t *testing.T) {
	r := NewRegistry(fakeFrontend("a", true), fakeFrontend("b", true))
	artifact := &model.Artifact{ID:"can://artifact/app/file", Kind:"artifact", Path:"file", Source:"x"}
	_, err := r.Detect(ArtifactContext{Artifact: artifact, Artifacts: map[string]*model.Artifact{"file":artifact}})
	if !errors.Is(err, ErrAmbiguousDialect) { t.Fatalf("got %v", err) }
}
```

Define the test double completely:

```go
type stubFrontend struct { name string; match bool }
func fakeFrontend(name string, match bool) Frontend { return stubFrontend{name:name, match:match} }
func (f stubFrontend) Name() string { return f.name }
func (f stubFrontend) Detect(ArtifactContext) (Detection, bool, error) {
	return Detection{Dialect:f.name, Kind:f.name+"_artifact"}, f.match, nil
}
func (f stubFrontend) Parse(context.Context, *model.Artifact, Detection) (model.Delta, error) {
	return model.Delta{}, nil
}
func (f stubFrontend) Resolve(context.Context, *model.Application) (model.Delta, error) {
	return model.Delta{}, nil
}
func (f stubFrontend) Evaluate(context.Context, *model.Application, EvaluationInput) (model.Delta, error) {
	return model.Delta{}, nil
}
```

- [ ] **Step 2: Write Helm classification cases**

Assert content plus nearest-chart context, including:

- `Chart.yaml` parses as a chart only when `apiVersion` is `v1` or `v2` and required metadata exists;
- YAML outside a chart is not Helm merely because its extension is `.yaml`;
- nearest ancestor Chart wins for nested and vendored charts;
- values inputs explicitly selected by a render profile get override role;
- `_helpers.tpl`, notes, test, hook, CRD, lock, requirements, values schema, and ignore kinds/roles are exact;
- README, LICENSE, packaged `.tgz`, and unrelated files remain raw;
- a file cannot receive two source dialect matches.

- [ ] **Step 3: Run tests to verify missing registry/detector**

Run: `go test ./internal/dialect ./internal/dialects/helm -run 'Registry|Detect'`

Expected: FAIL with undefined registry and Helm frontend.

- [ ] **Step 4: Implement the compiled registry**

Sort frontends by `Name()` in `NewRegistry`, call every detector for one artifact, return zero/one match, and produce `IAC_AMBIGUOUS_DIALECT` with sorted frontend names for multiple matches. Do not use Go `plugin`, dynamic libraries, executable discovery, or runtime registration.

`EvaluationInput` is:

```go
type EvaluationInput struct {
	Artifacts map[string]*model.Artifact
	ConfigArtifactID string
	Jobs int
	TempRoot string
}
```

- [ ] **Step 5: Implement chart-context indexing and classification**

Build an immutable sorted index of valid chart anchors before classifying members. For each path, choose the longest ancestor anchor. Use file basename and relative position only after the anchor is proven from Chart content. Return `Detection{Dialect:"helm",Kind,Roles,ChartArtifactID}`.

- [ ] **Step 6: Pass detector tests**

Run: `go test -race ./internal/dialect ./internal/dialects/helm -run 'Registry|Detect'`

Expected: PASS.

- [ ] **Step 7: Commit registry and detection**

```bash
git add internal/dialect internal/dialects/helm/detect* testdata/helm/l1-v2
git commit -m "feat(helm): detect chart artifacts"
```

---

### Task 6: Parse Helm chart metadata, dependencies, locks, values, schema, CRDs, and ignore rules at L1

**Files:**
- Create: `codeanalyzer-iac/internal/dialects/helm/chart.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/chart_test.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/values.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/values_test.go`
- Create: `codeanalyzer-iac/testdata/helm/l1-v1/Chart.yaml`
- Create: `codeanalyzer-iac/testdata/helm/l1-v1/requirements.yaml`
- Create: `codeanalyzer-iac/testdata/helm/l1-v1/requirements.lock`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/Chart.lock`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/values.schema.json`

**Interfaces:**
- Consumes: Helm `Detection`, typed facets, IDs/spans, and Artifact source from Tasks 2 and 5.
- Produces: `parseChart`, `parseDependencies`, `parseLock`, `parseValues`, `parseValuesSchema`, `parseCRD`, `parseIgnore`, and L1 `model.Delta` from `Frontend.Parse`.

- [ ] **Step 1: Write exact field and source-span tests**

For v2, assert every metadata field: api version, name, SemVer string, kube version, description, application/library type, keywords, home, sources, maintainer name/email/url, icon, app version, deprecated, annotations. Assert dependency name, alias, repository, constraint, condition, tags, string import-value, child/parent import-value, ID, and span.

For v1, assert dependencies come from `requirements.yaml`, not an absent Chart dependency list, and legacy lock role is correct. Reject Chart API v3 with `IAC_HELM_UNSUPPORTED_API_VERSION`.

For nested values, assert exact paths and spans for maps, sequences, quoted keys, dotted literal keys, empty maps/lists, anchors, and aliases. `ConfigKey.iac` must be exactly `HelmValueFacet{Kind:"helm_value"}`.

- [ ] **Step 2: Run the L1 parser tests to verify failure**

Run: `go test ./internal/dialects/helm -run 'Chart|Dependency|Lock|Values|CRD|Ignore'`

Expected: FAIL with undefined parser functions.

- [ ] **Step 3: Parse source-aware YAML and metadata**

Use `parser.ParseBytes(source, parser.ParseComments)` from `github.com/goccy/go-yaml/parser` for locations. Convert parser token offsets to the contract's one-based line/column and half-open byte offsets through one tested `spanOf(token.Token) model.Span` helper. Decode typed metadata with `github.com/goccy/go-yaml` after the AST is accepted.

Preserve Chart versions and dependency constraints as strings. Never normalize away the source spelling used for identity or spans. Produce a partial facet plus `IAC_HELM_YAML_PARSE` diagnostic when confidently parsed children exist before an error.

For every accepted Chart anchor, add `IdentityAlias{ID:can://iac/<app>/helm/chart/<chart-directory>,Kind:"helm_chart",Target:<Chart Artifact ID>}` plus the matching `iac_alias_of` edge. Whole-file chart semantics remain on the Artifact facet; no second chart node is created.

- [ ] **Step 4: Reuse ConfigKey nodes for Helm values**

Walk the YAML AST in source order. Escape path segments so a literal key containing `.` cannot collide with nested `image.tag`. Build `ConfigKeyID(artifact.ID,path)`, store the key's source span, and attach a `HelmValueFacet`; do not copy scalar values into the IaC facet.

- [ ] **Step 5: Parse ancillary typed facets**

- Validate `values.schema.json` with `jsonschema/v6` and emit only the typed validation-schema facet plus diagnostics.
- Parse CRD YAML metadata for kind/name/group/plural facts used later by resource plural lookup, while keeping the whole file one `HelmCRD` artifact.
- Parse `.helmignore` as ordered non-comment patterns on the facet.
- Parse lock entries as typed dependency snapshots; do not invent a `pkg:helm` PURL.

- [ ] **Step 6: Run the Helm L1 non-template gate**

Run:

```bash
go test ./internal/dialects/helm -run 'Chart|Dependency|Lock|Values|CRD|Ignore'
go test -race ./internal/dialects/helm -run 'Chart|Dependency|Lock|Values|CRD|Ignore'
```

Expected: PASS with exact values and spans.

- [ ] **Step 7: Commit non-template Helm L1**

```bash
git add internal/dialects/helm testdata/helm/l1-v1 testdata/helm/l1-v2
git commit -m "feat(helm): parse chart and values source facts"
```

---

### Task 7: Parse Helm template source facts at L1

**Files:**
- Create: `codeanalyzer-iac/internal/dialects/helm/templates.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/templates_test.go`
- Modify: `codeanalyzer-iac/testdata/helm/l1-v2/templates/deployment.yaml`
- Modify: `codeanalyzer-iac/testdata/helm/l1-v2/templates/_helpers.tpl`
- Create: `codeanalyzer-iac/testdata/helm/l1-v2/templates/hook.yaml`

**Interfaces:**
- Consumes: Helm template Artifact/Detection, model template child types, span/anonymous-ID helpers.
- Produces: `parseTemplate(artifact, detection) (*model.HelmTemplate, []model.Diagnostic)` and AST walkers for definitions, calls, value references, resource documents, and lookup references.

- [ ] **Step 1: Write exact template-node tests**

Assert exact names, kinds, source IDs, and spans for:

- `define` and `block` named-template declarations;
- `template`, `include`, `block`, and statically recoverable `tpl` call sites;
- dynamic `tpl` retained without a target name;
- every repeated `.Values.image.tag` use as a distinct `HelmValueReference`;
- chained/indexed value expressions, including an unresolved dynamic index;
- `lookup` group/version/kind/namespace/name expressions without execution;
- YAML document resource-template spans;
- helper, notes, test, hook, and resource roles;
- duplicate definitions retained as separate source nodes and diagnosed.

- [ ] **Step 2: Run template tests to verify failure**

Run: `go test ./internal/dialects/helm -run Template`

Expected: FAIL with undefined `parseTemplate`.

- [ ] **Step 3: Parse Go-template syntax with a complete function-name map**

Use `text/template/parse.Parse` with stub functions from `sprig.TxtFuncMap()` plus Helm names `include`, `tpl`, `required`, `lookup`, `toYaml`, `fromYaml`, `fromYamlArray`, `toToml`, `fromJson`, and `fromJsonArray`. Stubs exist only to let the standard parser accept identifiers; they are never executed.

Walk `ListNode`, `ActionNode`, `IfNode`, `RangeNode`, `WithNode`, `TemplateNode`, `CommandNode`, `FieldNode`, `ChainNode`, and `VariableNode`. Convert each `parse.Pos` byte position with a precomputed newline index; test multibyte UTF-8 before the template action.

- [ ] **Step 4: Emit distinct declarations, calls, values, resources, and lookups**

Named keys are semantic names when unique and `name@line:column` on duplicates. Use `line:column` map keys for call/reference occurrences. A `tpl` string literal is statically recoverable; any other first argument remains an unresolved call node. Resource-template documents are source regions, not decoded Kubernetes objects at L1.

- [ ] **Step 5: Add parse-error degradation**

On one malformed template, retain the Artifact, any definitions from separately parsed template definitions, and an `IAC_HELM_TEMPLATE_PARSE` diagnostic. Mark the facet `partial`; do not abort unrelated templates or charts.

- [ ] **Step 6: Run the complete L1 gate**

Run:

```bash
go test ./internal/dialects/helm -run 'Chart|Values|Template|Detect'
go test -race ./internal/dialects/helm -run 'Chart|Values|Template|Detect'
```

Expected: PASS. Serialize the L1 fixture through `model.Validate` and the embedded schema; assert there are no L2 target IDs or L3 render/profile objects.

- [ ] **Step 7: Commit Helm template L1**

```bash
git add internal/dialects/helm/templates* testdata/helm/l1-v2/templates
git commit -m "feat(helm): parse template source facts"
```

---

### Task 8: Resolve chart membership, dependencies, templates, and values at L2

**Files:**
- Create: `codeanalyzer-iac/internal/dialects/helm/resolve.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/resolve_test.go`
- Create: `codeanalyzer-iac/testdata/helm/nested/Chart.yaml`
- Create: `codeanalyzer-iac/testdata/helm/nested/charts/worker/Chart.yaml`
- Create: `codeanalyzer-iac/testdata/helm/nested/charts/worker/values.yaml`
- Create: `codeanalyzer-iac/testdata/helm/nested/charts/worker/templates/job.yaml`

**Interfaces:**
- Consumes: fully parsed L1 `model.Application`.
- Produces: `resolve(app *model.Application) (model.Delta,error)`, L2 target refinements, `HelmChartReference` nodes, optional `Package` links, and identity-only L2 edges.

- [ ] **Step 1: Write hand-computed resolution tests**

Assert exact edge endpoint pairs for:

- each non-anchor Helm Artifact to its nearest chart;
- v2 Chart dependencies and v1 requirements to declaration nodes;
- external declarations to deterministic `HelmChartReference` IDs;
- aliased vendored dependency reference to the nested Chart Artifact;
- static template/include/block call to a definition in the chart's global template namespace;
- duplicate named templates diagnosed and resolved according to Helm's deterministic load order;
- value reference to the highest-precedence source declaration available at L2 source analysis;
- unresolved dynamic `tpl` and dynamic value references retained without dangling edges.

```go
func assertEdge(t *testing.T, d model.Delta, rel model.Relationship, src, dst string) {
	t.Helper()
	for _, edge := range d.Edges[rel] {
		if edge.Src == src && edge.Dst == dst { return }
	}
	t.Fatalf("missing %s edge %s -> %s", rel, src, dst)
}
```

- [ ] **Step 2: Run L2 tests to verify failure**

Run: `go test ./internal/dialects/helm -run Resolve`

Expected: FAIL with undefined `resolve`.

- [ ] **Step 3: Build deterministic per-chart indexes**

Index artifacts by chart anchor, definitions by name and Helm load order, ConfigKeys by escaped logical path and source role, and vendored charts by dependency alias/name. Sort every candidate list by canonical ID before selecting or diagnosing.

- [ ] **Step 4: Emit reference nodes, refinements, and identity-only edges**

For each dependency declaration, build a stable reference ID from owning chart ID plus dependency key. Add a standard PURL only when a registered PURL type can represent the actual repository; never emit `pkg:helm`. Add target IDs only to previously absent fields. All edge objects remain:

```go
type Edge struct {
	Src string `json:"src"`
	Dst string `json:"dst"`
}
```

- [ ] **Step 5: Prove L1 is an unchanged subset of L2**

Marshal L1 before resolution and L2 after applying its Delta. Use a test helper that permits only added map keys and target refinements from absent to canonical ID. Assert every new edge endpoint exists in `model.AllNodes`.

Run: `go test ./internal/dialects/helm -run Resolve`

Expected: PASS.

- [ ] **Step 6: Commit Helm L2**

```bash
git add internal/dialects/helm/resolve* testdata/helm/nested
git commit -m "feat(helm): resolve cross-file chart facts"
```

---

### Task 9: Parse optional typed configuration and construct render profiles

**Files:**
- Create: `codeanalyzer-iac/internal/dialects/helm/config.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/config_test.go`
- Create: `codeanalyzer-iac/testdata/helm/profiles/.codeanalyzer-iac.yaml`
- Create: `codeanalyzer-iac/testdata/helm/profiles/Chart.yaml`
- Create: `codeanalyzer-iac/testdata/helm/profiles/values.yaml`
- Create: `codeanalyzer-iac/testdata/helm/profiles/values-production.yaml`
- Create: `codeanalyzer-iac/testdata/helm/profiles/templates/deployment.yaml`

**Interfaces:**
- Consumes: selected config Artifact ID, application Artifact map, ConfigKey/Helm profile/value-layer types.
- Produces: `parseConfig(app, artifactID) (model.Delta,error)`, `defaultProfile(chart) *model.HelmRenderProfile`, `BuildProfiles(app, configArtifactID) (model.Delta,error)`, and typed config/profile/layer facts for L3.

- [ ] **Step 1: Write tests for config's exact graph meaning**

Assert:

- without `--config`, every Chart gets exactly one deterministic `origin:"default"` profile contained by the Chart;
- selecting config attaches `CodeAnalyzerIaCConfig` to the same Artifact, never `Artifact.iac`;
- each named config profile is contained by the config Artifact and targets one Chart;
- `values` order is preserved by zero-padded keys `0000`, `0001` and integer ordinals;
- each literal `set` key becomes a ConfigKey child of the config Artifact and a layer references that ID;
- raw config text exists only on Artifact source;
- filesystem selection outside the ordinary input filter works only beneath workspace root;
- graph selection resolves only an already-loaded Artifact;
- a missing chart/value Artifact, duplicate profile name, invalid version, unknown field, or malformed YAML produces a config diagnostic and no explicit profile nodes.

- [ ] **Step 2: Run config tests to verify failure**

Run: `go test ./internal/dialects/helm -run 'Config|Profile'`

Expected: FAIL with undefined `parseConfig` and `defaultProfile`.

- [ ] **Step 3: Define and validate the typed config input**

Decode with known-field rejection into:

```go
type configFile struct {
	Version int `yaml:"version"`
	Renders []renderConfig `yaml:"renders"`
}
type renderConfig struct {
	Name string `yaml:"name"`
	Chart string `yaml:"chart"`
	ReleaseName string `yaml:"release_name"`
	Namespace string `yaml:"namespace"`
	Values []string `yaml:"values"`
	Set map[string]string `yaml:"set"`
	KubeVersion string `yaml:"kube_version"`
	APIVersions []string `yaml:"api_versions"`
}
```

Accept omitted `version` as 1 and reject any other value. Resolve chart/value selectors as Artifact IDs or app-relative paths; never read through a path during config parsing.

- [ ] **Step 4: Build deterministic profiles and layers**

Default release name is a stable DNS-safe form of chart name plus the first 8 hex characters of the chart Artifact digest; namespace is `default`; capabilities are the pinned Helm defaults. Explicit profile ID is `can://iac/<app>/config/profile/<name>`; default profile ID is `<chart-alias-id>/profile/default`. Layer IDs append `/value-layer/<zero-padded-ordinal>`.

Convert sorted `set` keys to ConfigKeys with spans from the config YAML AST. Append their layers after file layers so Helm's last-wins order is explicit.

- [ ] **Step 5: Pass config and profile tests**

Run: `go test -race ./internal/dialects/helm -run 'Config|Profile'`

Expected: PASS and no profile contains merged values.

- [ ] **Step 6: Commit typed configuration**

```bash
git add internal/dialects/helm/config* testdata/helm/profiles
git commit -m "feat(helm): model render configuration"
```

---

### Task 10: Render isolated Helm profiles and decode Kubernetes desired state at L3

**Files:**
- Create: `codeanalyzer-iac/internal/dialects/helm/render.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/render_test.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/kubernetes.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/kubernetes_test.go`
- Create: `codeanalyzer-iac/testdata/helm/render-failures/Chart.yaml`
- Create: `codeanalyzer-iac/testdata/helm/render-failures/templates/good.yaml`
- Create: `codeanalyzer-iac/testdata/helm/render-failures/templates/bad.yaml`
- Create: `codeanalyzer-iac/testdata/helm/security/Chart.yaml`
- Create: `codeanalyzer-iac/testdata/helm/security/templates/secret.yaml`

**Interfaces:**
- Consumes: Chart/profile/value-layer facts and complete Artifact sources.
- Produces: `renderProfile(ctx context.Context,app *model.Application,chart *model.HelmChart,profile *model.HelmRenderProfile,tempRoot string) (model.Delta,error)`, `materializeChart`, `mergeProfileValues`, `decodeDocuments`, `sanitizeResource`, `canonicalSHA256(any) string`, `escapeSetKey(string) string`, `escapeSetValue(string) string`, `failedRender(profile *model.HelmRenderProfile,phase,code string,cause error) (model.Delta,error)`, and L3 render/resource/address/provenance facts.

- [ ] **Step 1: Write no-network, traversal, status, and resource tests**

Assert exact behavior for:

- default profile and two explicit profiles produce different deterministic Deployment image facts/hashes and do not overwrite each other;
- application and library charts load; chart API v1 and v2 render;
- unpacked vendored dependencies render; `.tgz` is inventoried but never expanded from graph text;
- `lookup` is recorded from source but rendering cannot obtain a Kubernetes client;
- DNS is disabled;
- `../`, absolute, NUL, and symlink traversal cannot escape the private chart directory;
- load, dependency, values, template, and decode failures set exact `status`, `phase`, and diagnostic code;
- a partial multi-document result keeps only successfully decoded resources;
- named resources target stable addresses; `generateName`-only resources do not;
- Kubernetes `kind` maps to `resource_kind`, and no `status` field survives;
- Secret `data` and `stringData` keys survive with SHA-256 values, while plaintext never appears in marshaled JSON, logs, diagnostics, or projected properties.

- [ ] **Step 2: Run render tests to verify failure**

Run: `go test ./internal/dialects/helm -run 'Render|Kubernetes|Secret|Traversal'`

Expected: FAIL with undefined render/decode functions.

- [ ] **Step 3: Materialize a private virtual chart safely**

Create one `os.MkdirTemp(tempRoot,"caniac-render-")` per chart/profile and `defer os.RemoveAll(dir)`. Select only Artifacts belonging to the target chart, validate each app-relative member path before `os.MkdirAll`/`os.WriteFile`, and write exactly `[]byte(Artifact.Source)` with mode 0600. Reject symlinks and packaged binary dependencies.

Load through `loader.Load(dir)`, verify `chart.NewAccessor(chrt)` succeeds, and reject metadata API versions outside v1/v2 before render.

- [ ] **Step 4: Merge inputs through Helm and render without a client**

Read each Artifact-backed layer with `common.ReadValues`. Fold layers in ordinal order with a deep copy and `chartutil.MergeTables(next, merged)` because Helm treats the destination as authoritative. For a ConfigKey-backed layer, reparse the config Artifact source, recover that key's raw string, and apply it with `strvals.ParseIntoString(escapeSetKey(key)+"="+escapeSetValue(value), merged)`; test escaping for backslash, comma, equals, and dotted key paths. The model retains only source IDs, never a copied value.

```go
caps := common.DefaultCapabilities.Copy()
if profile.KubeVersion != "" {
	kv, err := common.ParseKubeVersion(profile.KubeVersion)
	if err != nil { return failedRender(profile, "values", "IAC_HELM_KUBE_VERSION", err) }
	caps.KubeVersion = *kv
}
caps.APIVersions = append(caps.APIVersions, profile.APIVersions...)
renderValues, err := chartutil.ToRenderValues(chrt, merged, common.ReleaseOptions{
	Name: profile.ReleaseName,
	Namespace: profile.Namespace,
	Revision: 1,
	IsInstall: true,
}, caps)
if err != nil { return failedRender(profile, "values", "IAC_HELM_VALUES", err) }
effectiveValuesHash := canonicalSHA256(renderValues["Values"])
rendered, err := (engine.Engine{Strict:false, LintMode:false, EnableDNS:false}).Render(chrt, renderValues)
```

Import `helm.sh/helm/v4/pkg/chart/common/util` as `chartutil` and `helm.sh/helm/v4/pkg/strvals` as `strvals`. Hash the final coalesced `.Values` subtree, including chart defaults, then discard it. Use `engine.Engine{}` rather than `engine.New`, so no client provider exists and `lookup` cannot reach a cluster.

Set render identity to `<chart-alias-id>/render/<percent-encoded-profile-name>@<first-16-input-hash>`. The full input hash covers sorted chart member IDs/digests, ordered profile/layer facts, renderer name `helm`, and renderer version `4.2.4`; it is also stored in full on the render node.

- [ ] **Step 5: Decode, identify, and sanitize resources**

Sort rendered filename keys, then stream every YAML document with `k8s.io/apimachinery/pkg/util/yaml.NewYAMLOrJSONDecoder`. Wrap maps in `unstructured.Unstructured`. Remove `status`; extract apiVersion/group, resource kind, metadata namespace/name/generateName/labels/annotations; compute `manifest_sha256` from deterministic sanitized JSON.

Render-scoped resource ID is `<render-id>/kubernetes/<group-or-core>/<resource-kind>/<namespace-or-default>/<name>` with a document ordinal suffix for collisions. Stable address excludes API version. Use built-in plural data or in-repository CRD facts only.

For Secrets, replace `data` and `stringData` with:

```go
type KubernetesSecretDatum struct {
	Key string `json:"key"`
	SHA256 string `json:"sha256"`
}
```

Hash decoded `data` bytes and raw `stringData` bytes. Do not retain or format those values after hashing.

- [ ] **Step 6: Prove render security and deterministic output**

Run:

```bash
go test ./internal/dialects/helm -run 'Render|Kubernetes|Secret|Traversal'
go test -race ./internal/dialects/helm -run 'Render|Kubernetes|Secret|Traversal'
go test ./internal/dialects/helm -run TestRenderDeterministic -count=20
```

Expected: PASS; the security test recursively searches JSON and captured stderr for every fixture plaintext and finds none.

- [ ] **Step 7: Commit Helm L3 rendering**

```bash
git add internal/dialects/helm/render* internal/dialects/helm/kubernetes* testdata/helm/render-failures testdata/helm/security
git commit -m "feat(helm): render profiles into Kubernetes facts"
```

---

### Task 11: Wire the L1–L3 orchestrator, parallelism, diagnostics, and JSON emission

**Files:**
- Create: `codeanalyzer-iac/internal/core/analyzer.go`
- Create: `codeanalyzer-iac/internal/core/analyzer_test.go`
- Create: `codeanalyzer-iac/internal/emit/json/write.go`
- Create: `codeanalyzer-iac/internal/emit/json/write_test.go`
- Modify: `codeanalyzer-iac/cmd/codeanalyzer-iac/main.go`
- Modify: `codeanalyzer-iac/internal/cli/root.go`

**Interfaces:**
- Consumes: `ingest.Source`, compiled registry containing `helm.New()`, model Delta/Validate, and options.
- Produces: `core.New(opts, source, registry) *Analyzer`, `(*Analyzer).Analyze(ctx) (*model.Analysis,error)`, `jsonemit.Marshal`, `jsonemit.Write`, and production `cli.Runner`.

- [ ] **Step 1: Write phase, level, failure, and determinism tests**

Use fake Source and Frontend implementations to assert:

- phase order is load → detect → parse → resolve → config/default profiles → evaluate → validate;
- L1 never calls Resolve/Evaluate; L2 calls Resolve but not Evaluate; L3 calls both;
- one artifact parse failure produces a diagnostic and permits others;
- invalid config is analyzer-wide fatal after an inspectable partial model is available;
- default mode exits successfully on nonfatal artifact/render diagnostics; strict mode returns nonzero after writing output when any error diagnostic exists;
- the same fixture at jobs 1, 2, and 8 marshals byte-identically;
- L1 is a subset of L2 and L2 a subset of L3;
- graph/Cypher always runs L3.

- [ ] **Step 2: Run orchestrator tests to verify failure**

Run: `go test ./internal/core ./internal/emit/json`

Expected: FAIL with undefined Analyzer/Marshal/Write.

- [ ] **Step 3: Implement ordered phase orchestration**

```go
type Analyzer struct {
	opts options.Options
	source ingest.Source
	registry *dialect.Registry
}

type keyedDelta struct {
	Key string
	Delta model.Delta
}

func (a *Analyzer) Analyze(ctx context.Context) (*model.Analysis, error) {
	loaded, err := a.source.Load(ctx)
	if err != nil { return nil, fmt.Errorf("load input: %w", err) }
	app := model.NewApplication(a.opts.AppName, loaded.Artifacts)
	app.Diagnostics = maps.Clone(loaded.Diagnostics)
	detections, detectDelta := a.registry.DetectAll(app)
	if err := model.Apply(app, detectDelta); err != nil { return nil, err }
	if err := applySorted(app, a.parseArtifacts(ctx, detections)); err != nil { return nil, err }
	if a.opts.AnalysisLevel >= 2 {
		if err := model.Apply(app, a.registry.ResolveAll(ctx, app)); err != nil { return nil, err }
	}
	if a.opts.AnalysisLevel >= 3 {
		profiles, err := helm.BuildProfiles(app, a.opts.Config)
		if err != nil { return nil, err }
		if err := model.Apply(app, profiles); err != nil { return nil, err }
		if err := applySorted(app, a.evaluateProfiles(ctx, app)); err != nil { return nil, err }
	}
	analysis := model.NewAnalysis(a.opts.AnalysisLevel, app)
	return analysis, model.Validate(analysis.Application)
}
```

Define `Registry.DetectAll(*model.Application) (map[string]dialect.Detection, model.Delta)` and `Registry.ResolveAll(context.Context,*model.Application) model.Delta` in Task 5. `parseArtifacts` and `evaluateProfiles` return `[]keyedDelta`; `applySorted` sorts by `keyedDelta.Key` before calling `model.Apply`. Workers never mutate Application.

- [ ] **Step 4: Bound parallel work without changing IDs/order**

Use `errgroup.Group.SetLimit(opts.Jobs)` for per-artifact parse and per-profile render. Assign every ID from source identity before scheduling or from deterministic content after completion; never use completion order. Collect results by key and sort before Apply.

- [ ] **Step 5: Implement canonical JSON output and schema validation**

`jsonemit.Marshal` calls `model.Validate`, marshals with `encoding/json` (compact by default), validates the unmarshaled document against embedded `schema.json`, and appends exactly one newline for stdout/files. `Write` uses a temp file plus `os.Rename` for `<output>/analysis.json`; no progress or diagnostics go to stdout.

- [ ] **Step 6: Wire production CLI behavior**

Delete `bootstrapRunner`, add `type coreRunner struct{}` whose `Run` builds the selected Source and registry, calls `core.New(...).Analyze`, writes the selected output, and returns the strict-mode error after the write. Output behavior:

- `--emit auto`: JSON for filesystem, direct reconciliation to the positional database for graph mode;
- `--emit json`: compact stdout or `<output>/analysis.json`, with no graph mutation;
- `--emit cypher`: `<output>/graph.cypher` or stdout;
- `--emit neo4j`: direct Bolt write; filesystem mode requires `--neo4j-uri`, graph mode reuses its positional URI unless the same explicit URI is supplied;
- `--emit schema`: write embedded `schema.neo4j.json` and require no input;
- `--strict`: finish the requested write, then return a typed diagnostics error when error-severity diagnostics exist.

- [ ] **Step 7: Pass level and determinism gates**

Run:

```bash
go test ./internal/core ./internal/emit/json
go test -race ./internal/core ./internal/emit/json
go test ./internal/core -run TestOutputIndependentOfJobs -count=20
```

Expected: PASS and byte-identical outputs for all worker counts.

- [ ] **Step 8: Commit the executable analysis pipeline**

```bash
git add internal/core internal/emit/json internal/cli cmd/codeanalyzer-iac
git commit -m "feat: run deterministic IaC analysis pipeline"
```

---

### Task 12: Project exact graph rows and reconcile Neo4j safely

**Files:**
- Create: `codeanalyzer-iac/internal/emit/neo4j/rows.go`
- Create: `codeanalyzer-iac/internal/emit/neo4j/project.go`
- Create: `codeanalyzer-iac/internal/emit/neo4j/project_test.go`
- Create: `codeanalyzer-iac/internal/emit/neo4j/catalog.go`
- Create: `codeanalyzer-iac/internal/emit/neo4j/catalog_test.go`
- Create: `codeanalyzer-iac/internal/emit/neo4j/cypher.go`
- Create: `codeanalyzer-iac/internal/emit/neo4j/cypher_test.go`
- Create: `codeanalyzer-iac/internal/reconcile/reconcile.go`
- Create: `codeanalyzer-iac/internal/reconcile/reconcile_test.go`
- Create: `codeanalyzer-iac/internal/reconcile/guard.go`
- Create: `codeanalyzer-iac/internal/reconcile/guard_test.go`
- Create: `codeanalyzer-iac/internal/reconcile/bolt.go`
- Create: `codeanalyzer-iac/internal/reconcile/bolt_integration_test.go`
- Modify: `codeanalyzer-iac/cmd/codeanalyzer-iac/main.go`
- Modify: `codeanalyzer-iac/internal/cli/root.go`

**Interfaces:**
- Consumes: fully validated L3 `model.Analysis`, embedded graph catalog, Neo4j credentials/options.
- Produces: `neo4jemit.Project(*model.Analysis) (GraphRows,error)`, `GraphRows`, `Catalog() ([]byte,error)`, `WriteCypher(io.Writer,GraphRows) error`, `reconcile.Guard(ingest.Source,ArtifactLookup) ingest.Source`, `reconcile.BuildPlan(GraphRows,ExistingState,bool) (Plan,error)`, `reconcile.Apply(context.Context,Store,Plan) error`, and transactional `BoltStore`.

- [ ] **Step 1: Write exact projection tests before row types**

```go
func TestChartArtifactIsOneProgressivelyTypedNode(t *testing.T) {
	rows, err := Project(fullFixtureAnalysis())
	if err != nil { t.Fatal(err) }
	node := rows.Node("can://artifact/payments/charts/api/Chart.yaml")
	want := []string{"Artifact", "HelmArtifact", "HelmChart", "IaCArtifact"}
	if diff := cmp.Diff(want, node.Labels); diff != "" { t.Fatal(diff) }
	if rows.HasSeparateNodeKind("helm_chart", node.ID) { t.Fatal("duplicated chart artifact") }
}
```

Also assert all representative labels/properties and every relationship endpoint from spec section 5, exact JSON-to-row parity, alias node plus one `IAC_ALIAS_OF`, shared ConfigKey/Package reuse, identity-only relationships, producer metadata only on owned semantic nodes, and no source on contained nodes.

- [ ] **Step 2: Write reconciliation ownership tests**

Seed fake existing rows with `:Artifact:TSModule`, neutral properties, `TS_*` edges, stale IaC nodes/labels/properties, shared `HAS_ARTIFACT`, and a foreign Package. Assert default upsert removes nothing; eager removes only stale producer-owned IaC facts and never neutral/foreign facts.

Seed a formerly selected config Artifact with `:CodeAnalyzerIaCConfig`, `iac_config_version`, and profile children, then run eager without `--config`. Assert the config facet label/property and owned profile subgraph are removed while the Artifact, source, hash, and foreign labels remain.

Wrap a filesystem Source with `Guard` and seed its target Artifact ID at the same hash and at a conflicting hash. The equal hash remains eligible. The conflict becomes an ineligible raw Artifact with `IAC_ARTIFACT_HASH_CONFLICT`; no Helm facet is produced and the target's source/hash/foreign properties remain unchanged.

- [ ] **Step 3: Run projector/reconciler tests to verify failure**

Run: `go test ./internal/emit/neo4j ./internal/reconcile`

Expected: FAIL with undefined row/project/plan types.

- [ ] **Step 4: Define canonical rows and a pure projector**

```go
type NodeRow struct {
	ID string
	Labels []string
	Properties map[string]any
}
type EdgeRow struct {
	Type string
	Src string
	Dst string
}
type GraphRows struct {
	Nodes []NodeRow
	Edges []EdgeRow
}
```

`Project(*model.Analysis) (GraphRows,error)` has no I/O. Merge labels/properties for equal node IDs, reject property conflicts, dedupe edges on `(type,src,dst)`, and sort labels, properties during encoding, nodes by ID, and edges by type/src/dst. Artifact source/sha/path/format stay neutral properties; typed facet properties use `iac_*` and `helm_*` names.

Encode structured maps as deterministic JSON string properties with a `_json` suffix (`labels_json`, `annotations_json`, `secret_data_json`) because Neo4j properties cannot contain nested objects. Owned contained nodes carry `producer`, `analyzer_version`, and `iac_app_id`; a shared Artifact facet uses namespaced `iac_producer`, `iac_analyzer_version`, and `iac_app_id` rather than claiming the neutral node wholesale.

- [ ] **Step 5: Generate and byte-check the catalog**

`Catalog() ([]byte,error)` deterministically describes every label, property type, relationship endpoint family, uniqueness constraint, and index. Test:

```go
func TestCatalogMatchesTrackedSchema(t *testing.T) {
	got, err := Catalog()
	if err != nil { t.Fatal(err) }
	want, err := os.ReadFile("../../../schema.neo4j.json")
	if err != nil { t.Fatal(err) }
	if !bytes.Equal(got, want) { t.Fatal("run make schema after intentional contract change") }
}
```

- [ ] **Step 6: Render deterministic parameterized Cypher**

Generate constraint/index statements followed by parameter blocks and `UNWIND` upserts. Never interpolate source, credentials, property values, IDs, labels discovered from input, or diagnostics into Cypher syntax. Labels and relationship types come only from the compiled catalog allowlist.

Snapshot-test `graph.cypher`, including a source string containing quotes, backticks, `$`, and Unicode. A second generation must byte-match.

- [ ] **Step 7: Implement producer-scoped reconciliation planning**

```go
type Plan struct {
	UpsertNodes []neo4jemit.NodeRow
	UpsertEdges []neo4jemit.EdgeRow
	DeleteOwnedNodeIDs []string
	DeleteOwnedEdges []neo4jemit.EdgeRow
	RemoveFacetLabels map[string][]string
	RemoveFacetProperties map[string][]string
}

type ExistingState struct {
	Nodes []neo4jemit.NodeRow
	Edges []neo4jemit.EdgeRow
}
```

```go
type Store interface {
	ReadExisting(context.Context, string) (ExistingState, error)
	WriteGeneration(context.Context, Plan) error
}

type ArtifactLookup interface {
	ExistingArtifactHashes(context.Context, []string) (map[string]string, error)
}
```

Desired-set comparison is scoped by `iac_app_id` and `producer:"codeanalyzer-iac"`. Eager deletion allowlists only owned labels from the catalog, `iac_*|helm_*` properties, and `IAC_*` edges. `Artifact`, `ConfigKey`, `Package`, `Application`, `HAS_ARTIFACT`, and `DEFINES_CONFIG` are immutable deletion-denylist entries.

- [ ] **Step 8: Implement one-generation Bolt transaction**

When filesystem input will write directly to Neo4j, `coreRunner` wraps it with `reconcile.Guard` before constructing `core.Analyzer`. Create constraints first. Inside one managed write transaction, use `MERGE ... ON CREATE SET` for neutral Artifact source/hash/path/format, then upsert owned nodes/facets, shared relationships, and owned relationships. Apply eager cleanup last. Store generation metadata and mark current only after all statements succeed. A forced failure before the last statement must roll back every IaC change.

- [ ] **Step 9: Pass unit and live Neo4j integration gates**

Run unit gates:

```bash
go test ./internal/emit/neo4j ./internal/reconcile
go test -race ./internal/emit/neo4j ./internal/reconcile
```

Run integration with a disposable Neo4j 5.x instance:

```bash
NEO4J_TEST_URI=neo4j://localhost:7687 \
NEO4J_TEST_USERNAME=neo4j \
NEO4J_TEST_PASSWORD=test-password \
go test ./internal/reconcile -run Integration -count=1
```

Expected: first run creates facts, second run is idempotent, eager removes seeded stale IaC facts, foreign facts remain byte-for-byte equal, and forced failure rolls back.

- [ ] **Step 10: Commit graph projection and reconciliation**

```bash
git add internal/emit/neo4j internal/reconcile schema.neo4j.json cmd/codeanalyzer-iac internal/cli
git commit -m "feat: project and reconcile IaC graphs"
```

---

### Task 13: Prove exact Helm representation against live repositories

**Files:**
- Create: `codeanalyzer-iac/tests/live/repositories.json`
- Create: `codeanalyzer-iac/tests/live/harness_test.go`
- Create: `codeanalyzer-iac/tests/live/acceptance_test.go`
- Create: `codeanalyzer-iac/tests/live/graph_test.go`
- Create: `codeanalyzer-iac/.github/workflows/live.yml`
- Modify: `codeanalyzer-iac/Makefile`
- Modify: `codeanalyzer-iac/README.md`

**Interfaces:**
- Consumes: the production CLI/analyzer, JSON and graph contracts, a disposable Neo4j 5.x database, Git, the two public repositories, and Helm CLI 4.2.4 as an independent render oracle used only by tests.
- Produces: `make test-live`, reproducible pinned-revision acceptance, an opt-in `CANIAC_LIVE_REF_MODE=head` drift lane, and exact filesystem/graph representation evidence.

- [ ] **Step 1: Add the immutable live-repository manifest and failing harness tests**

`repositories.json` records URL, pinned full commit, default branch, chart root, expected tracked-file count, and the application name. Use exactly:

```json
[
  {
    "name": "daytrader",
    "url": "https://github.com/sample-daytrader/sample.daytrader.microservices.git",
    "commit": "8a68b59430a94a242c54384763da9eb7682728b4",
    "default_branch": "main",
    "chart": "platform/helm",
    "tracked_files": 21
  },
  {
    "name": "quarkuscoffeeshop",
    "url": "https://github.com/quarkuscoffeeshop/quarkuscoffeeshop-helm.git",
    "commit": "aa3c842658e0fc7e44fa25132d8b817eab225cbe",
    "default_branch": "master",
    "chart": "charts/quarkuscoffeeshop-charts",
    "tracked_files": 14
  }
]
```

Write tests first for malformed/short refs, unexpected origin, failed checkout, dirty checkout, missing chart roots, and Helm versions other than 4.2.4. The harness clones into `t.TempDir()`, verifies `HEAD`, inventories `git ls-files`, never reuses a developer checkout, and deletes its temporary clone through normal test cleanup. `CANIAC_LIVE_REF_MODE=head` selects the declared default branch but keeps every semantic assertion active.

Run: `go test -tags=live ./tests/live -run TestHarness`

Expected: FAIL because the manifest loader and checkout/oracle helpers do not exist.

- [ ] **Step 2: Implement the isolated live harness**

Implement subprocess boundaries for `git` and the test-only Helm oracle with explicit argument arrays, captured stdout/stderr, timeouts, and credential-free public URLs. Require Helm's reported semantic version to equal `v4.2.4`. Build `caniac` once into the test temp root, analyze the complete cloned repository with an explicit stable `--workspace-root` and `--app-name`, and exclude `.git` through the production filesystem walker—not by narrowing the live input to the chart directory.

The harness writes an untracked `.caniac-live.yaml` typed config Artifact inside each temporary clone. It declares explicit release names/namespaces and named profiles so the same source material exercises default rendering and value-controlled branches. Graph-input tests seed this config Artifact along with all repository Artifacts.

Run: `go test -tags=live ./tests/live -run TestHarness`

Expected: PASS with pristine output.

- [ ] **Step 3: Assert the DayTrader chart exactly**

Analyze the repository root and require all 21 tracked files plus the generated config to exist as canonical `can://artifact/daytrader/...` Artifacts with matching source/digest. `docker-compose.yml`, `README.md`, and non-Helm files remain raw Artifacts without a Helm facet. Assert the Chart is `apiVersion:v1`, name `daytrader`, version `1.1.0`; all 13 template files belong to it; and source facts retain the conditional `.Values.psp.enabled` and `.Values.ocCreateRoute` references.

Use explicit release `daytrader` and namespace `daytrader`. Compare the analyzer's canonical `(apiVersion,kind,namespace,name)` resource set and normalized document digest with independent `helm template` output for all three profiles:

| profile | exact resource result |
|---|---|
| default | 5 Deployments + 5 Services = 10 |
| `psp.enabled=true` | default plus 1 ClusterRole, 1 ClusterRoleBinding, 1 ServiceAccount = 13 |
| `ocCreateRoute=true` | default plus 5 OpenShift Routes = 15 |

The exact default names are `daytrader-{accounts,gateway,portfolios,quotes,web}` and `daytrader-{accounts,gateway,portfolios,quotes,web}-service`. Do not collapse the conditional Route documents or the PSP templates merely because they are absent from the default render.

- [ ] **Step 4: Assert the Quarkus Coffee Shop chart exactly**

Analyze the repository root and require all 14 tracked files plus the generated config as canonical Artifacts. `.github` workflow/configuration YAML and the repository README remain raw; only files under `charts/quarkuscoffeeshop-charts` gain Helm roles. Assert `apiVersion:v2`, type `application`, name `quarkuscoffeeshop-charts`, version `3.5.0`, and appVersion `5.0.3`.

Require all six named templates from `_helpers.tpl` (`name`, `fullname`, `chart`, `labels`, `selectorLabels`, `serviceAccountName`) and resolution of every one of its eight `include` calls. Preserve 16 source resource-template documents across multi-document files and mark `templates/tests/test-connection.yaml` with both test and hook roles plus hook value `test-success`.

For release `coffee` and namespace `quarkuscoffeeshop-demo`, compare exactly with Helm 4.2.4: 7 Deployments + 7 Services + 1 ServiceAccount + 1 test Pod = 16. Preserve the upstream Deployment name `quarkuscoffeshop-web` exactly, including its spelling, and distinguish it from Service `quarkuscoffeeshop-web`. Require helper-derived names `coffee-quarkuscoffeeshop-charts` and `coffee-quarkuscoffeeshop-charts-test-connection`. A second profile with `serviceAccount.create=false` must produce exactly 15 resources and retain the ServiceAccount source template as an L1 fact.

- [ ] **Step 5: Prove schema validity and filesystem/graph equality for both repositories**

For each repository, validate L1/L2/L3 JSON with the embedded accepted schema and run the accepted `scripts/check_iac.py` semantic checker against the L3 document. Seed a disposable Neo4j database with only the complete neutral Artifacts, their lowercase SHA-256 values, the generated config Artifact, and foreign labels/properties. Analyze the positional Neo4j URI using the config Artifact ID; compare canonical typed IaC node and identity-only edge row sets with filesystem mode exactly, then verify foreign facts remain unchanged.

Also compare direct Cypher projection with the same row set. No test may replace exact row/resource equality with substring assertions or a golden generated by `caniac` itself.

Run:

```bash
NEO4J_TEST_URI=neo4j://localhost:7687 \
NEO4J_TEST_USERNAME=neo4j \
NEO4J_TEST_PASSWORD=test-password \
go test -tags=live ./tests/live -count=1
```

Expected: both external repositories pass schema, semantic, Helm-oracle, filesystem/graph, and foreign-fact preservation assertions.

- [ ] **Step 6: Add pinned and moving-head CI lanes**

Create `live.yml` with a pinned-ref job for pull requests and manual runs, and a weekly scheduled job using `CANIAC_LIVE_REF_MODE=head`. Install/check Helm 4.2.4 explicitly and provide Neo4j 5.x as a service. The moving-head lane reports the resolved commits and fails visibly on drift; it does not commit or upload regenerated expectations. Keep ordinary unit tests network-independent.

Add `test-live` and `test-live-head` Make targets and document prerequisites, URLs, pins, exact chart paths, expected semantics, and how an intentional upstream change is reviewed before updating a pin/assertion.

- [ ] **Step 7: Run and commit the live acceptance gate**

Run both pinned and current-head modes locally against Helm 4.2.4 and the disposable database. If current head has advanced, record the resolved SHA and resulting mismatch without weakening pinned acceptance; update expectations only after inspecting the upstream diff.

```bash
make test-live
CANIAC_LIVE_REF_MODE=head make test-live
git diff --check
```

Expected: pinned mode passes. Head mode passes when the upstream default branches still satisfy the reviewed expectations, otherwise it fails with the exact semantic drift.

```bash
git add tests/live .github/workflows/live.yml Makefile README.md
git commit -m "test: validate live Helm repositories"
```

---

### Task 14: Complete end-to-end parity, fuzzing, CI, and analyzer documentation

**Files:**
- Modify: `codeanalyzer-iac/internal/cli/root_test.go`
- Modify: `codeanalyzer-iac/internal/core/analyzer_test.go`
- Create: `codeanalyzer-iac/tests/e2e_test.go`
- Create: `codeanalyzer-iac/tests/parity_test.go`
- Create: `codeanalyzer-iac/tests/security_test.go`
- Create: `codeanalyzer-iac/internal/dialects/helm/fuzz_test.go`
- Create: `codeanalyzer-iac/.github/workflows/ci.yml`
- Modify: `codeanalyzer-iac/Makefile`
- Modify: `codeanalyzer-iac/README.md`
- Modify: `codeanalyzer-iac/.claude/SCHEMA_DECISIONS.md`

**Interfaces:**
- Consumes: all source, frontend, model, emit, and reconcile APIs from Tasks 1–12.
- Produces: release-candidate conformance gates for `codeanalyzer-iac` 0.1.0, including the live-repository acceptance from Task 13.

- [ ] **Step 1: Write filesystem and graph-input parity tests**

Analyze `testdata/helm/profiles` from filesystem into a model. Seed a disposable Neo4j database with only the resulting neutral Artifact IDs/source/digests plus foreign labels, then analyze via the positional Neo4j URI. Strip only input-mode diagnostics and compare canonical typed IaC node/edge row sets exactly.

Assert filesystem JSON at L1/L2/L3 validates against `schema.json`; assert graph projection always contains L3. Assert every lower-level fact is unchanged at higher levels.

- [ ] **Step 2: Write CLI channel and exit tests**

Build `./caniac` and assert:

```bash
go build -o ./caniac ./cmd/codeanalyzer-iac
./caniac testdata/helm/l1-v2 --app-name payments --analysis-level 1 > /tmp/caniac.json 2>/tmp/caniac.err
python3 -m json.tool /tmp/caniac.json >/dev/null
test ! -s /tmp/caniac.err
```

Add subprocess tests for default `.`, multiple/overlapping paths, workspace-root identity, graph app requirement, invalid config, strict partial output, msgpack rejection, level 4 rejection, mixed URI/path rejection, schema output without input, `--version`, and credential redaction.

- [ ] **Step 3: Add fuzz targets with seed fixtures**

Create Go fuzz tests:

```go
func FuzzHelmTemplateNeverPanics(f *testing.F) {
	f.Add([]byte(`{{ .Values.image.tag }}`))
	f.Fuzz(func(t *testing.T, source []byte) {
		artifact := fixtureTemplateArtifact(source)
		_, _ = parseTemplate(artifact, fixtureDetection())
	})
}
```

Add equivalent targets for YAML values, Chart metadata, config YAML, values JSON Schema, and rendered multi-document YAML. Each target may return diagnostics/errors but must not panic, escape a temp root, contact network, or leak Secret fixture values.

- [ ] **Step 4: Run short fuzz smoke tests**

```bash
go test ./internal/dialects/helm -run '^$' -fuzz FuzzHelmTemplateNeverPanics -fuzztime=10s
go test ./internal/dialects/helm -run '^$' -fuzz FuzzHelmValuesNeverPanics -fuzztime=10s
go test ./internal/dialects/helm -run '^$' -fuzz FuzzRenderedDocumentsNeverPanic -fuzztime=10s
```

Expected: each completes without a crash.

- [ ] **Step 5: Document the actual 0.1.0 behavior**

README sections must include:

- installation/build and `caniac` alias;
- filesystem examples with arbitrary paths and stable workspace root;
- graph enrichment example using positional URI and mandatory app name;
- optional typed `--config` behavior in both modes and its exact graph labels/edges;
- analysis levels, output modes, strict/eager behavior, credentials, and exit channels;
- progressive Artifact facet example and alias query;
- no-network/no-cluster/no-fetch guarantees and Secret-derived-data policy;
- Helm support matrix including v1/v2 charts and the `.tgz` inventory limitation;
- future dialect architecture, explicitly stating that native-runtime workers may emit the same typed Delta contract without becoming separate backends;
- all explicit non-goals from spec section 10.

- [ ] **Step 6: Add reproducible Make and CI gates**

```make
.PHONY: test race vet schema-check fuzz-smoke
test:
	go test ./...
race:
	go test -race ./...
vet:
	go vet ./...
schema-check:
	cmp schema.json internal/contract/schema.json
	cmp schema.neo4j.json internal/contract/schema.neo4j.json
	go test ./internal/contract ./internal/emit/neo4j
fuzz-smoke:
	go test ./internal/dialects/helm -run '^$$' -fuzz FuzzHelmTemplateNeverPanics -fuzztime=10s
```

CI runs format check, vet, unit/race tests, schema drift, and an integration job with a Neo4j 5.x service and no outbound dependency during tests. Dependency download occurs before the test network restriction.

- [ ] **Step 7: Run the complete backend gate**

```bash
gofmt -w cmd internal tests
test -z "$(gofmt -l cmd internal tests)"
go mod tidy
go vet ./...
go test ./...
go test -race ./...
make schema-check
go test ./internal/core -run TestOutputIndependentOfJobs -count=20
git diff --check
```

Then run the live Neo4j parity/integration job from Task 12 and `make test-live` from Task 13. Expected: every command passes; repeated JSON, Cypher, and graph row sets are identical; both external Helm charts match the independent Helm 4.2.4 oracle.

- [ ] **Step 8: Commit the release-candidate gates and docs**

```bash
git add .github Makefile README.md .claude tests internal/dialects/helm/fuzz_test.go internal/cli/root_test.go internal/core/analyzer_test.go
git commit -m "test: complete Helm backend conformance gates"
```

---

## Plan Completion Gate

Before claiming implementation complete, invoke `cldk-devtools:finishing-cldk-work`, `superpowers:verification-before-completion`, and `superpowers:requesting-code-review`. That release rung must verify both repository histories, the accepted schema commit, the tracker child issues, documentation integration, and the 0.1.0 artifact/release path.

The backend implementation itself is complete only when:

```bash
go vet ./...
go test ./...
go test -race ./...
make schema-check
git status --short
```

all succeed, the live Neo4j integration/parity suite and pinned live-repository suite pass, both current upstream heads have been checked for drift, and status contains only intentional branch changes.
