# Requirements: CX Agent V2 Demo — Use-Case Specification

**Defined:** 2026-07-08
**Core Value:** A specification concrete enough that an implementation team (CX Agent Studio access, no line to Scott/Akash/this research) can build a rehearsable, show-don't-tell WOW demo from it alone — landing every wow moment.

> These requirements define "done" for the **specification document**, not for the demo build. Each is testable: the spec either contains the section/use-case with the required detail, or it doesn't.

## v1 Requirements

### Narrative & GTM Framing (NARR)

- [ ] **NARR-01**: Spec contains a single coherent demo narrative/storyline (prose + beats) stitching every wow moment into one white-label FNOL scenario, ≤10–15 min
- [x] **NARR-02**: Narrative is white-label/generic (no customer-specific branding) and states which insurance sectors it generalizes to
- [ ] **NARR-03**: Spec includes GTM/competitive framing vs. Microsoft Copilot + Nuance, with all competitive claims flagged for stakeholder (Scott/Google) validation, not asserted as fact
- [ ] **NARR-04**: Spec references the V1 baseline (chat FNOL happy path) and states explicitly what V2 adds on top

### Platform Capability Map (CAP)

- [ ] **CAP-01**: Spec maps every wow moment to a specific CX Agent Studio primitive, tagged `[BUILT-IN]` / `[CUSTOM TOOL]` / `[MOCK DATA]`
- [ ] **CAP-02**: Spec cites current Google Cloud documentation for each platform-capability claim, and flags unverified/press-only claims as open questions

### Decision Logic (DEC)

- [x] **DEC-01**: Spec contains an explicit if/then business-rules table (no prose ambiguity) covering the ~$1,000 auto-approve vs. human-assessor threshold, the injury/high-risk escalation flag, and the cross-sell trigger
- [x] **DEC-02**: Spec mandates the threshold be computed by a deterministic Python tool routed via a Handoff Rule on a session variable — explicitly NOT RAG or instruction-only LLM judgment
- [x] **DEC-03**: Spec frames auto-approval as a human-authorized, configurable, auditable rule set (with a visible audit-trail artifact) — never "the AI decides" — and gives the $1,000 threshold a stated, illustrative/configurable rationale

### Mock Data (DATA)

- [x] **DATA-01**: Spec includes a Mock Data Appendix with literal, versioned seed records (policies, claimants, coverage, claim amounts) as the canonical demo dataset
- [ ] **DATA-02**: Seed claim amounts are engineered to land on both sides of every threshold, and include "negative" records that must NOT trigger a branch (over-fire check)
- [ ] **DATA-03**: Mock data uses synthetic (non-real) PII only, and includes multilingual test phrases and seed damage images
- [x] **DATA-04**: Every dollar figure / coverage term spoken in the demo traces to a named mock-data field (nothing free-generated on stage)

### Agent Architecture (ARCH)

- [ ] **ARCH-01**: Spec defines a root "Claims Concierge" agent plus a shallow layer of sub-agents (Intake, Decisioning, Escalation/Human-Handoff, Backend Claims-Processing, Upsell/Cross-Sell), each with responsibility, trigger, instruction guidance, and inputs/outputs
- [ ] **ARCH-02**: Spec specifies the Backend Claims-Processing sub-agent as shared across both autonomous and HITL branches, with dual output modes (customer email vs. assessor briefing packet)
- [ ] **ARCH-03**: Spec treats damage assessment as a capability of Intake (native Gemini image understanding), not a separate deep-nested agent, and states the shallow-nesting guardrail

### Tool & Data Inventory (TOOL)

- [ ] **TOOL-01**: Spec includes a single-source-of-truth tool & data inventory (one row per tool/data store) cross-referenced by every use-case spec
- [ ] **TOOL-02**: Each backend lookup (policy, coverage, claim creation, boat-insurance/upsell check, damage valuation) is specified as a mock Python code tool returning deterministic seeded JSON

### Use-Case Specs (UC)

Each use case uses a fixed template: trigger → agents involved → flow → tools/data → expected output → acceptance criteria → edge cases, with example dialogue.

- [ ] **UC-01**: Chat Intake (FNOL happy path)
- [ ] **UC-02**: Voice Intake (Google Telephony channel)
- [ ] **UC-03**: Mid-Conversation Language Switch (restricted to confirmed audio-to-audio language set, with captioned-text fallback)
- [ ] **UC-04**: Multimodal Damage Assessment (photo upload → natural-language findings → structured valuation)
- [ ] **UC-05**: Decision Threshold Branch — Autonomous (small claim auto-approves, visibly)
- [ ] **UC-06**: Decision Threshold Branch — HITL (large claim routes to human assessor, visibly)
- [ ] **UC-07**: Backend Claims-Processing Reveal (transcript → specialist summary → auto-drafted customer email; spec'd as in-session sub-agent chain / simulated reveal screen, not an off-the-shelf pipeline)
- [ ] **UC-08**: Cross-Sell / Upsell Moment (uninsured-boat bundle offer; "cost center → profit center" closing beat)
- [ ] **UC-09**: Escalation / Out-of-Scope Handling (injury escalation + out-of-domain deflection)

### Differentiator Add-Ons (ADD)

Cheap, high-impact "beyond Scott's list" moments layered onto existing use cases (no new architecture).

- [ ] **ADD-01**: Real-time sentiment/empathy detection with visible tone adaptation (bundled into UC-09 injury scenario)
- [ ] **ADD-02**: Fraud-signal surfacing in the specialist summary (rider on UC-07)
- [ ] **ADD-03**: Real-time knowledge grounding with visible policy-clause citation (addresses "will it hallucinate coverage")
- [ ] **ADD-04**: Personalization via seeded prior-interaction data (makes UC-08 cross-sell feel earned)

### Demo Runbook (RUN)

- [ ] **RUN-01**: Spec includes a presenter runbook: pre-flight checklist, literal walkthrough with timing, scenario/seed-data selection
- [ ] **RUN-02**: Runbook includes live-failure fallback guidance (recovery scripts, backup recordings, latency fallback), so a presenter who did not build the demo can run it solo

### Acceptance Criteria & Risks (AC)

- [ ] **AC-01**: Spec includes global demo-level acceptance criteria rolling up per-use-case criteria ("what good looks like")
- [ ] **AC-02**: Every deterministic wow moment has an N/N repeatability acceptance test; voice moments have a defined latency threshold tested on venue-class network
- [ ] **AC-03**: Spec includes an Open Questions / Risks / Assumptions section listing platform items to confirm against the live console (audio-to-audio language set, transcript→summary→email mechanics, Handoff Rule syntax) and GTM items to confirm with stakeholders

## v2 Requirements

Deferred to a later fast-follow iteration. Tracked, not in current roadmap.

### Deferred Capabilities (DEF)

- **DEF-01**: Cross-channel context continuity (chat → voice resume) — different, two-leg demo shape
- **DEF-02**: Proactive / catastrophe-triggered outbound outreach — different, outbound demo shape
- **DEF-03**: Agent Assist (real-time summarization/coaching/knowledge for human agents) — Scott: customer-dependent, only if engagement expands to full contact-center
- **DEF-04**: Speed/efficiency live metric overlay (elapsed-time counter for ROI anchoring) — nice-to-have polish

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Building the demo itself | This project delivers the spec; a separate team implements in CX Agent Studio |
| Real system integrations (live claims/policy) | Demo is mock-data-only; real integrations need credentials, premature, and a live-env failure kills deals |
| Real Gemini Enterprise ingestion pipeline | No confirmed off-the-shelf transcript→summary→email feature; spec the reveal as a built simulated chain instead |
| Integration Connectors / BigQuery / Cloud Function backend | Over-engineering for mock-data ASAP scope; use Python code tools |
| Flow-based (legacy Dialogflow CX) agent design | Platform is flow-free; flows are only a migration escape hatch |
| Custom branding / bespoke UI | Deliberately white-label; branding waits for POV/implementation |
| Full autonomous payment execution | Regulatory/trust risk; out of demo scope |
| External / non-Google telephony carriers | Akash: keep tight Studio integration via Google Telephony Platform |
| Exposing raw model chain-of-thought | Reads as unpolished/risky in a sales demo |
| Real customer PII | Synthetic data only |
| Multi-sector branching in one sitting | Dilutes the 10–15 min demo ceiling; white-label narrative stays single-sector per run |
| Full underwriting automation | Wrong buyer persona for this demo |

## Traceability

Which phases cover which requirements. Populated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| NARR-01 | Phase 1 — Foundations | Pending |
| NARR-02 | Phase 1 — Foundations | Complete |
| NARR-03 | Phase 1 — Foundations | Pending |
| NARR-04 | Phase 1 — Foundations | Pending |
| CAP-01 | Phase 1 — Foundations | Pending |
| CAP-02 | Phase 1 — Foundations | Pending |
| DEC-01 | Phase 1 — Foundations | Complete |
| DEC-02 | Phase 1 — Foundations | Complete |
| DEC-03 | Phase 1 — Foundations | Complete |
| DATA-01 | Phase 1 — Foundations | Complete |
| DATA-02 | Phase 1 — Foundations | Pending |
| DATA-03 | Phase 1 — Foundations | Pending |
| DATA-04 | Phase 1 — Foundations | Complete |
| ARCH-01 | Phase 2 — Component Architecture | Pending |
| ARCH-02 | Phase 2 — Component Architecture | Pending |
| ARCH-03 | Phase 2 — Component Architecture | Pending |
| TOOL-01 | Phase 2 — Component Architecture | Pending |
| TOOL-02 | Phase 2 — Component Architecture | Pending |
| UC-01 | Phase 3 — Use-Case Specs | Pending |
| UC-02 | Phase 3 — Use-Case Specs | Pending |
| UC-03 | Phase 3 — Use-Case Specs | Pending |
| UC-04 | Phase 3 — Use-Case Specs | Pending |
| UC-05 | Phase 3 — Use-Case Specs | Pending |
| UC-06 | Phase 3 — Use-Case Specs | Pending |
| UC-07 | Phase 3 — Use-Case Specs | Pending |
| UC-08 | Phase 3 — Use-Case Specs | Pending |
| UC-09 | Phase 3 — Use-Case Specs | Pending |
| ADD-01 | Phase 3 — Use-Case Specs | Pending |
| ADD-02 | Phase 3 — Use-Case Specs | Pending |
| ADD-03 | Phase 3 — Use-Case Specs | Pending |
| ADD-04 | Phase 3 — Use-Case Specs | Pending |
| RUN-01 | Phase 4 — Runbook & Synthesis | Pending |
| RUN-02 | Phase 4 — Runbook & Synthesis | Pending |
| AC-01 | Phase 4 — Runbook & Synthesis | Pending |
| AC-02 | Phase 4 — Runbook & Synthesis | Pending |
| AC-03 | Phase 4 — Runbook & Synthesis | Pending |

**Coverage:**
- v1 requirements: 36 total (corrected from initial header count of 33 — direct ID count across all 10 categories is 36)
- Mapped to phases: 36/36
- Unmapped: 0

---
*Requirements defined: 2026-07-08*
*Last updated: 2026-07-08 after roadmap creation — traceability populated, 100% coverage across 4 phases*
