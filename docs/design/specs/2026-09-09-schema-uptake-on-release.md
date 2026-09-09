# Spec: analyzers record their schemas in codeanalyzer-schema on release

Status: accepted
Date: 2026-09-09
Scope: release automation across the three v2 analyzers, cross-repo

---

## 1. Summary

Each `codeanalyzer-<lang>` release workflow gains a final step that records what that release
emits into the shared contract repo, `codellm-devkit/codeanalyzer-schema`, as a pull request:

- the Neo4j graph contract → `v<major>/neo4j/<lang>/schema.neo4j.json`
- the `analysis.json` samples → `v<major>/json/<lang>/*.sample.json`

Both destinations are derived from the `schema_version` inside the emitted documents. Every
analyzer reads `2.0.0` on both projections today, so everything currently lands under `v2/`.

### Why

The contract repo's README states its own provenance rule: the samples are "unedited analyzer
output on the analyzers' own test fixtures", refreshed by hand. Refreshed by hand means refreshed
late. On 2026-09-09 the tracked copies were three releases behind — python 1.4.0 against a released
1.5.0, java 3.0.1 against 3.1.1, typescript 1.2.0 against 1.5.2 — and the python Neo4j contract
still carried a `_module` property that 1.4.1 had removed. Nothing was wrong with any analyzer; the
record of what they emit had simply drifted, and only a manual refresh could reveal it.

That drift is silent by construction. Nothing fails when the contract repo is stale, so the
staleness surfaces only when someone regenerates and reads the diff.

## 2. Contract-Impact Triage

**Does this change schema v2 output?** No. No analyzer changes what it emits; this only records it.

**Which repos are touched:**

| Change | Analyzers | SDKs | Docs |
| --- | --- | --- | --- |
| Release-time schema uptake | all three `codeanalyzer-*` release workflows | — | the contract repo receives PRs |

## 3. The design

One step, last in each `release` job, gated on `startsWith(github.ref, 'refs/tags/')` so it runs
only for a real tagged release and only after that release has published.

1. Run the analyzer on its own fixtures, staging each sample under the exact filename the
   contract repo keeps it under.
2. Take the graph contract. Java and typescript already emit theirs as a release asset earlier in
   the job and it is reused; python emits it here.
3. Derive `v<major>` for each projection **from its own document**, not from a constant.
4. Clone the contract repo, copy both sets in, and open a PR from `uptake/<lang>-v<version>`.

### Decisions

- **A pull request, not a push.** Three analyzers release on their own clocks; a push races when
  two land close together, and a contract change would arrive unreviewed. A branch per release
  cannot race, and a re-run of the same tag force-updates its own branch rather than opening a
  second PR.
- **The destination is derived, not hard-coded.** `v2` is a fact about today's analyzers, not
  about this step. Reading the major from the emitted `schema_version` means the first analyzer to
  move to a v3 line writes to `v3/` without anyone editing three workflows. The two projections are
  read independently because their versions are independent lines, which the contract meta-schema
  states explicitly.
- **Samples as well as the contract.** The contract alone would leave the JSON half drifting, which
  is the half that was three releases stale.
- **Token: `ORG_DISCUSSIONS_TOKEN`.** The org PAT these workflows already use for the cross-repo
  announcement. It needs `contents:write` and `pull-requests:write` on the contract repo; the two
  repos describe its scope differently, so the step fails with an explicit message naming
  `CLDK_AUTH_TOKEN` — documented as carrying repo scope, and already used for a cross-repo issue
  write — as the alternative if it 403s.
- **`analysis.schema.json` is never touched.** That file is the hand-written schema, not analyzer
  output. Only `*.sample.json` and `schema.neo4j.json` are overwritten.

### Not decided here

Whether the contract repo should gate its own PRs on `scripts/check.py` in CI. It has no workflow
today, so these PRs arrive unverified and a reviewer runs the checker by hand. Worth its own issue.

## 4. Decomposition and release plan

One work item per repo, since each is one PR against a different repository, tracked under one
epic. No lockstep: each analyzer's step is independent and takes effect at that analyzer's next
release. The step is inert until then, and inert forever on a repo whose token lacks the scope,
which is why it fails loudly rather than silently skipping.

## 5. Definition of done

- All three release workflows carry the step, gated on a tag, as the last step of the release job.
- The step's destination paths are derived from the emitted documents and match the contract repo's
  existing layout exactly.
- A release on any one analyzer opens a PR on `codeanalyzer-schema` containing only that
  analyzer's files.
