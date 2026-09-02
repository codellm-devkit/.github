# Spec: an artifact's `source` is the whole file, or nothing

Status: draft for review
Date: 2026-09-02
Scope: the artifact layer across all analyzers, cross-repo

---

## 1. Summary

`PyArtifact.source` / `JArtifact.source` / `TSArtifact.source` stops having three possible
meanings and starts having two:

| | before | after |
| --- | --- | --- |
| capture on, small file | whole file, `text_truncated: false` | whole file |
| capture on, large file | **first 262144 bytes**, `text_truncated: true` | whole file |
| capture off | `""`, `text_truncated: false` | `""` |

The byte cap, its CLI flag (`--artifact-text-max-bytes`), and the `text_truncated` field all
retire. `--no-artifact-text` stays: opting out of the payload entirely is a real choice with an
unambiguous meaning, and it is the only remaining reason `source` can be empty for a decodable
file.

### Why

**A truncated `source` reads exactly like a complete small file.** Nothing about the value says
it is a prefix. Every consumer must carry the flag alongside the text to know whether the text can
be trusted, and a consumer that forgets is silently reasoning about the first 256 KB of a file
while believing it has the whole thing. The failure is quiet and the value looks correct.

**The flag cannot carry that meaning cleanly even when it is checked.** `text_truncated: false`
covers two different states — "this is the whole file" and "capture is off, `source` is empty" — so
a consumer reading only the flag cannot tell them apart. It takes `source != "" && !text_truncated`
to mean "trustworthy", which is a two-field invariant nothing enforces.

**It buys very little.** Measured on `microsoft/vscode`: 32 of 4,953 artifacts exceeded the cap —
**0.6%**. The payload saving is negligible against the interpretation cost.

**Integrity never depended on it.** `sha256` and `size_bytes` were always computed over the full
file regardless of the cap, so no integrity check changes.

## 2. Contract-Impact Triage

**Does this change schema v2 output?** Yes, in both projections. `text_truncated` leaves the
artifact model in `analysis.json` and leaves the language-neutral `:Artifact` node in Neo4j. A
removed field is breaking by every analyzer's own stated bump policy.

**Which repos are touched:**

| Change | Analyzers | SDKs | Docs |
| --- | --- | --- | --- |
| Drop the cap, the flag, and `text_truncated` | `codeanalyzer-python`, `codeanalyzer-java`; `codeanalyzer-typescript` **already done** | none — verified: no SDK models the field (§5) | each analyzer's graph-vocabulary reference |

## 3. Where the three analyzers stand today

| | java | python | typescript |
| --- | --- | --- | --- |
| `text_truncated` on the artifact | `JArtifact.textTruncated` | `PyArtifact.text_truncated` | **gone** (#117) |
| `text_truncated` on the graph node | `V2SchemaCatalog` + `V2GraphProjector` | `neo4j/schema.py`, `neo4j/project.py` | **gone** |
| byte cap | `--artifact-text-max-bytes`, default 262144 | `--artifact-text-max-bytes`, default 262144 | **gone** |
| opt out entirely | `--no-artifact-text` | `--no-artifact-text` | `--no-artifact-text` |
| carve-outs from the cap | none | `dependency-manifest` roles are captured in full regardless | n/a |

TypeScript decided this in `c1c27f3` (#117) and recorded it in its own repo, in
`docs/design/specs/2026-08-30-artifact-layer-v130-parity.md` §3: *"No byte cap: `source` is the
whole file, or `""` under `--no-artifact-text`."* That decision is what this spec generalises —
python and java are the remaining divergence, and this spec exists so all three agree by record
rather than by coincidence.

**The neutral node is why this cannot be left half-done.** `:Artifact` is language-neutral by
design (`can://artifact/<app>/<path>`, no `Py`/`J`/`TS` prefix). A consumer reading it in a shared
database currently gets `text_truncated` from two analyzers and not from the third — the exact
divergence that motivated putting `source` on the graph in the first place.

## 4. What retires, per analyzer

- the `text_truncated` / `textTruncated` field, in the artifact model and in the graph catalog
- the `--artifact-text-max-bytes` option and its 262144 default
- the truncating branch of text capture (python: `_capture_source`), and with it
- **python's `dependency-manifest` carve-out.** Manifests were exempted from the cap because
  `build_dependency_view` parses `source` and a truncated manifest would silently change the
  dependency graph. With no cap there is no exemption to state: every decodable file is captured in
  full, so the special case disappears rather than moving.

That last point generalises the argument. The cap forced every downstream reader to either carry
the flag or claim an exemption from it; removing the cap removes both obligations.

## 5. Verified: no SDK reads the field

`python-sdk` contains no occurrence of `text_truncated` in `cldk/` — it does not model the field,
so no SDK pin or model change is gated on this. Each analyzer should re-check its own SDK's
artifact model while implementing rather than trusting this line, but the removal is not
conditional on the answer.

## 6. The cost, stated plainly

Uncapped capture means a repository with very large checked-in files produces a correspondingly
larger payload, and that is **unbounded in principle** — the vscode measurement says it is rare,
not that it cannot happen. The mitigation is the flag that stays: `--no-artifact-text` drops the
payload entirely while keeping the inventory, hashes, and sizes.

This is the deliberate trade. A bounded payload whose contents cannot be trusted without a second
field is worse than an unbounded one that is either complete or absent, because the first failure
mode is silent and the second is obvious.

## 7. Decomposition and release plan

Tracking follows PR granularity: one issue per analyzer, one PR each. Both children already exist —
`codeanalyzer-python#172` and `codeanalyzer-java#214` — and this spec's epic adopts them rather
than filing new ones. TypeScript needs no child; #117 shipped its half.

No lockstep is required. Each analyzer removes a field from its own output and cuts its own
release; nothing reads another analyzer's artifacts. The consumer-visible parity arrives when the
last one lands, which is a docs update, not a code dependency.

Order is free. Suggested: python first (the carve-out removal makes it the largest simplification
and it is picked up now), java second, docs after.

## 8. Open questions for review

1. **Should this share a MAJOR bump with the prune-scope work?** `2026-09-02-prune-scope-on-can-id-prefix.md`
   also removes a property (`_module`) from the same Neo4j catalogs, and also takes a MAJOR by each
   analyzer's stated policy. Two removals landing in one release should cost one bump, not two —
   but that depends on whether the two land in the same train per analyzer, which is a release
   question, not a design one.
2. **Does anything outside the SDKs read `text_truncated`?** §5 clears `python-sdk`. `cocoa` and
   `cocoa-ts` consume analyzer output and have not been checked.
