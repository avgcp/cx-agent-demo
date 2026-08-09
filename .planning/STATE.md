---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: executing
stopped_at: Phase 2 context gathered
last_updated: "2026-08-09T22:05:00.000Z"
last_activity: "2026-08-09 - Phase 5: decision card rendering diagnosed (payload schema mismatch vs the SDK order_summary contract) and fixed in chat v7 bb14cdcc; deployment d7bfbb93 repointed; awaiting the user's visual confirmation"
progress:
  total_phases: 5
  completed_phases: 1
  total_plans: 7
  completed_plans: 5
  percent: 20
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-07-08)

**Core value:** A specification concrete and compelling enough that the implementation team can build a show-don't-tell demo that lands every wow moment — voice + mid-demo language switching, a visible autonomous-vs-human decision branch, the backend claims-processing reveal, and the cross-sell "cost center → profit center" moment.
**Current focus:** Phase 01 — foundations

## Current Position

Phase: 5 (active) — Phase 2 still unstarted
Plan: 05-01 complete; 05-02 in progress (decision-card rendering diagnosed and fixed in chat v7 `bb14cdcc`, awaiting the user's visual confirmation at http://localhost:3000; cover_offer_actions still never fired and now believed broken the same way; wrong-subject guard resolved by removal 2026-08-06); 05-03/04 outstanding
Status: Build work running ahead of the spec phases. Phases 2-4 author specification text and remain unstarted; Phase 5 changes the deployed agent and is where activity currently is.
Last activity: 2026-08-09 - Phase 5: decision card rendering diagnosed (payload schema mismatch vs the SDK order_summary contract) and fixed in chat v7 bb14cdcc; deployment d7bfbb93 repointed; awaiting the user's visual confirmation

Progress: [██████████] 100%

## Performance Metrics

**Velocity:**

- Total plans completed: 5
- Average duration: - min
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 01 | 5 | - | - |

**Recent Trend:**

- Last 5 plans: -
- Trend: -

*Updated after each plan completion*
| Phase 01-foundations P05 | 2min | 3 tasks | 3 files |

## Accumulated Context

### Roadmap Evolution

- Phase 5 added: Demo Build - Multimodal, Backend Reveal & Multilingual (build phase, not spec authoring)

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Roadmap: Deliverable is a spec, not code — phase success criteria describe properties of spec text (unambiguous, buildable, internally consistent), not running-software behaviors.
- Roadmap: 4-phase structure adopted directly from research/ARCHITECTURE.md's validated authoring-order dependency chain (Foundations → Component Architecture → Use-Case Specs → Runbook & Synthesis).
- [Phase 01-foundations]: Gap closure (01-05): inserted D-03/D-05/D-07/D-08/D-12/D-13 citation tokens at heading/field-row anchors matching existing Section 2/4 citation convention; no spec content rewritten

### Pending Todos

None yet.

### Blockers/Concerns

- Phase 1: Audio-to-audio translation language set for mid-demo voice language switching is press-sourced only, not confirmed in current CX Agent Studio technical docs — needs console/stakeholder confirmation before the narrative commits to specific language pairs (see CAP-02, DATA-03, UC-03).
- Phase 1: Transcript → specialist-summary → drafted-email mechanics has no confirmed off-the-shelf Gemini Enterprise pipeline — must be spec'd as a custom in-session sub-agent chain, not an assumed built-in feature (see CAP-02, UC-07).
- Phase 2: Exact Handoff Rule UI/syntax and Python-tool session-variable mechanics are MEDIUM-confidence (docs shift on a 6-month-old product) — flag for live-console verification early in Phase 2 (see DEC-02, ARCH work).
- GTM: Competitive framing vs. Microsoft Copilot/Nuance is LOW-MEDIUM confidence — treat as stakeholder input (Scott/Srini/Stephanie/Hallie) to validate, not asserted fact (see NARR-03).
- Demo build: the customer-facing "FNOL Demo Narrative - V2.docx" describes a build that does not exist — it promises chat intake, photo upload, an EN↔ES switch, a $100 deductible, a $3,500 limit, a $1,000 threshold and $450/$2,400 claim amounts. The delivered agent is voice-only with a diagnostic tree, $25/$3,000/50%-of-coverage and $840/$3,000. Reconcile before the document goes to any customer or implementation team (Phase 5 scope).
- Demo build: the implemented PoC diverges from the Phase 1 Mock Data Appendix (§4) — tariff-based pricing, a 50%-of-coverage auto-approve threshold, $25 excess and no photo-upload path. §4's $1,000 threshold, $100 deductible and $3,500 limit are stale against what a caller actually hears. Reconcile §4 (or mark it superseded) before the spec ships to any other implementation team.
- Phase 5 (RESOLVED 2026-08-05): the model's vision accuracy on a real photo is confirmed — a cracked-screen image was read correctly in the simulator ("I can see several cracks across the screen"), confirmed, and auto-approved at $840. Plan 05-02 is unblocked. Original concern: Every branch of `assess_screen_crack` is tested, but only by supplying the observation values directly — the API can be driven with text, not with a convincing photo of a cracked screen. If it misreads a demo photo, the confirm path silently becomes the contradiction path in front of an audience. One upload through the deployed widget (`d7bfbb93`) settles it; blocks plan 05-02.
- Phase 5 ACCEPTED LIMITATION (resolved 2026-08-06, quick task 260806-u21): photo assessment cannot verify WHICH device is in the image. History: v4 asked the model for the device category, v5 asked for physical features (keyboard visible, hinged lid) and inferred in code. Both failed live: against PDP100746 (iPhone) the model reported keyboard_visible=no, hinged_lid=no and "a black phone" for an unmistakable MacBook — see evidence-laptop-photo.png in the phase dir, decoded from the conversation record. Root cause: the model knows what the policy covers and conforms its report to that, not to the pixels. Damage reporting is accurate; only identity is contaminated. DECISION (user): option (b) — a guard that does not work is worse than none, because it reads as protection in both the code and the runbook. Option (a) (agent-as-tool vision isolation) was not attempted. Removed in chat v6 `56a8b22a`, now served by deployment `d7bfbb93`: the `keyboard_visible`/`hinged_lid` parameters, the device-inference block, the `WRONG_SUBJECT` branch and the `photo_device_seen` variable are all gone; `assess_screen_crack(crack_visible, what_you_see)` asks about damage only. STANDING DEMO CONSTRAINT: photo assessment confirms whether the reported damage is visible; it cannot verify that the photographed device is the insured device — run demos with matching device/policy pairs (PDP100294 = MacBook, use a laptop photo). Unchanged and still working: the contradiction path (crack reported, none visible → photo_contradiction, DL-5, human review, non-accusatory), the one-retry-then-human path for unusable photos, and deterministic tariff pricing. A junk or subject-less photo now lands in unclear → retry → human rather than a rejection, the same end state. NOT YET VERIFIED LIVE against chat v6: the contradiction path — proven offline in phototest2.py only; needs a photo of an undamaged device through the widget.
- Phase 5 DIAGNOSED AND FIXED, awaiting the user's eyes (2026-08-09, quick task 260809-n1b): widget rendering. ROOT CAUSE: the `claim_decision_card` tool declared seven flat claim fields (`issue_label`, `claim_amount`, `deductible`, ...) as its `parameters`, and a widget tool's parameters ARE its emitted payload. The deployed web-widget SDK's `order_summary` renderer reads only `productItem`, `costBreakdown`, `paymentMethod` and `actions`, found all four undefined, and drew nothing — so the raw payload is what appeared. NOT an SDK limitation, NOT an agent defect, and NOT the console-Preview limitation it was originally attributed to. CORRECTION to the previous entry's claim that "payload shape is confirmed correct": it was well-FORMED (the platform tags it `"type": "order_summary"`) which is NOT the same as matching the renderer's contract — that inference is what kept the real cause hidden for a week. FIXED in chat v7 `bb14cdcc-d723-4be1-85af-9f4451e22ed5`, now served by deployment `d7bfbb93`: the card is a single `productItem` row (title/subtitle/price/imageUri), with `costBreakdown` deliberately dropped (its hardcoded, unsuppressable "Sales tax" row would print $0.00 on an insurance claim) and `actions` deliberately dropped (its labels are live buttons wired to sendQuery, so a presenter tapping one would inject it into the conversation). The decision and the $1,500 cutoff moved into the subtitle. VERIFIED LIVE: the emitted payload is byte-identical to what `resolve_claim` computes in Python, with `units` as an int — the model supplies no figure. The full SDK contract is written down in 05-02-SUMMARY.md and is reusable for every future widget. WHAT REMAINS: only the human visual confirmation — run policy PDP100294, an Apple MacBook Pro 16", keyboard fault, at http://localhost:3000 and check the card draws.
- Phase 5: `cover_offer_actions` is very likely broken the same way and is UNFIXED. It declares `{prompt, options[{label,value}]}` with no dataMapping, but the SDK's `quick_actions` builder reads `{actions: [{content, description, utterance}]}` — `prompt` and `options` are read by nothing. Worse failure mode than the decision card's: `b.actions.map(...)` throws on undefined rather than degrading to a JSON blob. Still has never fired. Contract captured in 05-02-SUMMARY.md; needs a follow-up task.
- Phase 5: the decision turn draws the card but the agent says NOTHING alongside it (observed live on chat v7, pre-existing). The agent puts its sentence into the widget's `summary` parameter, and `textResponseConfig: NONE` — which per the CES discovery doc means "the LLM dynamically decides whether to generate a text response", NOT "no text" — leaves it free to stay silent. This contradicts `claim_intake`'s own "Never send the card without a sentence". That instruction block is self-contradictory and is the likely root cause: it says read the explanation WORD FOR WORD, then says do not repeat the figures and say only the short human part. Suggested fix (not applied, needs a live test): `textResponseConfig: {type: "LLM_GENERATED", textResponseInstruction: ...}` plus resolving the instruction contradiction.
- Phase 5 (resolved): CES quota is the binding constraint on testing. `RunSession LLM tokens` defaults to 1,000/min per project/region/model while a single conversation costs 120k-150k input tokens (3.5k-10.8k per turn). Raise it in IAM → Quotas before any demo; also `Concurrent BidiRunSession Operations` (10 per 30 min) if more than one person will use the phone.
- Demo build: the claim email can only reach ONE mailbox. Resend's shared `onboarding@resend.dev` sender delivers only to the address owning the Resend account (`akash.vinayak@nerdery.com`) — confirmed by a 403 from Resend when tested against another address. The phone deployment is pinned to v11 `b17c9a26`, which mails that address; both paths verified live. Sending to a second recipient requires verifying a domain at resend.com/domains and changing the `from` address.

### Quick Tasks Completed

| # | Description | Date | Commit | Directory |
|---|-------------|------|--------|-----------|
| 260805-aze | Create presenter demo runbook for deployed Meridian voice claim agent | 2026-08-05 | (see log) | [260805-aze-create-presenter-demo-runbook-for-deploy](./quick/260805-aze-create-presenter-demo-runbook-for-deploy/) |
| 260806-u21 | Remove wrong-subject photo guard from chat app; document the accepted limitation | 2026-08-06 | (see log) | [260806-u21-remove-wrong-subject-photo-guard-from-ch](./quick/260806-u21-remove-wrong-subject-photo-guard-from-ch/) |
| 260809-n1b | Reshape claim_decision_card to the SDK order_summary contract (chat v7) | 2026-08-09 | (see log) | [260809-n1b-reshape-claim-decision-card-to-the-sdk-o](./quick/260809-n1b-reshape-claim-decision-card-to-the-sdk-o/) |

## Deferred Items

Items acknowledged and carried forward from previous milestone close:

| Category | Item | Status | Deferred At |
|----------|------|--------|-------------|
| v2 | DEF-01 Cross-channel context continuity (chat→voice resume) | Deferred | Requirements definition |
| v2 | DEF-02 Proactive/catastrophe-triggered outbound outreach | Deferred | Requirements definition |
| v2 | DEF-03 Agent Assist | Deferred | Requirements definition |
| v2 | DEF-04 Speed/efficiency live metric overlay | Deferred | Requirements definition |

## Session Continuity

Last session: 2026-07-09T23:03:54.209Z
Stopped at: Phase 2 context gathered
Resume file: .planning/phases/02-component-architecture/02-CONTEXT.md
