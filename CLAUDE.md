<!-- GSD:project-start source:PROJECT.md -->
## Project

**CX Agent V2 Demo — Use-Case Specification**

A use-case specification package for a white-labeled **"Version 2.0" WOW demo** of an insurance first-notice-of-loss (FNOL) claims CX agent, built on **Google CX Agent Studio** (Gemini Enterprise for Customer Experience). The spec is handed to an implementation team so *they* build the demo in the Studio — this project produces the specification, not the demo itself.

It is a **go-to-market (GTM) asset**: a reusable, generic demo the sales team can put in front of multiple insurance accounts to sell the CX-agent capability (Google account rep Scott Hitchcock alone has ~18 accounts). The guiding principle is Scott's: **"show, don't tell"** — real interactive workflows over slides.

**Core Value:** A specification concrete and compelling enough that the implementation team can build a show-don't-tell demo that lands every wow moment — voice + mid-demo language switching, a visible autonomous-vs-human decision branch, the backend claims-processing reveal, and the cross-sell "cost center → profit center" moment.

If everything else fails, this must hold: **the implementation team can build the demo directly from the spec, and it wows customers.**

### Constraints

- **Platform**: Target Google CX Agent Studio (Gemini Enterprise for CX) — flow-free, instruction + tool + MCP/connector model — because that is where V1 lives and where the implementation team will build V2.
- **Deliverable**: Specification documents only (no demo build, no runnable code) — this is a GTM/spec project; a separate team implements.
- **Data**: Mock/simulated data only — demo-safe, no real customer systems.
- **Branding**: White-label / generic — reusable across accounts and sectors; no per-customer branding.
- **Timeline**: ASAP (weeks) — prioritize the highest-impact wow moments first for a fast follow-up demo.
- **Principle**: Show, don't tell — real interactive workflows, not a deck.
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Platform Components
### Core Platform
| Component | Status (as of research date) | Purpose | Why It's the Answer |
|-----------|------------------------------|---------|----------------------|
| **CX Agent Studio** | GA since 2026-02-04 | The build surface — minimal-code, AI-augmented conversational agent builder | Built on Agent Development Kit (ADK), purpose-built for enterprise CX; this is where V1 already lives and where the implementation team will build V2. [HIGH] |
| **Gemini Enterprise for Customer Experience** | Announced 2026-01-11 (NRF), CX Agent Studio component now GA | The umbrella product connecting Shopping agent, Food Ordering agent, and CX Agent Studio to a shared customer-lifecycle context | Positions the "backend claims-processing reveal" — Gemini Enterprise's broader data-store/summarization capabilities are the natural home for the transcript → summary → drafted-email flow. [HIGH] |
| **Agent Development Kit (ADK)** | Underlying framework | CX Agent Studio is a layer on top of ADK, with enterprise simplifications | Explains why "flow-based agents" (legacy Dialogflow CX) still exist as an *escape hatch*, not the primary paradigm. [HIGH] |
| **Gemini models** — `gemini-2.5-flash` (text/orchestration), `gemini-3.1-flash-live` (voice) | Current default model options | Powers generative reasoning, instruction-following, and multimodal understanding | Model is selectable per agent (root or sub-agent); `gemini-2.0` models are being deprecated June 1, 2026 with auto-migration to 2.5 — do not spec against 2.0. [HIGH] |
### Agent Model — Root Agent + Sub-Agents (Flow-Free)
| Concept | Status | What It Enables | Tag |
|---|---|---|---|
| **Root ("steering") agent** | GA | Primary entry point/orchestrator; owns top-level conversation, delegates to sub-agents | [BUILT-IN] |
| **Sub-agents (child agents)** | GA | Focused agents per domain/task (e.g., "Intake Agent," "Damage Assessment Agent," "Escalation Agent"); sub-agents can call other sub-agents (nested delegation) | [BUILT-IN] |
| **Instructions** | GA | Natural-language directives per agent. Optional XML structuring (`<role>`, `<persona>`, `<constraints>`, `<taskflow>`, `<step>` with `<trigger>`/`<action>`) improves reliability. Reference syntax: `{@AGENT: Name}`, `{@TOOL: name}`, `{variable}` | [BUILT-IN] — this is the primary mechanism for spec'ing agent behavior in the demo |
| **Examples (few-shot)** | GA | Inline `[user]`/`[model]`/`tool_code`/`tool_outputs` blocks embedded in instructions to demonstrate expected behavior for tricky scenarios (e.g., the escalation path) | [BUILT-IN] |
| **Agent-as-tool** | GA (added 2026-04-17) | A sub-agent can be exposed as a callable tool to another agent, sync or async | [BUILT-IN] |
| **Flow-based agents (legacy)** | GA, coexists | Reuse of classic Dialogflow CX flows for highly deterministic sequences the instruction model struggles with (e.g., strict data collection/validation) | Available but **explicitly positioned as a migration bridge, not the recommended path** — best practice is to treat any flow as an "encapsulated black box" called from the instruction-based root agent, not the primary architecture. **Do not spec V2 around flows**, per project constraint. |
| **Application-level settings** | GA | Global defaults: default model/language, voice/speech synthesis params, response length, interruption handling, tool execution mode (parallel/sequential), logging | [BUILT-IN] |
### Tools & Backend Integration
| Tool Type | Status | What It's For | Tag |
|---|---|---|---|
| **Python code tools** | GA | Inline Python with access to a `context`/`ToolContext` object (session state, conversation history). Includes `ces_requests` (HTTP) and `Pydantic`. **This is the recommended mechanism to simulate the claims/policy backend** — a Python tool can return fully mocked JSON (deductible, coverage, claim number, prior claims) with zero external dependencies, deterministically. Tools can also chain: one tool can internally call others (`tools.tool_name({args})`), avoiding multi-step agent orchestration flakiness. | [CUSTOM TOOL] — write one Python tool per backend "API" (e.g., `lookup_policy`, `create_claim`, `get_repair_partners`) seeded with mock data |
| **OpenAPI tools** | GA | Point the agent at any OpenAPI schema (real or mock REST endpoint) | [CUSTOM TOOL] — alternative to Python tools if the team stands up a lightweight mock API service instead of inline logic |
| **Client function tools** | GA | Functionality that lives in the client app (widget) rather than the CX Agent Studio backend; always executes client-side | [CUSTOM TOOL] — only relevant if the demo needs client-side logic (e.g., local camera capture UI) |
| **Integration Connector tools** | GA (Google Maps, Confluence, Jira, SharePoint added 2026-05-26) | 60+ prebuilt enterprise connectors: Salesforce, ServiceNow, Dynamics 365, Zendesk, Workspace, Slack, Teams, BigQuery, Cloud SQL, MongoDB, Jira, Asana, Box, Dropbox, etc. Requires connections configured in **Application Integration** (separate GCP product) first, in the same region as the agent. | [BUILT-IN CONNECTOR, requires setup] — **overkill for this demo**; the project is explicitly mock-data-only, no real systems (per PROJECT.md), so Python tools are simpler and sufficient. Worth a one-line mention in the spec as "path to production" but not the demo mechanism. |
| **MCP tools** | GA | Agent can call any remote MCP server as a tool (same auth model as OpenAPI: server address, auth type, custom headers) | [CUSTOM TOOL / BUILT-IN protocol] — useful if the team wants to demo "MCP" explicitly as a SOTA/interoperability wow moment, e.g. wiring to a small mock MCP server exposing claims operations |
| **CX Agent Studio's own MCP server** | GA | A *remote* MCP server exposed *by* CX Agent Studio itself, letting external AI tools (Gemini CLI, Antigravity) edit/build the agent app programmatically | [BUILT-IN] — an authoring-time convenience, not relevant to the demo's runtime behavior, but worth knowing about for the implementation team's own workflow |
| **Data store tools (Website / Cloud Storage)** | GA | RAG-style **unstructured** content retrieval (indexed docs/website pages) — good for FAQ/policy-wording lookups, **not** for structured transactional data | [BUILT-IN, if using unstructured mock docs] — appropriate for e.g. "what does my policy cover" grounding content; **not** the right tool for claim/policy record lookups (use Python tools instead) |
| **File Search tools** | GA | Search over uploaded files | Same category as data store tools — unstructured content only |
| **Google Search tools / Google Maps tools** | GA | Web grounding / location-aware answers | [BUILT-IN] — could support the "repair partner near you" cross-sell/service moment |
| **Widget tools / rich response widgets** | GA (current implementation is web-widget-only; standalone version planned) | UI cards, carousels, comparison widgets, custom JSON widgets rendered in the web chat widget | [BUILT-IN, configuration required] — usable for a visual claim-summary card or bundle-offer card |
| **System tools: `end_session`, `customize_response`** | GA | `end_session` is the **built-in human-handoff/escalation mechanism** — takes a `reason`, a `session_escalated` boolean, and forwards `params`/metadata. `customize_response` gives voice-specific control (disable barge-in, DTMF, timeouts) for e.g. delivering a critical disclaimer uninterrupted. | [BUILT-IN] — this is the mechanism to spec for the "route to human assessor" branch |
| **Guardrails** (prompt guards, blocklists) | GA | Safety/compliance rails on agent behavior | [BUILT-IN] |
| **Callbacks** | GA | Pre/post-processing hooks around model calls; used for deterministic enforcement (e.g., forcing a disclaimer to appear), hold-music during slow tool calls, agent-to-agent transfer (`Part.from_agent_transfer(agent=...)`) on tool failure | [BUILT-IN, requires code] |
### Voice & Multilingual
| Capability | Status | Detail | Tag / Confidence |
|---|---|---|---|
| **Text/chat language support** | GA | Any language supported by Vertex AI Gemini models (very broad) | [BUILT-IN] HIGH |
| **Voice language variants** | GA / Preview mix | **28 distinct voice language variants** documented. GA: English (US), French (Canada/France), German, Portuguese (Brazil), Spanish (Spain/US). Preview: Mandarin, Danish, Dutch, English (AU/CA/IN/UK), Finnish, Hindi, Hinglish, Indonesian, Italian, Japanese, Korean, Marathi, Polish, Romanian, Spanglish, Swedish, Tamil, Telugu, Turkish, Vietnamese | [BUILT-IN] HIGH — marketing materials round this up to "40+ languages," which likely blends TTS voice count (220+ voices, 40+ languages) with the narrower voice-*interaction* list above. **Use the 28-variant GA/Preview list, not "40+," when specifying exactly which languages the live demo can reliably use for voice.** |
| **Automatic language detection & response matching** | GA, documented in Agents page | "Agents can automatically detect the language of end-user input, and they will automatically respond using the same language." If the agent app is configured multilingual, it "will automatically switch languages to match the user input." | [BUILT-IN] HIGH — this is the mechanism underlying the "mid-demo language switching" wow moment for **chat**. |
| **Audio-to-audio (A2A) real-time voice translation, mid-conversation switching, 10 languages, built with Google DeepMind** | Reported consistently across multiple independent press sources (Constellation Research, TTEC Digital, CMSWire coverage of the Jan 11 2026 announcement) describing a live demo where a speaker switched English↔Mandarin mid-call and the agent adapted instantly | Enables the single most "wow" voice moment Scott described | **[BUILT-IN, per vendor/press] — MEDIUM confidence.** I could not locate this specific "audio-to-audio translation" capability, or its exact 10-language list, in the current `docs.cloud.google.com` CX Agent Studio reference pages (the technical "Languages" reference page documents the 28 voice variants and does not mention A2A translation by that name). It may be (a) a Gemini Enterprise for CX platform capability not yet reflected in CX Agent Studio's own docs, (b) shipped under different terminology, or (c) still rolling out post-GA. **Action for implementation team: confirm directly in the CX Agent Studio console (or with Google account team) which specific languages support live audio-to-audio translation and mid-call switching before building this as a scripted demo moment** — do not assume all 28 voice variants support it. |
| **Speech-to-Text / Text-to-Speech backbone** | GA | Google Cloud STT/TTS engines + Gemini Speech-to-Speech (STS) models under the hood; 220+ TTS voices across 40+ languages for synthesis | [BUILT-IN] HIGH |
| **Ultra-low-latency bi-directional voice streaming** | GA | Documented architecture goal: natural conversational flow during backend tool calls (truly async, no awkward silence); "prefix messages" (`partial=True`) let the agent give a quick acknowledgment while a tool call/model turn is still processing | [BUILT-IN] HIGH — relevant to keeping the claims-lookup tool call from feeling like dead air during the demo |
| **Voice-specific control (`customize_response`)** | GA | Disable barge-in, control DTMF, set timeouts — e.g., to deliver an uninterrupted disclaimer or hold message | [BUILT-IN] HIGH |
### Multimodal (Image / Damage Assessment)
| Capability | Status | Detail | Tag |
|---|---|---|---|
| **Native multimodal understanding (text, audio, image)** | GA | Gemini foundation models power multimodal reasoning directly in agent instructions — no separate "vision API" integration needed | [BUILT-IN] HIGH |
| **Image upload in the web chat widget** | GA | Confirmed in official docs: adding `enable-file-upload` to the `chat-messenger-container` component of the deployed web widget "enables image uploads," letting end users attach photos mid-conversation | [BUILT-IN, one config flag] HIGH — this is exactly the mechanism V1 already used for the "photo of damaged item" step; V2 can rely on it directly |
| **Damage-assessment reasoning over the photo** | Requires instruction/prompt design | The agent's instructions tell it what to look for/conclude from an attached image (e.g., estimate severity, flag for human review if ambiguous) — the vision capability is native to Gemini, but the *assessment logic* (thresholds, categories) is authored via instructions + a Python tool that turns the model's read of the image into a structured decision | [BUILT-IN model] + [CUSTOM TOOL for structured output/decisioning] |
| **Precedent: Google's own Jan 2026 materials use "photo of a damaged appliance" as the flagship multimodal CX example** | Confirmed directly in the official NRF press release: "With visual processing capabilities, they can 'see' what a shopper sees — such as a photo of a damaged appliance." | Validates that damage-photo-assessment is a Google-endorsed, on-brand showcase scenario, not a stretch use case | HIGH confidence, directly sourced |
### Gemini Enterprise Backend (Transcript → Summary → Drafted Email)
| Capability | Status | Detail | Tag / Confidence |
|---|---|---|---|
| **Conversation history (within CX Agent Studio)** | GA | Agent builder has a conversation-history view (accessed via Agent Preview) to browse saved conversations | [BUILT-IN] HIGH for viewing; docs don't detail export/API access for downstream ingestion — **LOW confidence on export mechanics**, needs hands-on confirmation |
| **Gemini Enterprise "apps" and "data stores"** | GA (broader Gemini Enterprise app product, separate docs tree from CX Agent Studio: `docs.cloud.google.com/gemini/enterprise/docs/apps-data-stores`) | Data stores ingest first-party (Cloud Storage, BigQuery, Drive, Gmail, Calendar) and third-party (Jira, Confluence, ServiceNow, Salesforce, Box, Slack) content; apps expose search/answers/actions/agents over that ingested data | [BUILT-IN at the Gemini Enterprise platform level] MEDIUM — this is a **separate, adjacent product surface** from CX Agent Studio; official docs describe search/grounding well but do **not** explicitly document an "auto-draft an email from a transcript" workflow |
| **Gemini Enterprise ready-made agents (Deep Research, NotebookLM Enterprise)** | GA (Gemini Enterprise app, not CX Agent Studio) | NotebookLM Enterprise is described as summarizing/extracting information across dense/complex sources; Gemini Enterprise's "taskforce" of agents can process meeting transcripts (from Google Meet), extract action items, and send follow-up summaries | MEDIUM confidence — these are documented for **meeting transcripts**, not explicitly for CX-agent conversation transcripts; the analogy is close but unverified for this exact use case |
| **Recommended demo approach for the "backend claims-processing reveal"** | — | Given the above, **do not assume a plug-and-play "conversation → drafted email" pipeline exists out of the box.** The most reliable path for the spec is: (1) CX Agent Studio conversation produces a transcript + structured claim data (via Python tool output/session variables); (2) a **custom tool or a second, backend-facing agent** takes that structured data + transcript text and, using Gemini's native generative/summarization ability (same underlying capability, invoked either as an agent instruction or via the Gemini API/Vertex AI directly), produces the specialist summary and drafted customer email. This can be staged entirely as a **scripted/simulated "reveal" screen** (e.g., a second UI pane showing the generated summary+email) rather than a live production-grade ingestion pipeline — consistent with the project's mock-data constraint. | [CUSTOM TOOL / MOCK] — flag as the single biggest platform-capability uncertainty in this research; **recommend a spike/confirmation call with the Google account team before finalizing this section of the spec** |
### Decision Logic / Conditional Branching (Flow-Free)
| Mechanism | Status | How It Expresses "If claim > $1,000, escalate" | Tag |
|---|---|---|---|
| **Instructions with `<taskflow>`/`<step>`/`<trigger>`/`<action>` XML structuring** | GA | The primary, agent-level way to express conditional logic: a step's `<trigger>` names the condition (e.g., claim amount exceeds a threshold), its `<action>` names the outcome (auto-approve vs. route to assessor). This is natural-language-adjacent, not code — the LLM interprets the trigger against conversation/session state each turn. | [BUILT-IN] HIGH for authoring; **inherently probabilistic** — Google's own best-practices guidance warns that "a tool call orchestrated by an agent is not deterministic" |
| **Handoff rules** | GA | The **deterministic** mechanism: rules defined over session variables (text/number/boolean) with AND/OR conditions that force or block a transfer between parent/child agents automatically — e.g., "if `claim_amount` variable > 1000 → force transfer to Escalation Agent." Configurable via UI (variable conditions) or Python (complex logic). | [BUILT-IN, deterministic] HIGH — **this is the recommended mechanism to spec for the dollar-threshold branch**, not instruction-only logic, precisely because it's deterministic and demo-reliable |
| **Python tool computing the decision, writing a session variable, handoff rule reading it** | GA (composition of the above) | Most robust pattern: a Python tool (`evaluate_claim`) computes `auto_approved = claim_amount <= 1000` and writes it to session state; a handoff rule on that boolean variable triggers the transfer | [BUILT-IN + CUSTOM TOOL] HIGH — recommended pattern for the spec's decision-logic requirement, combining deterministic computation with deterministic routing |
| **Callbacks for agent-to-agent transfer on failure/exception** | GA | `Part.from_agent_transfer(agent='escalation agent')` — used e.g. if a tool call fails or data is missing, not just for business-rule branching | [BUILT-IN] — secondary escalation path (technical failure vs. business-rule escalation) |
### Human-in-the-Loop / Escalation & Deployment Channels
| Capability | Status | Detail | Tag |
|---|---|---|---|
| **`end_session` system tool (escalation)** | GA | Ends the session with `session_escalated=true` + `reason` + forwarded params — the built-in signal that a human must take over | [BUILT-IN] HIGH |
| **Handoff rules (parent/child, forward and backward)** | GA | Forward: escalate to a specialized sub-agent (e.g., high-value claims, authentication retry). Backward: return to parent agent to re-gather info or re-qualify | [BUILT-IN] HIGH |
| **Google Telephony Platform (GTP)** | GA deployment channel | **Native, first-party voice channel** — attach a new phone number to the agent in minutes; no third-party carrier needed. Supports call termination via `end_session`, human-agent transfer w/ escalation messaging, SIP signaling (headers/UUI), DTMF input, barge-in/interruption control, remote-disconnect handling. Requires Dialogflow API enabled on the project. Confirms Akash's meeting note: voice is straightforward via GTP; avoid external carriers for a tight Studio-native demo. | [BUILT-IN] HIGH |
| **Other telephony/CCaaS channels** | GA | AudioCodes, Five9, Twilio, Google Cloud CCaaS — for integrating with an existing contact-center stack | [BUILT-IN connector, requires setup] — not needed for this demo since GTP is simpler and Studio-native |
| **Web widget** | GA | Embeddable chat widget; supports rich response widgets and (with one config flag) image upload | [BUILT-IN] HIGH |
| **API access** | GA | Direct API access to the agent app for custom front-ends | [BUILT-IN] |
| **Traffic splitting (A/B testing)** | GA | Route a % of traffic to a different agent version | [BUILT-IN] — not core to the demo, but usable if the team wants to show "V1 vs V2" side by side |
| **Simulator + Evaluations (golden/scenario-based, batch import/export)** | GA | Built-in testing harness for scripting and validating demo scenarios before the live pitch | [BUILT-IN] HIGH — directly useful for rehearsing/QA'ing the seeded auto-approve vs. escalate vs. upsell scenarios so they reliably fire during a live sales demo |
### Prebuilt Templates Relevant to Insurance/Claims/FNOL
| Finding | Confidence |
|---|---|
| **As of the current docs (noted "as of November 2025" in the docs themselves), the only sample agent application is "Cymbal Retail"** — a fictitious home-and-garden retail assistant demonstrating multi-agent architecture, structured instructions, Python/Search/OpenAPI tools, state management, guardrails, and evaluations. **No insurance, claims, FNOL, or financial-services template exists in CX Agent Studio today.** | HIGH — confirmed directly in official docs, which explicitly flag "more samples will be added in the future" |
| The only insurance-adjacent precedent anywhere in Google's own materials is the NRF press release's "photo of a damaged appliance" multimodal example (retail/appliance context, not insurance) | HIGH |
| **Implication:** V2 cannot lean on an existing insurance template for structure or credibility — the demo narrative, sub-agent breakdown, and mock data all have to be built from scratch, using Cymbal Retail's *architectural pattern* (multi-agent + tools + guardrails + evaluations) as the closest available reference implementation. | — |
## What NOT to Rely On
| Avoid Assuming | Why | Use Instead |
|---|---|---|
| Flow-based (visual) agent design as the primary V2 architecture | Explicitly out of scope per PROJECT.md; also Google's own best practices treat flows as a migration bridge/black-box, not the recommended paradigm | Root agent + sub-agents driven by instructions, tools, and handoff rules |
| Data store tools (website/Cloud Storage RAG) for structured claim/policy lookups | These are unstructured-content retrieval tools, not a database — unreliable for exact-match fields like deductible amounts or claim IDs | Python code tools returning structured mock JSON |
| "40+ languages" as the number of languages that support **voice** interaction | Marketing rounding conflates TTS voice/synthesis language count (40+, 220+ voices) with the narrower, currently GA+Preview voice-*interaction* language list (28 variants) | Use the 28-variant list from the Languages reference page when scripting which languages the live voice demo will use |
| Audio-to-audio translation as a documented, confirmed-available CX Agent Studio feature with a fixed 10-language list | Only found in press coverage of the Jan 2026 announcement, not in the current CX Agent Studio technical docs reviewed | Confirm directly with the Google account team / in-console before building the mid-call language-switch demo moment around it |
| A ready-made "transcript → summary → drafted email" pipeline inside Gemini Enterprise | Documented Gemini Enterprise capabilities (data stores, NotebookLM Enterprise, meeting-transcript action-item extraction) are adjacent but not confirmed for this exact CX-transcript-to-email workflow | Spec it as a custom tool / simulated "reveal" screen built on Gemini's generative/summarization capability, not an off-the-shelf feature |
| Integration Connectors (Salesforce/ServiceNow/etc.) as the mock-backend mechanism | Requires Application Integration setup, real connections, same-region config — heavyweight for a demo that's explicitly mock-data-only | Python code tools with inline mocked data |
## Platform Patterns by Demo Requirement
- Use a Python tool to compute the decision deterministically from mock claim data, write it to a session variable, and use a **handoff rule** (not instruction-only logic) on that variable to route to the Escalation sub-agent. This is the only combination that's both demo-reliable (deterministic) and natively supported.
- Chat-based language auto-detect-and-switch is well-documented and safe to script.
- Voice-based audio-to-audio translation is press-confirmed but not yet found in CX Agent Studio's own technical docs — treat as a stretch goal pending direct platform confirmation, with the Google Telephony Platform + the 28 documented voice-language variants as the confirmed fallback (agent responds in whichever supported language the caller speaks, without live translation).
- Enable `enable-file-upload` on the web widget; let Gemini's native multimodal reasoning assess the photo per agent instructions; use a Python tool to turn that assessment into a structured severity/decision output feeding the threshold logic above. This is the most solidly built-in wow moment in the whole spec (Google's own launch materials use nearly this exact scenario).
- Treat as the platform's biggest open question. Recommend the spec describe it as a custom-built second stage (tool or backend agent) using Gemini's generative capability on the captured transcript/structured data, staged as a simulated "reveal" UI — not marketed to the implementation team as an off-the-shelf Gemini Enterprise feature.
## Sources
- [CX Agent Studio overview](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio) — product overview, architecture, GA claims [HIGH]
- [Agents | CX Agent Studio](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/agent) — root/sub-agent model, language auto-detect/switch, model options [HIGH]
- [Instructions | CX Agent Studio](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/instruction) — instruction structure, XML tags, conditional-logic-via-taskflow, examples [HIGH]
- [Handoff rules | CX Agent Studio](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/handoff) — deterministic conditional routing mechanism [HIGH]
- [Tools overview / MCP tools / OpenAPI tools / Python code tools / Connector tools / System tools](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/tool) — full tool taxonomy [HIGH]
- [Python code tools](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/tool/python) — mock-backend mechanism confirmed [HIGH]
- [Data store tools](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/tool/data-store) — unstructured vs. structured data distinction [HIGH]
- [Integration Connector tools](https://docs.cloud.google.com/customer-engagement-ai/conversational-agents/ps/tool/connector) — 60+ connectors, Application Integration dependency [HIGH]
- [Rich response widgets](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/rich-response-widget) — widget gallery, current limitations [MEDIUM — page itself notes feature set will expand]
- [Web widget deployment](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/deploy/web-widget) — `enable-file-upload` image-attach confirmed [HIGH]
- [Google Telephony Platform deployment](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/deploy/google-telephony-platform) — native voice channel, setup, SIP/DTMF details [HIGH]
- [Flow-based agents](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/flow) — legacy/migration positioning [HIGH]
- [Best practices and patterns](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/best-practices) — tool design, determinism guidance, callbacks, prefix messages [HIGH]
- [Languages reference](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/reference/language) — 28 voice-language variants (GA + Preview), chat = any Vertex AI Gemini language [HIGH]
- [Sample agent applications](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/agent-sample) — Cymbal Retail is the only sample; no insurance/claims template exists [HIGH]
- [Release notes](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/resources/release-notes) — GA date 2026-02-04, feature timeline through June 2026, Gemini 2.0 deprecation [HIGH]
- [Gemini Enterprise for CX — official press release, Jan 11 2026](https://www.googlecloudpresscorner.com/2026-01-11-Google-Cloud-Brings-Shopping-and-Customer-Service-Together-with-Gemini-Enterprise-for-Customer-Experience) — "photo of a damaged appliance" multimodal example, 40+ language phone/mobile/web support [HIGH]
- [Gemini Enterprise apps and data stores](https://docs.cloud.google.com/gemini/enterprise/docs/apps-data-stores) — data store ingestion model (adjacent product surface) [MEDIUM]
- Constellation Research, TTEC Digital, and CMSWire coverage of the Jan 2026 announcement — audio-to-audio translation, 10 languages, mid-conversation switching, "built with Google DeepMind" [MEDIUM — press-only, not found in current CX Agent Studio technical docs; needs direct platform confirmation]
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
