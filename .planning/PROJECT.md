# CX Agent V2 Demo — Use-Case Specification

## What This Is

A use-case specification package for a white-labeled **"Version 2.0" WOW demo** of an insurance first-notice-of-loss (FNOL) claims CX agent, built on **Google CX Agent Studio** (Gemini Enterprise for Customer Experience). The spec is handed to an implementation team so *they* build the demo in the Studio — this project produces the specification, not the demo itself.

It is a **go-to-market (GTM) asset**: a reusable, generic demo the sales team can put in front of multiple insurance accounts to sell the CX-agent capability (Google account rep Scott Hitchcock alone has ~18 accounts). The guiding principle is Scott's: **"show, don't tell"** — real interactive workflows over slides.

## Core Value

A specification concrete and compelling enough that the implementation team can build a show-don't-tell demo that lands every wow moment — voice + mid-demo language switching, a visible autonomous-vs-human decision branch, the backend claims-processing reveal, and the cross-sell "cost center → profit center" moment.

If everything else fails, this must hold: **the implementation team can build the demo directly from the spec, and it wows customers.**

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Current scope. Building toward these. All are hypotheses until the spec ships and the demo is built. -->

- [ ] Demo narrative / storyline that stitches every wow moment into one coherent, white-label FNOL scenario
- [ ] Voice channel use case, including mid-demo language switching (leaning on Studio's built-in 40+ languages / audio-to-audio translation)
- [ ] Backend claims-processing reveal: conversation transcript → Gemini Enterprise ingestion → senior-specialist summary → auto-drafted customer email (e.g. "sorry about your laptop, here's a recommended repair partner")
- [ ] Both autonomous and human-in-the-loop (HITL) paths shown end-to-end (intake → triage → backend processing)
- [ ] Decision logic with a dollar threshold (~$1,000): small claims auto-approve, larger claims route to a human assessor — the branch must be visibly demonstrated
- [ ] Cross-sell / upsell moment (e.g. customer mentions a tree hit their car; system sees they don't insure their boat and prompts a bundle offer)
- [ ] White-label / generic framing (no customer-specific branding) so the demo is reusable across insurance accounts and sectors
- [ ] Mock/simulated data specification: seeded scenarios and canned policy/coverage/claim data that reliably trigger the auto-approve vs. escalate branch and the upsell moment
- [ ] Mapping of each use case to CX Agent Studio's state-of-the-art capabilities (agent instructions, tools/MCP/connectors, data stores, voice/multilingual, Gemini Enterprise)
- [ ] Additional SOTA wow moments beyond Scott's list, surfaced via research (e.g. personalization, cross-channel context) — echoing Akash's and Srini's multi-sector framing
- [ ] Acceptance criteria / "what good looks like" per use case, so the implementation team knows when the demo is done

### Out of Scope

<!-- Explicit boundaries. Includes reasoning to prevent re-adding. -->

- Actually building the demo — this project produces the spec; a separate implementation team builds it in CX Agent Studio
- Real system integrations (live claims/policy systems) — demo uses mock/simulated data; real integrations need credentials and are premature
- Custom branding / bespoke UI — deliberately white-label; branding waits until an actual POV/implementation
- Agent Assist (real-time summarization, coaching, knowledge retrieval for human agents) — Akash's suggestion, not Scott's; customer-dependent and only relevant if the engagement expands into full contact-center territory. Deferred to a later version.
- Flow-based agent design — CX Agent Studio is flow-free; agents are defined by instructions + tools, not visual flows

## Context

- **Origin:** V1 is a white-labeled, generic FNOL demo inspired by a highly-customized Furniture Care Protection (FCP) implementation. V1 shows a **chat** intake happy path: agent "Alex" greets, collects claimant name + policy ID, returns deductible/coverage, requests a photo of the damaged item, runs multimodal damage analysis, and returns a claim number. It also demonstrates escalation/HITL (injury case) and out-of-scope handling (weather question). Built on CX Agent Studio with mock data.
- **The meeting:** V2 wishlist comes from a demo to Scott Hitchcock (Google account rep) on 2026-07-07. Scott endorsed V1 as a solid baseline and the white-label approach; the V2 list below is what he wants before taking it to customers.
- **Platform (post-Jan-2026 product):** CX Agent Studio is part of Gemini Enterprise for Customer Experience, announced ~Jan 11 2026 / NRF 2026. It is low-code and **flow-free** — a root agent (+ sub-agents) driven by natural-language instructions + tools, with Gemini doing generative reasoning. Voice + multilingual are largely built-in (multimodal chat/voice/image, human-like voices in 40+ languages, audio-to-audio translation in ~10 core languages). Backend integration is via out-of-the-box connectors + MCP support + custom tools. This capability set is newer than the assistant's training data and should be verified against current Google Cloud docs during research.
- **Voice/telephony note:** Akash — voice is straightforward to demo using Google's Telephony platform (Studio voice channel already available); avoid bringing external carriers if tight Studio integration is desired.
- **GTM angle:** Insurance buyers are often Microsoft/Azure-centric (Copilot + Nuance) and resist disruption; the demo must show clear, superior value. Suggested sales motion: start with an isolated high-value use case (FNOL chat/voice) as an MVP to win a champion, then expand into contact-center and broader backend automation. Scott offered to connect Nerdery to Google's **Propel program** (POC/MVP funding) via his contact Shrey.
- **Audience of the spec:** primary — the implementation team building in CX Agent Studio; secondary — internal GTM stakeholders (Srini, Stephanie, Hallie/marketing) and Scott.

## Constraints

- **Platform**: Target Google CX Agent Studio (Gemini Enterprise for CX) — flow-free, instruction + tool + MCP/connector model — because that is where V1 lives and where the implementation team will build V2.
- **Deliverable**: Specification documents only (no demo build, no runnable code) — this is a GTM/spec project; a separate team implements.
- **Data**: Mock/simulated data only — demo-safe, no real customer systems.
- **Branding**: White-label / generic — reusable across accounts and sectors; no per-customer branding.
- **Timeline**: ASAP (weeks) — prioritize the highest-impact wow moments first for a fast follow-up demo.
- **Principle**: Show, don't tell — real interactive workflows, not a deck.

## Key Decisions

<!-- Decisions that constrain future work. Add throughout project lifecycle. -->

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Deliverable is a use-case specification, not a built demo | GTM play; a separate implementation team builds in CX Agent Studio | — Pending |
| Target CX Agent Studio (flow-free, instruction+tool model) | V1 lives there; it's where V2 will be built | — Pending |
| Mock/simulated data only | Demo-safe, no real integrations needed for a sales demo | — Pending |
| Keep white-label / generic | Reusable across ~18+ accounts and multiple sectors; branding waits for POV | — Pending |
| Defer Agent Assist | Scott's guidance: customer-dependent, only if engagement expands to contact-center | — Pending |
| Showcase SOTA features beyond Scott's list | "Show, don't tell" + differentiate vs. Microsoft/Nuance; maximize wow | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-07-08 after initialization*
