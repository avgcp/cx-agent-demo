# Architecture Research

**Domain:** Insurance FNOL claims demo on Google CX Agent Studio (Gemini Enterprise for Customer Experience) — a flow-free, instruction+tool multi-agent platform. Two architectures are in scope: (1) the demo's runtime composition on the platform, (2) the structure of the specification document the implementation team builds from.
**Researched:** 2026-07-08
**Confidence:** MEDIUM — CX Agent Studio / Gemini Enterprise for CX shipped ~Jan 2026, after training-data cutoff. Findings below are triangulated from Google's official docs (`docs.cloud.google.com/customer-engagement-ai/...`, `docs.cloud.google.com/gemini-enterprise-agent-platform/...`), a Google Developer forum architecture-advice thread, and a community "deep dive" blog series (Yash Kavaiya, Google Cloud Community, May 2026) — not independently re-verified line-by-line against the live console. Treat component names/settings as directionally correct; **the implementation team should confirm exact UI labels and tool-config fields against the live Agent Studio console before/while building**, since docs for a 6-month-old product can shift.

## Standard Architecture

### System Overview (Demo Runtime on CX Agent Studio)

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CHANNELS (Gemini Enterprise for CX)                │
│  ┌────────────────┐              ┌───────────────────────────────┐   │
│  │  Chat widget    │              │  Voice (Google Telephony      │   │
│  │  (web demo page)│              │  channel, native audio model, │   │
│  │  40+ languages  │              │  audio-to-audio translate,    │   │
│  │                 │              │  ~10 core languages)          │   │
│  └────────┬────────┘              └────────────────┬──────────────┘   │
└───────────┼────────────────────────────────────────┼──────────────────┘
            └───────────────────┬────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                 ROOT / ORCHESTRATOR AGENT ("Claims Concierge")        │
│  - Single entry point for both channels                               │
│  - Greeting, goal understanding, persona/tone (global instructions)   │
│  - Delegates via {@AGENT: Sub Agent Name} based on conversation state │
│  - Owns top-level session variables (claimant, policy, claim state)   │
└───────┬───────────┬────────────┬─────────────┬────────────┬──────────┘
        ▼           ▼            ▼             ▼            ▼
   ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐
   │ Intake  │ │ Damage   │ │Decisioning│ │ Escalation│ │ Upsell /     │
   │ Sub-    │ │Assessment│ │ Sub-Agent │ │/ Human    │ │ Cross-Sell   │
   │ Agent   │ │(capability│ │ (threshold│ │ Handoff   │ │ Sub-Agent    │
   │         │ │ of Intake │ │ + rules)  │ │ Sub-Agent │ │              │
   │         │ │ or own    │ │           │ │           │ │              │
   │         │ │ sub-agent)│ │           │ │           │ │              │
   └────┬────┘ └────┬─────┘ └─────┬─────┘ └─────┬─────┘ └──────┬───────┘
        │           │             │             │              │
        └─────┬─────┴──────┬──────┘             │              │
              ▼             ▼                     ▼              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │              BACKEND CLAIMS-PROCESSING SUB-AGENT                 │
   │  (the "reveal" — shared by both autonomous & HITL branches)      │
   │  transcript/state → specialist summary → draft customer email    │
   │  OR → human-assessor handoff packet                              │
   └────────────────────────────┬───────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        TOOLS (agent "hands")                          │
│  Policy Lookup (Python)   Coverage/Asset Lookup (Python)               │
│  Claim-Number Generator (Python)   Draft-Email Composer (Python)       │
│  Repair-Partner Search (Data Store/RAG)   end_session (system tool)    │
└──────────────────────────────────────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          DATA (mock/seeded)                           │
│  ┌───────────────┐  ┌────────────────┐  ┌─────────────────────────┐  │
│  │ Policy &      │  │ Knowledge Base │  │ Mock Claims Ledger /     │  │
│  │ Coverage store│  │ (unstructured: │  │ Demo Outbox (drafted     │  │
│  │ (structured   │  │ FAQ, repair-   │  │ emails, claim numbers)   │  │
│  │ JSON, deter-  │  │ partner        │  │                          │  │
│  │ ministic)     │  │ directory)     │  │                          │  │
│  └───────────────┘  └────────────────┘  └─────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| Root/Orchestrator Agent | Entry point for both chat and voice; sets persona/tone; understands goal; routes to the right sub-agent; owns cross-cutting session variables | Root agent with global instructions + `{@AGENT:}` routing references; global model override optional |
| Intake Sub-Agent | Verify identity, look up policy, capture FNOL details (loss date/description), request photo | Sub-agent with narrow instructions + Policy Lookup Python tool |
| Damage Assessment (capability or sub-agent) | Multimodal reasoning over the submitted photo to produce a severity/cost estimate | Native Gemini image understanding (multimodal model call) — no bespoke "vision tool" needed; agent just receives the image in-conversation |
| Decisioning Sub-Agent | Apply deterministic business rules (dollar threshold, injury flag) to decide auto-approve vs. escalate | Python tool for the actual comparison (deterministic) + **Handoff Rule** on a session variable (e.g. `claim_estimate_amount`) to fork the conversation path |
| Escalation / Human-Handoff Sub-Agent | Construct a handoff packet, deliver an escalation message, formally transfer the session | `end_session` system tool (`ESCALATION_MESSAGE`, `LIVE_AGENT_HANDOFF`, `PHONE_GATEWAY_TRANSFER`) |
| Backend Claims-Processing Sub-Agent | The "reveal": take the accumulated conversation/session state and produce (a) a senior-specialist-style summary and (b) a drafted follow-up artifact (customer email or human-assessor briefing) | Sub-agent invoked at/near end of conversation; consumes session variables + short transcript recap; outputs via a Draft-Email Composer Python tool |
| Upsell/Cross-Sell Sub-Agent (or conditional capability) | Detect a mentioned asset (e.g. boat) not present in the customer's coverage, and surface a bundle offer | Instruction-triggered call to Coverage/Asset Lookup tool; fires opportunistically during or after Intake |
| Tools | External-system "hands": lookups, generation, drafting, search, session termination | Python code tools (deterministic mock logic) + a Data Store/RAG tool (unstructured KB) + system tool (`end_session`) |
| Data Stores | Source of truth for policies, coverage, claims, and knowledge content | Structured mock data lives *inside* Python tools (seeded dicts/JSON), not in Vertex AI Search — RAG is reserved for unstructured content only |
| Channels | Chat widget + Google Telephony voice, both pointed at the same agent application | Configured at the agent-application level; multilingual (40+ language chat, ~10 language real-time audio-to-audio) is a native platform capability, not a custom build |

## Recommended Agent/Tool/Data Composition

```
Agent Application: "FNOL Claims Concierge" (white-label)
├── Root Agent: Claims Concierge
│   ├── Global instructions: persona, tone, white-label framing, language handling
│   └── Routes to:
├── Sub-Agent: Intake
│   ├── Instructions: verify identity, capture loss details, request photo
│   └── Tools: policy_lookup (Python)
├── Sub-Agent: Decisioning
│   ├── Instructions: apply threshold + complexity rules, set routing variables
│   ├── Tools: evaluate_claim (Python — deterministic $ threshold + injury flag)
│   └── Handoff Rule: claim_estimate_amount > 1000 OR injury_flag == true → Escalation
├── Sub-Agent: Escalation / Human Handoff
│   ├── Instructions: build handoff packet, deliver escalation message
│   └── Tools: end_session (system tool)
├── Sub-Agent: Backend Claims Processing
│   ├── Instructions: summarize like a senior specialist; draft output artifact
│   └── Tools: draft_email (Python — formats + logs to Demo Outbox)
├── Sub-Agent: Upsell / Cross-Sell
│   ├── Instructions: check for asset/coverage gap, offer bundle, don't be pushy
│   └── Tools: coverage_lookup (Python — same store as Intake, reused)
├── Tools (shared)
│   ├── policy_lookup (Python, mock policy+claimant records)
│   ├── coverage_lookup (Python, mock coverage/asset records)
│   ├── evaluate_claim (Python, deterministic rules)
│   ├── claim_number_generator (Python, sequential/seeded)
│   ├── draft_email (Python, formatting + write to outbox)
│   ├── repair_partner_search (Data Store / Vertex AI Search — unstructured KB)
│   └── end_session (system tool)
└── Data
    ├── Policy & Coverage store — seeded inside Python tools (JSON/dict), not RAG
    ├── Knowledge Base — Vertex AI Search data store (repair-partner directory, FAQ)
    └── Demo Outbox / Claims Ledger — seeded inside Python tools, mutated in-session for the demo run
```

### Structure Rationale

- **Structured mock data lives in Python tools, not a Vertex AI Search Data Store.** A Google Developer forum thread on exactly this question ("Data Store vs. Connectors vs. OpenAPI/MCP") concluded RAG-backed Data Stores "inherently struggle with deterministic logic" (e.g., comparing a dollar amount against a threshold), and recommended a middleware/code layer for anything requiring guaranteed, repeatable outcomes. For a demo that must **reliably** trigger the auto-approve vs. escalate branch and the upsell moment every single time in front of a customer, deterministic Python tools returning seeded JSON are the right choice — RAG's probabilistic retrieval is the wrong tool for "is $1,200 > $1,000."
- **Data Store/RAG is reserved for unstructured content** (repair-partner directory, FAQ-style knowledge) where grounded, flexible language generation is actually valuable and non-determinism is acceptable.
- **Damage assessment is not a separate "tool"** — CX Agent Studio's underlying Gemini models are natively multimodal (image understanding is a model capability, confirmed via Google's official "Image understanding" docs), so the Intake or a dedicated Damage-Assessment sub-agent just receives the photo in-conversation and reasons over it directly. This matches how V1 already works and needs no new integration.
- **The Backend Claims-Processing sub-agent is shared by both the autonomous and HITL branches.** This is a deliberate architectural reuse: the same "ingest conversation → produce specialist-quality summary" capability powers the auto-drafted customer email in the autonomous path AND the human-assessor briefing packet in the escalation path. One sub-agent, two output modes (email draft vs. handoff-packet), selected by which branch called it. This keeps the "reveal" logic in one place and makes the demo narrative ("here's what happens behind the scenes either way") consistent.
- **Ingestion is in-session, not a real BigQuery/Cloud Function pipeline.** CX Agent Studio does support conversation-history export to BigQuery and Cloud Logging for production analytics, but standing up a real export→ingest→summarize pipeline is unnecessary engineering for a mock-data ASAP demo. The Backend Claims-Processing sub-agent should simply consume the current session's accumulated state variables and a short transcript recap (both available in-session to a Python tool / to the sub-agent's own context) — this achieves the visual "reveal" without needing a real data pipeline. Flag this explicitly in the spec so the implementation team doesn't over-build.
- **Agent-as-Tool vs. full sub-agent handoff:** use full sub-agent delegation (conversation control transfers) for Intake, Decisioning, Escalation, and Backend Processing — each owns a distinct phase of the interaction. Consider Agent-as-Tool (capability reuse without full handoff) for the Upsell check, since it's a lightweight, opportunistic lookup that shouldn't take over the conversation.

## Architectural Patterns

### Pattern 1: Deterministic Decision Fork via Handoff Rule + Python Tool

**What:** A Python tool (`evaluate_claim`) computes the estimate and a boolean/enum decision; a Handoff Rule on the resulting session variable (e.g. `escalation_required: true/false`) deterministically routes to either the Decisioning sub-agent's auto-approve output or the Escalation sub-agent — no LLM judgment call on the threshold itself.
**When to use:** Any moment the demo needs a guaranteed, repeatable branch (the $1,000 threshold, injury flag).
**Trade-offs:** Slightly more setup than "just ask the LLM to decide," but removes any risk of the model fudging the branch live in front of a customer — non-negotiable for a sales demo where the branch itself is the wow moment.

### Pattern 2: Single Backend Sub-Agent, Dual Output Mode

**What:** One "Backend Claims Processing" sub-agent, invoked from both branches, whose instructions say: "if `escalation_required` is false, produce a customer-facing draft email; if true, produce a human-assessor handoff summary."
**When to use:** Whenever two branches share the same underlying transformation (transcript → structured insight) but differ only in audience/output format.
**Trade-offs:** Keeps the "ingestion → summary" logic DRY and demo-narrative-consistent; requires instructions to clearly branch on the mode variable rather than duplicating a second sub-agent.

### Pattern 3: Opportunistic Sub-Agent Trigger for Upsell

**What:** Rather than a scripted, always-fires upsell step, the Upsell sub-agent (or agent-as-tool) is instructed to activate when the conversation surfaces an asset-mention pattern (e.g. "tree hit my car... and my boat") and a coverage-gap lookup confirms the asset is uninsured.
**When to use:** Contextual, natural-feeling cross-sell moments — the demo's "cost center → profit center" story beat depends on this feeling organic, not scripted.
**Trade-offs:** Requires the mock data to be deliberately seeded so the trigger reliably fires in the demo script (see Mock Data Appendix in the recommended spec structure below) — this is a controlled "reliable accident," not a random model decision.

## Data Flow

### Autonomous Path (auto-approve)

```
Customer (chat or voice)
    ↓
Root Agent → {@AGENT: Intake}
    ↓ (policy_lookup tool)                      ↓ (photo submitted)
Claimant + policy verified               Damage Assessment (multimodal)
    ↓___________________________________________↓
                        ↓
         claim_estimate_amount set (session variable)
                        ↓
        Root Agent → {@AGENT: Decisioning}
                        ↓ (evaluate_claim tool: amount ≤ $1,000, no injury)
              escalation_required = false  →  Handoff Rule → AUTO-APPROVE
                        ↓
        claim_number_generator tool → claim number issued to customer
                        ↓
       (opportunistic) Root Agent → {@AGENT: Upsell}
        coverage_lookup tool → gap found → bundle offer surfaced
                        ↓
        Root Agent → {@AGENT: Backend Claims Processing} (mode = customer_email)
        consumes session state + transcript recap
                        ↓
        draft_email tool → specialist-style summary + drafted customer
        email ("sorry about your laptop, here's a recommended repair
        partner") written to Demo Outbox — this is the "reveal" shown
        to the sales audience
                        ↓
        end_session tool (normal close, no live-agent transfer)
```

### HITL / Escalation Path

```
... same Intake + Damage Assessment as above ...
                        ↓
         claim_estimate_amount set (session variable)
                        ↓
        Root Agent → {@AGENT: Decisioning}
                        ↓ (evaluate_claim tool: amount > $1,000 OR injury_flag)
              escalation_required = true  →  Handoff Rule → ESCALATE
                        ↓
        Root Agent → {@AGENT: Escalation / Human Handoff}
        composes ESCALATION_MESSAGE for the customer
                        ↓
        Root Agent → {@AGENT: Backend Claims Processing} (mode = assessor_briefing)
        consumes session state + transcript recap
                        ↓
        draft_email tool (reused, different template) → structured
        handoff packet / senior-specialist summary for the (simulated)
        human assessor — this is the demo's proof that HITL isn't a
        dead end, it's a warm, context-rich handoff
                        ↓
        end_session tool (ESCALATION_MESSAGE, LIVE_AGENT_HANDOFF=true,
        optionally PHONE_GATEWAY_TRANSFER)
```

### Divergence Point

The **fork happens at Decisioning**, immediately after the damage/cost estimate is known and before any resolution is communicated to the customer. Both paths reconverge at the **Backend Claims-Processing sub-agent** — same component, different output mode — which is the architecturally cleanest place to demo "look, autonomous and human-assisted claims both get this same intelligent backend treatment."

### Key Data Flows

1. **Live conversation → session state:** Every turn appends `<state_update>` events (dynamic variables) to conversation history; Decisioning and Backend Processing read this accumulated state rather than re-parsing raw transcript text.
2. **Transcript → specialist summary → draft email:** Modeled as an in-session sub-agent call (not an external pipeline) consuming session variables + a short recap, for ASAP/mock-data scope. Note for the spec: real production deployments would export conversation history to BigQuery / trigger via Cloud Function for true post-call automation — explicitly out of scope for this demo, call this out so the implementation team doesn't over-build.
3. **Coverage gap → upsell offer:** Coverage/Asset Lookup tool call, gated by an instruction-level trigger condition (asset mentioned + not in coverage list), not a hardcoded flow step — keeps it feeling like an "aha" the agent noticed rather than a scripted beat.

## Scaling Considerations

This is a sales demo, not a production system — "scale" here means *robustness across repeated live presentations and reuse across ~18+ accounts*, not user load.

| Scale | Architecture Adjustments |
|-------|--------------------------|
| Single scripted run (one presenter, one scenario) | Everything above is sufficient — deterministic Python-tool mock data, in-session backend reveal |
| Reused across many accounts/sectors (Scott's 18 accounts + others) | Keep all branding/copy in instructions and mock data (not hardcoded agent names), so swapping "insurance" flavor text or claimant names doesn't require re-architecting; keep mock data sets clearly separated per demo "scenario" so presenters can pick a scenario without cross-contamination |
| Eventual real POV/pilot (post-demo, out of current scope) | Structured mock data would migrate from in-tool Python dicts to real Integration Connectors/MCP against actual policy/claims systems; unstructured KB Data Store would point at real documents; Backend Claims-Processing would move from in-session state to a real conversation-history export + Cloud Function pipeline. Flag this explicitly as a "not now" in the spec so the implementation team isn't tempted to build it prematurely. |

### Scaling Priorities (for this project)

1. **First risk: mock data reliability.** The single most important "scaling" concern for a live sales demo is that the seeded data *always* deterministically produces the intended branch (auto-approve vs. escalate) and the intended upsell trigger. Any RAG/probabilistic component in that decision path is the first thing that will break trust live. Mitigate with Pattern 1 (deterministic tool + Handoff Rule).
2. **Second risk: reusability drift.** As the demo gets reused and lightly customized per account, instructions/mock data will get copy-pasted and diverge. Mitigate by keeping all account-specific flavor isolated to a small, clearly-labeled section of the mock-data appendix (see spec structure below), not scattered through agent instructions.

## Anti-Patterns

### Anti-Pattern 1: Using a Vertex AI Search Data Store for the Decision Threshold

**What people do:** Put policy/claim/coverage records into a Data Store and ask the agent to "look up whether the claim is under $1,000" via RAG retrieval.
**Why it's wrong:** RAG is probabilistic retrieval + generation — it can misread a number, hallucinate a threshold, or phrase the same fact two different ways from run to run. For a demo whose entire "wow" is a *visibly reliable* autonomous-vs-human branch, this is the single highest-risk mistake.
**Do this instead:** Deterministic Python tool + session variable + Handoff Rule (Pattern 1).

### Anti-Pattern 2: Building a Real BigQuery/Cloud Function Ingestion Pipeline for the Backend Reveal

**What people do:** Take "conversation transcript → Gemini Enterprise ingestion → summary → draft email" literally and stand up conversation-history export to BigQuery plus a Cloud Function trigger.
**Why it's wrong:** Massive over-engineering for a mock-data, ASAP-timeline sales demo; adds infrastructure, auth, and failure surface the implementation team doesn't need and the spec's constraints (mock data, deliverable is a spec not real integrations) explicitly rule out.
**Do this instead:** In-session Backend Claims-Processing sub-agent consuming session state (Pattern 2). Reserve the real pipeline as a documented "how this would work in production" footnote, not a build requirement.

### Anti-Pattern 3: Deep Sub-Agent Nesting

**What people do:** Sub-agents that invoke sub-agents that invoke sub-agents, mirroring an org chart.
**Why it's wrong:** The community architecture deep-dive on this platform explicitly warns that hierarchies should stay shallow — added routing hops increase latency and routing-confusion risk, both bad in a live, time-boxed sales demo.
**Do this instead:** Root + one layer of specialist sub-agents (Intake, Decisioning, Escalation, Backend Processing, Upsell) is sufficient for this use case; keep Damage Assessment as a capability of Intake rather than its own sub-agent unless instruction complexity demands separation.

### Anti-Pattern 4: Conflating This Demo with Agent Assist

**What people do:** Reach for Agent Assist (real-time summarization/coaching for human agents) to build the "senior-specialist summary" moment, since it sounds like a natural fit.
**Why it's wrong:** Explicitly out of scope per project constraints — Agent Assist is customer-dependent and deferred; using it here would misrepresent what the white-label demo needs to reproducibly show and add a dependency the spec doesn't want.
**Do this instead:** Build the "specialist summary" as an ordinary Backend Claims-Processing sub-agent output (Pattern 2), which stays entirely inside CX Agent Studio's flow-free agent model.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Google Telephony (voice channel) | Native channel configuration at the agent-application level | Preferred over external carriers per Akash's guidance in PROJECT.md — avoids extra integration surface for a demo |
| Vertex AI Search (Data Store) | RAG tool for unstructured KB content only (repair-partner directory, FAQ) | Do not use for structured/deterministic data (see Anti-Pattern 1) |
| MCP servers / OpenAPI / Integration Connectors | Available for real backend integration | Not needed for this mock-data demo; document as the "path to production" in the spec, not a build item |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Root Agent ↔ Sub-Agents | `{@AGENT: Name}` instruction-level delegation | Root owns top-level routing; each sub-agent should have a crisp, non-overlapping description so the LLM router doesn't get confused about which sub-agent owns a given user need |
| Sub-Agent ↔ Tool | `{@TOOL: tool_name}` instruction-level invocation | Tools are stateless-ish helpers; put business logic (threshold math) in tools, not in agent instructions, to keep it deterministic |
| Decisioning ↔ Escalation/Auto-Approve | Handoff Rule on a session variable | Deterministic, not LLM-judged — this is the load-bearing boundary for the demo's core wow moment |
| Autonomous branch ↔ HITL branch | Both call the same Backend Claims-Processing sub-agent with a different mode variable | Reuse point — avoids duplicated "summarize the conversation" logic |

---

# Recommended Specification Document Structure

The deliverable is not the demo — it is a spec an implementation team builds directly from. Below is the recommended structure, what each section contains, why it's ordered this way, and the authoring-order dependencies that should inform roadmap phasing.

## Section-by-Section Structure

| # | Section | Contents | Depends On |
|---|---------|----------|-------------|
| 0 | **Cover / Purpose** | One page: what this demo is, who it's for (Scott + ~18 accounts), the "show don't tell" principle, white-label framing, link back to V1 as baseline | — |
| 1 | **Demo Narrative / Storyline** | The single coherent end-to-end script that stitches every wow moment together in order: e.g. chat opens → intake → photo → estimate → threshold branch reveal (run once each way) → backend reveal → upsell moment → (optional) voice + language-switch variant. Written as prose/beats, not yet technical. | Section 0 |
| 2 | **Platform Capability Map** | A table mapping every wow moment in the narrative to a CX Agent Studio primitive: root/sub-agent, tool type, data store, channel, or native capability (e.g. "mid-conversation language switch → native audio-to-audio translation, no custom build"). This is where research (this ARCHITECTURE.md) gets translated into spec form. | Section 1 |
| 3 | **Decision Logic Spec** | Explicit, unambiguous business rules as a table: the $1,000 threshold, the injury auto-escalate rule, the upsell trigger condition(s). No prose ambiguity — literal `if/then` rows an engineer can encode directly into a tool. | Section 1 |
| 4 | **Mock Data Appendix** | Actual seed data: sample policies/claimants (some with gaps in coverage, e.g. no boat insurance), sample claims scenarios engineered to hit both sides of the threshold, sample damage descriptions/photos, multilingual test phrases for the language-switch demo. This is the literal JSON/table the Python tools will return. | Section 3 |
| 5 | **Agent Architecture Spec** | Root agent + each sub-agent, one subsection each: name, responsibility, persona/tone notes, trigger condition (when root delegates to it), instruction guidance (bulleted behaviors — not final prompt text, but enough for the implementation team to write instructions), inputs/variables consumed, outputs/variables produced, tools it calls. | Sections 2, 3 |
| 6 | **Tool & Data Inventory** | One table row per tool (name, type — Python/Data Store/system, purpose, inputs, outputs, which mock data it reads) and one row per data store (contents, schema, seed record count, structured vs. unstructured). Single source of truth cross-referenced by every use-case spec. | Sections 2, 4, 5 |
| 7 | **Per-Use-Case Specs** | One spec per wow moment, using a fixed template (below). This is the bulk of the document and the part the implementation team builds turn-by-turn from. | Sections 3, 4, 5, 6 |
| 8 | **Demo Script / Runbook** | Literal presenter walkthrough: what the presenter says/does, what appears on screen at each step, expected timing, which of the 2-3 canned scenarios to run (autonomous happy path, HITL path, language-switch path), and fallback guidance if something misfires live. | Section 7 |
| 9 | **Acceptance Criteria (global)** | Demo-level "what good looks like": every wow moment lands, no dead air, both branches demoable on demand, timing under target minutes, white-label framing intact. Rolls up the per-use-case acceptance criteria from Section 7. | Section 7 |
| 10 | **Open Questions / Risks / Assumptions** | Anything the implementation team needs to confirm against the live console (exact tool-config UI, Handoff Rule syntax, telephony setup steps) plus GTM ambiguities to confirm with Scott/Akash/Srini. Maintained throughout, finalized last. | All |

### Per-Use-Case Spec Template (used inside Section 7)

```
### Use Case: [name, e.g. "Autonomous Decision Branch — Auto-Approve"]

**Trigger:** [what user/system event starts this]
**Agent(s) involved:** [root + which sub-agent(s), referencing Section 5]
**Behavior / conversation flow:** [numbered steps of what the agent does/says]
**Tools & data required:** [reference Section 6 rows by name]
**Expected output:** [concrete artifact — claim number issued, email drafted, offer surfaced]
**Acceptance criteria:** [bullet list — testable, e.g. "given seeded claim X ($800, no injury),
  agent auto-approves within N turns and issues a claim number without human handoff"]
**Edge cases / fallbacks:** [what happens if photo is ambiguous, if user answers out of order, etc.]
```

Recommended use cases for Section 7, derived from the demo narrative in PROJECT.md:
1. Chat Intake (baseline, carried from V1)
2. Voice Intake
3. Mid-Conversation Language Switch (voice)
4. Multimodal Damage Assessment
5. Decision Threshold Branch — Autonomous (auto-approve)
6. Decision Threshold Branch — HITL (escalate)
7. Backend Claims-Processing Reveal (both output modes: customer email + assessor briefing)
8. Cross-Sell / Upsell Moment
9. Escalation / Out-of-Scope Handling (carried from V1 — injury case, weather question)

## What Makes This Spec Unambiguous and Buildable

- **Every business rule is a table, not a paragraph.** The $1,000 threshold, injury flag, and upsell trigger belong in Section 3 as literal conditions, not described narratively inside a use-case section — this is what lets the implementation team write a deterministic Python tool directly rather than interpreting prose.
- **Mock data is concrete, not illustrative.** Section 4 should contain actual seed records ("Policy #PL-10234, claimant Jane Doe, coverage: auto, home; NOT boat") that the implementation team drops straight into tools — not "a policy without boat coverage" as a placeholder.
- **Every use-case spec cross-references the Tool & Data Inventory by name**, never re-describes data inline — this prevents the two from drifting out of sync as the spec is edited.
- **Acceptance criteria are testable, not aspirational.** "Given seeded scenario X, the agent does Y within N turns" is buildable; "the agent should feel smart about upselling" is not.
- **Instruction guidance, not final prompts.** Section 5 gives the implementation team behavioral bullets and boundaries (what this sub-agent owns, what it must never do) — leaves the actual instruction-writing/tuning to the people with console access, avoiding a spec that's already stale the moment it's written against a UI the author didn't have open.
- **The "not now" list is explicit.** Section 10 (and the Anti-Patterns above) should explicitly rule out real BigQuery pipelines, real Integration Connectors, and Agent Assist — so the implementation team doesn't over-build against an ASAP timeline.

## Suggested Authoring Order (Roadmap Phasing Implication)

1. **Foundations first:** Demo Narrative (§1) → Platform Capability Map (§2) → Decision Logic Spec (§3) → Mock Data Appendix (§4). These four are tightly coupled and should be authored together/in one phase — nothing downstream can be written correctly until the narrative, the exact threshold rules, and the exact seed data are locked.
2. **Structural layer second:** Agent Architecture Spec (§5) → Tool & Data Inventory (§6). These translate the foundations into the concrete component inventory the per-use-case specs will reference.
3. **Bulk content third:** Per-Use-Case Specs (§7) — the largest section, written one use case at a time, each pulling from §3–§6. This is naturally parallelizable across use cases once §1–§6 are stable.
4. **Synthesis last:** Demo Script/Runbook (§8) and global Acceptance Criteria (§9) can only be finalized once all use-case specs exist — they roll everything up into what a presenter actually does live.
5. **Ongoing:** Open Questions/Risks (§10) is maintained from day one but only "closed out" at the end.

This ordering directly implies a 3-4 phase roadmap: **(1) Foundations** (narrative + capability map + decision logic + mock data), **(2) Component Architecture** (agent spec + tool/data inventory), **(3) Use-Case Specs** (the bulk, per wow moment), **(4) Runbook & Acceptance** (synthesis + open-questions closeout). Research flags: Phase 1 and Phase 2 are where platform-capability uncertainty is highest (voice/telephony setup specifics, Handoff Rule exact syntax, Data Store config fields) and may warrant a quick console-verification pass before or during authoring, since this research is MEDIUM confidence on exact UI mechanics.

## Sources

- [Agents | CX Agent Studio | Google Cloud Documentation](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/agent) — root/sub-agent structure, settings
- [Instructions | CX Agent Studio | Google Cloud Documentation](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/instruction) — `{@AGENT:}` / `{@TOOL:}` syntax, instruction structuring
- [Tools | CX Agent Studio | Google Cloud Documentation](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/tool) — full tool-type inventory
- [MCP tools | CX Agent Studio | Google Cloud Documentation](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/tool/mcp)
- [Python code tools | CX Agent Studio | Google Cloud Documentation](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/tool/python)
- [CX Agent Studio handoff | Agent Assist | Google Cloud Documentation](https://docs.cloud.google.com/agent-assist/docs/handoff-cxas) — `end_session` tool, escalation/handoff mechanics
- [Conversation history | Dialogflow CX | Google Cloud Documentation](https://docs.cloud.google.com/dialogflow/cx/docs/concept/conversation-history) — transcript retention, BigQuery export
- [Image understanding | Gemini Enterprise Agent Platform | Google Cloud Documentation](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/capabilities/image-understanding) — native multimodal capability
- [Customer Experience Agent Studio | Google Cloud](https://cloud.google.com/gemini-enterprise-cx/cx-agent-studio) — product overview, voice/multilingual capabilities
- [Gemini Enterprise for Customer Experience | Google Cloud](https://cloud.google.com/products/gemini-enterprise-for-customer-experience) — omnichannel gateway, voice
- Architecture advice for CX Agent Studio: Data Store vs. Connectors vs. OpenAPI/MCP — Google Developer forums (`discuss.google.dev/t/architecture-advice-for-cx-agent-studio-data-store-vs-connectors-vs-openapi-mcp/365273`) — MEDIUM confidence, community discussion, informed the Data-Store-vs-tool anti-pattern
- CX Agent Studio Architecture Deep Dive: Root Agents, Sub-Agents, Tools, and the Agentic Paradigm — Yash Kavaiya, Google Cloud Community/Medium, May 2026 — MEDIUM confidence, third-party synthesis of official docs
- Mastering Variables and State Management in AI Agents: CX Agent Studio — Yash Kavaiya, Google Cloud Community/Medium, May 2026 — MEDIUM confidence, informed Handoff Rule / session-variable patterns

---
*Architecture research for: Insurance FNOL claims demo on Google CX Agent Studio (Gemini Enterprise for CX)*
*Researched: 2026-07-08*
