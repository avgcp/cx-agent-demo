# Roadmap: CX Agent V2 Demo — Use-Case Specification

## Overview

This is a specification-authoring project, not a software build. Each phase below produces a section (or set of sections) of the use-case specification document that an implementation team will build the demo from on Google CX Agent Studio. "Success criteria" per phase describe properties of the *spec text* (unambiguous, buildable, internally consistent, traceable) — not running-software behaviors. The four phases follow the dependency order validated by `research/ARCHITECTURE.md`'s "Suggested Authoring Order": Foundations (narrative, capability map, decision logic, mock data) must lock before Component Architecture (agents, tools) can be specified correctly; Component Architecture must exist before the 9 parallelizable Use-Case Specs can be written; and the Runbook/Synthesis phase can only be finalized once every use case exists to roll up.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Foundations** - Lock the demo narrative, platform capability map, decision logic, and mock data appendix that every later section depends on _(completed 2026-07-09)_
- [ ] **Phase 2: Component Architecture** - Specify the agent architecture and tool/data inventory that per-use-case specs will reference
- [ ] **Phase 3: Use-Case Specs** - Write the 9 fixed-template use-case specs plus 4 differentiator add-ons (parallelizable)
- [ ] **Phase 4: Runbook & Synthesis** - Produce the presenter runbook, roll up global acceptance criteria, and close out open questions/risks
- [ ] **Phase 5: Demo Build — Multimodal, Backend Reveal & Multilingual** - Add photo damage verification, the backend claims-processing reveal, a web channel, and Spanish to the deployed agent _(BUILD PHASE — see caveat below)_

## Phase Details

### Phase 1: Foundations
**Goal**: The spec's foundational sections — demo narrative, platform capability map, decision logic, and mock data — are locked, unambiguous, and internally consistent, so nothing downstream needs to re-derive them.
**Depends on**: Nothing (first phase)
**Requirements**: NARR-01, NARR-02, NARR-03, NARR-04, CAP-01, CAP-02, DEC-01, DEC-02, DEC-03, DATA-01, DATA-02, DATA-03, DATA-04
**Success Criteria** (what must be TRUE of the spec):
  1. The demo narrative section contains a single ≤10–15 min beat-by-beat script sequencing every wow moment (voice + language switch, decision branch shown both ways, backend reveal, cross-sell) into one coherent white-label FNOL story, states which insurance sectors it generalizes to, and explicitly states what V2 adds over the V1 chat-happy-path baseline.
  2. The narrative's competitive framing vs. Microsoft Copilot + Nuance is present but every competitive claim is explicitly flagged for Scott/Google stakeholder validation, not asserted as settled fact.
  3. The platform capability map contains one row per wow moment, each tagged `[BUILT-IN]` / `[CUSTOM TOOL]` / `[MOCK DATA]` and cited against current Google Cloud documentation, with any unverified/press-only claim (notably audio-to-audio language set, transcript→summary→email mechanics) explicitly flagged as an open question rather than stated as fact.
  4. The decision logic table expresses the ~$1,000 auto-approve/human-assessor threshold, the injury/high-risk escalation flag, and the cross-sell trigger as literal if/then rows with no prose ambiguity; it mandates routing via a deterministic Python tool + Handoff Rule on a session variable (never RAG or instruction-only LLM judgment); and it frames auto-approval as a human-authorized, configurable, auditable rule set with a visible audit-trail artifact and a stated illustrative rationale for the $1,000 figure — never "the AI decides."
  5. The Mock Data Appendix contains versioned, literal seed records (policies, claimants, coverage, claim amounts, damage images, multilingual test phrases) engineered to land on both sides of every threshold, including "negative" records that must NOT trigger a branch, uses synthetic (non-real) PII only, and every dollar figure/coverage term used anywhere in the narrative traces to a named mock-data field.
**Plans**: 5 plans (incl. 1 gap-closure)

Plans:
**Wave 1**
- [x] 01-01-PLAN.md — Mock Data Appendix (canonical seed dataset; source of truth for all figures/records/images/phrases)
- [x] 01-02-PLAN.md — Platform Capability Map (one tagged, cited row per wow moment; unverified claims flagged)

**Wave 2** *(blocked on Wave 1 completion)*
- [x] 01-03-PLAN.md — Decision Logic table (literal if/then rules; deterministic tool + Handoff Rule; audit-trail framing)

**Wave 3** *(blocked on Wave 2 completion)*
- [x] 01-04-PLAN.md — Demo Narrative (≤10–15 min beat-by-beat storyline; V1→V2 delta; flagged competitive framing)

**Wave 4 — Gap closure** *(closes 01-VERIFICATION.md decision-ID traceability gaps)*
- [x] 01-05-PLAN.md — Decision-ID citation insertions (D-05/D-12/D-13 in §3, D-08 in §4, D-03/D-07 + fraud-out-of-scope sentence in §1)

### Phase 2: Component Architecture
**Goal**: The concrete component inventory — agent architecture and tool/data inventory — is specified precisely enough that the per-use-case specs can reference it by name without re-deriving anything.
**Depends on**: Phase 1
**Requirements**: ARCH-01, ARCH-02, ARCH-03, TOOL-01, TOOL-02
**Success Criteria** (what must be TRUE of the spec):
  1. The Agent Architecture Spec defines the root "Claims Concierge" agent plus a shallow layer of sub-agents (Intake, Decisioning, Escalation/Human-Handoff, Backend Claims-Processing, Upsell/Cross-Sell), each with stated responsibility, trigger, instruction guidance, and inputs/outputs — with no deep nesting beyond one sub-agent layer.
  2. The Backend Claims-Processing sub-agent is specified as shared across both the autonomous and HITL branches, with two explicit output modes (customer email vs. human-assessor briefing packet) selected by a mode variable, rather than duplicated as two separate components.
  3. Damage assessment is specified as a native-multimodal capability of the Intake sub-agent (not a separate deep-nested agent), with the shallow-nesting guardrail stated explicitly as a design rule other authors must follow.
  4. The Tool & Data Inventory contains exactly one row per tool/data store (name, type, purpose, inputs, outputs, source mock data) as a single source of truth, and every backend lookup (policy, coverage, claim creation, boat-insurance/upsell check, damage valuation) is specified as a deterministic mock Python code tool returning seeded JSON — not a Data Store/RAG lookup.
  5. Exact Handoff Rule UI/syntax and exact Python-tool session-variable mechanics are explicitly flagged in the spec as items the implementation team must confirm against the live console before building, rather than asserted as settled.
**Plans**: 2 plans

Plans:
**Wave 1**
- [ ] 02-01-PLAN.md — Agent Architecture Spec (§5): root Claims Concierge + 5 sub-agents, shallow-nesting design rule, session-variable contract table (ARCH-01/02/03)

**Wave 2** *(blocked on Wave 1 completion — §6 references §5 agents + session variables by name)*
- [ ] 02-02-PLAN.md — Tool & Data Inventory (§6): one row per tool, all deterministic Python (zero RAG), SC#5 console-confirm flags (TOOL-01/02)

### Phase 3: Use-Case Specs
**Goal**: Every wow moment has a complete, buildable, testable use-case spec that pulls from — never re-derives — the Foundations and Component Architecture sections.
**Depends on**: Phase 2
**Requirements**: UC-01, UC-02, UC-03, UC-04, UC-05, UC-06, UC-07, UC-08, UC-09, ADD-01, ADD-02, ADD-03, ADD-04
**Success Criteria** (what must be TRUE of the spec):
  1. All 9 use-case specs (UC-01 Chat Intake through UC-09 Escalation/Out-of-Scope) exist using the fixed template — trigger, agents involved, flow, tools/data, expected output, acceptance criteria, edge cases, example dialogue — and each cross-references the Tool & Data Inventory and Decision Logic table by name instead of re-describing them.
  2. UC-05 and UC-06 (the autonomous and HITL decision-branch use cases) are specified to be run back-to-back with two contrasting claim amounts, and UC-07 (backend reveal) is spec'd as an in-session sub-agent chain / simulated reveal, not an off-the-shelf ingestion pipeline.
  3. The 4 differentiator add-ons (ADD-01 sentiment/empathy on UC-09, ADD-02 fraud-signal rider on UC-07, ADD-03 policy-clause citation, ADD-04 personalization on UC-08) are specified as riders layered onto their host use cases with no new architecture introduced.
  4. UC-03 (mid-conversation language switch) is explicitly restricted to the confirmed audio-to-audio language set with a captioned-text fallback specified for any language outside it.
  5. Every use case's acceptance criteria are testable statements (e.g., "given seeded claim X, agent auto-approves within N turns and issues a claim number without human handoff"), not aspirational language, and every dollar figure/coverage claim in example dialogue sources from a named Mock Data Appendix field.
**Plans**: TBD

Plans:
- [ ] 03-01: TBD

### Phase 4: Runbook & Synthesis
**Goal**: A presenter who did not build the demo can run it solo from the spec alone, and the spec closes with rolled-up acceptance criteria and no unresolved ambiguity left silent.
**Depends on**: Phase 3
**Requirements**: RUN-01, RUN-02, AC-01, AC-02, AC-03
**Success Criteria** (what must be TRUE of the spec):
  1. The Demo Runbook contains a pre-flight checklist, a literal timed walkthrough, and explicit scenario/seed-data selection guidance covering at minimum the autonomous, HITL, and language-switch paths.
  2. The Runbook includes live-failure fallback guidance (recovery scripts, backup recordings, latency fallback) sufficient for a presenter with no build knowledge of the demo to recover live without the implementation team present.
  3. Global acceptance criteria (AC-01) roll up every per-use-case criterion from Phase 3 into demo-level "what good looks like" statements (every wow moment lands, no dead air, both decision branches demoable on demand, timing under target minutes, white-label framing intact).
  4. Every deterministic wow moment has a stated N/N repeatability acceptance test, and every voice/language-switch moment has a defined latency threshold tested against venue-class (not lab) network conditions.
  5. The Open Questions/Risks/Assumptions section is finalized and complete, explicitly listing every platform item still needing live-console confirmation (audio-to-audio language set, transcript→summary→email mechanics, Handoff Rule syntax) and every GTM item still needing stakeholder confirmation (competitive claims), with none left as a silent assumption elsewhere in the spec.
**Plans**: TBD

Plans:
- [ ] 04-01: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundations | 5/5 | Complete | 2026-07-09 |
| 2. Component Architecture | 0/2 | Planned | - |
| 3. Use-Case Specs | 0/TBD | Not started | - |
| 4. Runbook & Synthesis | 0/TBD | Not started | - |

### Phase 5: Demo Build — Multimodal, Backend Reveal & Multilingual

> **⚠ This phase departs from the roadmap's framing.** Phases 1–4 produce *specification text*; their success criteria are properties of that text. This phase changes a **running agent** — CX Agent Studio app `6e01e4a5-42a8-5213-b3da-c9053ff8ea52`, live on version **v11 `b17c9a26`** via GTP deployment `d28bbcb0`. Its success criteria are therefore observable behaviours of a deployed system, verified by driving real conversations. The demo was built and delivered ahead of the spec; this phase records the work that follows from that.

**Goal**: The deployed FNOL agent gains the three capabilities the demo narrative promises but the build never had — photo-based damage verification, a visible backend claims-processing artifact, and Spanish — without weakening any determinism guarantee established during the v1→v11 hardening.

**Depends on**: Nothing in this roadmap. Operates directly on the deployed agent; independent of Phases 2–4.

**Requirements**: (to be assigned — this phase predates its requirement IDs)

**Success Criteria** (what must be TRUE of the deployed agent):
  1. A customer can attach a photo of the damage, and the agent reports what it can actually see in that image — confirming or contradicting what the customer reported — while the **claim amount continues to come from the deterministic tariff and never from the vision read**.
  2. The anti-fraud path holds: a reported crack that is absent from the photo sets `photo_contradiction`, routes to human review under DL-5, and is communicated to the customer without accusing them of anything. An unusable photo gets exactly one retry before a human is involved, and a photo of the wrong object is rejected.
  3. Every resolved claim produces a reviewable artifact — a customer confirmation email on the autonomous path, a structured assessor briefing packet (SUMMARY / ACTION / CLAIM / DIAGNOSTIC / RULES FIRED / FLAGS) on the escalated path — and the assessor packet is delivered as a real message so the artifacts are demonstrable without a bespoke reveal screen.
  4. A `WEB_UI` deployment with file upload runs alongside the existing GTP phone deployment, and **the phone deployment continues to serve its pinned version unchanged**.
  5. A caller can complete the full claim in Spanish (es-US), with every customer-facing string — the verbatim-read decision explanation, the claim email, diagnostic questions, empathy lines, and the send-away checklist — available in that language rather than English text spoken with a Spanish voice.
  6. All hardening from v1→v11 still holds after the changes: deterministic pricing, single-send email, policy-ID digit rescue, liquid-ingress disambiguation, no self-narration, no double-speaking.

**Source material**: `assess_screen_crack` and the `case_summary` agent (with its `generate_case_summary` agent-as-tool) already exist in app `9ae7a0c3-6511-413c-8cdb-0efe9e90d2b9` ("Meridian Claim - Chat") and are worth porting. **That app must not be deployed or used as a base** — it is a fork of the pre-hardening build and carries none of the v1→v11 fixes, including a placeholder Resend key. Port the two components into the hardened app instead.

**Plans:** 1 complete, 3 outstanding

Plans:
- [x] 05-01 — Chat channel with photo damage verification *(executed without a PLAN.md; see 05-01-SUMMARY.md)* — covers criteria 1, 2, 4
- [ ] 05-02 — Decision card + quick actions (`widgetTool`: `ORDER_SUMMARY`, `QUICK_ACTIONS`) — **blocked** on confirming the vision read works in the real widget
- [ ] 05-03 — `case_summary` / assessor briefing packet delivered as a second email — covers criterion 3
- [ ] 05-04 — Spanish (es-US) across every customer-facing string — covers criterion 5

**Built so far**: `Meridian Claim - Chat (hardened)` `a2f621e4-9faf-505a-b804-22471f022366`, deployment `d7bfbb93` (`WEB_UI`/`CHAT_ONLY`) on version `97f44790`. Voice app `6e01e4a5` untouched on v11 `b17c9a26`.
