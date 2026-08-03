---
name: Epic
about: Cross-repo coordination record for a structural CLDK change
title: 'Epic: '
labels: ["epic"]
assignees: ''
projects_v2: codellm-devkit/1
---

**Summary**
What changes and why, in 2-4 sentences. Name the schema v2 impact explicitly — for example "adds a `comment` body-node kind", or "no schema change, SDK surface only".

**Affected repos**
One line per repo: which repo, its role in this change, and its ladder rung.
- 

**Design decisions**
Decisions locked before build starts.
- 
- Scope guard: what is explicitly out of scope for this change.

**Design record**
Link the spec this epic came from, rather than restating it here.

**Children**
Attach each child as a native GitHub sub-issue — do not maintain a checklist here. One child per ladder rung or PR-unit, filed just-in-time when that unit is picked up.

**Definition of done**
- Every child PR merged with its gate green.
- Analyzer output validates against the SDK v2 models at its max level; the L1 ⊆ … ⊆ L4 superset gate holds.
- SDK public API unchanged, or the major bump and its shims are documented.
- Docs and CHANGELOG updated; versions pinned in lockstep.
