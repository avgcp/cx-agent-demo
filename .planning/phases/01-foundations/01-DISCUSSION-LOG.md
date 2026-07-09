# Phase 1: Foundations - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-07-09
**Phase:** 1-Foundations
**Areas discussed:** Anchor scenario & sector, Language-switch pair, Decision threshold & triggers, Competitive framing depth

---

## Gray Areas Selected

User selected all four offered gray areas to discuss:

| Area | Description | Selected |
|------|-------------|----------|
| Anchor scenario & sector | Insurance line, concrete loss event, claimant persona | ✓ |
| Language-switch pair | Which language(s) the mid-demo voice switch uses | ✓ |
| Decision threshold & triggers | $1,000 ceiling, rationale, escalation + cross-sell triggers | ✓ |
| Competitive framing depth | How prominent the Copilot/Nuance framing is | ✓ |

---

## Anchor Scenario — Insurance Line

| Option | Description | Selected |
|--------|-------------|----------|
| Personal auto | Car-accident FNOL; cleanest, most universal; boat cross-sell as second vehicle | (initially picked, then superseded) |
| Homeowner / property | Home-damage FNOL; matches Google's "damaged appliance" example | |
| Multi-line personal carrier | Frame carrier as multi-line, run one claim type | |

**User's choice:** Initially "Personal auto," then redirected via free text: *"can we do computer loss / warranty the team already built it"* — reuse the V1 computer/electronics-loss asset the implementation team already built.
**Notes:** Reuse minimizes implementation lift and makes the V1→V2 continuity story literal. Then user said *"decide what is best"*, delegating the reconciliation to Claude.

## Anchor Scenario — Loss Event (offered before the pivot)

| Option | Description | Selected |
|--------|-------------|----------|
| Parking-lot / low-speed collision | Cosmetic vs. heavy damage; passenger whiplash injury variant | |
| Intersection collision | More serious crash; harder to make auto-approve believable | |
| Weather / hail damage | Great photo moment; injury less natural | |
| (Free text) computer loss / warranty | Reuse V1's already-built computer-loss FNOL | ✓ |

**User's choice:** Free-text redirect to computer-loss reuse (see above).

## Reconciliation Decisions (Claude's discretion, per "decide what is best")

Claude resolved four ripple effects of the scenario pivot and presented them for confirmation:

1. **Insurance vs. warranty framing** → **insurance** (deductible/coverage/threshold vocabulary preserved).
2. **Two claim sizes** → cracked-screen (auto-approve) vs. liquid-damaged high-end machine (HITL), run back-to-back.
3. **Escalation flag** → "injury" replaced by deterministic "total-loss / business-critical device with data loss" flag; sentiment/empathy add-on rides on customer distress.
4. **Cross-sell** → "uninsured boat" replaced by "add protection for other uninsured devices"; boat flagged as Scott-validation item.

## Language-Switch Pair

**Claude's recommendation (accepted):** English ↔ Spanish (US) — both GA voice variants, #1 US secondary language, broadly relatable across ~18 accounts. Captioned-text fallback for anything outside the confirmed audio-to-audio set; EN↔ES A2A support flagged for in-console confirmation.

## Decision Threshold & Triggers

**Claude's recommendation (accepted):** Keep literal $1,000; rationale = ~cost of a manual adjuster touch, configurable per carrier. Four literal if/then rows routed by deterministic Python tool + Handoff Rule on session variable.

## Competitive Framing Depth

**Claude's recommendation (accepted):** Light, clearly-flagged callout vs. Microsoft Copilot + Nuance, every claim tagged `[VALIDATE — Scott/Google]`. Not a full head-to-head table.

**Final confirmation:** User selected "Lock all of it" — all decisions written to CONTEXT.md.

---

## Claude's Discretion

- Full reconciliation of the scenario pivot (insurance framing, claim-size pair, escalation-flag swap, cross-sell swap), the language pair, the threshold rationale, and the competitive-framing depth — all delegated via "decide what is best" and confirmed via "Lock all of it."
- Persona names, policy numbers, exact dollar figures, coverage wording, and the specific seeded prior-interaction — deferred to Mock Data Appendix authoring.

## Deferred Ideas

- Boat / marine cross-sell (Scott's original example) — retained as a flagged validation item.
- Additional language pairs (Hindi/Hinglish, French-Canada) — captioned fallback / per-account variants.
- Heavier competitive head-to-head table — out of scope for Phase 1.
- V2/DEF items (cross-channel continuity, proactive outreach, Agent Assist, live-metric overlay).
