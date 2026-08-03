# Contributing to codellm-devkit

## How we track work

**Tracking granularity follows PR granularity — never step count, never repo count.**

The only question is: *does a pull request close this?*

- **Yes** → it is an issue.
- **No, it is a step inside a PR** → it is a checkbox in that issue's **Goals**.

That one rule replaces counting. A twelve-step plan that lands in one PR is **one issue with
twelve checkboxes**, not twelve issues. A change touching three repos in three PRs is an epic
with three sub-issues — because there are three PRs, not because there are three repos.

### The two templates

| Template | Use when |
| --- | --- |
| **Work item** | One unit of work, closed by one PR. The default. |
| **Epic** | Work spanning repos or several PRs. Holds the design summary and a link to the spec. |

If in doubt, open a Work item. Promoting one to an epic later is cheap; splitting a premature
epic back into nothing is not.

### Sub-issues, not checklists

Epics track children as **native GitHub sub-issues** — use the *Create sub-issue* control on the
epic. Do not hand-maintain a `CHILDREN` checklist, and do not use `Part of #N` trailers; both
drift the moment anything moves, and GitHub now does the rollup for you.

Attaching an existing issue as a sub-issue from the CLI takes the child's **id**, not its number:

```sh
child_id=$(gh api repos/<owner>/<repo>/issues/<child-number> --jq .id)
gh api -X POST repos/<owner>/<repo>/issues/<parent-number>/sub_issues -F sub_issue_id="$child_id"
```

### File issues just-in-time

Open the issue **when you pick up the work**, not when you finish planning. A plan may list ten
future units; unit 3 gets a number when someone starts unit 3.

Issues filed in bulk ahead of the work are inventory, and inventory rots — it goes stale, it
buries the issues that are actually live, and it makes the backlog unreadable. A backlog nobody
can read does not preserve a design record; it hides one.

### Specs and plans are committed

Design docs live in the repo (`docs/superpowers/specs/`, `docs/superpowers/plans/`) and are
committed as **provenance**. An issue *links* the spec it came from; it does not paste the design
into the body. The doc is reviewable in a PR and diffable over time — an issue body is neither.

### Branches and PRs

Each issue gets a branch `<type>/issue-NNN-<short-title>` and one PR that closes it
(`Closes #NNN`). An epic closes when its sub-issues do.

## Writing a good issue

Two sections carry most of the weight, and both templates require them:

- **Scope boundary** — what this issue does *not* do. Usually the most useful sentence in the
  issue; it is what stops a PR sprawling.
- **Definition of done** — exact conditions. Prefer an exact expected set over "non-empty", and a
  demonstrated behaviour over an asserted one. "Works correctly" is not a definition of done.

Cite `file:line` wherever you can. An issue that names the line is one someone can pick up cold.
