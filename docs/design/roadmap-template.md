# <org or area> roadmap

<!-- Copy this file to docs/design/roadmap.md and fill it in. One roadmap, amended in
     place — do not add a second. Git history is the record of how thinking changed. -->

**Pass:** <date> · **Planned with:** <who>

## Candidates

<!-- One row per contract decision, not per theme. A theme ("microservice analysis")
     is several rows. -->

| # | Candidate | Moves schema? | Collision group | Blocked by |
| - | --------- | ------------- | --------------- | ---------- |
| 1 | <one line> | <yes — what vocabulary / no> | <A / —> | <# / —> |
| 2 | <one line> | <> | <> | <> |

## Collision groups

<!-- Candidates touching the same schema vocabulary. Each group is one design session
     for the shared part, however far apart the candidates ship. -->

- **<A — vocabulary name>**: candidates <#, #>
  - Decide once: <kinds/fields, and who emits them>
  - Carried by: <which candidate>

## Dependency order

<!-- What unblocks what. A graph, not a list. -->

    <1> ──▶ <2> ──▶ <3>

## Release trains

| Train | Carries | Notes |
| ----- | ------- | ----- |
| <repo x.y.0> | <#> | <schema major? lockstep constraint?> |

## Not now

<!-- Excluded this pass, with the reason. Deleting a row loses the fact it was
     considered. -->

- <candidate> — <blocked / not yet worth it / superseded / no owner>

## Starting now

<!-- The single candidate entering design mode, and its epic link once filed.
     Everything else here has no issue yet, by design. -->

- <candidate> — <epic link>
