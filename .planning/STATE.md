---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: executing
stopped_at: Phase 2 context gathered
last_updated: "2026-08-06T02:20:36.140Z"
last_activity: "2026-08-06 - Phase 5 plan 05-02 in progress: decision card + quick actions built; wrong-subject photo defect found"
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
Plan: 05-01 complete; 05-02 in progress (widgets built, rendering unverified, defect open); 05-03/04 outstanding
Status: Build work running ahead of the spec phases. Phases 2-4 author specification text and remain unstarted; Phase 5 changes the deployed agent and is where activity currently is.
Last activity: 2026-08-06 - Phase 5 plan 05-02 in progress: decision card + quick actions built; wrong-subject photo defect found

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
- Phase 5 DEFECT (open): the wrong-subject photo guard does not work. A laptop photo was accepted and auto-approved against PDP100583 (iPhone 16 Pro Max) — payload showed device "Apple iPhone 16 Pro Max", cutoff 750, amount 420. `assess_screen_crack` asks the MODEL whether the photo matches the policy (`image_shows_device`); it answered yes while looking at a laptop on a phone policy. Fix: have the model report what device it SEES (`device_in_photo`) and compare against `device_category` in code — same observe-vs-judge split used everywhere else in this build. Inherited from the ported tool and not re-examined.
- Phase 5: widget rendering unverified. The console Preview panel shows "custom payload in JSON" rather than drawing the card; rich response widgets are web-widget-only, so this needs a test through the real embed on deployment `d7bfbb93`. Payload shape is confirmed correct (platform tags it `"type": "order_summary"`).
- Phase 5 (resolved): CES quota is the binding constraint on testing. `RunSession LLM tokens` defaults to 1,000/min per project/region/model while a single conversation costs 120k-150k input tokens (3.5k-10.8k per turn). Raise it in IAM → Quotas before any demo; also `Concurrent BidiRunSession Operations` (10 per 30 min) if more than one person will use the phone.
- Demo build: the claim email can only reach ONE mailbox. Resend's shared `onboarding@resend.dev` sender delivers only to the address owning the Resend account (`akash.vinayak@nerdery.com`) — confirmed by a 403 from Resend when tested against another address. The phone deployment is pinned to v11 `b17c9a26`, which mails that address; both paths verified live. Sending to a second recipient requires verifying a domain at resend.com/domains and changing the `from` address.

### Quick Tasks Completed

| # | Description | Date | Commit | Directory |
|---|-------------|------|--------|-----------|
| 260805-aze | Create presenter demo runbook for deployed Meridian voice claim agent | 2026-08-05 | (see log) | [260805-aze-create-presenter-demo-runbook-for-deploy](./quick/260805-aze-create-presenter-demo-runbook-for-deploy/) |

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
