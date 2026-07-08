# Feature Research

**Domain:** Insurance FNOL / claims-servicing CX agent — GTM sales demo (white-label, multi-sector)
**Researched:** 2026-07-08
**Confidence:** MEDIUM-HIGH (platform capabilities HIGH via Google Cloud docs/search; insurance-industry practice MEDIUM via multiple 2026 industry sources; live "wow demo" patterns MEDIUM via sales-demo best-practice sources)

## Context Recap

This is not a product feature backlog — it is a **sales-demo use-case menu**. The question isn't "what will users use daily" but "what makes an insurance buyer lean forward in the first 90 seconds, and what makes them trust this over their incumbent Microsoft/Nuance stack." Every feature below is scored for demo-impact and demo-effort, not production maturity. V1 baseline (already built) is chat-only, single-language, single-path happy flow: greet → collect name+policy → coverage/deductible lookup → photo damage analysis (multimodal) → claim number, plus one injury-escalation branch and one out-of-scope deflection.

## Feature Landscape

### Table Stakes (Buyers Expect These in Any 2026 CX Agent Demo)

Missing these makes the demo feel like a 2023-era chatbot, not a 2026 agentic platform.

| Feature | Why Expected | Wow Factor | Demo Effort | Sectors | Notes |
|---------|---------------|------------|--------------|---------|-------|
| Structured FNOL intake (identity + policy verification, guided Q&A) | Baseline of every claims chatbot since ~2019; buyers assume it works | LOW | LOW (already in V1) | All (auto, home, pet, marine, commercial, warranty/FCP) | Keep tight — don't over-narrate, it's not the hook |
| Coverage / deductible lookup against mock policy data | Table stakes for any "self-serve claims" pitch | LOW | LOW (V1 has it) | All | Grounds the agent in real(-looking) policy data, sets up the dollar-threshold logic later |
| Multimodal photo damage assessment | Every 2025-26 claims-AI vendor demos this; Nuance/Copilot also show document/image ingestion | MEDIUM | LOW (V1 has it) | Auto, home, property, marine, equipment/warranty — not applicable to pure liability/life | Already validated in V1; extend with a second modality (e.g., video walk-around) only if time allows — see differentiator |
| Injury / high-risk escalation to a human | Buyers immediately ask "what happens when it's not a fender-bender" — this is the trust question | MEDIUM (as reassurance, not spectacle) | LOW (V1 has it) | All | This is actually your BEST anti-"black box AI" answer — reframe from table-stakes checkbox to trust proof point in narration |
| Out-of-scope / off-topic deflection | Buyers test edge cases live; a bot that answers "what's the weather" like a claims question kills credibility instantly | LOW | LOW (V1 has it) | All | Keep — this is a live-demo risk mitigant, see Anti-Features |
| Basic multilingual chat (text) | Any enterprise buyer assumes Unicode/i18n support in 2026; Microsoft/Nuance both support this | LOW | LOW | All | Not differentiating alone — differentiator is *mid-conversation switching* and *voice* (below) |
| Claim number / confirmation issuance | Closes the loop; without it the demo feels incomplete | LOW | LOW (V1 has it) | All | — |
| Natural, non-robotic conversational tone | Buyers compare directly against legacy IVR/Nuance scripted flows; stilted phrasing reads as "old tech" | MEDIUM | LOW (prompt/instruction tuning) | All | Cheap win — invest disproportionate care in agent persona instructions since it's free and highly visible |

### Differentiators (Scott's Confirmed V2 Must-Haves — The Core Wow Moments)

These are the five items explicitly requested from the 2026-07-07 meeting. Prioritize these first; they are the spine of the demo narrative.

| Feature | Value Proposition | Wow Factor | Demo Effort | Sectors | Notes |
|---------|---------------------|------------|--------------|---------|-------|
| **Voice channel + mid-demo language switching** | Google's Live API supports human-like voices in 40+ languages with direct audio-to-audio (A2A) translation in ~10 core languages, and *native mid-conversation language switching* without the caller re-announcing themselves — this is a capability Microsoft's Copilot Studio real-time voice agents do not yet match (GA'd April 2026, North-America-only region lock as of writing). Switching languages live, on a phone call, in front of the buyer, is the single most visceral "this isn't a script" moment available. | **VERY HIGH** | MEDIUM (Studio voice channel is built-in per Akash's note — avoid external telephony carriers to keep it tightly demoable; main effort is scripting a clean bilingual scenario and rehearsing the switch) | All — language need is universal, but resonates hardest with accounts with Spanish/bilingual-heavy books (auto, home, health) | Use Google's own telephony, not a bolted-on carrier — reduces live-demo failure surface and keeps the win squarely "built-in Google capability" vs. integration risk |
| **Backend claims-processing reveal** (transcript → Gemini Enterprise ingestion → senior-specialist agent summary → auto-drafted customer email with repair-partner referral) | Buyers' AI skepticism is usually "fine, the chatbot talks to customers, but then what — does a human still have to do all the real work?" This answer is: no, the system does the specialist's paperwork too. Showing the "screen behind the screen" (a structured case file + an auto-drafted, personalized, on-brand email) converts "cute chatbot" into "this changes our operating model." | **VERY HIGH** — buyers consistently describe this class of reveal as the moment they mentally shift from evaluating a chatbot to evaluating a claims-operations platform | MEDIUM-HIGH (needs: a second "specialist" view/UI or transcript panel, a summarization step, and an auto-drafted email artifact — CX Agent Studio supports MCP/connector tool calls and multi-agent orchestration, so this is a tool-call + sub-agent pattern, not custom engineering) | All — every insurance vertical has a "back-office does the real work" narrative; especially strong for warranty/FCP (repair-partner referral is literally the FCP model already validated in V1) | This is the single highest-leverage "beyond Scott's list" opportunity too: don't just show the artifact, show the transcript-to-summary *transformation* live (split-screen or sequential reveal) — the transformation is the wow, not the endpoint |
| **Autonomous vs. human-in-the-loop paths, shown end-to-end** | Directly answers the #1 objection every insurance buyer has about agentic AI: "who's accountable when it's wrong." Showing BOTH paths in one demo (not just claiming HITL exists) proves the system knows its own limits. | **HIGH** | MEDIUM (requires two full scripted scenarios plus a visible "handoff" moment — e.g., a case file appearing in a mock adjuster queue) | All — this is a universal insurance trust requirement, arguably strongest for high-severity lines (auto liability, commercial, health/injury) | Pair with the dollar-threshold logic below — they are naturally the same demonstration, don't build them as separate moments |
| **Dollar-threshold decision logic (~$1,000): auto-approve small claims vs. route large claims to a human assessor, visibly demonstrated** | This is where "agentic AI" becomes "agentic AI with governance." It's the single feature that most directly rebuts the Microsoft/Nuance narrative of "we also have a chatbot" — the differentiator is the *visible, explainable decision boundary*, not just automation. Real-world STP benchmarks: leading P&C carriers hit 60%+ straight-through-processing for claims under a defined severity threshold; the industry trend is now toward risk-adjusted (not flat) thresholds (claim type + customer history + fraud score), which is a strong "beyond Scott's list" upsell to the base ask (see Differentiator below). | **VERY HIGH** | LOW-MEDIUM (two claim amounts, one simple IF/branch in agent instructions — cheapest high-wow feature in the whole spec) | All | Run BOTH branches back-to-back in the same demo session with two claim scenarios (e.g., $400 phone screen vs. $8,000 collision) — the side-by-side contrast is what sells it, not either branch alone |
| **Cross-sell / upsell moment ("cost center → profit center")** | Reframes the whole pitch from "cost savings on claims handling" to "claims handling as a revenue channel" — this is a CFO/CRO-level argument, not just an ops argument, and it's the line Scott will use to get budget released. Industry pattern: insurers are already using post-claim moments (e.g., home claim → prompt flood coverage) as the highest-intent cross-sell moment in the entire customer lifecycle, because the customer is actively engaged and trust is high. | **HIGH** (especially for business-buyer / VP audiences, less flashy for technical audiences than the voice-switch or backend-reveal moments) | LOW (mock policy data flags one uninsured asset category; one proactive offer line in the closing turn) | All — auto→boat/RV, home→flood/umbrella, pet→multi-pet, commercial→cyber, life→disability — genuinely universal pattern, easy to re-skin per sector | Land this as the *closing* beat of the demo, right after claim resolution — "and by the way, we noticed X" — mirrors the real conversational moment insurers cite as highest-conversion |

### Differentiators — Beyond Scott's List (SOTA Capabilities Worth Adding)

Researched beyond the confirmed V2 requirements per the "echo Akash's and Srini's multi-sector framing" instruction in PROJECT.md. Ranked by wow-per-effort.

| Feature | Value Proposition | Wow Factor | Demo Effort | Sectors | Notes |
|---------|---------------------|------------|--------------|---------|-------|
| **Real-time sentiment/empathy detection with visible tone adaptation** | Industry data: carriers pairing live agents with emotion analytics report claims processed ~47% faster and satisfaction up ~41% (industry-reported, single-source — MEDIUM confidence). More importantly for a demo: showing the agent *detect* distress in a claimant's voice/word choice and visibly shift tone/pacing is one of the few AI capabilities buyers viscerally feel rather than just believe. Directly rebuts "AI has no empathy" — the single most common insurance-buyer objection to agentic claims handling. | **VERY HIGH** | LOW-MEDIUM (scripted distressed-caller scenario + instruction tuning for tone-adaptive responses; no new backend needed — this can be entirely prompt/persona work) | All, but strongest for auto/health/life (injury, loss-of-use scenarios) | Cheapest "beyond the list" high-wow addition — recommend bundling into the injury-escalation scenario already in V1 rather than building a new scenario from scratch |
| **Fraud-signal surfacing in the specialist summary** | 2026 industry pattern: best conversational-AI claims platforms surface fraud signals inline (inconsistent timelines, mismatched geography, duplicate claim patterns) as part of the case file handed to the human. This is a natural extension of the backend-reveal moment already planned — adds "risk management" credibility to the specialist summary artifact for minimal extra scripting. | **HIGH** | LOW (one flagged data point added to the mock case file/specialist summary already being built for the backend reveal — no new UI, no new flow) | All, esp. auto and property (highest fraud-incidence lines) | Free rider on the backend-reveal build — do not skip this, it's the cheapest add on the list relative to impact |
| **Cross-channel context continuity (chat → voice handoff, no re-explaining)** | 2026 Salesforce State of Service data: 71% of customers now use 3+ channels per issue; 64% expect AI to carry full context across every touchpoint; Gartner estimates fragmented handoffs add 42% to handle time and cost 18 CSAT points when context is lost. This is a stated, quantified pain point for every contact-center buyer and a direct shot at Microsoft's chat-first architecture (voice bolted on as text-to-speech wrapper is a documented weakness of several competing platforms). | **HIGH** | MEDIUM (requires scripting two channel legs of one scenario — e.g., start FNOL in chat, "call back" and resume in voice with the agent already knowing the claim status) | All | Strong complement to the voice/language-switching moment — could be staged as one continuous narrative: chat intake → phone follow-up → resumed with zero repetition |
| **Real-time knowledge grounding / RAG against policy documents (cite the actual clause)** | Rather than the agent asserting "you're covered," show it cite the specific policy clause/PDF section it pulled the answer from. This addresses the #2 insurance-buyer objection after empathy: "will it hallucinate coverage answers." Grounding in an actual (mock) policy document with visible citation is the antidote. | **HIGH** (technical/compliance-minded buyers specifically) | LOW-MEDIUM (CX Agent Studio data stores support grounded retrieval by design — mostly a matter of loading a mock policy PDF/knowledge base and having the agent reference it explicitly) | All — especially resonant for regulated lines (health, life, commercial) where "explainability" is a compliance requirement, not a nicety | Consider pairing narration explicitly: "note it didn't just say 'yes you're covered' — it cited section 4.2" |
| **Proactive outreach (catastrophe/weather-triggered)** | Industry pattern: insurers using AI to identify affected policyholders in a geography during a weather event and proactively reach out before the customer even calls — reframes FNOL from reactive intake to proactive service. Strong strategic story ("we don't wait for the phone to ring") but requires an outbound-trigger scenario, which is a different demo shape than the rest of the (inbound, conversational) narrative. | **MEDIUM-HIGH** (strategically compelling, less visually dramatic live) | MEDIUM-HIGH (requires a distinct "trigger event" staging mechanism — e.g., simulate a storm alert kicking off an outbound message — doesn't reuse the existing conversational demo flow) | Auto, home, property, commercial — not applicable to life/health/pet | Good "Act 2" addition for a longer-form demo or a dedicated follow-up meeting rather than the core live-conversation demo; higher effort-to-wow ratio than the others above, so sequence it lower priority unless there's a specific weather-exposed account in the room |
| **Personalization using prior interaction/claim history** | Agent references the claimant's tenure, prior claims, or preferences ("since your last claim we noted you prefer email over calls") — makes the mock customer feel like a real relationship, not a fresh intake every time. Cheap to fake with seeded mock data; strong emotional resonance without new architecture. | **MEDIUM** | LOW (seed one prior-interaction fact into the mock customer record; reference it in agent instructions) | All | Easy, cheap addition — can be layered onto almost any scenario without new build, good "sprinkle throughout" rather than a standalone moment |
| **Speed/efficiency metric overlay ("claim triaged in under 5 minutes" or similar live counter)** | Industry-reported benchmarks vary widely (agentic FNOL-to-triage dropping from 4-8 hours to under 5 minutes is repeatedly cited in 2026 claims-automation sources), but showing a live elapsed-time counter during the demo — "that took 90 seconds, a human intake call for this averages 12 minutes" — anchors the ROI conversation concretely rather than abstractly. | **MEDIUM** | LOW (a visible timer/comparison stat card; no functional build) | All | Cheap, high-leverage for the economic-buyer audience (CFO/COO) in the room — pair with the cross-sell moment as the "profit center" closing 1-2 punch |

### Anti-Features (Do Not Show — Real Risk to a Sales Demo)

| Feature | Why It Seems Appealing | Why Problematic | Alternative |
|---------|---------------------------|---------------------|--------------|
| Real system integrations (live policy admin, claims, or payment systems) | "Wow, it's really live!" feels more impressive than mock data | Live systems fail live, mid-pitch, in front of the buyer — the single highest-severity risk identified across sales-demo best-practice research ("if your demo environment breaks, the deal is at risk, period"); also a data-security/compliance non-starter for a generic white-label asset with no signed contract in place | Curated, seeded mock data that reliably triggers every branch (auto-approve, escalate, cross-sell) every time — PROJECT.md already scopes this correctly, reinforce it |
| Full autonomous claim *payment* execution (moving real or simulated money out) | Feels like the ultimate "no human needed" proof point | Insurance buyers' actual trust threshold is precisely at the point money changes hands — showing unconditional autonomous payout undercuts the carefully built HITL/dollar-threshold trust narrative rather than reinforcing it, and risks looking reckless to risk/compliance stakeholders in the room | Show the auto-*approval* decision and a "payment initiated" confirmation state, not a live payment rail — the decision is the wow, not the money movement |
| External/non-Google telephony carriers for the voice channel | More "real-world" feeling, possibly seen as more flexible | Adds a second point of failure and dilutes the differentiation story — the wow is Google's own native voice/language capability; bringing an outside carrier makes the win harder to attribute to the platform being sold, and (per Akash's note) is unnecessary since Studio's voice channel is already available | Use Google's built-in Studio voice/telephony channel exclusively |
| Exposing raw model chain-of-thought / internal reasoning traces | "Look how smart it is" transparency instinct | Raw reasoning traces read as unpolished/unfinished to a business buyer, can reveal awkward phrasing or model uncertainty language mid-pitch, and are not how the production experience will look | Show the *outcome* of reasoning (structured case summary, decision with rationale line) — curated, not raw |
| Deep, unscoped personalization requiring real customer PII in a white-label generic demo | Feels more "real" and relationship-driven | Directly conflicts with the white-label/no-customer-branding constraint and raises data-handling questions with zero benefit — buyers know it's a demo, they don't need real people's data to be convinced | Seeded, clearly-fictional mock customer with a rich-enough profile to demonstrate personalization patterns |
| Building bespoke, sector-specific branching for every insurance vertical shown in one sitting | Tempting to prove "it works for auto AND home AND pet AND commercial" | Turns a tight, rehearsable 10-15 minute demo (the sales-demo best-practice ceiling before attention drops) into an unfocused tour; dilutes every wow moment instead of landing a few hard | Keep ONE white-label scenario spine (the confirmed FNOL narrative) and note in the spec which mock data swaps make it trivially re-skinnable per sector for a specific account — don't build multiple full scenarios for one demo |
| Full underwriting / risk-pricing automation as part of this demo | Adjacent capability, could seem like a natural "go big" addition | Out of scope per PROJECT.md (this is a claims/FNOL demo, not an underwriting demo) and conflates two different buyer personas (claims ops vs. underwriting) — mixing them dilutes the pitch and risks scope creep into a much larger, unvetted build | Stays out of scope; flag as a possible *future* demo module, not part of this spec |
| Agent Assist (real-time human-agent coaching/summarization) | Natural extension once the specialist-summary backend reveal exists | Explicitly deferred in PROJECT.md — Scott didn't ask for it, and it shifts the pitch from "isolated high-value use case" (the recommended sales motion) into full contact-center territory prematurely | Defer to a later version once the FNOL use case has won a champion account, per the documented GTM sequencing (isolated use case first → contact-center expansion later) |

## Feature Dependencies

```
[Voice channel] ──requires──> [Studio native telephony/voice API]
[Mid-demo language switching] ──requires──> [Voice channel]
[Cross-channel context continuity] ──requires──> [Voice channel] AND [Chat channel (V1 baseline)]

[Backend claims-processing reveal] ──requires──> [MCP/connector or sub-agent tool-call pattern]
[Fraud-signal surfacing] ──enhances──> [Backend claims-processing reveal]  (free rider, same artifact)
[Auto-drafted customer email] ──requires──> [Backend claims-processing reveal]

[Dollar-threshold decision logic] ──requires──> [Coverage/policy mock data (V1 baseline)]
[Autonomous vs. HITL paths] ──requires──> [Dollar-threshold decision logic]  (same demonstration, two claim scenarios)
[Autonomous vs. HITL paths] ──enhances──> [Backend claims-processing reveal]  (HITL path routes into the specialist-summary artifact)

[Cross-sell/upsell moment] ──requires──> [Mock customer policy data with a flagged coverage gap]
[Personalization] ──enhances──> [Cross-sell/upsell moment]  (prior-interaction data makes the offer feel earned, not generic)

[Real-time knowledge grounding/RAG] ──enhances──> [Coverage/deductible lookup (V1 baseline)]
[Sentiment/empathy detection] ──enhances──> [Injury/high-risk escalation (V1 baseline)]  (bundle, don't build separately)

[Proactive outreach] ──conflicts with──> [Single inbound-conversation demo narrative]  (different demo shape; stage separately if used at all)
[Multi-sector branching in one sitting] ──conflicts with──> [10-15 minute demo ceiling / tight rehearsable narrative]
```

### Dependency Notes

- **Mid-demo language switching requires the voice channel:** Google's audio-to-audio translation and native language-switching behavior are Live API voice capabilities; there is no equivalent mid-conversation switch behavior to demo in text chat, so the voice channel must be built first (or simultaneously) for this to be demonstrable at all.
- **Fraud-signal surfacing enhances (doesn't require new work beyond) the backend claims-processing reveal:** both consume the same case-file/specialist-summary artifact. Sequence this as an addition to that build task, not a separate feature.
- **Autonomous vs. HITL paths and the dollar-threshold logic are effectively one demonstration:** don't scope or build these as two separate deliverables — script two claim amounts and let the branch point *be* the demonstration of both requirements simultaneously.
- **Personalization enhances the cross-sell moment:** an upsell offer lands better dramatically when the agent shows it "knows" the customer (tenure, prior claim) rather than appearing to guess. Cheap to layer on, not a blocker if time runs short.
- **Proactive outreach conflicts with the core narrative shape:** every other confirmed feature is a single continuous inbound conversation; proactive outreach is an outbound-triggered vignette. If included, stage it as a distinct "Act 2" rather than trying to weave it into the FNOL conversation flow — forcing it in creates narrative confusion, the opposite of a wow moment.
- **Multi-sector branching conflicts with the demo-length ceiling:** sales-demo best practice research consistently flags 10-15 minutes as the point where enterprise-buyer attention degrades. The white-label requirement is satisfied by *swappable mock data* (documented in the spec for the implementation team), not by showing multiple sector variants live in one sitting.

## MVP Definition

Reframed for a demo spec: "MVP" = the minimum set of wow moments that make this V2 demo clearly superior to the V1 baseline and to a Microsoft/Nuance pitch, buildable in weeks.

### Launch With (V2 Core — Scott's Confirmed List)

- [ ] Voice channel with mid-demo language switching — the single most differentiating, hardest-to-replicate-on-Azure capability
- [ ] Backend claims-processing reveal (transcript → summary → auto-drafted email) — converts "chatbot" framing into "claims-ops platform" framing
- [ ] Autonomous vs. HITL paths shown end-to-end, driven by the dollar-threshold branch — the trust/governance proof point
- [ ] Cross-sell/upsell closing moment — the CFO-level "profit center" argument
- [ ] White-label mock data set that reliably triggers every branch above, every time, live

### Add After Validation (V2.x — Cheap, High-Impact "Beyond the List" Additions)

- [ ] Fraud-signal surfacing in the specialist summary — free rider on the backend reveal, add once that artifact exists
- [ ] Sentiment/empathy tone adaptation — bundle into the existing injury-escalation scenario, low incremental cost
- [ ] Real-time knowledge grounding with visible policy-clause citation — strengthens the existing coverage-lookup moment, addresses the "will it hallucinate" objection
- [ ] Personalization via seeded prior-interaction data — cheap layer onto the cross-sell moment
- [ ] Speed/efficiency metric overlay — cheap, strengthens the economic-buyer close

### Future Consideration (V3+ — Higher Effort, Different Demo Shape or Out of Current Scope)

- [ ] Cross-channel context continuity (chat → voice resume) — high value but requires a two-leg scenario structure distinct from the current single-conversation narrative; strong candidate for the *next* fast-follow demo iteration
- [ ] Proactive/catastrophe-triggered outreach — strategically compelling but a different (outbound) demo shape; best suited to a dedicated follow-up meeting once FNOL has landed
- [ ] Agent Assist / human-agent coaching — explicitly deferred per PROJECT.md; revisit only if an account's engagement expands into full contact-center scope
- [ ] Full underwriting/pricing automation — explicitly out of scope; a different buyer persona and a much larger build

## Feature Prioritization Matrix

| Feature | User Value (Wow) | Implementation Cost | Priority |
|---------|--------------------|------------------------|----------|
| Dollar-threshold auto-approve vs. HITL branch | HIGH | LOW-MEDIUM | P1 |
| Voice + mid-demo language switching | HIGH | MEDIUM | P1 |
| Backend claims-processing reveal | HIGH | MEDIUM-HIGH | P1 |
| Cross-sell/upsell moment | HIGH | LOW | P1 |
| Fraud-signal surfacing (bundled into reveal) | HIGH | LOW | P2 |
| Sentiment/empathy tone adaptation | HIGH | LOW-MEDIUM | P2 |
| Real-time knowledge grounding with citation | HIGH | LOW-MEDIUM | P2 |
| Personalization via seeded history | MEDIUM | LOW | P2 |
| Speed/efficiency metric overlay | MEDIUM | LOW | P2 |
| Cross-channel context continuity (chat→voice) | HIGH | MEDIUM | P3 |
| Proactive/catastrophe outreach | MEDIUM-HIGH | MEDIUM-HIGH | P3 |
| Agent Assist / human coaching | MEDIUM | HIGH | P3 (deferred, out of current scope) |
| Full underwriting automation | LOW (wrong buyer persona for this demo) | HIGH | P3 (out of scope) |

**Priority key:**
- P1: Must have — Scott's confirmed V2 list, build first
- P2: Should have — cheap-to-demo, high-impact additions that strengthen P1 moments without new architecture
- P3: Future consideration — higher effort, different demo shape, or explicitly deferred/out of scope

## Competitor Feature Analysis

| Feature | Microsoft (Copilot Studio + Dynamics 365 Contact Center + legacy Nuance) | Google (Gemini Enterprise CX Agent Studio) | Our Demo Approach |
|---------|----------------------------------------------------------------------------|---------------------------------------------------------------------------|------------------------|
| Real-time voice AI | GA'd April 2026 for Dynamics 365 Contact Center; speech-to-speech, but real-time voice model hosted North-America-only as of the announcement | Native audio-to-audio in ~10 core languages, human-like voices in 40+ languages, built into CX Agent Studio's Live API from day one | Lead with voice + live language switching as the opening wow moment — Google's global multilingual reach vs. Microsoft's region-locked GA is a direct, citable differentiator |
| FNOL/claims summarization | Copilot can generate call transcripts and summaries from Teams Phone calls for FNOL; positioned as an agent-productivity aid | Gemini Enterprise ingestion → structured specialist summary → auto-drafted customer artifact (email), positioned as a claims-ops transformation, not just a note-taking aid | Emphasize the *auto-drafted, ready-to-send customer email* as the concrete artifact — goes one step further than transcript summarization |
| Legacy Nuance sunset | On-premise Nuance support ends June 2026, hosted support ended December 2025; Microsoft is explicitly telling Nuance customers to migrate to Dynamics 365/Copilot Studio | N/A — clean modern platform, no legacy migration baggage | Where relevant/appropriate in sales conversation (not necessarily inside the demo itself), note the forced-migration timing as a window of opportunity — incumbents are mid-transition right now |
| Flow-based vs. instruction-based agent design | Copilot Studio historically flow/topic-based (though evolving) | CX Agent Studio is explicitly flow-free — natural-language instructions + tools/MCP + Gemini reasoning | Frame the backend-reveal and dollar-threshold logic as *emergent from instructions*, not hard-coded flow branches — reinforces "this adapts, it doesn't just follow a script" |
| Governance / human-in-the-loop | Present via Dynamics 365 Contact Center supervisor tooling; positioned primarily as agent-productivity/coaching | Demonstrated live, end-to-end, as a visible customer-facing decision branch (auto-approve vs. route to human) tied to a business-meaningful dollar threshold | This is a *narrative* differentiator more than a raw-capability one — show the decision boundary transparently rather than just claiming "human oversight exists" |
| Data grounding / RAG | Present (Copilot grounds in M365/Dynamics data) | Present via CX Agent Studio data stores + MCP connectors | Not a clear platform differentiator — de-emphasize as a competitive point, use it instead to defeat the "will it hallucinate" objection generically |

## Sources

- [Customer Experience Agent Studio | Google Cloud](https://cloud.google.com/gemini-enterprise-cx/cx-agent-studio)
- [Gemini Enterprise for CX | Google Cloud Documentation](https://docs.cloud.google.com/customer-engagement-ai)
- [Configure language and voice | Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/live-api/configure-language-voice)
- [Gemini Live API overview | Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/live-api)
- [MCP tools | CX Agent Studio | Google Cloud Documentation](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/tool/mcp)
- [Set up your custom MCP server data store | Gemini Enterprise](https://docs.cloud.google.com/gemini/enterprise/docs/connectors/custom-mcp-server/set-up-custom-mcp-server)
- [Microsoft Dynamics 365 Contact Center is now generally available](https://www.microsoft.com/en-us/dynamics-365/blog/business-leader/2024/07/01/microsoft-dynamics-365-contact-center-is-now-generally-available/)
- [Microsoft Copilot Studio Real-Time Voice Agents for Dynamics 365 Contact Center](https://windowsforum.com/threads/microsoft-copilot-studio-real-time-voice-agents-for-dynamics-365-contact-center.415566/)
- [Insurance - Modernize claims processes – Microsoft Adoption](https://adoption.microsoft.com/en-us/scenario-library/financial-services/process-a-claim/)
- [Nuance to Stop Supporting On-Premise Contact Centers: Now What? - CX Today](https://www.cxtoday.com/customer-analytics-intelligence/nuance-to-stop-supporting-on-premise-contact-centers-now-what/)
- [Google Cloud Next 2026 Wrap Up | Google Cloud Blog](https://cloud.google.com/blog/topics/google-cloud-next/google-cloud-next-2026-wrap-up)
- [The Ultimate Guide to FNOL Process Automation [2026 Updated] - Strada](https://www.getstrada.com/blog/fnol-automation)
- [AI Claims Processing: The Complete 2026 Guide for Insurance Leaders](https://vcasoftware.com/ai-for-claims-processing/)
- [AI in Insurance Customer Service in 2026: The State of Conversational Carriers | Perspective AI](https://getperspective.ai/blog/ai-insurance-customer-service-2026-state-of-conversational-carriers)
- [AI Insurance Fraud Detection in 2026: From Pattern Anomalies to Conversational Red Flags | Perspective AI](https://getperspective.ai/blog/ai-insurance-fraud-detection-in-2026-from-pattern-anomalies-to-conversational-red-flags)
- [How Empathy Shapes Insurance Customer Experience](https://convin.ai/en-us/blog/insurance-customer-experience-empathy-strategy)
- [AI adds an unexpected trait to loss-claim calls: Empathy | Digital Insurance](https://www.dig-in.com/news/ai-adds-an-unexpected-trait-to-loss-claim-calls-empathy)
- [The Best Omnichannel AI Customer Support Platform [2026] - Voiceflow](https://www.voiceflow.com/blog/omnichannel-ai-customer-support)
- [Using AI to predict and prevent weather catastrophe home insurance claims](https://www.santafenewmexican.com/life/features/using-ai-to-predict-and-prevent-weather-catastrophe-home-insurance-claims/article_900c461a-5d6c-5301-99f3-f70756c9dd2b.html)
- [Insurance AI Chatbots: 20 Use Cases & Examples - Master of Code](https://masterofcode.com/blog/insurance-chatbot)
- [7 tips to improve your sales demos - Modjo](https://www.modjo.ai/en/blog/7-keys-to-creating-the-wow-effect-in-a-demo)
- [Sales Demo Environments: Should You Build or Buy? - Reprise](https://www.reprise.com/resources/blog/sales-demo-environments-should-you-build-or-buy)
- [.planning/PROJECT.md — internal project context, meeting notes with Scott Hitchcock (2026-07-07)]

---
*Feature research for: Insurance FNOL claims CX-agent GTM demo (white-label, Google CX Agent Studio)*
*Researched: 2026-07-08*
