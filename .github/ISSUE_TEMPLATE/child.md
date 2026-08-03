---
name: Child issue (ladder rung)
about: A single unit of implementation work on one repo, closed by a single PR
title: ''
labels: ["enhancement"]
assignees: ''
projects_v2: codellm-devkit/1
---

**Problem**
What this repo lacks today, and what this issue adds. One paragraph.

**Scope boundary**
What this issue does NOT do. The provider/client line especially: an analyzer emits the graph and stops — slicing and taint are frontend SDK queries over that graph, not analyzer features.

**Goals**
The contract, as a checklist.
1. 

**Caveats and known risks**
- Substrate or tooling risk — be concrete, and name the workaround.
- Inherited unsoundness or known gaps — documented, not silently absorbed.
- Cost, determinism, and incrementality notes.

**Definition of done**
- An exact-set gate, not "non-empty" — for example, the backward slice on the fixture equals the hand-computed node set.
- Output validates against the SDK v2 models; the parity clause holds.

**Design record**
Link the spec this came from, if there is one. Attach this issue to its epic as a native GitHub sub-issue rather than writing a "Part of #N" trailer here.
