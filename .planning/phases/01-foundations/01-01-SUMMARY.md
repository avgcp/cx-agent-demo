---
phase: 01-foundations
plan: 01
subsystem: mock-data-appendix
tags: [spec-authoring, mock-data, fnol, decision-logic-inputs]
dependency_graph:
  requires: []
  provides:
    - ".planning/spec/section-4-mock-data-appendix.md"
    - "canonical field registry (claim_amount, deductible, coverage_limit, total_loss_flag, data_loss_flag, uninsured_device, policy_id, claim_id, damage_image_ref)"
  affects:
    - "Section 1 (Demo Narrative) — must reference these record IDs/values, not invent new ones"
    - "Section 3 (Decision Logic) — thresholds and routing rules must cite this dataset"
tech_stack:
  added: []
  patterns:
    - "Every on-stage dollar figure/coverage term traces to a named mock-data field via the Traceability Field Registry"
    - "Negative/over-fire records seeded per branch to catch false-positive escalation/cross-sell"
key_files:
  created:
    - ".planning/spec/section-4-mock-data-appendix.md"
  modified: []
decisions:
  - "Carrier named 'Meridian Device Protection' (D-03 generic/white-label placeholder)"
  - "Two anchor claims CLM-24001 ($450, auto-approve) and CLM-24002 ($2,400, HITL/total-loss) straddle the literal $1,000 threshold (D-04, D-05)"
  - "CLM-24003 ($950) and claimant Sam Okafor seeded as over-fire negative controls (DATA-02)"
  - "EN<->ES(US) trigger phrase verbatim: '¿Podemos continuar en español?' (D-10)"
metrics:
  duration_minutes: 15
  completed: "2026-07-09"
---

# Phase 01 Plan 01: Mock Data Appendix Summary

Authored the canonical, versioned seed dataset (v1.0) that every other spec section must cite by named field/record rather than inventing values — carrier, claimant, policy/coverage, two anchor laptop claims straddling the $1,000 threshold, over-fire negative records, a pre-validated damage-image library, verbatim EN↔ES(US) trigger phrases, and a traceability registry mapping every dollar figure to its owning record.

## What Was Built

`.planning/spec/section-4-mock-data-appendix.md` — a single Markdown file containing:

1. **Synthetic-Data Standard** — mandates fictitious names, `555-01xx` phone numbers, `.test`/`example-*` email domains, no valid-format SSNs, and an explicit prohibition on presenters improvising real personal details live.
2. **Carrier** — Meridian Device Protection (generic, white-label, swappable placeholder per D-03).
3. **Primary claimant profile** — Jordan Rivera / PDP-100294, with a seeded uninsured smartphone driving the cross-sell moment (D-08).
4. **Policy & coverage** — $100 deductible, $3,500 coverage limit, coverage valid.
5. **Two anchor laptop claims run back-to-back** (D-04): CLM-24001 ($450, cracked screen, auto-approve) and CLM-24002 ($2,400, liquid-damaged/total loss/data loss, always-escalates HITL per D-05) — straddling the literal $1,000 threshold.
6. **Negative/over-fire records** (DATA-02): CLM-24003 ($950, near-threshold, must auto-approve — escalation over-fire check) and claimant Sam Okafor (fully insured, no uninsured device — cross-sell over-fire check).
7. **Pre-validated damage-image library** (DATA-03): 5 named image refs (IMG-CRACK-01/02/03, IMG-LIQUID-01/02) each with an expected damage-analysis output, with spares held in reserve as swap-in recovery images.
8. **Multilingual test phrases** (DATA-03): verbatim English FNOL open, the exact language-switch trigger phrase (`¿Podemos continuar en español?`), and the Spanish FNOL phrase, with a note that unconfirmed language pairs use the captioned-text fallback (D-11).
9. **Traceability Field Registry** (DATA-04): every dollar figure appearing anywhere in the file ($100, $450, $950, $1,000, $2,400, $3,500) mapped to its canonical named field and owning record, with the explicit rule that nothing may be spoken on stage unless it appears in this registry.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Canonical seed records + synthetic-PII standard | 5eb598b | .planning/spec/section-4-mock-data-appendix.md |
| 2 | Negative/over-fire records, damage-image library, multilingual phrases, traceability registry | 297cc17 | .planning/spec/section-4-mock-data-appendix.md |

## Deviations from Plan

None — plan executed exactly as written. All literal values specified in the plan (carrier name, claimant profile, policy/coverage figures, both anchor claims, both negative records, all image refs, all phrases) were transcribed verbatim into the spec file; no architectural changes, bug fixes, or scope additions were required.

## Verification

- `grep -q "## Mock Data Appendix"` — pass
- `grep -q "CLM-24001"` / `"CLM-24002"` — pass (10 combined occurrences)
- `grep -q "total_loss_flag"` — pass (true on CLM-24002, false on CLM-24001/CLM-24003)
- `grep -q "555-01"` — pass
- `grep -q "### Negative / Over-Fire Records"` / `"CLM-24003"` — pass
- `grep -q "IMG-CRACK-01"` / `"IMG-LIQUID-01"` — pass
- `grep -q "continuar en español"` — pass
- `grep -q "### Traceability Field Registry"` — pass
- All dollar figures in the file ($100, $450, $950, $1,000, $2,400, $3,500) confirmed present as rows in the Traceability Field Registry
- $450/$950 confirmed below $1,000; $2,400 confirmed above $1,000; both sides of the threshold are seeded

## Known Stubs

None. This plan produces spec text only (per the plan's `<threat_model>` — spec-authoring phase, no runtime/attack surface). Damage-image references (IMG-CRACK-01..03, IMG-LIQUID-01..02) are explicitly documented as placeholder references the implementation team must attach and re-validate before the live demo — this is a stated, intentional deferral (not a gap) called out directly in the "Pre-Validated Damage-Image Library" subsection.

## Threat Flags

None. Per the plan's threat model, this is spec-authoring only — no code, service, or live data path was introduced.

## Self-Check: PASSED

- FOUND: .planning/spec/section-4-mock-data-appendix.md
- FOUND: 5eb598b (git log --oneline --all)
- FOUND: 297cc17 (git log --oneline --all)
