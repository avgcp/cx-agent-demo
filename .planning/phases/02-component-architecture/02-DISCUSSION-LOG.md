# Phase 2: Component Architecture - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-07-09
**Phase:** 2-component-architecture
**Areas discussed:** Upsell component form, Repair-partner lookup, Session-variable contract, Instruction-guidance depth

---

## Upsell Component Form

| Option | Description | Selected |
|--------|-------------|----------|
| Sub-agent, listed — agent-as-tool noted | Spec as one of the 5 named sub-agents (SC#1 literal), but note it's invoked opportunistically and MAY be an agent-as-tool so it never hijacks the conversation | ✓ |
| Pure agent-as-tool / capability | Spec as a capability/agent-as-tool of root, not a peer sub-agent — most faithful to research but deviates from SC#1's explicit 5-sub-agent list | |
| Full sub-agent, full handoff | Spec as a full conversation-owning sub-agent — uniform but risks the cross-sell feeling scripted | |

**User's choice:** Sub-agent, listed — agent-as-tool noted (Recommended)
**Notes:** Reconciles ROADMAP SC#1's explicit 5-sub-agent list with research Pattern 3 (opportunistic cross-sell). Cross-sell target is uninsured other devices per Phase 1 D-08. → D-15

---

## Repair-Partner Lookup

| Option | Description | Selected |
|--------|-------------|----------|
| In scope, mock Python tool | Keep the "recommended repair partner" beat, spec as a deterministic mock Python tool returning seeded records — consistent with SC#4, one ordinary inventory row | ✓ |
| In scope, RAG Data Store exception | Spec as the one unstructured Data Store/RAG lookup (research's model) — adds a second paradigm + non-determinism risk on stage | |
| Out of scope for now | Drop the recommendation entirely — simplest inventory but loses a concierge flourish and needs a Phase 1 narrative tweak | |

**User's choice:** In scope, mock Python tool (Recommended)
**Notes:** Consequence — the demo now has ZERO RAG/Data Store components; every tool is a deterministic Python tool. Supersedes research's `repair_partner_search` Data Store. → D-16

---

## Session-Variable Contract

| Option | Description | Selected |
|--------|-------------|----------|
| Named contract table + console-confirm flag | Table with variable name, type, produced-by, consumed-by, and which decision-row/handoff-rule reads it; exact Studio syntax flagged as console-confirm | ✓ |
| Names only, described per agent | List variables narratively inside each agent's I/O subsection, no consolidated table — less traceable | |
| Defer variable naming entirely | Describe handoffs conceptually with no committed names — weakest for buildability | |

**User's choice:** Named contract table + console-confirm flag (Recommended)
**Notes:** The interface glue that makes handoffs traceable back to the Phase 1 Decision Logic table (§3). Names are Claude's discretion; the contract is mandatory. → D-17

---

## Instruction-Guidance Depth

| Option | Description | Selected |
|--------|-------------|----------|
| Behavioral bullets + XML skeleton hint | Behavioral-guidance bullets as authoritative content + a short illustrative `<role>/<constraints>/<taskflow>` XML shape hint per agent — buildable, resistant to console drift | ✓ |
| Behavioral bullets only | Pure bullets, no XML structure — most durable but less of a head-start | |
| Near-final instruction text | Close-to-final paste-ready prompts — most immediately buildable but highest staleness risk (research anti-pattern) | |

**User's choice:** Behavioral bullets + XML skeleton hint (Recommended)
**Notes:** Enough shape to build from, short of paste-ready prompts that go stale against a live console the author can't see. → D-18

---

## Claude's Discretion

- Exact session-variable names (within the mandatory D-17 contract structure) and their types.
- Tool & Data Inventory column set beyond SC#4's required fields.
- Full tool-roster completeness (evaluate_claim, claim_number_generator, draft_email dual-template, repair-partner lookup, `end_session` system tool) — completeness is correctness, not a new decision.
- Root-agent persona/tone wording and `{@AGENT:}` routing-description phrasing.
- §5/§6 cross-reference style back to Phase 1 sections; damage-assessment sub-section placement inside Intake and exact shallow-nesting guardrail wording.

## Deferred Ideas

- Repair-partner directory as a real RAG Data Store → "path to production" footnote only.
- Real BigQuery/Cloud Function backend-reveal pipeline → out of scope (research Anti-Pattern 2); belongs to Phase 4 open-questions.
- Near-final instruction prompt text → deferred to the implementation team with live console access.
