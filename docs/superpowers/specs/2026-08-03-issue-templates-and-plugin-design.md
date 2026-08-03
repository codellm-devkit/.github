# Org Issue Templates + cldk-devtools Reconciliation

**Repos:** `codellm-devkit/.github`, `codellm-devkit/cldk-devtools`
**Date:** 2026-08-03

Companion to `2026-08-01-readme-cleanup-design.md`. That spec cleans up the org
profile README; this one fixes the issue-filing machinery the README work exposed.
This spec must land first — the README issue is filed with a template added here.

## Problem

Three separate defects, discovered while deciding how to file one small issue.

### 1. The org has two issue templates, both shaped for external reporters

`.github/ISSUE_TEMPLATE/` holds `bug_report.md` ("To Reproduce", "Logs") and
`feature_request.md`. Neither fits work a maintainer files against their own
audit, and there is no docs template at all — so a link-rot cleanup either
pretends to be a bug or goes freeform.

Verified: none of `python-sdk`, `typescript-sdk`, `codeanalyzer-java`,
`codeanalyzer-python`, `codeanalyzer-typescript`, `cldk-devtools`, or `cocoa`
defines its own `.github/ISSUE_TEMPLATE` or `pull_request_template.md`. All seven
inherit from this repo. These org templates are not merely defaults — they are
the only templates anywhere in the org, so a gap here is an org-wide gap.

### 2. The plugin's `gh issue create` invocations silently drop labels and project

`skills/designing-cldk-changes/references/epic-and-issue-templates.md` uses
`--body-file` throughout, and line 135 makes that an explicit rule so multi-line
templates survive intact. That rule is correct on its own terms, but it means:

- Issue templates never apply. Templates pre-fill GitHub's **web form**; the REST
  API that `gh` uses does not consult them.
- The `labels:` and `projects_v2: codellm-devkit/1` frontmatter therefore never
  applies either.

The epic invocation compensates by hand — line 114 passes `--label epic`, with a
comment to create the label if needed. The child invocations (lines 119–129) pass
no `--label` at all, and **nothing** passes `--project`. So every child issue the
plugin has filed is unlabeled, and every issue it has filed, epic or child, is
absent from project board `codellm-devkit/1`.

"Use the org templates" cannot mean invoking them from a script; it means
matching their structure in the body and passing `--label` / `--project`
explicitly.

### 3. The plugin carries its own issue body forms, diverging from the org's

The plugin defines ALL-CAPS sectioned forms (`SUMMARY`, `AFFECTED REPOS`,
`SCOPE BOUNDARY`, `CAVEATS AND KNOWN RISKS`, `DEFINITION OF DONE`) that exist
nowhere in the org templates. Two sources of truth for the same artifact.

### Not the problem

`maintaining-cldk` was suspected of driving epic-and-children overhead for small
work. It does not: `SKILL.md:15` already says "No spec and no epic are needed to
start," and `finishing-cldk-work/references/docs-and-closeout.md:41` already says
most maintaining entries close with a single issue. The unconditional
"spec ⇒ epic + children" rule lives in the user's global `CLAUDE.md`, which is
out of scope here. The plugin's real gap is narrower — see change 4 below.

## Approach

Make the org templates the single source of truth. Add the three shapes the org
is missing, then strip the duplicate body forms out of the plugin so it points at
the org templates and keeps only what is genuinely ladder-specific: the
one-child-per-rung rule, the placement convention, and the `gh` invocations.

## Change 1 — add `epic.md`

`.github/ISSUE_TEMPLATE/epic.md`. Frontmatter: `name: Epic`,
`title: 'Epic: '`, `labels: ["epic"]`, `projects_v2: codellm-devkit/1`.

Sections, converted from the plugin's epic form to the org's `**Bold**` heading
style:

- **Summary** — what changes and why, 2–4 sentences; name the schema-v2 impact
  explicitly ("adds a `comment` body-node kind" / "no schema change, SDK surface
  only").
- **Affected repos** — one line per repo: repo, role, ladder rung.
- **Design decisions** — locked with the user before build starts, including a
  scope guard naming what is explicitly out of scope.
- **Children** — checklist, one entry per ladder rung / PR-unit.
- **Definition of done** — epic-level gates: every child merged and green,
  analyzer output validating against SDK v2 models, `L1 ⊆ … ⊆ L4` superset gate,
  parity clause, versions pinned in lockstep, docs updated.

The `epic` label does not exist in the org yet and must be created once
(`gh label create epic`).

## Change 2 — add `child.md`

`.github/ISSUE_TEMPLATE/child.md`. Frontmatter: `name: Child issue (ladder rung)`,
`labels: ["enhancement"]`, `projects_v2: codellm-devkit/1`.

Preserves the sections that make an implementation issue honest:

- **Problem** — what this repo lacks today and what this adds, one paragraph.
- **Scope boundary** — what this issue does NOT do; the provider/client line
  especially (an analyzer emits the graph and stops; slicing and taint are
  frontend SDK queries, out of scope on the analyzer).
- **Goals** — the contract, as a numbered checklist.
- **Caveats and known risks** — substrate/tooling risk with its workaround,
  inherited unsoundness documented rather than silently absorbed, cost and
  determinism notes.
- **Definition of done** — exact-set gates, never "non-empty".
- **Design record** — a link to the originating spec. Epic linkage is a native
  GitHub sub-issue relationship, attached with
  `gh api -X POST repos/<owner>/<repo>/issues/<parent>/sub_issues -F sub_issue_id=<child_id>`,
  not a `Part of #N` trailer in the body.

## Change 3 — add `docs.md`

`.github/ISSUE_TEMPLATE/docs.md`. Frontmatter: `name: Documentation`,
`labels: ["documentation"]`, `projects_v2: codellm-devkit/1`.

Sections: **What's wrong** (which page or file, and what is inaccurate, missing,
or broken) / **Where** (path or URL) / **Expected** (what it should say or link
to instead) / **Additional context**.

Deliberately short. A docs issue that needs "Logs" is a bug report.

The `documentation` label may not exist in the org and must be created once if
absent.

## Change 4 — `maintaining-cldk/SKILL.md`

Two edits, unrelated to templates but discovered in the same exercise.

1. **Entry Preconditions** — add the missing entry path. Work may arrive from
   brainstorming with a spec already written and no issue filed. That is a
   normal entry, not a promotion to design mode: file one standalone issue and
   continue. The spec stays where it is and is not pasted into the issue.

2. **Red Flags** — add a row:

   | "Brainstorming produced a spec, so this needs an epic." | Spec ≠ epic. The epic gate belongs to `designing-cldk-changes` and is triggered by contract impact, not by the existence of a written design. |

## Change 5 — `epic-and-issue-templates.md`

Rewrite to reference rather than duplicate.

**Removed:** the inline epic template (lines 39–68) and child-issue template
(lines 70–104). Replaced by pointers to `epic.md` and `child.md` in the org
`.github` repo, with a one-line note that the org copy is authoritative and this
file must not restate it.

**Kept:** the shape description, the triage-table-to-child mapping, the
one-child-per-rung rule, the placement convention, and the worked
native-dataflow example. These are ladder-specific and have no org equivalent.

**Superseded:** the `Part of <owner>/<epic-repo>#<epic-number>` trailer
convention (line 37, and the trailing comments on the child invocations). Epic
linkage is now a native GitHub sub-issue relationship. Also revisit the
hand-maintained `CHILDREN` checklist in the epic form — sub-issues track children
natively, so the checklist is now duplicate bookkeeping.

**Fixed:** every `gh issue create` invocation gains explicit `--label` and
`--project codellm-devkit/1`, with a note explaining why — `--body-file` bypasses
template frontmatter, so the flags are how the label and board membership
actually get applied. The epic invocation already passes `--label epic` and needs
only `--project`; the three child invocations need both. The existing
`--body-file` rule stays; it is correct.

`tests/baselines/designing-cldk-changes/s1.md` contains the epic body form and
will need regenerating when the inline templates are removed.

## Plugin repo mechanics

No local clone exists. The version read during design was the installed cache at
`~/.claude/plugins/cache/codellm-devkit/cldk-devtools/0.2.0`, which is
overwritten on plugin update. **Do not edit the cache.** Clone
`codellm-devkit/cldk-devtools` and work there.

Before opening the plugin PR:

- `tests/baselines/maintaining-cldk/{s1,s2}.md` may need regenerating after the
  `SKILL.md` edits. Unread at design time — confirm, do not assume.
- `tests/consistency/check-readme-dispatcher-sync.sh` mirrors the dispatcher's
  routing table into the plugin README. No change here touches that table, so it
  should stay green. Run it and confirm.
- Whether the plugin version needs bumping from `0.2.0` is a repo convention
  question; follow `cldk-devtools`' own `CLAUDE.md`.

## Delivery

```
PR A  .github         epic.md + child.md + docs.md   (+ create epic/documentation labels)
PR B  cldk-devtools   maintaining-cldk edits + epic-and-issue-templates rewrite
```

PR A must merge before the README issue in the companion spec can be filed under
`docs.md`. PR B depends on PR A only in that it references templates that should
exist by then.

Each PR uses the org `pull_request_template.md`. PR A is a **Documentation
update**; PR B is a **Documentation update** as well — both are content-only, no
runtime behavior. Mark the tests and error-handling checklist items N/A rather
than checking them, unless the plugin's baseline tests do change, in which case
"New and existing tests pass locally" genuinely applies to PR B.

## Success criteria

1. `.github/ISSUE_TEMPLATE/` contains five templates; the three new ones render
   in GitHub's issue-picker with correct names and labels.
2. Every label referenced in template frontmatter exists in the org.
3. `epic-and-issue-templates.md` contains no inline issue body template — only
   pointers to the org templates.
4. Every `gh issue create` invocation in the plugin passes `--label` and
   `--project`.
5. `maintaining-cldk/SKILL.md` describes the brainstorming-with-a-spec entry and
   carries the spec-≠-epic red flag.
6. The plugin's consistency check and baseline tests pass.

## Out of scope

- The user's global `CLAUDE.md` and its unconditional "spec ⇒ epic" rule. Scoping
  it to structural work would fix the root collision; that file is the user's.
- `ISSUE_TEMPLATE/config.yml` — the org allows blank issues and has no
  contact-links routing. Both are defensible defaults; changing them is a
  separate decision.
- `pull_request_template.md` content.
- `workflow-templates/release_config.json`, which is 0 bytes and looks like a
  defect. Noted during the audit, unrelated to issue filing, not investigated.
- Other ladder rungs. Only `maintaining-cldk` and `designing-cldk-changes`'
  reference file change.
- Retrofitting labels or project membership onto issues the plugin filed before
  this fix.
