## Decision Logic

This section is the **single source of truth for routing** in the CX Agent V2 Demo. Every threshold, flag, and condition below references a named field in the Mock Data Appendix (Section 4) — no rule row invents or restates a value not already canonical there. If a presenter or agent instruction states a dollar figure or flag not traceable to a Section 4 record, that is a spec violation, not a demo variation.

### Business-Rules Table (per D-13)

| Rule ID | Condition (literal if/then) | Outcome | Exercised by (Section 4 record) |
|---|---|---|---|
| DL-1 | `claim_amount ≤ $1,000 AND coverage_valid = true AND total_loss_flag = false` | **Auto-approve, issue claim number** | CLM-24001 ($450, cracked screen) and CLM-24003 ($950, near-threshold negative record — must auto-approve, must NOT escalate despite proximity to the $1,000 ceiling) |
| DL-2 | `claim_amount > $1,000` | **Route to human assessor** | CLM-24002 ($2,400, liquid-damaged high-end laptop) |
| DL-3 | `total_loss_flag = true` | **Route to human (regardless of amount)** | CLM-24002 (`total_loss_flag = true`, `data_loss_flag = true`) |
| DL-4 | `claim resolved AND profile has uninsured_device` | **Fire cross-sell** | Primary claimant Jordan Rivera (`uninsured_device` = recently purchased smartphone, present); must **NOT** fire for Sam Okafor (`uninsured_device` = none) |

**Notes on precedence and negatives:**
- DL-3 is evaluated independently of DL-1/DL-2 and always wins: a claim can be under $1,000 and still route to a human if `total_loss_flag = true`. CLM-24002 satisfies both DL-2 (amount > $1,000) and DL-3 (total-loss) — either rule alone is sufficient to route it to a human; the two are not mutually exclusive. (per D-05)
- CLM-24003 ($950) exists solely to catch a decision-logic bug that escalates on *proximity* to the $1,000 ceiling rather than on the literal `claim_amount > $1,000` comparison. It must auto-approve exactly like CLM-24001.
- Sam Okafor (fully insured, `coverage_valid = true`, no coverage gap) exists solely to catch a decision-logic bug that offers a device bundle regardless of whether an actual coverage gap exists. DL-4 must NOT fire for this profile.

### Determinism Mandate (DEC-02)

The comparisons in DL-1 through DL-4 are **not** evaluated by the LLM's judgment at conversation time. They are computed by a **deterministic Python code tool** (`evaluate_claim`) that reads the structured claim fields (`claim_amount`, `coverage_valid`, `total_loss_flag`, `uninsured_device`) from session state, applies the literal comparisons above, and writes the result to a boolean/enum **session variable** (e.g. `escalation_required: true/false`, `cross_sell_eligible: true/false`). A **Handoff Rule** configured on that session variable then performs the routing — forcing a transfer to the Escalation sub-agent when `escalation_required = true`, or to the auto-approve output path when `escalation_required = false` — with no LLM judgment call on the threshold itself.

This is `[CUSTOM TOOL] + [BUILT-IN]`: the Python tool is custom-authored per this spec; the Handoff Rule is a built-in CX Agent Studio primitive for deterministic parent/child routing on session variables.

This is explicitly **NOT** a RAG / Data Store lookup and **NOT** instruction-only LLM judgment. Per Anti-Pattern 1 (research/ARCHITECTURE.md): "is $1,200 > $1,000" must never be answered via probabilistic retrieval or generative inference over unstructured content — RAG is probabilistic retrieval + generation and can misread a number, hallucinate a threshold, or phrase the same fact two different ways from run to run. A Data Store / Vertex AI Search tool is appropriate for unstructured content (e.g., "what does my policy cover" grounding text) but is **never** the mechanism for this threshold decision.

This determinism is what makes the autonomous-vs-human branch reliably repeatable live: the same seeded record must produce the same routing outcome in every rehearsal and every live delivery (N/N), not "usually."

### Governance, Configurability & Audit Trail (DEC-03)

**1. Configurable rationale for the $1,000 threshold (per D-12).** The $1,000 auto-approve ceiling is set at roughly the **cost of a manual adjuster touch** — claims that cost more to hand-process than to simply pay are auto-settled, so the carrier avoids spending more in adjuster labor than the claim itself is worth. This is an **illustrative, configurable business rule**, not a hardcoded universal truth: each carrier tunes the threshold to its own cost-to-serve, claim mix, and risk appetite. The demo narrative and agent instructions must state this rationale out loud (e.g., "this account's policy routes claims under $1,000 with no total-loss flag to auto-approval; thresholds and rules are configurable per line of business") rather than presenting $1,000 as an unexplained magic number.

**2. Human-authorized framing.** The rule set (DL-1 through DL-4, including the $1,000 ceiling) is authored and owned by the carrier's claims leadership — it is a human-authorized policy, not an autonomous judgment the AI invents at runtime. The system **pre-approves within a human-authorized policy/rule set**; it does not "decide" in the sense of exercising independent discretion. Demo narration, agent instructions, and any marketing language built on this spec must **never** say "the AI decides" or "the AI approves the claim" unqualified. The only correct framing is: **"the system applied the carrier's configured rule."**

**3. Visible audit-trail artifact.** Every auto-approve outcome is paired with a reviewable artifact, not a black-box decision: a logged rule reference (which Rule ID fired — e.g. "DL-1" — and which session variable value drove it, e.g. `escalation_required = false` because `claim_amount = $450 ≤ $1,000`) plus the reviewable conversation transcript. This is a concrete on-screen artifact the presenter can surface live ("look, it logs and explains which rule it applied"), turning auditability into a wow moment rather than a compliance landmine. This same audit artifact is the prepared answer to the buyer question "what happens if it's wrong / who's accountable" — the rule reference plus transcript is the auditable record a human reviewer (or regulator) would inspect.

**4. Secondary factors beyond dollar amount.** The branch is not a single naive number. `total_loss_flag` (DL-3) is a second, independent routing input that overrides the dollar threshold in either direction. In production, this same pattern extends to additional secondary factors such as fraud signals and prior-claims history, which would compose with DL-1/DL-2/DL-3 the same way `total_loss_flag` does — as additional deterministic session-variable inputs to the same Handoff Rule mechanism, not new LLM judgment calls.
