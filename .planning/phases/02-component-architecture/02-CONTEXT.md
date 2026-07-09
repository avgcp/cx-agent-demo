# Phase 2: Component Architecture - Context

**Gathered:** 2026-07-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 2 produces the two **structural-layer** sections of the use-case specification (§5 and §6 of the recommended spec structure in `research/ARCHITECTURE.md`):

1. **Agent Architecture Spec (§5)** — the root "Claims Concierge" agent plus a single shallow layer of sub-agents (Intake, Decisioning, Escalation/Human-Handoff, Backend Claims-Processing, Upsell/Cross-Sell). One subsection per agent: name, responsibility, trigger (when root delegates), instruction guidance, inputs/variables consumed, outputs/variables produced, tools called.
2. **Tool & Data Inventory (§6)** — a single-source-of-truth table, one row per tool and per data store, that every Phase 3 use-case spec references **by name** instead of re-deriving.

Requirements ARCH-01, ARCH-02, ARCH-03, TOOL-01, TOOL-02 are fixed by REQUIREMENTS.md/ROADMAP.md. This discussion locked the *authoring choices* the research and success criteria left genuinely open — the four gray areas below — while the WHAT (5-sub-agent list, shared dual-mode Backend agent, damage-assessment-inside-Intake, one-row-per-tool inventory, every backend lookup = deterministic mock Python tool) is already locked by the Phase 2 ROADMAP success criteria and is not re-litigated here.

**This phase writes spec text, not software.** Success = §5 and §6 are unambiguous, buildable, internally consistent, and traceable to the Phase 1 Decision Logic and Mock Data Appendix.

### Locked by ROADMAP success criteria (restated for downstream agents — NOT re-decided here)
- **SC#1:** Root "Claims Concierge" + shallow one-layer sub-agents (Intake, Decisioning, Escalation/Human-Handoff, Backend Claims-Processing, Upsell/Cross-Sell); each with responsibility, trigger, instruction guidance, inputs/outputs. **No deep nesting beyond one sub-agent layer.**
- **SC#2:** Backend Claims-Processing sub-agent is **one shared component** across the autonomous and HITL branches, with **two output modes** (customer email vs. human-assessor briefing packet) selected by a mode variable — never duplicated as two components.
- **SC#3:** Damage assessment is a **native-multimodal capability of the Intake sub-agent**, not a separate deep-nested agent; the **shallow-nesting guardrail is stated explicitly** as a design rule other authors must follow.
- **SC#4:** Tool & Data Inventory = exactly one row per tool/data store; **every backend lookup is a deterministic mock Python code tool returning seeded JSON**, not a Data Store/RAG lookup.
- **SC#5:** Exact Handoff Rule UI/syntax and exact Python-tool session-variable mechanics are **explicitly flagged as console-confirm items**, not asserted as settled.

</domain>

<decisions>
## Implementation Decisions

Decision IDs continue Phase 1's sequence (D-01..D-14) to stay globally unique across the spec, so Phase 3 use-case specs can cite any decision unambiguously.

### Upsell Component Form (ARCH-01) — reconciles SC#1 with research Pattern 3
- **D-15:** Spec **Upsell/Cross-Sell as one of the five named sub-agents** (keeps SC#1 literal and the "root + 5 sub-agents" story clean), but its §5 subsection **must explicitly note** that it is invoked *opportunistically* — triggered when the conversation surfaces an uninsured-device signal and a coverage-gap lookup confirms the gap — and **may be implemented as an "agent-as-tool"** rather than a full conversation-owning handoff, so it never hijacks the conversation. This satisfies the success criterion while preserving the research's "organic aha, not scripted step" intent (research §Pattern 3). The cross-sell target is *uninsured other devices* per Phase 1 **[[D-08]]**, not boat.

### Repair-Partner Lookup (TOOL-01, TOOL-02)
- **D-16:** Keep the **"recommended repair partner" beat** (it enriches the autonomous-path backend-reveal email from Phase 1 §1) and spec it as a **deterministic mock Python tool** returning seeded partner records — **one ordinary inventory row like the other lookups**. Consequence: **the demo has ZERO RAG / Data Store components — every tool in the inventory is a deterministic Python tool.** This **supersedes** the research's `repair_partner_search` Vertex AI Search Data Store (`research/ARCHITECTURE.md` §Recommended Composition) and directly honors SC#4's "no Data Store/RAG for backend lookups" and the demo-reliability principle (deterministic on stage every time). The §6 inventory should note the "data store" row count as **0 (all structured mock data lives inside Python tools)** and may add a one-line "path to production" footnote that a real deployment could move the partner directory to a RAG Data Store.

### Session-Variable Contract (ARCH-01, ARCH-02; ties §5 to Phase 1 Decision Logic §3)
- **D-17:** Author an **explicit session-variable contract table** as part of §5 (or a shared sub-section referenced by §5 and §6): columns = **variable name, type, produced-by agent, consumed-by agent(s), and which Decision-Logic row / Handoff Rule reads it.** This is the interface glue that makes every multi-agent handoff traceable end-to-end back to the Phase 1 Decision Logic table (§3). Candidate variables (names are Claude's discretion, but the *contract* is mandatory): `claim_estimate_amount`, `coverage_valid`, `total_loss_flag` (the D-05 always-escalate flag), `escalation_required` (the boolean the Handoff Rule forks on), `backend_output_mode` (`customer_email` | `assessor_briefing`, the SC#2 mode selector), `uninsured_devices_present` (the D-08 upsell trigger), `claim_number`. **Add an explicit SC#5 flag** that exact CX Agent Studio session-variable syntax and Handoff Rule UI mechanics must be confirmed against the live console — the table specifies the *names and data-flow contract*, not the unverified UI mechanics.

### Instruction-Guidance Depth in §5 (ARCH-01, ARCH-03)
- **D-18:** Each agent's §5 "instruction guidance" subsection leads with **behavioral-guidance bullets** as the authoritative content — what the agent owns, must-dos, must-nevers, trigger condition, inputs/outputs — **plus a short illustrative XML structure hint** using the platform's instruction tags (`<role>` / `<persona>` / `<constraints>` / `<taskflow>` with `<trigger>`/`<action>`) so the implementation team sees the intended shape. **Do NOT write near-final/paste-ready prompt text** (research Anti-Pattern: a spec that's stale the moment it's written against a console the author can't see). Buildable head-start, resistant to console drift.

### Claude's Discretion
- **Exact session-variable names** (within the mandatory contract-table structure of D-17) and their exact types.
- **Tool & Data Inventory column set** beyond SC#4's required fields (name, type, purpose, inputs, outputs, source mock data) — e.g. whether to add a "called-by agent" column for traceability.
- **Full tool roster completeness:** the inventory must include not just the five backend lookups named in SC#4 (policy, coverage, claim creation, boat-insurance→*device*-upsell check, damage valuation) but also the supporting tools the architecture needs — e.g. `evaluate_claim` (deterministic threshold/flag math), `claim_number_generator`, `draft_email` (dual-template for D-16's two output modes), the repair-partner lookup (D-16), and the `end_session` **system tool** for escalation/close. Inventory completeness is correctness, not a new decision.
- **Root-agent persona/tone wording** and exact `{@AGENT:}` routing-description phrasing.
- **Section numbering / cross-reference style:** how §5/§6 cite the Phase 1 sections (Decision Logic §3, Mock Data Appendix §4) by name — follow the "cross-reference, never re-derive" discipline established in Phase 1.
- **Damage-assessment sub-section placement** inside the Intake agent's §5 subsection (as a capability), and the exact wording of the shallow-nesting guardrail design rule (SC#3).

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements & Roadmap (locked scope for this phase)
- `.planning/REQUIREMENTS.md` — ARCH-01, ARCH-02, ARCH-03 and TOOL-01, TOOL-02 definitions, plus the Out-of-Scope table (anti-scope-creep guardrails).
- `.planning/ROADMAP.md` §"Phase 2: Component Architecture" — the 5 success criteria that define "TRUE of the spec text" for §5 and §6.
- `.planning/PROJECT.md` — core value, constraints (mock-data-only, white-label, flow-free, ASAP), "show, don't tell."

### Phase 1 outputs (the four sections §5/§6 must reference by name — NEVER re-derive)
- `.planning/spec/section-1-demo-narrative.md` — the end-to-end storyline; source of the autonomous-path email "recommended repair partner" beat (D-16) and the two-claims decision-branch structure.
- `.planning/spec/section-2-platform-capability-map.md` — the per-wow-moment → CX Agent Studio primitive mapping that §5/§6 make concrete.
- `.planning/spec/section-3-decision-logic.md` — the literal if/then rows and Handoff-Rule routing that the D-17 session-variable contract must trace to.
- `.planning/spec/section-4-mock-data-appendix.md` — the seeded records every Python tool in §6 returns (single source of truth for figures/records/images).
- `.planning/phases/01-foundations/01-CONTEXT.md` — Phase 1 locked decisions **D-01..D-14** (esp. D-05 total-loss/data-loss always-escalate flag, D-08 device-bundle cross-sell, D-12/D-13 $1,000 threshold + deterministic tool + Handoff Rule).

### Project Research (grounds the component composition)
- `.planning/research/ARCHITECTURE.md` — recommended root + shallow-sub-agent composition, the dual-output-mode Backend pattern (§Pattern 2), the deterministic-fork pattern (§Pattern 1), the opportunistic-upsell pattern (§Pattern 3), and the deep-nesting anti-pattern. **NOTE: its `repair_partner_search` Data Store is superseded by D-16 (mock Python tool, zero RAG).**
- `.planning/research/STACK.md` — CX Agent Studio primitive taxonomy; agent-as-tool (D-15); Python code tools; `end_session` system tool; Handoff Rules; session variables.
- `.planning/research/PITFALLS.md` — non-determinism and RAG-for-structured-data pitfalls that D-16 and the deterministic-tool discipline pre-empt.
- `CLAUDE.md` (project root) — full platform capability reference tables with source citations and confidence tags (mirror source of STACK.md; includes doc URLs and the "flow-free / do-not-spec-around-flows" constraint).

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **V1 built demo:** the laptop/computer-loss FNOL intake + coverage lookup + photo-based damage assessment + claim-number happy path already exists in the implementation team's CX Agent Studio project. §5's Intake agent (incl. its damage-assessment capability, SC#3) and the corresponding §6 tools are specified to **map onto and extend** that existing structure, not replace it.
- **Phase 1 spec sections** (`.planning/spec/section-1..4`) are the direct inputs §5/§6 build on — the Decision Logic table and Mock Data Appendix are the single sources of truth §5/§6 cite rather than re-describe.

### Established Patterns
- **Spec-authoring project, not code:** all Phase 2 outputs are Markdown spec sections. "Acceptance" = properties of the spec text (unambiguous, buildable, traceable), not running-software behavior.
- **Determinism discipline (carried from Phase 1):** every backend lookup is a deterministic mock Python tool over seeded data; every must-land branch routes via Python tool + Handoff Rule on a session variable — never RAG or LLM judgment. D-16 extends this to eliminate RAG from the demo entirely.
- **Cross-reference, never re-derive:** §5/§6 reference Phase 1 sections and each other by name; the §6 inventory is the single source of truth Phase 3 use-case specs cite.

### Integration Points
- §5 and §6 are the components Phase 3's 9 use-case specs reference by name. The **session-variable contract (D-17)** and the **Tool & Data Inventory (§6)** are the two artifacts Phase 3 depends on most — they must be complete and internally consistent before Phase 3 can be written without re-deriving.

</code_context>

<specifics>
## Specific Ideas

- **Zero-RAG demo (D-16):** the deliberate, notable simplification of this phase — every tool is a deterministic Python tool, no Data Store/RAG anywhere. Call this out as a clean architectural property (and a reliability win for a live sales demo), superseding the research's one RAG carve-out.
- **Session-variable table as the "glue" (D-17):** the user specifically wanted the multi-agent handoffs to be traceable back to the Phase 1 Decision Logic rows — the contract table is the mechanism, with exact Studio syntax flagged as console-confirm.
- **Upsell stays "organic" (D-15):** even though listed as the 5th sub-agent, it must read as an opportunistic capability so the "cost center → profit center" cross-sell feels earned, not scripted.
- **Buildable but not stale (D-18):** behavioral bullets + illustrative XML skeleton — enough shape to build from, short of paste-ready prompts that would go stale against the live console.

</specifics>

<deferred>
## Deferred Ideas

- **Repair-partner directory as a real RAG Data Store** — deferred to "path to production"; the demo uses a deterministic mock Python tool (D-16). Retained as a one-line production footnote in §6, not a build item.
- **Real BigQuery/Cloud Function backend-reveal pipeline** — remains out of scope (research Anti-Pattern 2); the Backend Claims-Processing sub-agent consumes in-session state. Belongs to Phase 4 Open-Questions "path to production," not Phase 2.
- **Near-final instruction prompt text** — deliberately deferred to the implementation team with live console access (D-18); §5 gives guidance + XML shape only.
- V2/DEF items (Integration Connectors/MCP against real systems, Agent Assist, cross-channel continuity) remain out of scope per REQUIREMENTS.md.

</deferred>

---

*Phase: 2-Component-Architecture*
*Context gathered: 2026-07-09*
