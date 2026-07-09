# Phase 1: Foundations - Context

**Gathered:** 2026-07-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 1 produces the four load-bearing sections of the use-case specification that every later section references without re-deriving: (1) the **Demo Narrative/Storyline**, (2) the **Platform Capability Map**, (3) the **Decision Logic** table, and (4) the **Mock Data Appendix**. Requirements NARR-01..04, CAP-01..02, DEC-01..03, DATA-01..04 are fixed by REQUIREMENTS.md/ROADMAP.md. This discussion locked the *creative and content choices* inside those requirements — the concrete scenario, personas, thresholds, language pair, and framing — so the researcher and planner don't invent them.

**This phase writes spec text, not software.** Success = the spec sections are unambiguous, buildable, internally consistent, and traceable.

</domain>

<decisions>
## Implementation Decisions

### Anchor Scenario & Sector (NARR-01, NARR-02, DATA-01)
- **D-01:** Anchor the single white-label run in a **personal electronics / gadget-insurance** claim — a covered **laptop**. This deliberately **reuses the computer-loss/damage FNOL intake + photo-assessment happy path the implementation team already built for V1**, so V2 visibly extends V1 rather than replacing it (makes NARR-04 literal). One insurance line per run; no multi-sector branching on stage.
- **D-02:** Framing is **insurance**, not manufacturer-warranty — deductible, coverage limit, and the ~$1,000 threshold govern the vocabulary of the decision table and mock data. "Repair vs. replace" may appear as narrative flavor, but insurance terms are canonical. Rationale: preserves the insurance-buyer GTM positioning and keeps DEC/DATA requirements meaningful.
- **D-03:** White-label carrier is generic/unbranded; narrative states it generalizes to any personal-lines / device / gadget / contents insurer (and the pattern generalizes to broader P&C FNOL). No per-account branding.
- **D-04:** The FNOL loss event is **accidental damage to a laptop**, seeded on both sides of the auto-approve threshold and run **back-to-back** in the same demo:
  - **Small claim → auto-approves:** cracked screen / cosmetic damage, under $1,000.
  - **Large claim → routes to human assessor:** liquid-damaged / destroyed high-end machine, over $1,000.
  - Both are photo-assessable (reuses V1 multimodal damage assessment) and both trace to named seed records.

### Escalation / High-Risk Flag (DEC-01, UC-09, ADD-01) — SUPERSEDES "injury" wording
- **D-05:** Replace the "injury" escalation trigger (which does not fit a device loss) with a deterministic **"total-loss of a business-critical device with data loss" flag** — a yes/no intake field that **always routes to a human regardless of dollar amount**. This preserves the original intent (a categorical event that escalates independently of the dollar branch, i.e. the analog of "injury = always escalate").
- **D-06:** The **sentiment/empathy add-on (ADD-01)** rides on this same moment — the customer's genuine distress about losing their work/data — a better fit than injury. Agent visibly adapts tone, then escalates to a human.
- **D-07:** Fraud stays where the roadmap already placed it: a rider on the backend-reveal specialist summary (ADD-02), **not** the escalation trigger. Out-of-scope / out-of-domain deflection remains UC-09's other half.
- **Requirement-text impact:** requirement IDs and intent are unchanged; only the illustrative event changes (injury → total-loss/data-loss). Flag for a light ROADMAP/REQUIREMENTS wording refresh at plan time.

### Cross-Sell / Upsell (UC-08, ADD-04) — SUPERSEDES "boat" wording
- **D-08:** Replace the "uninsured-boat bundle" with **"add protection for the customer's other uninsured devices"** (phone, tablet, home electronics / a whole-home device-protection plan). Fits a device customer and makes **ADD-04 personalization** land — seeded profile shows a recently-bought, uninsured phone, so the offer feels earned. Same "cost center → profit center" closing beat.
- **D-09:** Boat was Scott's specific named example, so **"device bundle vs. boat" is flagged as a Scott/Google validation item** in the narrative (consistent with NARR-03's flag-for-validation discipline). Do not assert as settled.

### Language-Switch Pair (NARR-01 wow moment, CAP-01/02, UC-03)
- **D-10:** Scripted mid-demo voice switch is **English ↔ Spanish (US)** — both are **GA** voice variants (not Preview-only), Spanish is the #1 US-market secondary language, and it is the most broadly relatable across Scott's ~18 accounts.
- **D-11:** Any language outside the confirmed audio-to-audio set uses the **captioned-text fallback**. Audio-to-audio support for the EN↔ES pair is still flagged as an **in-console confirmation item** (CAP-02 / open-questions discipline) — press-confirmed, not yet doc-confirmed.

### Decision Logic — Threshold & Triggers (DEC-01, DEC-02, DEC-03)
- **D-12:** Keep the **literal $1,000** auto-approve ceiling. **Stated illustrative/configurable rationale:** set at roughly the cost of a manual adjuster touch — claims that cost more to hand-process than to simply pay are auto-settled; each carrier tunes it to their own cost-to-serve. Framed as a human-authorized, configurable, auditable rule set with a visible audit-trail artifact — never "the AI decides" (DEC-03).
- **D-13:** Decision rows are literal if/then, all routed by a **deterministic Python tool + Handoff Rule on a session variable** (DEC-02 — never RAG or instruction-only LLM judgment):
  - `amount ≤ $1,000 AND coverage valid AND no high-risk flag` → **auto-approve**, issue claim number
  - `amount > $1,000` → **route to human assessor**
  - `total_loss_flag = true` → **route to human** (regardless of amount)
  - `claim resolved AND seeded profile has uninsured devices` → **fire cross-sell**

### Competitive Framing (NARR-03)
- **D-14:** **Light, clearly-flagged** competitive framing — a short callout positioning Google's native audio-to-audio voice + backend claims-processing reveal against **Microsoft Copilot + Nuance**, every claim tagged `[VALIDATE — Scott/Google]`. Not a full head-to-head table (stale/wrong claims are a liability in a live sales asset).

### Claude's Discretion
- Exact persona names, policy numbers, dollar figures (within the small/large split), coverage-clause wording, and the specific seeded "prior interaction" for personalization — to be authored in the Mock Data Appendix, engineered to land on both sides of every threshold and to include over-fire "negative" records (DATA-02). All synthetic PII only (DATA-03).
- Narrative prose/beat wording and the exact ordering of wow moments within the ≤10–15 min script, provided every locked wow moment is sequenced into one coherent story.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements & Roadmap (locked scope for this phase)
- `.planning/REQUIREMENTS.md` — NARR-01..04, CAP-01..02, DEC-01..03, DATA-01..04 definitions and the Out-of-Scope table (anti-scope-creep guardrails).
- `.planning/ROADMAP.md` §"Phase 1: Foundations" — the 5 success criteria that define "TRUE of the spec text."
- `.planning/PROJECT.md` — core value, constraints (mock-data-only, white-label, flow-free, ASAP), "show, don't tell" principle.

### Project Research (grounds the capability map + decision logic)
- `.planning/research/STACK.md` — CX Agent Studio primitive taxonomy; which capabilities are `[BUILT-IN]` / `[CUSTOM TOOL]` / `[MOCK DATA]`; the 28-variant voice-language list; audio-to-audio open question.
- `.planning/research/FEATURES.md` — V1 baseline definition, Scott's 5 V2 must-haves, P2 add-ons, anti-features.
- `.planning/research/ARCHITECTURE.md` — recommended ~10-section spec document structure and the demo runtime composition (root + shallow sub-agents).
- `.planning/research/PITFALLS.md` — hallucinated-dollar-figure, non-determinism, and voice-latency pitfalls that the decision-logic and mock-data sections must pre-empt.
- `.planning/research/SUMMARY.md` — synthesis + the four flagged platform uncertainties.
- `CLAUDE.md` (project root) — full platform capability reference tables with source citations and confidence tags (mirror source of STACK.md, includes doc URLs for CAP-02 citations).

### External docs to cite (for CAP-02 — platform-capability claims)
- The Google Cloud CX Agent Studio doc URLs are enumerated in `CLAUDE.md` §Sources — use those exact URLs for per-capability citations; flag the audio-to-audio + transcript→summary→email items as press-only/open.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **V1 built demo (implementation team):** the computer/laptop-loss FNOL intake + coverage lookup + photo-based damage assessment + claim-number happy path already exists. Phase 1's narrative and mock data are authored to **reuse and extend** this, not replace it. (Asset lives in the implementation team's CX Agent Studio project, not this repo — this repo is spec text only.)

### Established Patterns
- **Spec-authoring project, not code:** all Phase 1 outputs are Markdown spec sections under `.planning/` / the eventual spec document. No runnable code, no tests in the software sense — "acceptance" = properties of the spec text.
- **Determinism discipline:** every dollar figure/coverage term on stage must trace to a named mock-data field via a deterministic tool call; every "must-land" branch routes via Python tool + Handoff Rule, never LLM judgment.

### Integration Points
- Phase 1's four sections are the inputs Phase 2 (Component Architecture) and Phase 3 (Use-Case Specs) reference by name. The Mock Data Appendix and Decision Logic table are the single sources of truth those phases must cite rather than re-derive.

</code_context>

<specifics>
## Specific Ideas

- **Reuse over rebuild:** the user's explicit steer was "can we do computer loss / warranty — the team already built it." The anchor scenario is chosen specifically to minimize implementation lift and make the V1→V2 continuity story concrete.
- **Two claims, back-to-back, same loss type:** cracked-screen (auto-approve) immediately followed by liquid-damaged high-end machine (HITL) — the contrast IS the decision-branch wow moment.
- **English↔Spanish (US)** specifically because both are GA voice variants and Spanish is the highest-value US secondary language for Scott's account base.
- **$1,000 kept literal**, with the "cost of a manual adjuster touch" rationale as the CFO-credible, configurable story.

</specifics>

<deferred>
## Deferred Ideas

- **Boat / marine cross-sell** — Scott's original named example; deferred in favor of the device-bundle cross-sell for narrative fit, but retained as a flagged Scott-validation item in case he wants it restored. Not lost.
- **Additional language pairs (Hindi/Hinglish, French-Canada, etc.)** — kept out of the scripted live moment; available as captioned-text fallback or future account-specific variants. Belongs to per-account tailoring, not the generic Phase 1 narrative.
- **Heavier competitive head-to-head table vs. Copilot/Nuance** — deferred; only the light flagged callout is in scope for Phase 1.
- V2/DEF items (cross-channel continuity, proactive outreach, Agent Assist, live-metric overlay) remain out of scope per REQUIREMENTS.md.

</deferred>

---

*Phase: 1-Foundations*
*Context gathered: 2026-07-09*
