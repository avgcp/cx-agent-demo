# Roadmap: CX Agent V2 Demo — Use-Case Specification

## Overview

This is a specification-authoring project, not a software build. Each phase below produces a section (or set of sections) of the use-case specification document that an implementation team will build the demo from on Google CX Agent Studio. "Success criteria" per phase describe properties of the *spec text* (unambiguous, buildable, internally consistent, traceable) — not running-software behaviors. The four phases follow the dependency order validated by `research/ARCHITECTURE.md`'s "Suggested Authoring Order": Foundations (narrative, capability map, decision logic, mock data) must lock before Component Architecture (agents, tools) can be specified correctly; Component Architecture must exist before the 9 parallelizable Use-Case Specs can be written; and the Runbook/Synthesis phase can only be finalized once every use case exists to roll up.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundations** - Lock the demo narrative, platform capability map, decision logic, and mock data appendix that every later section depends on
- [ ] **Phase 2: Component Architecture** - Specify the agent architecture and tool/data inventory that per-use-case specs will reference
- [ ] **Phase 3: Use-Case Specs** - Write the 9 fixed-template use-case specs plus 4 differentiator add-ons (parallelizable)
- [ ] **Phase 4: Runbook & Synthesis** - Produce the presenter runbook, roll up global acceptance criteria, and close out open questions/risks

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
- [ ] 01-05-PLAN.md — Decision-ID citation insertions (D-05/D-12/D-13 in §3, D-08 in §4, D-03/D-07 + fraud-out-of-scope sentence in §1)

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
**Plans**: TBD

Plans:
- [ ] 02-01: TBD

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
| 1. Foundations | 0/TBD | Not started | - |
| 2. Component Architecture | 0/TBD | Not started | - |
| 3. Use-Case Specs | 0/TBD | Not started | - |
| 4. Runbook & Synthesis | 0/TBD | Not started | - |
