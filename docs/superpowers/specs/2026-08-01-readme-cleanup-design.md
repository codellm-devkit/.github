# Org Profile README Cleanup

**Repo:** `codellm-devkit/.github`
**File:** `profile/README.md` (renders at github.com/codellm-devkit)
**Date:** 2026-08-01, revised 2026-08-03

## Problem

The org profile README carries links that no longer resolve to what they claim,
repeats itself, and has grown section-by-section without a governing shape.

1. **Stale repo names.** Two links survive only because GitHub redirects
   renamed repositories. If either name is reclaimed, the link breaks silently.
2. **A community link that works only for people who don't need it.** The
   Discord URL is a client deep-link into a specific channel; it resolves for
   existing members and fails for everyone else — exactly inverting its purpose.
3. **Broken logo markup.** The `<picture>` element's fallback `<img>` has no
   `src` attribute and a placeholder `alt` of `"Logo"`.
4. **Redundancy.** A whole section duplicates bullets that appear 20 lines
   above it, and a second code example restates the first with two words
   changed.

Alongside these, several links resolve to near-empty repositories or a package
published once as `0.0.0-pre-alpha` in April 2025. These are not broken, but
they spend the reader's attention on nothing.

## Approach

Adopt the structure of the Model Context Protocol org profile README
(`github.com/modelcontextprotocol/.github/blob/main/profile/README.md`): logo,
tagline, nav, one intro paragraph, `Getting Started` as a short emoji bullet
list, `Project Structure` as a flat `repo-name - description` list, then
`Contributing`.

Two deliberate departures from MCP, because CLDK is a library and MCP is a
protocol:

- **Keep the badge row.** MCP has none. CLDK's version, PyPI, and paper badges
  carry information a library's landing page should lead with.
- **Keep one code example.** MCP's call to action is "read the spec"; CLDK's is
  `pip install cldk`. The example is the strongest hook on the page.

CLDK-specific tail sections (Reference, IBM disclosure, Contact) have no MCP
equivalent and are appended after `Contributing`.

## Link audit

Verified 2026-08-01 by HTTP status, redirect inspection, npm registry query,
and the Discord invites API.

### Broken or stale

| Link | Finding | Action |
| --- | --- | --- |
| `github.com/codellm-devkit/codeanalyzer-ts` | 301 → `codeanalyzer-typescript` | Rewrite to canonical name |
| `github.com/codellm-devkit/cocoa-py` | 301 → `cocoa-mcp` | Removed — see COCOA below |
| `discord.com/channels/1333486179667935403/1334150434348208208` | Client deep-link; resolves only for members | Replace with `discord.gg/zEjz9YrmqN` (verified live, never expires) |
| `mailto:rangeet.pan@gmail.com` | Display text reads `rangeet.pan@ibm.com` — mailto disagrees | Point mailto at the displayed IBM address |
| `<img alt="Logo">` | No `src`; `<picture>` has no working fallback | Add `src="assets/cldk-light.png"`, `alt="CodeLLM-DevKit"` |

### Live but not worth linking

| Link | Finding | Action |
| --- | --- | --- |
| `npmjs.com/package/@codellm-devkit/cldk` | Latest dist-tag `0.0.0-pre-alpha`, published 2025-04-25, never updated | Drop the inline link; mark the SDK experimental |
| `github.com/codellm-devkit/codeanalyzer-go` | 124 KB stub | Drop the link; keep Go in the support sentence |
| `github.com/codellm-devkit/codeanalyzer-rust` | 93 KB stub, last pushed 2025-02-14 | Drop the link; keep Rust in the support sentence |

### Verified healthy — leave alone

`codellm-devkit.info`, `codellm-devkit.info/quickstart/`, `badge.fury.io/py/cldk`,
`arxiv.org/abs/2410.13007`, `opensource.org/licenses/Apache-2.0`,
`github.com/orgs/codellm-devkit/discussions`, `pypi.org/project/cldk/`,
`python-sdk`, `typescript-sdk`, `codeanalyzer-java`, `codeanalyzer-python`,
`codeanalyzer-typescript`, `cldk-tutorial`.

`CONTRIBUTING.md` is a relative link resolving to `profile/CONTRIBUTING.md`,
which exists. Leave as-is.

## Target structure

```
<picture>            logo, dark/light sources, img fallback repaired
badges               docs · pypi · arXiv · license   (npm comment block deleted)
tagline              one centered sentence
nav                  Documentation | Quickstart | Discussions
intro paragraph      unchanged
support sentence     Java/Python/TypeScript today; Go/Rust/C in development

## Getting Started
## Project Structure
## Contributing
## Reference
## IBM Public Repository Disclosure
## Contact
```

No `# CodeLLM-DevKit` H1. MCP has one, but the CLDK logo is a wordmark already
reading "Codellm-Devkit"; an H1 would stack a title on a title.

### Getting Started

Three bullets, MCP's emoji style:

- 📚 Read the Documentation for guides, core concepts, and common tasks
- ✨ Browse usage examples and tutorials
- 💻 Install and query, with `pip install cldk` and the existing Java example
  nested beneath, followed by one sentence: only the `language` argument
  changes across languages.

### Project Structure

Flat `repo-name - description` list, MCP's format:

- `python-sdk` - official Python SDK (`cldk` on PyPI)
- `typescript-sdk` - official TypeScript SDK (experimental)
- `codeanalyzer-java` - static analysis for Java/JavaEE using WALA and JavaParser
- `codeanalyzer-python` - static analysis backend for Python
- `codeanalyzer-typescript` - static analysis backend for TypeScript/JavaScript
- `cldk-tutorial` - start building with CLDK

## Removed

- **The `Supported Languages` table.** Its backend links move into
  `Project Structure`; its status information collapses to one sentence after
  the intro paragraph.
- **The `Language-Specific Analysis Backends` section.** Absorbed into
  `Project Structure`.
- **The `Build with CLDK` section, and every COCOA reference.** The org now has
  three COCOA repos (`cocoa`, `cocoa-mcp`, `cocoa-ts`) where the README
  describes one. Removed for now rather than picking a winner on the landing
  page.
- **The `Documentation and Examples` section.** Its two bullets already appear
  under Getting Started.
- **The second code example.** It differs from the first only in `language=`
  and `project_path`. The surviving example is the existing Java one, unchanged;
  the removed Python example becomes one sentence.
- **The commented-out npm badge block.** Dead markup, and npm is not linked
  anywhere else.

## Formatting

Follow MCP's separators, which reverts uncommitted working-tree changes.
Confirmed with the maintainer.

| Element | Current working tree | Target |
| --- | --- | --- |
| Nav separator | `<strong>` links joined by `•` | Plain links joined by `\|` |
| List separator | `•` between name and description | ` - ` |

## Deliberate non-changes

- **TypeScript is "Strong support."** The support sentence groups it with Java
  and Python; the `typescript-sdk` bullet is separately marked experimental.
  These describe different artifacts and do not contradict each other.
- **The Contact table stays**, despite Discussions and Discord covering similar
  ground.
- **The IBM disclosure stays** — likely organizationally required.
- **The Reference BibTeX stays.**
- **No prose rewriting** beyond what removal requires. The intro paragraph, the
  tagline, and the retained section wording are unchanged.

## Other fixes

- "Saurabh Sihna" → "Saurabh Sinha" in the Contact table.

## Success criteria

1. Every URL in the rendered README returns 200 without relying on a redirect.
2. No link targets a repository under ~200 KB or a package whose latest release
   is a pre-alpha.
3. No section duplicates content from another section.
4. Every `mailto:` matches the address displayed next to it.
5. No occurrence of "cocoa" in any form.
6. The logo renders in both light and dark themes, and its `<img>` fallback has
   a `src` and a descriptive `alt`.
7. Section order and list formatting match the target structure above.

## Delivery

One repo, one issue, one PR.

- **One issue**, filed with `.github/ISSUE_TEMPLATE/docs.md`. No epic, no child
  issues — this is `maintaining-cldk` work. The spec is not pasted into the
  issue; the issue references it, and the spec stays local (gitignored).
- **`docs.md` does not exist yet.** It is added by the companion spec,
  `2026-08-03-issue-templates-and-plugin-design.md`, which must land first.
  This is the only ordering dependency between the two.
- Template frontmatter (`labels:`, `projects_v2:`) is applied by GitHub's web
  form, not by the REST API that `gh` uses. Filing with `gh issue create`
  requires passing `--label documentation --project codellm-devkit/1`
  explicitly to get the same result.
- **PR** uses the repo's `pull_request_template.md`, which auto-populates. Tick
  **Documentation update** under Types of changes. "New and existing tests pass
  locally" and "I have added appropriate error handling" are N/A for a docs-only
  change and should be marked N/A rather than checked.

## Out of scope

- `profile/CONTRIBUTING.md` and `profile/CODE_OF_CONDUCT.MD` content.
- Issue and PR templates, and every cldk-devtools plugin change — all covered
  by `2026-08-03-issue-templates-and-plugin-design.md`.
- Deciding the future of the three COCOA repos. Removing the references
  sidesteps the question; it does not answer it.
