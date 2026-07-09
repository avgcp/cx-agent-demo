---
phase: 01-foundations
plan: 03
subsystem: decision-logic
tags: [spec-authoring, decision-logic, determinism, governance, fnol]
dependency_graph:
  requires:
    - phase: "01-foundations (plan 01)"
      provides: ".planning/spec/section-4-mock-data-appendix.md — canonical claim/policy records CLM-24001, CLM-24002, CLM-24003, claimant profiles"
  provides:
    - ".planning/spec/section-3-decision-logic.md"
    - "Business-Rules Table (DL-1..DL-4) as the single source of truth for routing"
    - "Determinism Mandate naming the Python-tool + Handoff Rule pattern, explicitly excluding RAG/LLM judgment"
    - "Governance/audit-trail framing forbidding 'the AI decides' language"
  affects:
    - "Phase 2 (Component Architecture) — must implement DL-1..DL-4 via the mandated Python tool (evaluate_claim) + Handoff Rule pattern, not RAG or instruction-only logic"
    - "Phase 3 (Use-Case Specs) — decision-branch and cross-sell use cases must cite DL-1..DL-4 by Rule ID rather than re-deriving thresholds"
tech_stack:
  added: []
  patterns:
    - "Every routing rule expressed as a literal if/then row citing a named Section 4 record — no prose ambiguity"
    - "Deterministic Python tool computes decision -> writes session variable -> Handoff Rule routes (never RAG/LLM judgment on the threshold)"
    - "Every auto-approve outcome paired with a visible audit-trail artifact (rule reference + transcript) for compliance/accountability framing"
key_files:
  created:
    - ".planning/spec/section-3-decision-logic.md"
  modified: []
decisions:
  - "DL-1..DL-4 authored verbatim from D-13's four literal if/then rows (auto-approve, human-assessor, total-loss always-escalate, cross-sell)"
  - "DL-3 (total_loss_flag) documented as independently sufficient and non-mutually-exclusive with DL-2 for CLM-24002, to avoid ambiguity about rule precedence"
  - "$1,000 threshold framed as illustrative/configurable ('cost of a manual adjuster touch') per D-12, with an explicit ban on 'the AI decides' language per DEC-03/Pitfall 7"
metrics:
  duration_minutes: 12
  completed: "2026-07-09"
---

# Phase 01 Plan 03: Decision Logic Spec Summary

Authored the literal if/then business-rules table (DL-1..DL-4) governing the $1,000 auto-approve/escalate threshold, the total-loss always-escalate flag, and the cross-sell trigger, plus the determinism mandate (deterministic Python tool + Handoff Rule, never RAG/LLM judgment) and the governance/audit-trail framing that bans "the AI decides" language.

## Performance

- **Tasks:** 2 completed
- **Files modified:** 1 created

## Accomplishments
- Business-Rules Table with DL-1..DL-4, each citing the exact Section 4 record(s) that exercise it, including both negative/over-fire controls (CLM-24003, Sam Okafor)
- Determinism Mandate (DEC-02) naming `evaluate_claim` (Python tool) + Handoff Rule on a session variable, explicitly excluding RAG/Data-Store lookup and instruction-only LLM judgment, citing Anti-Pattern 1
- Governance, Configurability & Audit Trail subsection (DEC-03): $1,000 rationale as illustrative/configurable "cost of a manual adjuster touch"; explicit ban on "the AI decides"; visible audit-trail artifact (rule reference + transcript) as both a wow moment and the compliance answer; secondary factors beyond dollar amount noted

## Task Commits

Each task was committed atomically:

1. **Task 1: Literal if/then rules table + determinism mandate** - `b04efdd` (docs)
2. **Task 2: Audit-trail + configurable-rationale framing (DEC-03)** - `bee8c2b` (docs)

## Files Created/Modified
- `.planning/spec/section-3-decision-logic.md` - Business-Rules Table (DL-1..DL-4), Determinism Mandate (DEC-02), Governance/Configurability/Audit Trail (DEC-03)

## Decisions Made
- Added an explicit "Notes on precedence and negatives" paragraph (not separately called out in the plan's acceptance criteria, but needed to avoid rule-precedence ambiguity given CLM-24002 satisfies both DL-2 and DL-3 simultaneously) — kept as spec-clarity content, not a scope addition, since it directly serves DEC-01's "no prose ambiguity" requirement.

## Deviations from Plan

None — plan executed exactly as written. All four rule rows, the determinism mandate, and all four governance sub-points were authored per the plan's literal instructions and D-12/D-13 source decisions; no architectural changes, bug fixes, or scope additions were required beyond the precedence clarification noted above (Rule 2 — auto-add missing critical clarity to prevent ambiguity, itself directly required by DEC-01).

## Verification

- `grep -q "## Decision Logic"` — pass
- `grep -q "DL-1"` / `"DL-4"` — pass (7 combined DL-N occurrences via `grep -c`)
- `grep -q "1,000"` — pass
- `grep -q "total_loss_flag"` — pass
- `grep -q "Handoff Rule"` — pass
- `grep -qi "not.*RAG\|never.*RAG\|not a RAG"` — pass
- `grep -q "CLM-24001"` / `"CLM-24002"` / `"CLM-24003"` — all pass, all three anchor/negative records cited
- `grep -q "### Governance, Configurability & Audit Trail"` — pass
- `grep -qi "manual adjuster touch"` — pass
- `grep -qi "configurable"` — pass
- `grep -qi "audit"` — pass
- `grep -qi "AI decides"` — pass
- Post-commit deletion check: no files deleted by either commit

## Known Stubs

None. This plan produces spec text only (per the plan's `<threat_model>` — spec-authoring phase, no runtime/attack surface).

## Threat Flags

None. Per the plan's threat model, this is spec-authoring only — no code, service, or live data path was introduced.

## Next Phase Readiness

- Section 3 (Decision Logic) is complete and internally consistent with Section 4 (Mock Data Appendix): all three anchor/negative claim records (CLM-24001, CLM-24002, CLM-24003) and both claimant profiles (Jordan Rivera, Sam Okafor) are cited correctly.
- Phase 2 (Component Architecture) can now implement DL-1..DL-4 directly against the mandated Python-tool + Handoff Rule pattern without re-deriving thresholds.
- No blockers.

## Self-Check: PASSED

- FOUND: .planning/spec/section-3-decision-logic.md
- FOUND: b04efdd (git log --oneline --all)
- FOUND: bee8c2b (git log --oneline --all)

---
*Phase: 01-foundations*
*Completed: 2026-07-09*
