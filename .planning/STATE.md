---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: executing
stopped_at: Phase 2 context gathered
last_updated: "2026-08-10T22:10:00.000Z"
last_activity: "2026-08-10 - Phase 5: the decision turn now SPEAKS the rule explanation. Chat v9 160dc3b2 (now served by d7bfbb93) sets claim_decision_card to LLM_GENERATED, maps textResponse <- resolve_claim.explanation deterministically, and resolves claim_intake's WORD-FOR-WORD vs say-only-the-short-human-part contradiction in favour of verbatim with a single-source sentinel. Proven live: byte-identical to the tool's own string, emitted EXACTLY ONCE, card/tariff/single-send-email all unregressed. On-screen render of the spoken line, the cross-sell buttons and the photo contradiction path remain open"
progress:
  total_phases: 5
  completed_phases: 1
  total_plans: 7
  completed_plans: 5
  percent: 20
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-07-08)

**Core value:** A specification concrete and compelling enough that the implementation team can build a show-don't-tell demo that lands every wow moment — voice + mid-demo language switching, a visible autonomous-vs-human decision branch, the backend claims-processing reveal, and the cross-sell "cost center → profit center" moment.
**Current focus:** Phase 01 — foundations

## Current Position

Phase: 5 (active) — Phase 2 still unstarted
Plan: 05-01 complete; 05-02 in progress (decision-card rendering VERIFIED LIVE through the real widget embed on chat v7 `bb14cdcc` (2026-08-09); cover_offer_actions FIXED and FIRED for the first time on chat v8 `3f85b1d8` (2026-08-10) — payload proven live, on-screen render still unconfirmed; the silent decision turn FIXED on chat v9 `160dc3b2` (2026-08-10) — the explanation is now spoken byte-identically and exactly once, on-screen render still unconfirmed; wrong-subject guard resolved by removal 2026-08-06); 05-03/04 outstanding
Status: Build work running ahead of the spec phases. Phases 2-4 author specification text and remain unstarted; Phase 5 changes the deployed agent and is where activity currently is.
Last activity: 2026-08-10 - Phase 5: the decision turn now SPEAKS the rule explanation. Chat v9 160dc3b2 (now served by d7bfbb93) sets claim_decision_card to LLM_GENERATED, maps textResponse <- resolve_claim.explanation deterministically, and resolves claim_intake's WORD-FOR-WORD vs say-only-the-short-human-part contradiction in favour of verbatim with a single-source sentinel. Proven live: byte-identical to the tool's own string, emitted EXACTLY ONCE, card/tariff/single-send-email all unregressed. On-screen render of the spoken line, the cross-sell buttons and the photo contradiction path remain open

Progress: [██████████] 100%

## Performance Metrics

**Velocity:**

- Total plans completed: 5
- Average duration: - min
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 01 | 5 | - | - |

**Recent Trend:**

- Last 5 plans: -
- Trend: -

*Updated after each plan completion*
| Phase 01-foundations P05 | 2min | 3 tasks | 3 files |

## Accumulated Context

### Roadmap Evolution

- Phase 5 added: Demo Build - Multimodal, Backend Reveal & Multilingual (build phase, not spec authoring)

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Roadmap: Deliverable is a spec, not code — phase success criteria describe properties of spec text (unambiguous, buildable, internally consistent), not running-software behaviors.
- Roadmap: 4-phase structure adopted directly from research/ARCHITECTURE.md's validated authoring-order dependency chain (Foundations → Component Architecture → Use-Case Specs → Runbook & Synthesis).
- [Phase 01-foundations]: Gap closure (01-05): inserted D-03/D-05/D-07/D-08/D-12/D-13 citation tokens at heading/field-row anchors matching existing Section 2/4 citation convention; no spec content rewritten

### Pending Todos

None yet.

### Blockers/Concerns

- Phase 1: Audio-to-audio translation language set for mid-demo voice language switching is press-sourced only, not confirmed in current CX Agent Studio technical docs — needs console/stakeholder confirmation before the narrative commits to specific language pairs (see CAP-02, DATA-03, UC-03).
- Phase 1: Transcript → specialist-summary → drafted-email mechanics has no confirmed off-the-shelf Gemini Enterprise pipeline — must be spec'd as a custom in-session sub-agent chain, not an assumed built-in feature (see CAP-02, UC-07).
- Phase 2: Exact Handoff Rule UI/syntax and Python-tool session-variable mechanics are MEDIUM-confidence (docs shift on a 6-month-old product) — flag for live-console verification early in Phase 2 (see DEC-02, ARCH work).
- GTM: Competitive framing vs. Microsoft Copilot/Nuance is LOW-MEDIUM confidence — treat as stakeholder input (Scott/Srini/Stephanie/Hallie) to validate, not asserted fact (see NARR-03).
- Demo build: the customer-facing "FNOL Demo Narrative - V2.docx" describes a build that does not exist — it promises chat intake, photo upload, an EN↔ES switch, a $100 deductible, a $3,500 limit, a $1,000 threshold and $450/$2,400 claim amounts. The delivered agent is voice-only with a diagnostic tree, $25/$3,000/50%-of-coverage and $840/$3,000. Reconcile before the document goes to any customer or implementation team (Phase 5 scope).
- Demo build: the implemented PoC diverges from the Phase 1 Mock Data Appendix (§4) — tariff-based pricing, a 50%-of-coverage auto-approve threshold, $25 excess and no photo-upload path. §4's $1,000 threshold, $100 deductible and $3,500 limit are stale against what a caller actually hears. Reconcile §4 (or mark it superseded) before the spec ships to any other implementation team.
- Phase 5 (RESOLVED 2026-08-05): the model's vision accuracy on a real photo is confirmed — a cracked-screen image was read correctly in the simulator ("I can see several cracks across the screen"), confirmed, and auto-approved at $840. Plan 05-02 is unblocked. Original concern: Every branch of `assess_screen_crack` is tested, but only by supplying the observation values directly — the API can be driven with text, not with a convincing photo of a cracked screen. If it misreads a demo photo, the confirm path silently becomes the contradiction path in front of an audience. One upload through the deployed widget (`d7bfbb93`) settles it; blocks plan 05-02.
- Phase 5 ACCEPTED LIMITATION (resolved 2026-08-06, quick task 260806-u21): photo assessment cannot verify WHICH device is in the image. History: v4 asked the model for the device category, v5 asked for physical features (keyboard visible, hinged lid) and inferred in code. Both failed live: against PDP100746 (iPhone) the model reported keyboard_visible=no, hinged_lid=no and "a black phone" for an unmistakable MacBook — see evidence-laptop-photo.png in the phase dir, decoded from the conversation record. Root cause: the model knows what the policy covers and conforms its report to that, not to the pixels. Damage reporting is accurate; only identity is contaminated. DECISION (user): option (b) — a guard that does not work is worse than none, because it reads as protection in both the code and the runbook. Option (a) (agent-as-tool vision isolation) was not attempted. Removed in chat v6 `56a8b22a`, now served by deployment `d7bfbb93`: the `keyboard_visible`/`hinged_lid` parameters, the device-inference block, the `WRONG_SUBJECT` branch and the `photo_device_seen` variable are all gone; `assess_screen_crack(crack_visible, what_you_see)` asks about damage only. STANDING DEMO CONSTRAINT: photo assessment confirms whether the reported damage is visible; it cannot verify that the photographed device is the insured device — run demos with matching device/policy pairs (PDP100294 = MacBook, use a laptop photo). Unchanged and still working: the contradiction path (crack reported, none visible → photo_contradiction, DL-5, human review, non-accusatory), the one-retry-then-human path for unusable photos, and deterministic tariff pricing. A junk or subject-less photo now lands in unclear → retry → human rather than a rejection, the same end state. The photo CONFIRM path is now verified live through the real widget on chat v7 `bb14cdcc` (2026-08-09, quick task 260809-nt7): a real cracked-MacBook photo uploaded through the browser, read correctly ("I can see the cracks on the screen there."), priced from the tariff at $840 and auto-approved as `CLM-24442`. The CONTRADICTION path remains NOT VERIFIED LIVE, proven offline in phototest2.py only, and still needs a photo of an undamaged device through the widget.
- Phase 5 VERIFIED LIVE (2026-08-09, quick tasks 260809-n1b diagnosis + fix, 260809-nt7 verification): widget rendering. ROOT CAUSE: the `claim_decision_card` tool declared seven flat claim fields (`issue_label`, `claim_amount`, `deductible`, ...) as its `parameters`, and a widget tool's parameters ARE its emitted payload. The deployed web-widget SDK's `order_summary` renderer reads only `productItem`, `costBreakdown`, `paymentMethod` and `actions`, found all four undefined, and drew nothing — so the raw payload is what appeared. NOT an SDK limitation, NOT an agent defect, and NOT the console-Preview limitation it was originally attributed to. CORRECTION to the previous entry's claim that "payload shape is confirmed correct": it was well-FORMED (the platform tags it `"type": "order_summary"`) which is NOT the same as matching the renderer's contract — that inference is what kept the real cause hidden for a week. FIXED in chat v7 `bb14cdcc-d723-4be1-85af-9f4451e22ed5`, now served by deployment `d7bfbb93`: the card is a single `productItem` row (title/subtitle/price/imageUri), with `costBreakdown` deliberately dropped (its hardcoded, unsuppressable "Sales tax" row would print $0.00 on an insurance claim) and `actions` deliberately dropped (its labels are live buttons wired to sendQuery, so a presenter tapping one would inject it into the conversation). The decision and the $1,500 cutoff moved into the subtitle. VERIFIED LIVE (n1b): the emitted payload is byte-identical to what `resolve_claim` computes in Python, with `units` as an int — the model supplies no figure. The full SDK contract is written down in 05-02-SUMMARY.md and is reusable for every future widget. VERIFICATION RECORD (2026-08-09, quick task 260809-nt7): the user drove a full claim through the real, console-generated chat-messenger SDK v1.16 widget embed at `http://localhost:3000` with a token broker, against deployment `d7bfbb93` serving chat v7 `bb14cdcc`, and the card RENDERED. It drew title `Apple MacBook Pro 16"` with right-aligned price `$840.00`, and subtitle `Screen replacement · CLM-24442 · $840 less $25 excess = $815 to you · Approved on the spot (on-the-spot limit $1,500)`. The two predicted cosmetic artifacts appeared (an empty grey image tile at left, two divider rules with nothing after them) and none of the warned-of failures did (no `$4,200.00`, no `$NaN`, no `$0.00` row, no broken-image glyph). The run was the photo CONFIRM path with a matching device/policy pair (Jordan Rivera / PDP100294 / MacBook). The SDK `order_summary` contract in 05-02-SUMMARY.md is now verified-by-observation, not only read-from-source.
- Phase 5 (RESOLVED 2026-08-09, quick task 260809-nt7): browser file upload through the web widget needs no Cloud Storage bucket configuration. The `enable-file-upload` flag on the console-generated v1.16 embed was sufficient on its own — a real photo uploaded, displayed inline in the conversation and reached the model, with no storage config anywhere. An image-upload error seen earlier in the same session was a race against the in-flight redeploy, not a defect. This was the first time the photo path had ever been exercised through the real embed rather than the API or the simulator.
- Phase 5 FIXED AND FIRED (2026-08-10, quick task 260810-ifr, chat v8 `3f85b1d8-4810-44eb-85e6-39adc42593c9` now served by `d7bfbb93`): `cover_offer_actions`. It WAS broken the same way as the decision card — it declared `{prompt, options[{label,value}]}` while the SDK's `quick_actions` builder reads `{actions: [{content, description?, utterance?}]}`, and `b.actions.map(...)` throws on undefined rather than degrading to a JSON blob, so nothing would have appeared at all. THE KEY FINDING, and it corrects a week-old assumption: **it had never fired because it had never been REACHED, not because it was miswired.** All three prerequisites were already correct — the tool was attached to `claim_intake`, `uninsured_device` was a declared variable written by `verify_identity`, and the instruction had an explicit Cross-sell step naming the tool. Zero of 14 conversations had ever taken a turn PAST the email confirmation on the auto-approve path, which is structurally what the offer requires; the one that got closest (the user's 2026-08-09 run) ended on the email turn. The payload defect was real but entirely latent. Three further instruction defects were fixed at the same time, because a correct payload alone would still have left the buttons dead: (1) `end_session` was instructed in the SAME turn as the widget, so the session would end before anyone could tap — a new `<step name="Close after the offer">` now owns `end_session` and runs only after the customer answers; (2) "the buttons ARE the question" was provably false — the sentence lived in the `prompt` parameter, which renders nowhere, so following the instruction produced two naked buttons with no question; the agent now says the offer aloud; (3) `textResponseConfig: NONE` (= "the LLM decides whether to emit text") let the sentence vanish, so it is now `LLM_GENERATED` ON THIS WIDGET ONLY — the decision card's `NONE` was deliberately left untouched as the visually-verified known-good baseline. Mechanism note: `LLM_GENERATED` adds a REQUIRED `textResponse` parameter to the tool's input schema, described by `textResponseInstruction` — that is what makes the sentence unskippable. VERIFIED LIVE on session `f9c3999f-2c0d-4908-8a1d-c22971547bb4` through deployment `d7bfbb93`: the tool fired for the first time ever, emitting exactly `{actions:[{content:"Add it",utterance:"Yes please, add it to my cover."},{content:"Not now",utterance:"Not right now, thanks."}]}` and nothing else, alongside the agent text *"One more thing while I have you: your Apple iPhone 16 Pro Max isn't on this policy. Would you like me to add it to your cover?"*, with NO same-turn `end_session`; sending the button's own utterance then produced a warm close plus `end_session` one turn later, with no price quoted anywhere. The decision card, tariff pricing and single-send email were all unregressed on the same run. STILL PENDING: the buttons' on-screen render is NOT confirmed by eye, and the conversation record shows the offer sentence emitted TWICE (once as the model's own text chunk, once as the widget's delivered `textResponse`) — whether that doubles up in the browser is not determinable from the record and needs a human at `http://localhost:3000`. If it does, the fix is to stop instructing the agent to say the offer separately and let the mandatory `textResponse` be the only source.
- Phase 5 STANDING WARNING (2026-08-10, re-stated after quick task 260810-k5x): the chat app `a2f621e4` was last modified by DIRECT `apps.tools.patch` / `apps.agents.patch`, not by export→import. **Every exported package on disk is stale**, including `chat-fresh-k5x.zip`, which was taken before the k5x patches landed. Any future export→edit→import task must take a fresh `exportApp` first or it will silently revert the `cover_offer_actions` schema, the `claim_decision_card` `textResponseConfig`/`dataMapping`, and the `claim_intake` cross-sell AND decision-announcement instructions. Current live version: chat v9 `160dc3b2-571c-480f-b901-e4dbe8947f70`.
- Phase 5 FIXED (2026-08-10, quick task 260810-k5x, chat v9 `160dc3b2-571c-480f-b901-e4dbe8947f70` now served by `d7bfbb93`, read back not inferred): **the silent decision turn.** Previously the agent drew the card and said nothing (chat v7) or said a figure-free paraphrase (chat v8) — non-deterministic, so a presenter could not depend on it, and the rule-and-why lived only in the card's small grey subtitle. TWO causes, both fixed together. (1) `claim_decision_card` had `textResponseConfig: NONE`, which per the CES discovery doc means "the LLM dynamically decides whether to generate a text response", NOT "no text" — the sentence went into the `summary` parameter, which renders nowhere. Now `LLM_GENERATED` with a `textResponseInstruction` pinning it to a word-for-word reproduction. (2) `claim_intake`'s decision-announcement block was self-contradictory — read the explanation WORD FOR WORD, then "do NOT repeat them in your own message - say the short human part only". **Resolved in favour of verbatim** per the user's locked decision; the contradicting clauses were DELETED and a greppable sentinel added: `THE CARD TEXT RESPONSE IS THE ONLY PLACE YOU SAY THE DECISION - do not also send a message of your own in this turn, or the customer hears it twice.` Instruction 18,310 → 18,496 chars, exactly one contiguous region changed. **THE MECHANISM THAT MATTERS:** the `dataMapping` `FIELD_MAPPING` entry `"textResponse": "explanation"` was **ACCEPTED by the API** — so the platform copies `resolve_claim`'s string into the widget's required text-response parameter and the model composes nothing. That is deterministic, not prompt-dependent. VERIFIED LIVE on session `dda81738-832f-455d-bc91-42f7abb3e308`, run against the DRAFT APP before any repoint (a `runSession` with no `config.deployment` is accepted, so the demo rig was never exposed to an unverified build): the spoken line was *"Good news - that keyboard replacement comes to $420, which is under the $1,500 I can approve on the spot, so I can approve that for you right now. Your excess is $25, and your reference is CLM-24005."* — **byte-identical** to `resolve_claim`'s `explanation`, and emitted **EXACTLY ONCE**. The `260810-ifr` duplication trap did NOT reproduce: the decision turn holds two text chunks total (the customer's own message, and one widget-delivered chunk), and the model emitted no text chunk of its own. The repoint was made CONDITIONAL on that count being 1. Unregressed on the same run: `productItem` byte-identical to `card_product_item` with `price.units` an int 420, `cover_offer_actions` byte-identical, tariff $420/$25/$1,500, `send_claim_email` in exactly one turn, 8 tools / 37 variables. STILL PENDING: nobody has watched the spoken line draw in a browser — specifically whether it prints once or doubles above the card. Also note the `summary` parameter is still filled and still renders nowhere; harmless.
- Phase 5 PROCESS NOTE (2026-08-10, quick task 260810-k5x): two executor sessions died from an API transport error while composing the 18,310-char `claim_intake` instruction **inline in tool-call arguments**. Volume of generated text was the cause, not the edit itself. The technique that worked: a Python script written to the scratchpad that GETs the agent, does an exact `str.replace()` of only the small contradictory region, asserts (unique anchors, six content assertions, ±1,200-char delta bound), PATCHes and reads back byte-for-byte — printing only lengths, booleans and short excerpts. **Anyone editing a large instruction on this project should do it by scripted replacement, never by re-typing the instruction inline.**
- Phase 5 (resolved): CES quota is the binding constraint on testing. `RunSession LLM tokens` defaults to 1,000/min per project/region/model while a single conversation costs 120k-150k input tokens (3.5k-10.8k per turn). Raise it in IAM → Quotas before any demo; also `Concurrent BidiRunSession Operations` (10 per 30 min) if more than one person will use the phone.
- Demo build: the claim email can only reach ONE mailbox. Resend's shared `onboarding@resend.dev` sender delivers only to the address owning the Resend account (`akash.vinayak@nerdery.com`) — confirmed by a 403 from Resend when tested against another address. The phone deployment is pinned to v11 `b17c9a26`, which mails that address; both paths verified live. Sending to a second recipient requires verifying a domain at resend.com/domains and changing the `from` address.

### Quick Tasks Completed

| # | Description | Date | Commit | Directory |
|---|-------------|------|--------|-----------|
| 260805-aze | Create presenter demo runbook for deployed Meridian voice claim agent | 2026-08-05 | (see log) | [260805-aze-create-presenter-demo-runbook-for-deploy](./quick/260805-aze-create-presenter-demo-runbook-for-deploy/) |
| 260806-u21 | Remove wrong-subject photo guard from chat app; document the accepted limitation | 2026-08-06 | (see log) | [260806-u21-remove-wrong-subject-photo-guard-from-ch](./quick/260806-u21-remove-wrong-subject-photo-guard-from-ch/) |
| 260809-n1b | Reshape claim_decision_card to the SDK order_summary contract (chat v7) | 2026-08-09 | (see log) | [260809-n1b-reshape-claim-decision-card-to-the-sdk-o](./quick/260809-n1b-reshape-claim-decision-card-to-the-sdk-o/) |
| 260809-nt7 | Record live verification of decision-card rendering and photo upload through the real web widget | 2026-08-09 | (see log) | [260809-nt7-record-live-verification-of-decision-car](./quick/260809-nt7-record-live-verification-of-decision-car/) |
| 260810-ifr | Fix cover_offer_actions to match the SDK quick_actions contract; cross-sell fires for the first time (chat v8) | 2026-08-10 | (see log) | [260810-ifr-fix-cover-offer-actions-to-match-the-sdk](./quick/260810-ifr-fix-cover-offer-actions-to-match-the-sdk/) |
| 260810-k5x | Make the agent speak the decision explanation verbatim, exactly once (chat v9) | 2026-08-10 | (see log) | [260810-k5x-make-the-agent-speak-the-decision-explan](./quick/260810-k5x-make-the-agent-speak-the-decision-explan/) |

## Deferred Items

Items acknowledged and carried forward from previous milestone close:

| Category | Item | Status | Deferred At |
|----------|------|--------|-------------|
| v2 | DEF-01 Cross-channel context continuity (chat→voice resume) | Deferred | Requirements definition |
| v2 | DEF-02 Proactive/catastrophe-triggered outbound outreach | Deferred | Requirements definition |
| v2 | DEF-03 Agent Assist | Deferred | Requirements definition |
| v2 | DEF-04 Speed/efficiency live metric overlay | Deferred | Requirements definition |

## Session Continuity

Last session: 2026-07-09T23:03:54.209Z
Stopped at: Phase 2 context gathered
Resume file: .planning/phases/02-component-architecture/02-CONTEXT.md
