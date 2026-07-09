---
phase: 01-foundations
plan: 04
subsystem: spec-authoring
tags: [demo-narrative, gtm-spec, fnol, decision-logic, competitive-framing]

# Dependency graph
requires:
  - phase: 01-foundations (plan 01)
    provides: Section 4 Mock Data Appendix (canonical claim/policy/multilingual fields)
  - phase: 01-foundations (plan 03)
    provides: Section 3 Decision Logic (DL-1..DL-4 rules and determinism/governance framing)
provides:
  - Section 1 Demo Narrative / Storyline — the single ≤10-15 min beat-by-beat script stitching every locked wow moment into one white-label FNOL run
  - V1-to-V2 delta subsection (NARR-04)
  - White-label / sector generalization subsection (NARR-02)
  - Flagged competitive framing vs. Microsoft Copilot + Nuance (NARR-03)
  - Flagged validation items list (boat-vs-device-bundle, audio-to-audio EN<->ES, backend-reveal custom chain)
affects: [02-component-architecture, 03-use-case-specs, 04-runbook-synthesis]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Narrative beats tagged [DETERMINISTIC — must land] / [FLEX — presenter may improvise] / [CURVEBALL] to separate rehearsable must-land moments from genuine improvisation (Pitfall 11)"
    - "Every spoken dollar figure/coverage term cited inline to its Section 4 field + owning record (traceability discipline, Pitfall 1/4)"
    - "Competitive claims use inline [VALIDATE — Scott/Google] tag rather than an asserted head-to-head table (D-14, Pitfall 12)"

key-files:
  created:
    - .planning/spec/section-1-demo-narrative.md
  modified: []

key-decisions:
  - "Sequenced all 10 locked wow moments (chat intake, coverage lookup, 2 anchor claims, EN<->ES voice switch, empathy, both decision branches, backend reveal, cross-sell) plus 1 FLEX and 1 CURVEBALL beat into a single continuous ≤10-15 min run rather than isolated feature vignettes"
  - "Competitive framing kept to a short prose callout (2 claims, both tagged [VALIDATE — Scott/Google]) rather than a full Copilot/Nuance comparison table, per D-14"
  - "Boat-vs-device-bundle, EN<->ES audio-to-audio support, and the backend-reveal custom-chain nature are explicitly flagged rather than asserted as settled, per D-09/D-11 and CAP-02 Open Questions"

patterns-established:
  - "Beat-level DETERMINISTIC/FLEX/CURVEBALL tagging as the narrative-authoring convention for future use-case sections in Phase 3"

requirements-completed: [NARR-01, NARR-02, NARR-03, NARR-04]

# Metrics
duration: 12min
completed: 2026-07-09
---

# Phase 01 Plan 04: Demo Narrative / Storyline Summary

**Single ≤10-15 min beat-by-beat FNOL demo script sequencing chat intake, back-to-back auto-approve/HITL claims (CLM-24001 $450 / CLM-24002 $2,400), EN↔ES voice switch, empathy adaptation, backend claims-processing reveal, and a personalized cross-sell close, with flagged V1→V2 delta, sector generalization, and Copilot/Nuance competitive framing.**

## Performance

- **Duration:** 12 min
- **Started:** 2026-07-09T20:51:xx Z (approx, per session start)
- **Completed:** 2026-07-09T21:02:50Z
- **Tasks:** 2 completed
- **Files modified:** 1 created

## Accomplishments
- Authored a single coherent beat-by-beat script (12 beats) sequencing every locked wow moment from `01-CONTEXT.md`, with every dollar figure ($100, $3,500, $450, $2,400) cited inline to its Section 4 field/record and both decision branches citing DL-1/DL-2/DL-3 by rule ID
- Wrote the "What V2 Adds Over V1" and "White-Label & Sector Generalization" subsections satisfying NARR-04 and NARR-02
- Appended a short, explicitly flagged competitive-framing callout vs. Microsoft Copilot + Nuance (NARR-03) and a "Flagged Validation Items" list carrying forward the three CONTEXT-level open items (boat-vs-device-bundle, EN↔ES audio-to-audio, backend-reveal custom chain)

## Task Commits

Each task was committed atomically:

1. **Task 1: Beat-by-beat storyline + V1→V2 delta + sector generalization** - `222742c` (docs)
2. **Task 2: Competitive framing (flagged) + validation-item callouts** - `0636ed7` (docs)

_Note: docs(01-04) commit type used per project convention (spec-authoring plan, no code)._

## Files Created/Modified
- `.planning/spec/section-1-demo-narrative.md` - Section 1 of the CX Agent V2 Demo spec: beat-by-beat script, V1→V2 delta, sector generalization, competitive framing, flagged validation items

## Decisions Made
- Ordered the 12 beats (Claude's discretion per CONTEXT) as: intake → coverage lookup → small claim → auto-approve branch → voice+language switch → large claim → empathy → HITL branch → backend reveal → cross-sell → FLEX → CURVEBALL, so the decision-branch contrast (Beat 4 vs. Beat 8) reads as the central dramatic turn of the run
- Placed the CURVEBALL beat as flexible-timing ("after Beat 2 or Beat 6") rather than fixed, since Pitfall 11 only requires it to exist and be genuinely off-script, not occupy a fixed slot
- Kept competitive framing to 2 claims (voice/language reach; backend reveal) rather than enumerating every row from Section 2's Competitor Feature Analysis table, to honor D-14's "light callout, not a full table" constraint

## Deviations from Plan

None - plan executed exactly as written. Both tasks' acceptance criteria and verify blocks pass as specified.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required. This is a spec-authoring plan producing Markdown text only.

## Next Phase Readiness

Section 1 (Demo Narrative) is complete and cross-references Sections 3 and 4 by exact record/rule ID, closing the loop on the four Phase 1 foundational sections (Sections 1-4). Phase 2 (Component Architecture) and Phase 3 (Use-Case Specs) can now cite this narrative's beat numbers and rule/field references directly rather than re-deriving the storyline. The three flagged validation items (boat-vs-device-bundle, EN↔ES audio-to-audio, backend-reveal custom chain) should be carried into Phase 4's Open Questions/Risks section (AC-03) rather than being silently resolved.

## Self-Check: PASSED

- FOUND: .planning/spec/section-1-demo-narrative.md
- FOUND commit 222742c in git log
- FOUND commit 0636ed7 in git log
- Both task-level verify grep checks passed (TASK1 PASS, TASK2 PASS confirmed above)

---
*Phase: 01-foundations*
*Completed: 2026-07-09*
