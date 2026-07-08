# Pitfalls Research

**Domain:** Live AI CX-agent sales demo (insurance FNOL/claims) built on Google CX Agent Studio / Gemini Enterprise for CX — deliverable is a use-case SPECIFICATION, not the demo itself
**Researched:** 2026-07-08
**Confidence:** MEDIUM-HIGH (platform capability claims verified against Google Cloud sources; demo-craft and insurance-domain pitfalls verified against multiple independent sources; competitive-positioning specifics are LOW-MEDIUM confidence and should be sanity-checked with Scott/Google before the spec asserts them as fact)

## Critical Pitfalls

### Pitfall 1: LLM Hallucination on Coverage/Dollar Figures, Live

**What goes wrong:**
Mid-demo, the agent states a deductible, coverage limit, or claim payout that doesn't match the mock policy data — or invents a plausible-sounding number when the field is ambiguous. In front of an insurance buyer, a wrong dollar figure isn't a cosmetic bug; it reads as "this AI can't be trusted with money," which is fatal for the exact audience this demo targets. This is the insurance-demo equivalent of the well-documented pattern where a confidently wrong AI answer becomes the headline of the demo instead of the product.

**Why it happens:**
LLMs optimize for fluent, plausible continuations, not verified accuracy — they don't distinguish "I retrieved this" from "I inferred this." Under-specified mock policy data (missing fields, inconsistent formats) increases the chance the model fills gaps with an invented number rather than escalating.

**How to avoid:**
Spec must require that every dollar figure and coverage term the agent says on stage is sourced from an explicit, structured mock-data lookup (tool/function call), never free-generated. Require the agent's instructions to include an explicit "cite from data, never estimate; if the field is missing, say so and escalate" rule. Spec should also mandate that every scripted demo path is dry-run tested against the exact seed data before delivery, with dollar figures cross-checked line by line.

**Warning signs:**
Agent instructions describe *behavior* ("discuss the customer's coverage") without specifying *data source*; mock data has optional/blank fields for numbers the script relies on; no explicit "grounding" or "never estimate financial figures" instruction in the spec.

**Phase to address:**
Mock Data & Scenario Specification section + Decision Logic & Business Rules section (define every number as coming from a named data field, not agent judgment).

---

### Pitfall 2: Non-Determinism Breaks the Scripted Wow Moment

**What goes wrong:**
Because CX Agent Studio agents are instruction+tool driven (flow-free, generative), the same input can yield a different path, wording, or even a different branch decision run to run. A wow moment scripted to land a specific way (e.g., "customer mentions boat → cross-sell fires") sometimes just doesn't trigger, or triggers with different wording that breaks the rehearsed narrative.

**Why it happens:**
Generative agents are probabilistic by design — that flexibility is the selling point, but it directly conflicts with the sales need for a repeatable, rehearsable script. Teams new to flow-free agent builders often carry over "click here → see X" mental models from flow-based tools and are surprised when instructions alone don't guarantee determinism.

**How to avoid:**
Spec should explicitly separate "must be deterministic" moments (decision branch trigger, cross-sell trigger, backend reveal) from "can vary" moments (small talk, phrasing). For deterministic moments, require the trigger condition to be an explicit, testable rule tied to structured mock data (not inferred from freeform conversation), and require the spec to define the *exact* input utterance(s) a presenter must say to reliably trigger each branch. Include a "repeatability acceptance criterion": each wow moment must fire correctly in N/N dry runs before sign-off.

**Warning signs:**
Spec describes triggers in vague natural language ("when the customer seems to mention a related asset") rather than exact phrases/fields; no repeatability test defined; implementation team reports "it worked yesterday, not today."

**Phase to address:**
Decision Logic & Business Rules section + Acceptance Criteria section (repeatability as a pass/fail test, not just a described behavior).

---

### Pitfall 3: Voice & Translation Latency Kills the Wow Moment

**What goes wrong:**
The mid-demo language switch and voice channel are two of the highest-stakes wow moments (per the meeting context) — and voice is the most latency-sensitive modality in the whole demo. Natural human turn-taking gap is ~200ms; production voice AI commonly runs 1,400–1,700ms; above ~1,500ms, users start talking over the agent, repeat themselves, or the exchange visibly feels "broken" — exactly the impression a sales demo cannot afford. Audio-to-audio translation adds an extra hop that can compound this.

**Why it happens:**
Latency accumulates at multiple points — turn/endpoint detection, ASR, LLM time-to-first-token, TTS — and network conditions at a customer's office/venue (not a controlled lab) are the least predictable variable of all.
*(Confidence: MEDIUM — general voice-AI latency figures are well-documented across vendors; CX Agent Studio's own measured latency was not independently benchmarked in this research and should be verified via a live test.)*

**How to avoid:**
Spec should require: (a) the language-switch and voice moments to be dry-run tested on the actual venue-class network conditions expected (e.g., conference wifi, not office fiber) before the live demo; (b) a defined maximum acceptable latency threshold as an acceptance criterion; (c) an explicit fallback script (see Pitfall 9) if latency degrades live. Spec should NOT assume audio-to-audio translation parity across all 40+ chat/voice languages — restrict the live language-switch moment to languages confirmed in the platform's audio-to-audio core set (~10 languages per current Google Cloud documentation) and treat any other language as a text/subtitle fallback, not a live-audio claim.

**Warning signs:**
Spec doesn't mention network conditions or a latency budget; language-switch scenario picks a language outside the confirmed audio-to-audio set; no test performed on non-lab network.

**Phase to address:**
Voice & Multilingual Use Case section + Demo Runbook / Fallback Plan section.

---

### Pitfall 4: Mock Data That Doesn't Reliably Trigger the Decision Branch or Upsell

**What goes wrong:**
The demo's two headline "show, don't tell" moments — the ~$1,000 auto-approve/escalate branch and the cross-sell/upsell (tree hits car → uninsured boat prompt) — both depend on mock data being internally consistent and unambiguous. If claim amounts, policy coverage, or "products not held" fields are underspecified, loosely worded, or left to the agent's interpretation, the branch may fire inconsistently, fail to fire, or fire on the wrong scenario during a live run.

**Why it happens:**
"Mock/simulated data" is often treated as a low-effort placeholder ("just make up some numbers") rather than a first-class spec artifact. Under an ASAP timeline, mock data is the piece most likely to be left implicit ("the implementation team will figure out reasonable values") — exactly the ambiguity a spec is supposed to eliminate.

**How to avoid:**
Spec must include a canonical, versioned seed dataset: named customers, policy IDs, coverage amounts, deductibles, claim amounts (one clearly under $1,000, one clearly over), and an explicit "product gap" field driving the upsell (e.g., customer has auto policy, does not have boat/watercraft policy). Each scenario should state, in the spec itself, the exact expected agent behavior for that data (which branch fires, what upsell copy appears) as a testable table — not prose description.

**Warning signs:**
Threshold and trigger values described only in prose ("a small claim") rather than as concrete data records; no single canonical dataset — different sections of the spec imply different numbers; no explicit "negative" data (claims that should NOT trigger escalation/upsell) to catch over-firing.

**Phase to address:**
Mock Data & Scenario Specification section (should be treated as a primary deliverable artifact, not an appendix).

---

### Pitfall 5: Photo/Damage-Analysis Unreliability On Stage

**What goes wrong:**
V1 already includes multimodal damage-photo analysis; V2 relies on it again as part of the intake happy path. Vision-based damage assessment is measurably sensitive to image quality, lighting, shadows, and framing — industry sources note false positives (shadows/dirt misread as damage) and severity misjudgments even in production systems with curated training data. A demo photo that looks fine to a presenter can still produce an odd or embarrassing severity read live.

**Why it happens:**
Presenters often use "whatever photo is handy" (a phone snap of a laptop, a stock image) instead of a pre-validated image, and multimodal models are more sensitive to real-world lighting/framing variance than text-only tasks — a variance that's invisible until it fails on stage.

**How to avoid:**
Spec must specify a small library (3–5) of pre-validated seed images per damage scenario, each tested to reliably produce the intended severity/coverage read, and must forbid ad hoc/live-captured photos in the standard demo path (live capture can be an optional "if we're feeling bold" stretch moment, clearly marked as such, with a scripted recovery line). Spec should define the expected damage-analysis output per seed image as an acceptance criterion.

**Warning signs:**
Spec says "customer uploads a photo of the damage" without naming/attaching the actual seed images; no documented expected-output per image; presenter improvising with a live photo during rehearsal.

**Phase to address:**
Mock Data & Scenario Specification section + Demo Runbook / Fallback Plan section.

---

### Pitfall 6: Non-Credible Auto-Approve/Escalate Threshold

**What goes wrong:**
The ~$1,000 threshold is a single flat number with no stated rationale. Insurance buyers — who live in actuarial, underwriting, and claims-reserving logic — will immediately probe "why $1,000? What about claim type, policy tier, fraud signals, prior claims history?" A threshold that looks arbitrary undercuts the entire "intelligent decisioning" pitch, turning the signature wow moment into a credibility gap.

**Why it happens:**
For a fast, white-label demo, a single round-number threshold is the path of least resistance — but "reusable across accounts" and "credible to an insurance underwriter" pull in opposite directions if the threshold isn't explicitly framed.

**How to avoid:**
Spec should frame the threshold explicitly as an *illustrative, configurable business rule* — not a hardcoded universal truth — and give the agent instructions/narrative language that says so out loud (e.g., "this account's policy routes claims under $1,000 with no injury or fraud flags to auto-approval; thresholds and rules are configurable per line of business"). This turns a potential objection into a proof point (rules-as-configuration is a selling point for a platform pitch). Spec should also define 1–2 secondary factors beyond dollar amount (e.g., injury flag, prior-claims flag) so the branch isn't a single naive number.

**Warning signs:**
Spec states the threshold with no accompanying "why"/configurability language; agent has no way to explain its own decision if asked "why did you approve this."

**Phase to address:**
Decision Logic & Business Rules section.

---

### Pitfall 7: Regulatory/Compliance Optics of AI "Deciding" Insurance Claims

**What goes wrong:**
Even in a mock demo, showing an AI autonomously "approving" a claim touches a regulated nerve: state insurance departments and NAIC guidance scrutinize automated adverse/approval decisions, adjuster-licensing requirements, and auditability of claims decisioning. If an insurance buyer in the room asks "is this actually allowed / how does this pass a regulatory audit," an unprepared presenter looks naive about the buyer's real-world constraints — undermining trust even though nothing is actually being decided for real.

**Why it happens:**
The demo's design goal (visible, snappy autonomous-vs-human branch) is in tension with how real insurers talk about automation: framed as adjuster augmentation with audit trails and human oversight, not "the AI decides." Teams building sales demos often optimize for visual drama over the compliance narrative their own buyers expect to hear.

**How to avoid:**
Spec should require the demo narrative to explicitly frame auto-approval as "system pre-approves within a human-authorized policy/rule set, with full audit trail" — not "AI decides." Every auto-approve moment should be paired with a visible artifact (a logged rule reference, a reviewable transcript) that suggests auditability, even if simulated. This turns compliance into a wow moment ("look, it explains and logs its reasoning") instead of a landmine.

**Warning signs:**
Demo narrative/marketing language uses "the AI approves the claim" without any "governed by configurable rules" framing; no simulated audit-trail artifact shown; spec has no answer prepared for "what happens if this is wrong / who's liable."

**Phase to address:**
Decision Logic & Business Rules section + Backend Claims-Processing Reveal section (the audit-trail/summary artifact should double as the compliance answer).

---

### Pitfall 8: PII / Real-Looking Sensitive Data in Mock Transcripts and Artifacts

**What goes wrong:**
Mock data that's "too real" — plausible SSNs, real address formats, a presenter's own name/phone number typed in live, or a screenshot accidentally containing a real customer's data from another project — creates privacy/legal exposure even though the demo itself is fake. Conversation transcripts and the "backend reveal" (specialist summary, auto-drafted email) are exactly the artifacts most likely to leak something real if presenters improvise.

**Why it happens:**
Mock data is treated as low-stakes because it's fake, so less care is taken sanitizing it than real production data would get — but the transcript/email artifacts are visually persuasive precisely because they look real, which is the same property that makes accidental real-PII leakage easy to miss.

**How to avoid:**
Spec must mandate that all seed data (names, addresses, policy numbers, phone numbers, emails) is synthetic and drawn from a clearly fictitious namespace (e.g., obviously fake domains, no valid-format SSNs), and must prohibit live improvisation of personal details during a demo (no typing a real prospect's or presenter's own info to "personalize" on the fly). Spec should also state that the backend reveal artifacts (specialist summary, auto-drafted customer email) are pre-approved templates populated from mock fields, not agent-generated freeform text pulled from whatever was typed live.

**Warning signs:**
Spec doesn't define a synthetic-data policy; scenario examples use realistic-looking SSN/address formats; no rule against presenters typing real personal information during rehearsal or live delivery.

**Phase to address:**
Mock Data & Scenario Specification section (explicit synthetic-data standard) + Backend Claims-Processing Reveal section.

---

### Pitfall 9: Over-Promising Real Gemini Enterprise / Platform Integration

**What goes wrong:**
The backend reveal (transcript → Gemini Enterprise ingestion → specialist summary → auto-drafted email) is described in the meeting notes as a *conceptual* pipeline. If the spec asserts specific platform mechanics (named connectors, specific ingestion APIs, specific model behaviors) that aren't actually current/GA in CX Agent Studio, the implementation team either builds something that doesn't work, or the sales team later gets caught overstating capability to a technical buyer — a much worse outcome than a slightly smaller but accurate demo.

**Why it happens:**
CX Agent Studio / Gemini Enterprise for CX launched ~Jan 2026 — after the assistant's/researcher's training cutoff in many cases — so claims about specific mechanics are easy to get wrong by extrapolating from older Dialogflow/CCAI mental models or by training-data assumptions rather than current docs.

**How to avoid:**
Spec must state platform capability claims (voice languages, audio-to-audio language count, connector/MCP support, backend automation mechanics) with a source and a "verify against current docs at build time" flag, and should default to describing the backend reveal as an *achievable simulated pipeline* (a defined tool/sub-agent chain the implementation team builds explicitly) rather than asserting it as an out-of-the-box platform feature. Any claim used in customer-facing sales language should be independently verified by the implementation team against current Google Cloud documentation before it's said out loud to a prospect.

**Warning signs:**
Spec cites a specific "Gemini Enterprise ingests the transcript automatically" mechanic without naming the actual tool/API/sub-agent that performs it; capability claims have no source or verification note; spec language matches marketing copy more than technical documentation.

**Phase to address:**
Capability Mapping (CX Agent Studio features) section + Backend Claims-Processing Reveal section.

---

### Pitfall 10: Language-Switch Failure On Stage (Accent, Audio, Mic Dependency)

**What goes wrong:**
The mid-demo language switch is one of the two headline wow moments Scott specifically asked for. Live audio is uniquely vulnerable to venue conditions: a laptop mic vs. a conference-room mic, background noise, a presenter's own accent when demonstrating the "customer" side, and codec/network quality all affect ASR accuracy and downstream translation quality in ways a scripted chat demo never faces.

**Why it happens:**
Voice demos are usually rehearsed in a quiet office with a good headset, then delivered in an unpredictable customer conference room or over a video call with compressed/lossy audio — a materially different acoustic environment that wasn't tested.

**How to avoid:**
Spec should require the language-switch scenario to specify: the exact phrase(s) that trigger the switch (to avoid ASR-ambiguous phrasing), the confirmed language pair(s) within the audio-to-audio core set, and a rehearsal requirement across at least two audio setups (laptop mic and a typical conference-room/video-call setup). Spec should also define a graceful-degradation fallback (e.g., switch to captioned bilingual text) if live audio quality is visibly poor, per the runbook (Pitfall 17).

**Warning signs:**
No specified trigger phrase (relies on the agent "understanding" a switch is wanted); no confirmed language pair tied to the actual audio-to-audio supported set; only ever rehearsed with a headset in a quiet room.

**Phase to address:**
Voice & Multilingual Use Case section + Demo Runbook / Fallback Plan section.

---

### Pitfall 11: Demo That Looks Like a "Deck," Not a Workflow

**What goes wrong:**
Scott's explicit principle is "show, don't tell." A demo that's actually a rigid, linear script with no real interactivity (presenter clicks through pre-set inputs, agent gives memorized-sounding responses) reads as a glorified slide deck with a chat window — the exact thing this project exists to avoid. Buyers who are Microsoft/Azure-centric and already skeptical will detect canned theater immediately.

**Why it happens:**
Under an ASAP timeline, the fastest path to a reliable demo is over-scripting every line — but over-scripting is in direct tension with looking like a real, flexible, generative agent. Teams optimize for reliability and accidentally kill authenticity.

**How to avoid:**
Spec should distinguish "guaranteed" moments (decision branch, upsell, language switch — must be deterministic per Pitfall 2) from "flex" moments where the presenter is instructed to genuinely improvise small talk or an off-script question, proving the agent isn't just a slideshow. Spec should include at least one scripted "curveball" (e.g., an out-of-scope question, as V1 already demonstrates) specifically to prove generative flexibility live, with a defined expected graceful-handling behavior.

**Warning signs:**
Every line of the demo script is fixed dialogue with no room for live variation; no planned "prove it's not scripted" moment; internal rehearsals never deviate from the script.

**Phase to address:**
Demo Narrative & Storyline section.

---

### Pitfall 12: No Explicit Differentiation vs. Microsoft Copilot + Nuance

**What goes wrong:**
Per the meeting context, insurance buyers are often Microsoft/Azure-centric with existing Nuance Contact Center AI / Copilot Studio investment and will resist disruption. If the spec doesn't equip the demo (or the sales narrative around it) with a clear, specific "why this over what you already have" answer, the demo wows in the room but doesn't survive the buyer's next internal conversation with their existing Microsoft rep.
*(Confidence: LOW-MEDIUM — Nuance/Copilot's own insurance-specific positioning is thin in current public sources; the general Gemini-vs-Copilot comparison is well covered, but insurance-vertical specifics should be validated with Scott/Google field team rather than asserted from this research alone.)*

**Why it happens:**
Competitive differentiation is usually treated as a sales-enablement artifact separate from the technical demo spec — but for a GTM asset explicitly built to unseat an incumbent (Nuance is already embedded in many insurance contact centers), the differentiation has to be *demonstrated*, not just argued in a follow-up slide.

**How to avoid:**
Spec should identify 1-2 concrete capability contrasts that can be *shown* live rather than claimed (e.g., flow-free natural-language agent authoring vs. flow-based bot design; native multimodal voice+chat+image in one agent vs. stitched-together point solutions; audio-to-audio translation latency vs. cascaded STT→MT→TTS). These should map to wow moments already in scope (voice/language switch, photo analysis, backend reveal) rather than adding new scope. Spec should flag this as a GTM/positioning input for Srini/Stephanie/Hallie, not something the implementation team invents on their own.

**Warning signs:**
Spec has no competitive-positioning notes at all; the differentiation argument only exists in verbal conversation with Scott, not written down; wow moments aren't mapped to a "why Google, not Microsoft" story.

**Phase to address:**
Demo Narrative & Storyline section (a short competitive-framing subsection), informed by GTM stakeholders.

---

### Pitfall 13: Too-Custom to Actually Be White-Label / Reusable

**What goes wrong:**
V1 originated from a highly-customized Furniture Care Protection (FCP) implementation. If V2's spec carries over FCP-specific structure, terminology, or data shapes (product categories, claim types, coverage language specific to furniture protection plans) without genericizing, the "reusable across ~18+ accounts and multiple sectors" goal quietly fails — the demo works great for the account it was unconsciously modeled on and feels awkward or irrelevant everywhere else.

**Why it happens:**
Reusing a working example is the fastest way to move under an ASAP timeline, and domain-specific details (a "recommended repair partner" for a laptop, boat/auto bundling) are exactly the kind of concrete, vivid detail that makes a demo compelling — but vivid and generic are in tension.

**How to avoid:**
Spec should explicitly state which elements are structural/reusable (the decision-branch pattern, the escalation pattern, the cross-sell pattern, the backend-reveal pattern) versus illustrative/swappable (the specific line of business — auto/home/renters — and specific product examples like "laptop repair" or "boat bundle"). Require that at least one alternate vertical/product example be spec'd as a drop-in substitute (proving genuine reusability) even if only one is built for the first demo.

**Warning signs:**
Spec hardcodes a single vertical's terminology throughout with no abstraction layer; no note distinguishing "pattern" from "example"; the same mock data would look strange if a presenter tried to relabel it for a different account.

**Phase to address:**
Demo Narrative & Storyline section + Mock Data & Scenario Specification section (data model should be parameterized, not hardcoded).

---

### Pitfall 14: Scope Creep — Agent Assist (or Other Deferred Scope) Creeping Back In

**What goes wrong:**
PROJECT.md explicitly defers Agent Assist (real-time summarization/coaching for human agents) to a later version. Under "show, don't tell" pressure and an ASAP timeline, it's easy for the implementation team (or a well-meaning stakeholder) to quietly fold Agent Assist-like features back in because they're adjacent and "would look cool," expanding scope, timeline, and complexity beyond what was actually agreed.

**Why it happens:**
Scope boundaries documented in a project charter are easy to forget once the spec-writing gets into the weeds of the HITL/escalation path, since Agent Assist and HITL escalation touch the same conceptual territory (a human handling a case with AI support) — the boundary is conceptually blurry even though it's explicitly decided.

**How to avoid:**
Spec should restate the Out of Scope boundary at the point in the document where it's most tempting to blur (the HITL/escalation and backend-reveal sections) with a one-line reminder of why (customer-dependent, contact-center-only relevant, deferred). Any place where the spec discusses "what the human specialist sees," it should be scoped to the specific backend-reveal artifact (transcript → summary → drafted email) and explicitly not extended into live coaching/real-time suggestions.

**Warning signs:**
Backend-reveal or escalation sections start describing real-time suggestions to a live human agent during the call (that's Agent Assist); scope discussions in review meetings start including "while we're in there, what if we also..."

**Phase to address:**
Out of Scope reaffirmation should appear at the start of the Backend Claims-Processing Reveal section and the HITL/Escalation section specifically, not just once at the top of the document.

---

### Pitfall 15: Spec Ambiguity Forces the Implementation Team to Guess

**What goes wrong:**
Because this project's entire value proposition is "the implementation team can build directly from the spec," any ambiguity is amplified: instead of a developer asking a clarifying question in a shared Slack channel, the implementation team may guess, build the wrong thing, and the gap isn't discovered until a rehearsal — or worse, in front of a customer. Vague behavioral descriptions ("the agent should handle this gracefully," "shows empathy") without concrete example dialogue are the most common source.

**Why it happens:**
Natural-language agent instructions (CX Agent Studio's core authoring model) make it tempting to write the spec itself in the same loose, descriptive prose style as agent instructions — but a spec needs to be more precise than an agent instruction, because a human implementer, not a forgiving LLM, has to interpret it.

**How to avoid:**
Spec should follow structured-PRD discipline: every use case gets an explicit "example dialogue" or scripted exchange, not just a behavioral description; every ambiguous judgment call (what counts as "small talk" vs. "off-script," what exact wording triggers a branch) gets resolved with a concrete example rather than left abstract. Require each spec section to end with an explicit "Open Questions / Assumptions" subsection so unresolved ambiguity is visible and flagged for review rather than silently guessed at during implementation.

**Warning signs:**
Sections describe *outcomes* without *mechanisms* ("the system recognizes the cross-sell opportunity" without saying which field/trigger); no example transcripts included; reviewers can't agree on what a section means.

**Phase to address:**
Applies across all spec sections; enforce via a spec-review checklist gate before handoff (Acceptance Criteria section should include a "spec passed ambiguity review" line item).

---

### Pitfall 16: Missing Acceptance Criteria ("What Good Looks Like")

**What goes wrong:**
PROJECT.md already flags this as an active requirement, which is a good sign — but it's also one of the easiest things to under-deliver on: teams write a rich narrative description of what the demo *should feel like* but never define the testable pass/fail conditions the implementation team can self-check against before calling a use case "done." Without this, "done" becomes a subjective negotiation late in the timeline.

**Why it happens:**
Acceptance criteria feel like the "boring" last step after the exciting narrative/wow-moment design work, and under an ASAP timeline they're the section most likely to get compressed or skipped.

**How to avoid:**
Every use case/wow moment in the spec must ship with explicit, observable, testable acceptance criteria (e.g., "given claim amount $650 and no injury flag, agent auto-approves within one turn and states the policy rule it applied," not "agent handles small claims well"). Acceptance criteria should be written before or alongside the narrative, not after, and should be re-checked against the mock data (Pitfall 4) to confirm they're actually achievable with the specified seed data.

**Warning signs:**
Use cases have narrative/prose descriptions but no criteria section; criteria that exist are subjective ("feels natural," "wows the customer") rather than observable; no criteria reference specific mock data records.

**Phase to address:**
Acceptance Criteria section (dedicated, not folded into narrative) — should be the final gate before spec handoff.

---

### Pitfall 17: No Demo Runbook / No Fallback for a Live Wow Moment Failing

**What goes wrong:**
Every pitfall above (hallucination, non-determinism, latency, photo misread, language-switch failure) can still happen even with a well-built demo — live software fails live, especially generative AI over voice on unpredictable networks. Without a rehearsed recovery plan, a single glitch mid-demo derails the entire pitch and the presenter is left improvising in front of the exact skeptical, incumbent-invested buyer this demo needs to win over.

**Why it happens:**
Runbooks and fallback plans are typically treated as "operational" concerns outside the scope of a "specification," so they get left to whoever ends up giving the demo — but for a GTM asset given to ~18+ accounts by people who may not have built it themselves, tribal knowledge doesn't scale.

**How to avoid:**
Spec must include a dedicated Demo Runbook / Fallback Plan section: pre-demo environment/data reset steps, a recording or screenshot backup for each wow moment (per the recovery strategies below), scripted recovery lines for each known failure mode ("looks like the system's thinking — let me show you that exact flow from a saved run"), and a pre-flight checklist (network check, mic check, seed-data reset) to run before every live delivery. This should be written for a presenter who did *not* build the demo.

**Warning signs:**
No runbook section exists at all; recovery guidance lives only in the builder's head; no backup recordings/screenshots specified as deliverables alongside the live build; spec assumes the demo environment is always in a known-good state with no reset procedure defined.

**Phase to address:**
Demo Runbook / Fallback Plan section — should be one of the last sections written but is a hard requirement for GTM readiness, not optional polish.

---

## Technical Debt Patterns

Shortcuts that seem reasonable under an ASAP timeline but create long-term problems for a reusable, multi-account GTM asset.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|-----------------|------------------|
| Hardcode the $1,000 threshold with no stated rationale in agent instructions | Faster spec, one less decision | Looks arbitrary under underwriter-level scrutiny; undermines "intelligent decisioning" pitch | Only if spec explicitly frames it in-narrative as "illustrative/configurable," never as a silent hardcode |
| Reuse V1's FCP-derived data shapes/terminology without genericizing | Fast to build on known-working base | Breaks white-label reusability claim across accounts/sectors | Never for the final deliverable; acceptable only as an internal first draft |
| Spec only the happy path, defer edge cases ("we'll handle that live") | Smaller, faster spec | No fallback when an off-script question or edge case comes up live; presenter improvises unprepared | Acceptable only for an early internal review draft, not the handoff spec |
| Treat voice as "chat with a microphone" (no separate latency/turn-taking spec) | Less spec-writing effort | Misses barge-in, latency-budget, and audio-quality requirements unique to voice — the exact area most likely to fail live | Never — voice is an explicit headline wow moment |
| Skip writing acceptance criteria until "later" | Faster narrative-writing phase | "Done" becomes a late, subjective negotiation; implementation team can't self-verify | Never for the final spec (PROJECT.md already flags this as required) |

## Integration Gotchas

Common mistakes specific to CX Agent Studio / Gemini Enterprise for CX capability claims.

| Integration/Capability | Common Mistake | Correct Approach |
|-------------------------|-----------------|-------------------|
| Backend policy/claims system "connector" | Assuming a named out-of-the-box connector exists for a generic insurance policy system without checking current docs | Spec a concrete mock tool/function contract (input/output schema) for the implementation team to build against, rather than naming an unverified connector |
| Audio-to-audio translation | Assuming all 40+ chat/voice languages get live audio-to-audio parity | Restrict the live language-switch wow moment to the confirmed audio-to-audio core language set (~10 languages per current Google Cloud docs); use text/captioned fallback for anything outside that set |
| "Gemini Enterprise ingestion" of the transcript for the backend reveal | Describing this as an automatic platform feature without naming the actual tool/sub-agent/prompt chain that performs it | Spec the exact tool/sub-agent responsible for transcript → summary → drafted email, so the implementation team builds a working chain rather than guessing at an assumed built-in feature |
| Telephony/voice channel | Bringing in an external carrier/telephony integration when Studio's own voice channel is sufficient (per Akash's meeting note) | Default to Google's built-in Studio telephony/voice channel for tight integration; only introduce external carriers if a specific customer requirement demands it (and flag that as added complexity) |

## Performance Traps

For a live sales demo, "scale" means presenter count, venue diversity, and scenario count growing — not user load.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|-----------------|
| Demo only ever tested on office wifi/headset | Latency and audio quality look fine in rehearsal, degrade unpredictably on-site | Spec requires testing on venue-class network (conference wifi, video-call compression) as part of acceptance criteria | First real customer site with uncontrolled network |
| Adding more scenarios/wow moments without re-testing interactions between them | New upsell or escalation trigger accidentally fires during a different scripted path | Spec includes a scenario regression matrix — every new scenario is checked against existing triggers before sign-off | Once the spec grows past ~3-4 interacting scenarios (likely, given the V2 wish list) |
| Demo built/tuned by one person, then handed to other AEs across 18+ accounts | Works perfectly for the builder, breaks (wrong phrasing, missed triggers) when a different presenter drives it off-script | Spec must be presenter-agnostic: exact trigger phrases, explicit runbook, no reliance on the builder's tribal knowledge | As soon as a second presenter (of the 18+ target accounts) tries to run it solo |

## Security Mistakes

Domain-specific issues beyond general demo hygiene — this is a claims/insurance context even though the data is mock.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Using realistic-format PII in mock data/transcripts (valid-looking SSNs, real address formats, real phone numbers) | Privacy/legal exposure if screenshots, recordings, or transcripts leak; erodes "this is clearly mock" framing if it looks too real | Spec mandates a synthetic-data standard: obviously fictitious names/domains, invalid-format identifiers, clearly fake addresses |
| Presenter improvises live with their own or a prospect's real personal info to "personalize" the demo | Real PII entering a recorded/logged AI conversation and possibly downstream summary/email artifacts | Spec explicitly prohibits live improvisation of personal details; all inputs must come from the canonical seed dataset |
| Backend-reveal artifacts (specialist summary, drafted customer email) generated freeform from whatever was said live | Unpredictable, potentially sensitive or off-brand content shown to a prospect | Spec artifacts as templates populated from defined mock fields, reviewed and pre-approved, not open-ended generation |
| System prompts/internal agent instructions inadvertently shown during the "backend reveal" | Exposes internal engineering detail/IP to prospects, and can reveal placeholder/rough language not meant for external eyes | Spec defines exactly what is and isn't shown in the reveal (customer-facing artifacts only, not raw instructions) |

## UX Pitfalls

| Pitfall | User (Audience) Impact | Better Approach |
|---------|--------------------------|-------------------|
| Autonomous-vs-human decision branch happens invisibly/instantly with no narration | Audience misses the actual value moment — the AI "just answered" with no visible judgment call | Spec requires the agent (or presenter narration) to explicitly state which path was taken and why, making the decision visible |
| Dialogue is stiff/over-scripted, sounding memorized | Breaks the "real interactive workflow, not a deck" illusion the whole project is built to deliver | Spec allows and even scripts a deliberate off-script/flex moment to demonstrate genuine generative flexibility |
| Cross-sell/upsell moment feels like a bolted-on sales pitch rather than a natural, context-aware observation | Feels manipulative/salesy rather than "wow, it noticed that" | Spec writes the upsell trigger and copy to sound like a helpful observation tied directly to what the customer just said, not a generic pitch |
| Backend reveal artifact (drafted email) is generic boilerplate | Undercuts the "personalization" wow factor of showing AI-generated, context-aware output | Spec requires the drafted email to visibly reference specific details from the actual conversation (item damaged, claim outcome, recommended partner) |

## "Looks Done But Isn't" Checklist

- [ ] **Decision branch:** Often missing the agent's visible reasoning/narration — verify the spec requires the agent (or demo flow) to state which rule/threshold it applied, not just silently produce an outcome.
- [ ] **Language switch:** Often "works" only via captioned text or a slow cascaded pipeline mistaken for true audio-to-audio — verify the confirmed language pair is actually in the platform's audio-to-audio core set, tested end-to-end with audio, not just described.
- [ ] **Mock data:** Often missing negative/edge-case records (a claim that should NOT escalate, a customer who should NOT get the upsell) — verify the seed dataset includes at least one "should not trigger" example per branch to catch over-firing.
- [ ] **Backend reveal artifact:** Often described conceptually ("summary is generated") without an actual sample artifact attached — verify the spec includes a literal example of the specialist summary and the drafted customer email, word for word.
- [ ] **Acceptance criteria:** Often written as feelings ("wows the customer") rather than checks — verify each wow moment has an observable, testable pass condition tied to specific mock data.
- [ ] **Runbook:** Often assumed to be "obvious" and left unwritten — verify a literal fallback script/recording exists for each of the 4-5 highest-risk live moments (hallucination, branch misfire, photo misread, language-switch failure, general "it broke").
- [ ] **White-label check:** Often only sanity-checked against the original vertical it was modeled on — verify at least one alternate product/vertical substitution is spec'd to prove the pattern actually generalizes.

## Recovery Strategies

When pitfalls occur despite prevention, how the presenter/spec should recover live.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|----------------|------------------|
| Hallucinated dollar figure/coverage term said live | MEDIUM | Presenter redirects to the "let me pull up the actual policy record" moment (shows the ground-truth mock data), reframing the glitch as a transparency feature; spec should pre-write this recovery line |
| Photo/damage analysis returns an odd or wrong result | LOW | Presenter switches to a pre-validated seed image ("let's use one of our sample claims"); spec should ship 3-5 validated images specifically to swap in |
| Language-switch audio garbles or fails to trigger | MEDIUM | Presenter falls back to captioned/text bilingual mode or narrates in English while showing the text translation; spec should define this degradation path explicitly, not leave it to improvisation |
| Decision branch produces the wrong outcome (e.g., auto-approves a claim that should escalate) | HIGH | This is the most credibility-damaging failure for an insurance-regulated audience; spec should require a rehearsed recovery line acknowledging the miss transparently and pivoting to a recorded backup of the correct run, rather than trying to argue the wrong output was actually right |
| General live failure/crash | MEDIUM | Pre-recorded full-scenario video/screen capture as final fallback, kept current with each spec revision; spec should mandate this recording as a required deliverable alongside the live build, not an afterthought |

## Pitfall-to-Phase Mapping

How the roadmap/spec should sequence work to address these pitfalls. (Spec section names are suggested groupings; actual roadmap phase numbers will be finalized during roadmap creation.)

| Pitfall | Spec Section (Prevention Phase) | Verification |
|---------|-----------------------------------|----------------|
| 1. Hallucination on dollar figures | Mock Data & Scenario Spec; Decision Logic & Business Rules | Every stated number traces to a named data field in a dry run |
| 2. Non-determinism breaks wow moments | Decision Logic & Business Rules; Acceptance Criteria | N/N repeatability test defined and passed |
| 3. Voice/translation latency | Voice & Multilingual Use Case; Demo Runbook | Latency budget tested on venue-class network, not just lab wifi |
| 4. Mock data doesn't trigger branch/upsell | Mock Data & Scenario Specification | Canonical dataset table with expected outcome per record |
| 5. Photo/damage-analysis unreliability | Mock Data & Scenario Specification; Demo Runbook | Pre-validated seed image library with documented expected outputs |
| 6. Non-credible threshold | Decision Logic & Business Rules | Threshold framed as configurable rule with stated rationale in narrative |
| 7. Regulatory optics of auto-deciding claims | Decision Logic & Business Rules; Backend Claims-Processing Reveal | Narrative frames decisions as rule-governed + auditable, not autonomous judgment |
| 8. PII in mock data/artifacts | Mock Data & Scenario Specification; Backend Claims-Processing Reveal | Synthetic-data standard applied; no real-format identifiers anywhere |
| 9. Over-promising platform integration | Capability Mapping; Backend Claims-Processing Reveal | Every capability claim sourced and marked for pre-delivery verification against current docs |
| 10. Language-switch failure on stage | Voice & Multilingual Use Case; Demo Runbook | Tested across 2+ audio setups; confirmed language pair in audio-to-audio core set |
| 11. Demo looks like a deck | Demo Narrative & Storyline | At least one genuine flex/curveball moment included and rehearsed |
| 12. No differentiation vs. Copilot+Nuance | Demo Narrative & Storyline (GTM framing subsection) | 1-2 concrete, demonstrable capability contrasts documented, reviewed with GTM stakeholders |
| 13. Too-custom to be white-label | Demo Narrative & Storyline; Mock Data & Scenario Specification | Alternate vertical/product substitution spec'd as proof of reusability |
| 14. Agent Assist scope creep | Backend Claims-Processing Reveal; HITL/Escalation section | Out-of-scope reminder present at both sections; reviewed against PROJECT.md boundary |
| 15. Spec ambiguity | All sections | Spec-review checklist gate; each section has example dialogue, not just description |
| 16. Missing acceptance criteria | Acceptance Criteria | Dedicated section exists with testable, observable criteria per use case |
| 17. No demo runbook/fallback | Demo Runbook / Fallback Plan | Dedicated section exists with pre-flight checklist, recovery scripts, and backup recordings |

## Sources

- [Customer Experience Agent Studio | Google Cloud](https://cloud.google.com/gemini-enterprise-cx/cx-agent-studio) — platform capability claims (voice, 40+ languages, audio-to-audio in ~10 core languages)
- [Gemini Enterprise for Customer Experience | Google Cloud](https://cloud.google.com/gemini-enterprise-cx)
- [Google Cloud Brings Shopping and Customer Service Together with Gemini Enterprise for Customer Experience (press release, Jan 11 2026)](https://www.googlecloudpresscorner.com/2026-01-11-Google-Cloud-Brings-Shopping-and-Customer-Service-Together-with-Gemini-Enterprise-for-Customer-Experience)
- [Understanding the Risks AI Hallucinations Create for Businesses](https://natlawreview.com/article/ai-hallucinations-are-creating-real-world-risks-businesses) — Google Bard live-demo hallucination / market-cap impact precedent
- [AI hallucination examples: 12 real cases and what they teach](https://www.open.cx/blog/ai-hallucination-examples)
- [AI Hallucinations in Customer Service: Why Quality Control Architecture Matters](https://yuma.ai/blogs/ai-hallucinations-in-customer-service-why-quality-control-architecture-matters)
- [How Insurance Claims Automation Transforms Auto FNOL Process](https://insillion.com/blog/ai-in-auto-insurance-claims-fnol) — NAIC/HIPAA/CCPA audit-trail and adjudication-consistency pitfalls
- [How AI Automates Insurance Claims Processing: FNOL to Settlement 2026](https://fieldnotesai.com/blog/ai-insurance-claims-processing-fnol-settlement-2026)
- [Computer Vision in Insurance: Vehicle Damage Assessment Case](https://medium.com/intelliarts-ai/computer-vision-in-insurance-vehicle-damage-assessment-case-1ac09e80140f) — image-quality/false-positive limitations in damage AI
- [SOSA — How computer vision is solving the claims bottleneck](https://www.sosa.co/blog/why-damage-assessment-is-the-choke-point-in-claims)
- [A Multimodal RAG Framework for Housing Damage Assessment](https://arxiv.org/pdf/2509.09721)
- [Engineering for Real-Time Voice Agent Latency — Cresta](https://cresta.com/blog/engineering-for-real-time-voice-agent-latency) — turn-taking latency benchmarks (200ms natural gap vs. 1,400-1,700ms production median)
- [Voice AI Latency: What's Fast, What's Slow, and How to Fix It — Hamming AI](https://hamming.ai/resources/voice-ai-latency-whats-fast-whats-slow-how-to-fix-it)
- [AI Voice Agent Challenges: 8 Failures & How to Fix Them](https://appinventiv.com/blog/why-ai-voice-agents-fail/)
- [Demo Fails: How to Turn Challenges Into Triumphs — Reprise](https://www.reprise.com/resources/blog/demo-fails-how-to-turn-challenges-into-triumphs) — live-demo recovery and fallback practices
- [Software Demo Best Practices: What Top Teams Do Differently — Reprise](https://www.reprise.com/resources/blog/software-demo-best-practices)
- [How to Write a Good Spec for AI Agents — O'Reilly Radar](https://www.oreilly.com/radar/how-to-write-a-good-spec-for-ai-agents/) — structured-PRD discipline, ambiguity handling
- [How to write a good spec for AI agents — Addy Osmani](https://addyo.substack.com/p/how-to-write-a-good-spec-for-ai-agents)
- [PII Redaction for Voice Agent Transcripts: Compliance & Architecture Guide — Hamming AI](https://hamming.ai/resources/pii-redaction-voice-agents-compliance-architecture-guide)
- [Streamline Regulatory Compliance with Insurance Redaction Software](https://redactor.ai/blog/streamline-regulatory-compliance-with-insurance-redaction-software)
- [Nuance Contact Center AI (Microsoft AppSource)](https://appsource.microsoft.com/en-us/product/web-apps/nuance_gskaff.nuance-ccai?tab=overview) — LOW-MEDIUM confidence, general positioning only; insurance-specific competitive claims not independently verified
- [Microsoft Copilot Studio Launches Realtime Voice Agents for Dynamics 365 Contact Center — CX Today](https://www.cxtoday.com/contact-center/copilot-studio-real-time-voice-agents-dynamics-365-contact-center/)
- `.planning/PROJECT.md` — project scope, constraints, and meeting-context source for demo requirements

---
*Pitfalls research for: Insurance FNOL/claims CX-agent WOW demo specification (Google CX Agent Studio / Gemini Enterprise for CX)*
*Researched: 2026-07-08*
