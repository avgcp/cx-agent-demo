# Project Research Summary

**Project:** CX Agent V2 Demo — Use-Case Specification
**Domain:** GTM/specification project — the deliverable is a use-case SPECIFICATION for a white-labeled insurance FNOL claims "WOW demo," built by a separate implementation team on Google CX Agent Studio (Gemini Enterprise for Customer Experience). This project does not write application code.
**Researched:** 2026-07-08
**Confidence:** MEDIUM-HIGH

## Executive Summary

This is a specification-authoring project, not a software build. The "product" experts build here is a document precise enough that an implementation team with CX Agent Studio console access — but no direct line to Scott, Akash, or this research — can construct a reliable, rehearsable sales demo from it alone. Research across all four tracks converges on the same architectural shape: a **flow-free, hierarchical multi-agent design** (one root agent + a shallow layer of purpose-built sub-agents — Intake, Damage Assessment, Decisioning, Escalation, Backend Claims-Processing, Upsell) driven by natural-language instructions and Python tools, not visual flows. The platform (CX Agent Studio, GA since 2026-02-04) is new enough that no insurance/FNOL template exists — the spec has to build the domain from scratch using the platform's own "Cymbal Retail" sample as an architectural reference, not a content reference.

The recommended approach is deliberately conservative about where determinism lives versus where generative flexibility is allowed to show. The single highest-stakes design decision — the ~$1,000 auto-approve vs. human-assessor branch — must be computed by a **deterministic Python tool** and routed by a **Handoff Rule** on a session variable, never left to instruction-only LLM judgment or RAG retrieval; the same discipline applies to the cross-sell trigger. Backend systems (policy lookup, coverage lookup, claim creation) should be **mocked as Python code tools returning seeded JSON**, not built against Integration Connectors or a real BigQuery/Cloud Function pipeline — that infrastructure is explicitly out of scope and over-engineering for an ASAP, mock-data-only GTM asset. Voice, multilingual response-matching, and multimodal image understanding are native, largely built-in platform capabilities requiring configuration rather than custom engineering. The "backend claims-processing reveal" (transcript → specialist summary → drafted email) should be spec'd and built as an **in-session sub-agent call consuming session state**, presented as a simulated "reveal" screen — not marketed or engineered as an off-the-shelf Gemini Enterprise ingestion pipeline, because no such pipeline is confirmed to exist for this exact workflow in current docs.

The primary risk to this project isn't platform capability — it's **spec ambiguity and unverified capability claims flowing downstream into a live sales demo**. Four items carry genuine platform uncertainty (see Gaps below) and should be flagged for implementation-team/console confirmation rather than asserted as fact in the spec. Pitfalls research independently converges on the same theme from a different angle: hallucinated dollar figures, non-deterministic wow moments, non-credible thresholds, and regulatory optics around "AI approving claims" are all avoidable by the same discipline — ground every number in named mock data, make every "must land" moment deterministic and testable, and frame auto-approval as governed/auditable rule-following rather than autonomous AI judgment. The roadmap should therefore treat the spec's own internal consistency (narrative → capability map → decision logic → mock data, in that dependency order) as the load-bearing early-phase work, with acceptance criteria and a demo runbook as a mandatory, non-optional closing phase.

## Key Findings

### Platform Capability Surface (from STACK.md)

CX Agent Studio (GA 2026-02-04, part of Gemini Enterprise for Customer Experience, announced NRF Jan 2026) is a flow-free, instruction+tool builder on top of the Agent Development Kit. It is not a code stack in the traditional sense — it's a capability surface the spec must map every wow moment onto, tagging each as **[BUILT-IN]**, **[CUSTOM TOOL]**, or **[MOCK DATA]**.

**Core platform elements the spec should assume:**
- **Root agent + sub-agents ([BUILT-IN]):** the primary architecture — natural-language instructions, optional XML structuring (`<taskflow>`, `<trigger>`, `<action>`), few-shot examples, `{@AGENT:}`/`{@TOOL:}` reference syntax. Flow-based (legacy Dialogflow CX) agents exist only as a migration bridge/escape hatch — explicitly not the recommended paradigm, and out of scope per PROJECT.md.
- **Python code tools ([CUSTOM TOOL], the recommended mock-backend mechanism):** inline Python with session-state access, returning fully mocked JSON deterministically — this is how policy/coverage/claim lookups should be spec'd, not Integration Connectors (which require real Application Integration setup — overkill for mock-data-only scope) and not Data Store/RAG tools (unstructured-content retrieval, wrong tool for exact-match structured fields like dollar amounts).
- **Handoff Rules ([BUILT-IN], deterministic):** rules over session variables (AND/OR conditions) that force a transfer between agents — this, not instruction-only logic, is the mechanism for the dollar-threshold branch, because Google's own best-practice docs warn that "a tool call orchestrated by an agent is not deterministic."
- **Voice + multilingual (mostly [BUILT-IN]):** native multimodal Live API, Google Telephony Platform (native voice channel, no external carrier needed), automatic language detection/response-matching for chat, 28 documented GA+Preview voice-language variants. **Caveat:** the specific "audio-to-audio real-time translation, ~10 languages" capability that drives the mid-call language-switch wow moment is confirmed only in press coverage of the Jan 2026 announcement, not in current CX Agent Studio technical docs — flagged as the platform's top open question (see Gaps).
- **Multimodal damage assessment ([BUILT-IN] model + [CUSTOM TOOL] for structured decisioning):** native Gemini image understanding — Google's own launch materials use "photo of a damaged appliance" as the flagship multimodal example, directly validating this as an on-brand showcase, not a stretch.
- **Backend reveal (transcript → summary → drafted email):** no confirmed off-the-shelf Gemini Enterprise pipeline for this exact workflow exists in current docs. Recommended: spec it as a custom tool/sub-agent consuming in-session state, staged as a simulated "reveal" screen.
- **No insurance/FNOL template exists** on the platform today — only "Cymbal Retail" (a fictitious retail assistant) as an architectural reference. The demo narrative, sub-agent breakdown, and mock data must be built from scratch.

### Wow Moments & Feature Landscape (from FEATURES.md)

This is a sales-demo use-case menu, scored for demo-impact and demo-effort, not production feature maturity. V1 baseline (already built) is chat-only, single-language, single happy path: intake → coverage lookup → photo damage assessment → claim number, plus one injury-escalation and one out-of-scope deflection.

**Table stakes (already in V1, keep tight, don't over-narrate):** structured FNOL intake, coverage/deductible lookup, multimodal photo assessment, injury/high-risk escalation (reframe as trust proof point, not checkbox), out-of-scope deflection, basic multilingual chat, claim number issuance, natural conversational tone.

**Scott's five confirmed V2 must-haves (the spine of the demo, build first):**
1. **Voice channel + mid-demo language switching** — VERY HIGH wow, MEDIUM effort. Google's native audio-to-audio capability is a direct, citable differentiator vs. Microsoft Copilot's North-America-only, April-2026-GA'd real-time voice.
2. **Backend claims-processing reveal** — VERY HIGH wow, MEDIUM-HIGH effort. Converts "cute chatbot" framing into "claims-ops platform" framing; show the transformation live (split-screen/sequential reveal), not just the artifact.
3. **Autonomous vs. HITL paths shown end-to-end** — HIGH wow, MEDIUM effort. Directly rebuts the #1 insurance-buyer objection ("who's accountable when it's wrong"). Pair with the dollar-threshold branch — they are the same demonstration.
4. **Dollar-threshold decision logic (~$1,000)** — VERY HIGH wow, LOW-MEDIUM effort (cheapest high-wow feature in the whole spec). Run both branches back-to-back with two claim amounts for contrast.
5. **Cross-sell/upsell moment ("cost center → profit center")** — HIGH wow (especially for CFO/CRO audiences), LOW effort. Land as the closing beat, right after claim resolution.

**Cheap, high-impact "beyond the list" additions (P2 — should-have, add once P1 lands):**
- **Real-time sentiment/empathy detection with visible tone adaptation** — VERY HIGH wow, LOW-MEDIUM effort, no new backend needed; bundle into the existing injury-escalation scenario rather than building new. Directly rebuts "AI has no empathy," the most common insurance-buyer objection.
- **Fraud-signal surfacing in the specialist summary** — HIGH wow, LOW effort; a free rider on the backend-reveal artifact already being built (one flagged data point, no new UI/flow).
- **Real-time knowledge grounding with visible policy-clause citation** — HIGH wow (technical/compliance buyers), LOW-MEDIUM effort; addresses the #2 objection ("will it hallucinate coverage answers") by citing the actual mock policy clause.
- **Personalization via seeded prior-interaction data** — MEDIUM wow, LOW effort; cheap layer that makes the cross-sell moment feel earned rather than generic.
- **Speed/efficiency metric overlay** (live elapsed-time counter) — MEDIUM wow, LOW effort; anchors the ROI conversation for economic buyers.

**P3/deferred:** cross-channel context continuity (chat→voice resume, different narrative shape), proactive/catastrophe-triggered outreach (different, outbound demo shape), Agent Assist (explicitly deferred per PROJECT.md), full underwriting automation (out of scope, wrong buyer persona).

**Anti-features to explicitly exclude from the spec:** real system integrations, full autonomous payment execution, external/non-Google telephony carriers, exposing raw model chain-of-thought, real customer PII, multi-sector branching in a single sitting (dilutes a 10-15 minute demo ceiling), and full underwriting automation.

### Spec Document Architecture (from ARCHITECTURE.md)

Two architectures are in scope: (1) the demo's runtime composition on CX Agent Studio, and (2) the structure of the specification document itself. Both converge on the same discipline: **keep determinism where the wow moment depends on it, keep flexibility where authenticity depends on it.**

**Demo runtime — recommended composition:** one root "Claims Concierge" agent + a shallow layer of sub-agents (Intake, Decisioning, Escalation/Human-Handoff, Backend Claims-Processing, Upsell/Cross-Sell), each with narrow instructions and its own tools. Damage assessment is a *capability* of Intake (native Gemini image understanding), not a separate sub-agent — avoid deep nesting (an explicit anti-pattern). The Backend Claims-Processing sub-agent is **shared by both the autonomous and HITL branches** — one component, two output modes (customer email vs. assessor briefing), selected by which branch called it. This is the architecturally cleanest way to demo "both paths get the same intelligent backend treatment."

**Recommended specification document structure (~10 sections):**
0. Cover/Purpose — 1 page, what/who/why, white-label framing, link to V1
1. Demo Narrative/Storyline — the single coherent script stitching every wow moment together, prose/beats, not yet technical
2. Platform Capability Map — table mapping every wow moment to a CX Agent Studio primitive (root/sub-agent, tool type, data store, channel)
3. Decision Logic Spec — explicit if/then business rules table ($1,000 threshold, injury flag, upsell trigger), no prose ambiguity
4. Mock Data Appendix — literal seed records (policies, claimants, claim amounts engineered to hit both sides of every threshold, multilingual test phrases)
5. Agent Architecture Spec — root + each sub-agent: responsibility, trigger, instruction guidance, inputs/outputs, tools called
6. Tool & Data Inventory — one row per tool/data store, single source of truth cross-referenced by every use-case spec
7. Per-Use-Case Specs — the bulk of the document; one spec per wow moment using a fixed template (trigger, agents, flow, tools/data, expected output, acceptance criteria, edge cases)
8. Demo Script/Runbook — literal presenter walkthrough, timing, scenario selection, live-failure fallback guidance
9. Acceptance Criteria (global) — demo-level "what good looks like," rolling up per-use-case criteria
10. Open Questions/Risks/Assumptions — platform mechanics to confirm against the live console, GTM ambiguities to confirm with stakeholders; maintained throughout, finalized last

**Nine recommended use cases for Section 7:** Chat Intake, Voice Intake, Mid-Conversation Language Switch, Multimodal Damage Assessment, Decision Threshold Branch (Autonomous), Decision Threshold Branch (HITL), Backend Claims-Processing Reveal, Cross-Sell/Upsell Moment, Escalation/Out-of-Scope Handling.

### Critical Pitfalls (from PITFALLS.md — 17 identified, top 5 below)

1. **LLM hallucination on coverage/dollar figures, live** — a wrong number reads as "this AI can't be trusted with money" to an insurance buyer. Avoid by requiring every dollar figure/coverage term spoken on stage to be sourced from a named structured mock-data field via tool call, never free-generated; require dry-run testing of every scripted path against seed data.
2. **Non-determinism breaks the scripted wow moment** — flow-free, generative agents are probabilistic by design, which conflicts with the sales need for a repeatable script. Avoid by explicitly separating "must be deterministic" moments (decision branch, cross-sell trigger, backend reveal) from "can vary" moments, and requiring an N/N repeatability acceptance test for every deterministic moment.
3. **Voice & translation latency kills the wow moment** — natural turn-taking gap is ~200ms; production voice AI commonly runs 1,400-1,700ms, and above ~1,500ms conversations visibly feel "broken." Avoid by dry-run testing the voice/language-switch moments on venue-class (not lab) network conditions, defining a latency acceptance threshold, and restricting the live language-switch to the confirmed audio-to-audio language set with a captioned-text fallback for anything outside it.
4. **Mock data that doesn't reliably trigger the decision branch or upsell** — under ASAP pressure, mock data is the piece most likely to be left implicit ("the implementation team will figure out reasonable values"), which is exactly the ambiguity a spec must eliminate. Avoid by treating the mock dataset as a primary, versioned, canonical deliverable artifact (Section 4) with explicit "negative" records that should NOT trigger a branch, to catch over-firing.
5. **Regulatory/compliance optics of AI "deciding" insurance claims** — even in a mock demo, an insurance buyer may ask "is this actually allowed / how does this pass a regulatory audit," and an unprepared presenter looks naive. Avoid by framing auto-approval throughout the spec as "system pre-approves within a human-authorized, configurable rule set, with a visible audit-trail artifact" — never "the AI decides" — turning compliance into a wow moment rather than a landmine.

Also load-bearing: **non-credible flat threshold** (frame the $1,000 as illustrative/configurable, give it a stated rationale, so it survives underwriter-level scrutiny), **over-promising real Gemini Enterprise integration** (source every platform-capability claim, flag unverified ones, default to describing the backend reveal as a spec'd/built simulated chain rather than an out-of-the-box feature), and **missing acceptance criteria / no demo runbook** (both explicitly required by PROJECT.md and structurally the last-to-get-cut items under an ASAP timeline — the roadmap should protect them as a mandatory closing phase, not an optional polish step).

## Implications for Roadmap

Architecture research's "Suggested Authoring Order" directly implies a natural 4-phase structure, independently reinforced by the pitfall-to-spec-section mapping (every pitfall maps cleanly onto one of these four phases' sections).

### Phase 1: Foundations
**Rationale:** Narrative, capability map, decision logic, and mock data are tightly coupled — nothing downstream can be written correctly until the storyline is locked, the exact threshold/trigger rules are unambiguous, and the exact seed data exists. This is also where the highest-risk platform-capability uncertainty lives (voice/audio-to-audio language set, backend-reveal mechanics), so it benefits most from research-phase treatment.
**Delivers:** Sections 0-4 — Cover/Purpose, Demo Narrative/Storyline, Platform Capability Map, Decision Logic Spec, Mock Data Appendix.
**Addresses:** All 5 of Scott's confirmed wow moments at the narrative level; establishes the white-label framing and the "beyond the list" additions to consider (sentiment/empathy, policy-clause citation, fraud signals, personalization).
**Avoids:** Pitfalls 1 (hallucination), 4 (mock data doesn't trigger), 6 (non-credible threshold), 7 (regulatory optics), 8 (PII in mock data), 9 (over-promising integration), 11 (demo looks like a deck), 12 (no differentiation vs. Copilot/Nuance), 13 (too-custom to be white-label).

### Phase 2: Component Architecture
**Rationale:** Translates the locked foundations into the concrete component inventory that per-use-case specs will reference — must follow Phase 1 because agent/tool boundaries can't be spec'd correctly until the decision logic and mock data shape are fixed.
**Delivers:** Sections 5-6 — Agent Architecture Spec (root + sub-agents: Intake, Decisioning, Escalation, Backend Claims-Processing, Upsell), Tool & Data Inventory.
**Uses:** Root+sub-agent flow-free model, Python code tools for structured mock data, Handoff Rules for the deterministic branch, native multimodal for damage assessment (all per STACK.md/ARCHITECTURE.md).
**Implements:** The shared Backend Claims-Processing sub-agent (dual output mode) and the shallow-nesting anti-pattern guardrail.

### Phase 3: Use-Case Specs
**Rationale:** The largest section, naturally parallelizable across the 9 use cases once Phases 1-2 are stable — each use-case spec pulls from Sections 3-6 without needing to re-derive anything.
**Delivers:** Section 7 — one fixed-template spec per wow moment (Chat Intake, Voice Intake, Mid-Conversation Language Switch, Multimodal Damage Assessment, Decision Branch Autonomous, Decision Branch HITL, Backend Reveal, Cross-Sell, Escalation/Out-of-Scope).
**Addresses:** Every P1 and P2 feature from FEATURES.md, each with testable acceptance criteria and example dialogue (not just behavioral description).
**Avoids:** Pitfalls 2 (non-determinism), 3 (voice latency), 5 (photo unreliability), 10 (language-switch failure), 14 (Agent Assist scope creep), 15 (spec ambiguity).

### Phase 4: Runbook & Synthesis
**Rationale:** Can only be finalized once every use-case spec exists — this phase rolls everything up into what a presenter who did not build the demo actually does live, and closes out any remaining open questions.
**Delivers:** Sections 8-10 — Demo Script/Runbook (pre-flight checklist, recovery scripts, backup recordings), global Acceptance Criteria (rollup), Open Questions/Risks/Assumptions closeout.
**Addresses:** GTM readiness — a spec that ~18+ accounts' worth of presenters (not just the builder) can run solo.
**Avoids:** Pitfalls 16 (missing acceptance criteria) and 17 (no runbook/fallback) — both explicitly flagged as "boring, easy to skip under ASAP pressure, and hard-required" by PROJECT.md and pitfalls research alike.

### Phase Ordering Rationale

- **Dependency-driven, not preference-driven:** ARCHITECTURE.md's own authoring-order analysis (Section-by-Section "Depends On" column) is the direct source for this ordering — Phase 2 cannot start correctly before Phase 1's decision logic and mock data are locked, and Phase 3 cannot start before Phase 2's component inventory exists.
- **Risk-front-loading:** the platform's least-certain capabilities (audio-to-audio translation language set, backend-reveal mechanics, Handoff Rule exact syntax) all live in Phase 1/2 content, so uncertainty is surfaced and flagged early rather than discovered mid-way through writing 9 use-case specs.
- **Pitfall coverage confirms the grouping:** independently, PITFALLS.md's pitfall-to-spec-section mapping clusters cleanly into these same four phases — no pitfall spans more than two adjacent phases, which validates the phase boundaries rather than just the section-dependency logic.

### Research Flags

Needs a console-verification pass before/during authoring (not necessarily a full `/gsd-research-phase`, but should not be taken on faith):
- **Phase 1** — confirm the audio-to-audio translation capability, its exact supported language set, and whether it supports true mid-call switching (currently press-sourced only, not found in current CX Agent Studio technical docs); confirm there is no off-the-shelf "transcript → summary → email" Gemini Enterprise pipeline before committing the spec to a specific mechanic.
- **Phase 2** — confirm exact Handoff Rule UI/syntax and exact Python-tool session-variable mechanics in the live console (research is MEDIUM confidence on exact UI labels/fields since docs shift monthly on a 6-month-old product).

Standard patterns, low research risk:
- **Phase 3** — the per-use-case template pattern itself is well-supported by both STACK.md and ARCHITECTURE.md with HIGH-confidence primitives (Python tools, native multimodal, Handoff Rules); most of Phase 3's risk is spec-writing discipline (ambiguity, example dialogue), not platform uncertainty.
- **Phase 4** — runbook/acceptance-criteria authoring is a documentation discipline task, not a platform-research task.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack (platform capability surface) | MEDIUM-HIGH | Core architecture, tools, deployment channels verified against current official Google Cloud docs (HIGH). A few claims — audio-to-audio translation language set/mechanics, Gemini Enterprise transcript-to-email pipeline — are press-only or adjacent-product-only and explicitly flagged LOW-MEDIUM pending hands-on console confirmation. |
| Features (wow moments, competitive framing) | MEDIUM-HIGH | Platform capabilities HIGH (Google Cloud docs/search); insurance-industry practice MEDIUM (multiple 2026 industry sources, some single-source stats like the 47%/41% empathy figures); live-demo craft patterns MEDIUM (sales-demo best-practice sources, not insurance-specific). |
| Architecture (demo runtime + spec structure) | MEDIUM | CX Agent Studio shipped ~Jan 2026, after training-data cutoff; findings triangulated from official docs, one Google Developer forum thread, and one third-party community deep-dive (May 2026) — not independently re-verified line-by-line against the live console. Component names/settings should be treated as directionally correct, confirmed at build time. |
| Pitfalls (risk identification) | MEDIUM-HIGH | Platform capability claims verified against Google Cloud sources; demo-craft and insurance-domain pitfalls verified against multiple independent sources. Competitive-positioning specifics (Microsoft Copilot/Nuance insurance-vertical claims) are explicitly LOW-MEDIUM and flagged for sanity-check with Scott/Google field team before being asserted in the spec. |

**Overall confidence:** MEDIUM-HIGH — strong enough to proceed directly to requirements definition and roadmap creation; four specific items (below) need implementation-team/console or stakeholder confirmation and should be tracked as open items rather than blockers.

### Gaps to Address

- **Audio-to-audio language set for voice mid-demo switching:** press coverage of the Jan 2026 announcement describes real-time audio-to-audio translation across ~10 core languages with mid-call switching, but this is not confirmed in current CX Agent Studio technical docs (which document 28 voice-language *variants* for standard voice interaction, not translation). **Handle during planning:** flag as an open question in spec Section 10; require the implementation team to confirm directly in-console or with the Google account team before committing the live demo script to specific language pairs; spec a text/captioned fallback for any language outside the confirmed set.
- **Transcript → summary → drafted-email pipeline mechanics:** no documented off-the-shelf Gemini Enterprise feature performs this exact CX-transcript-to-email workflow; adjacent capabilities exist (meeting-transcript action-item extraction, NotebookLM Enterprise summarization) but aren't confirmed for this use case. **Handle during planning:** spec this explicitly as a custom-built in-session sub-agent/tool chain (Pattern 2 from ARCHITECTURE.md), never as an assumed built-in feature; this is also the single most consequential Pitfall-9 (over-promising) risk in the whole project.
- **Handoff Rule exact syntax/UI and Python-tool session-variable mechanics:** described at the conceptual/architectural level with HIGH confidence that the *pattern* is correct (deterministic tool computes decision → writes session variable → Handoff Rule routes on it), but exact console field names/config steps are MEDIUM confidence, sourced partly from third-party community write-ups. **Handle during planning:** implementation team should verify against the live console early in Phase 2; not a blocker for spec-writing since the pattern itself is well-supported.
- **Regulatory/competitive framing specifics:** the "auto-approval must be framed as governed/auditable, not autonomous AI judgment" guidance is well-grounded in general NAIC/insurance-automation literature (MEDIUM confidence), but the specific Microsoft Copilot/Nuance insurance-vertical competitive claims are LOW-MEDIUM confidence (thin public sourcing). **Handle during planning:** treat competitive-positioning language as a GTM/stakeholder input (Srini/Stephanie/Hallie/Scott) to validate, not something asserted unilaterally in the spec.

## Sources

### Primary (HIGH confidence)
- [CX Agent Studio overview, Agents, Instructions, Handoff rules, Tools, Python code tools, Data store tools, Connector tools, Rich response widgets, Web widget deployment, Google Telephony Platform deployment, Flow-based agents, Best practices, Languages reference, Sample agent applications, Release notes](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio) — full platform capability surface, architecture, deployment channels
- [Google Cloud press release, Jan 11 2026](https://www.googlecloudpresscorner.com/2026-01-11-Google-Cloud-Brings-Shopping-and-Customer-Service-Together-with-Gemini-Enterprise-for-Customer-Experience) — "photo of a damaged appliance" multimodal precedent
- [Image understanding | Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/capabilities/image-understanding) — native multimodal capability
- [Conversation history | Dialogflow CX](https://docs.cloud.google.com/dialogflow/cx/docs/concept/conversation-history) — transcript retention/export mechanics

### Secondary (MEDIUM confidence)
- [Gemini Enterprise apps and data stores](https://docs.cloud.google.com/gemini/enterprise/docs/apps-data-stores) — adjacent product surface, not CX Agent Studio itself
- Architecture advice for CX Agent Studio (Data Store vs. Connectors vs. OpenAPI/MCP), Google Developer forums — informed the Data-Store-vs-tool anti-pattern
- CX Agent Studio Architecture Deep Dive (Yash Kavaiya, Google Cloud Community/Medium, May 2026) — third-party synthesis of official docs
- [Engineering for Real-Time Voice Agent Latency — Cresta](https://cresta.com/blog/engineering-for-real-time-voice-agent-latency) — turn-taking latency benchmarks
- Multiple 2026 insurance-industry AI/FNOL automation sources (Strada, VCA Software, Perspective AI, Convin.ai) — feature/wow-moment framing, empathy and fraud-signal industry patterns

### Tertiary (LOW confidence, needs validation)
- Constellation Research, TTEC Digital, CMSWire coverage of Jan 2026 announcement — audio-to-audio translation, 10-language claim, "built with Google DeepMind" — not found in current CX Agent Studio technical docs, needs direct platform confirmation
- Nuance/Microsoft Copilot Studio insurance-vertical competitive positioning — general sourcing only, insurance-specific claims not independently verified

---
*Research completed: 2026-07-08*
*Ready for roadmap: yes*
