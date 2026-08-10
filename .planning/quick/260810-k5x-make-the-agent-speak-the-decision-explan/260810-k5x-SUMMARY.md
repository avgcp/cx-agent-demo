---
quick_id: 260810-k5x
type: quick
status: complete
date: 2026-08-10
subsystem: cx-agent-chat-app
tags: [widget, rich-response, order-summary, decision-turn, verbatim, data-mapping, ces, sdk-contract]
server_changes:
  app: a2f621e4-9faf-505a-b804-22471f022366 (Meridian Claim - Chat (hardened))
  new_version: 160dc3b2-571c-480f-b901-e4dbe8947f70 (chat v9 - claim_decision_card speaks resolve_claim's explanation verbatim; claim_intake decision-announcement contradiction resolved, card text response is the single source)
  previous_version: 3f85b1d8-4810-44eb-85e6-39adc42593c9 (chat v8)
  deployment_repointed: d7bfbb93-8cee-43fe-9095-bc5775f353bd -> 160dc3b2 (read back, not inferred; conditional on the duplication count being exactly 1)
  mechanism: direct apps.tools.patch + apps.agents.patch (NO importApp)
  untouched: [6e01e4a5-42a8-5213-b3da-c9053ff8ea52 (voice, v11 b17c9a26), 9ae7a0c3-6511-413c-8cdb-0efe9e90d2b9 (fork), cover_offer_actions (byte-identical)]
files_modified:
  - .planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md
  - .planning/STATE.md
  - .planning/spec/DEMO-RUNBOOK.md
---

# 260810-k5x — Make the agent speak the decision explanation

**The agent now says the decision out loud, in the rule engine's own words, exactly once.**

At the moment of approval it states the amount, the limit it is compared against and why the
claim qualifies — reproduced **byte-for-byte** from `resolve_claim`'s computed `explanation`,
not composed by the model. The card draws below it with the same facts, deliberately, as
reinforcement.

Shipped as **chat v9 `160dc3b2-571c-480f-b901-e4dbe8947f70`**, now served by `d7bfbb93`.
**Only the on-screen render is unconfirmed** — that needs a human at `http://localhost:3000`.

## Answers to the specific questions asked

| | |
|---|---|
| **New version ID** | `160dc3b2-571c-480f-b901-e4dbe8947f70` — *"chat v9 - claim_decision_card speaks resolve_claim's explanation verbatim; claim_intake decision-announcement contradiction resolved, card text response is the single source"* |
| **Which mechanism carried the string** | **The deterministic `dataMapping` field mapping.** `"textResponse": "explanation"` was **ACCEPTED** by the API on the first attempt — no fallback was needed. The platform copies the string; the model composes nothing. `textResponseInstruction` is still in place as a second line of defence, but it is not what carried the text. |
| **The spoken line, verbatim** | *"Good news - that keyboard replacement comes to $420, which is under the $1,500 I can approve on the spot, so I can approve that for you right now. Your excess is $25, and your reference is CLM-24005."* |
| **Byte-identical to `resolve_claim`'s `explanation`?** | **Yes.** Not a degraded pass, not a paraphrase. Every figure originates with the pricing tool. |
| **How many times did it appear in the decision turn?** | **Exactly once.** PASS. |
| **Was `d7bfbb93` repointed?** | **Yes, and read back rather than inferred.** `3f85b1d8` (v8) → `160dc3b2` (v9), with `channelType` `WEB_UI` and `webWidgetConfig` byte-unchanged. |
| **Did the card still work?** | **Yes.** Emitted `productItem` byte-identical to `resolve_claim`'s `card_product_item`, `price.units` a JSON **int** `420`, `nanos` 0, `USD`, gstatic `imageUri`. |
| **Did the cross-sell still work?** | `cover_offer_actions`'s `widgetTool` is **byte-identical** before and after, and present in the v9 snapshot still declaring only `actions`. It was **not exercised on this run** — the run stopped at the email turn by plan, to conserve quota. |

## The duplication check — the headline result

`260810-ifr` found that `LLM_GENERATED` on `cover_offer_actions` emitted the offer sentence
**twice**: once as the model's own text chunk, once as the widget's delivered `textResponse`.
That trap was the whole reason this task's repoint was made conditional.

**It did not reproduce.** The decision turn contains **two text chunks in total**:

| Message | Source | Content |
|---|---|---|
| `msg0` | the customer's own turn | the fault description |
| `msg12` | **widget-delivered** (the message carrying `payload`) | the explanation |

**The model emitted no text chunk of its own** — which is precisely what the new sentinel
instructs. The single-source design worked as `260810-ifr` predicted it would.

The `summary` parameter was still filled — *"Your claim is approved. The keyboard replacement
costs $420, minus your $25 excess, leaving $395 for you. The reference is CLM-24005."* — and
still renders nowhere. Harmless, recorded for completeness.

## What was wrong, and what shipped

**Two causes, fixed together.** Fixing either alone would have left the beat unreliable.

1. **`claim_decision_card.textResponseConfig` was `{type: NONE}`** — which per the CES
   discovery document means *"the LLM dynamically decides whether to generate a text
   response"*, **not** "no text". The model routed its sentence into the `summary` parameter,
   which renders nowhere, and was free to stay silent. It was silent on 2026-08-09 and said a
   figure-free paraphrase on 2026-08-10. Non-deterministic, so undemoable.
   → Now `LLM_GENERATED`, with a `textResponseInstruction` requiring a word-for-word
   reproduction, plus the `textResponse → explanation` field mapping that makes it moot.

2. **`claim_intake`'s decision-announcement block contradicted itself.** It said read the
   `explanation` **WORD FOR WORD**, and then, ~900 characters later, *"Because the card
   carries the figures, do NOT repeat them in your own message - say the short human part only
   ('Good news, I can approve that right now')"*.
   → **Resolved in favour of verbatim**, per the user's locked decision. The contradicting
   clauses were **deleted**, not softened: *"do NOT repeat them in your own message"*, *"short
   human part only"*, *"Never send the card without a sentence"*, *"never send the figures
   twice"*, and *"The card's summary line is what you would naturally say"* are all gone.

**The single-source guard**, carried verbatim so it can be grepped:

> `THE CARD TEXT RESPONSE IS THE ONLY PLACE YOU SAY THE DECISION - do not also send a message of your own in this turn, or the customer hears it twice.`

The replacement also made the block **branch-agnostic** — the explanation is read as written
whether it approves, escalates, or deliberately withholds a payout figure on the
`photo_blocked` branch — and reinforced the no-invention rule (*"If a number is not in the
explanation, you do not say it"*). The *"never refer to the card itself"* hardening was
preserved.

Instruction **18,310 → 18,496 chars (+186)**, exactly one contiguous region changed.

## Deviations

**[Rule 3 — blocking, process] Two prior executor sessions died at the same step.** Both
crashed from an API transport error (connection closed mid-response) while composing the
18,310-char instruction **inline in tool-call arguments**. The volume of generated text was
the cause, not the edit.

**The mitigation, worth reusing on this project:** do the edit with a Python script written to
the scratchpad and executed, which GETs the agent, performs an exact `str.replace()` of only
the small contradictory region, asserts, PATCHes and reads back — with every large string kept
in files and only lengths, booleans and short excerpts ever printed. The only text authored
inline was the ~1.7 KB replacement block, written to a file by heredoc. **Anyone editing a
large instruction on this project should work this way.**

**Task 1's tool patch had landed before the first crash.** It was **verified read-only, not
reapplied** — a second `updateMask=widgetTool` write is the main way the visually-verified
card could have been destroyed, since that mask replaces the whole object. Confirmed intact:
`textResponseConfig.type` `LLM_GENERATED`, `dataMapping.fieldMappings`
`{"productItem": "card_product_item", "textResponse": "explanation"}`, mode `FIELD_MAPPING`,
`parameters` byte-identical to the baseline.

**[Rule 1 — bug, in this task's own tooling] The first assertion script read `productItem`
from the wrong place** and reported canaries 7a–7c as FAIL. Under `FIELD_MAPPING` the card's
tool-call arguments carry `payload`/`summary`/`textResponse`, with `productItem` nested inside
`payload` — not at the top level. Re-checked against the correct path: all three PASS. No
server state was involved and nothing was changed to make them pass.

**The draft app accepted a `runSession` with no `config.deployment`.** The plan's fallback
(repoint first, run through the deployment) was therefore **not needed**, and the mandatory
rollback-on-failure clause never became live. The demo rig was never exposed to an unverified
build.

**No 429 occurred**, so the authorised single spaced retry was never used. Turns were paced
~55 s apart. **One conversation, three turns.**

**No screen-path drift.** The precise fault description recommended by `260810-ifr` (physical
keys broken, screen explicitly fine and undamaged, no liquid) priced the keyboard correctly on
the first attempt — no photo was demanded, and none of the four corrective turns `260810-ifr`
needed were required here. That is a useful confirmation of the drift mitigation.

## Verification evidence

| Check | Result |
|---|---|
| Task 1 tool patch (pre-existing) verified read-only, not reapplied | `LLM_GENERATED` ✅, `textResponse → explanation` mapping ✅, `parameters` unchanged ✅ |
| Live instruction matched the on-disk snapshot before editing | ✅ 18,310 chars, sentinel absent |
| Anchor uniqueness gate | START and END anchors each occur **exactly once** in 18,310 chars |
| Six pre-PATCH assertions | **ALL PASS** — sentinel ×1; all five contradicting clauses removed; `WORD FOR WORD` survives; `<step name="Cross-sell">` ×1 plus `Close after the offer`, `Do NOT end the session in this turn`, `Skip this step entirely on the HUMAN_REVIEW path`, `cover_offer_actions`, `claim_decision_card`, `send_claim_email`, `end_session`, `assess_screen_crack`, `</taskflow>` all present; exactly one contiguous region changed; delta **+186** (bound ±1,200) |
| Agent read back after PATCH | **Byte-for-byte** equal to what was sent; `validationErrors` absent; 8 tools unchanged; `childAgents` and `transferRules` unchanged |
| v9 snapshot audit | Both widget tools present; card `LLM_GENERATED`; card `parameters` match baseline; both `dataMapping` entries present; `cover_offer_actions` declares `actions`, no `prompt`/`options`; snapshot contains the sentinel and `Close after the offer` |
| Deployment held at v8 until the run passed | ✅ asserted `3f85b1d8` before the conversation |
| Live: `textResponse` byte-identical to `explanation` | ✅ **PASS** (199 chars, exact) |
| Live: duplication count | ✅ **exactly 1** (exact match and normalised near-match both = 1) |
| Live: card `productItem` | ✅ byte-identical to `card_product_item`; `units` int `420`, `nanos` 0, `USD`, gstatic `imageUri`, `title`/`subtitle` present |
| Live: deterministic tariff | ✅ $420 keyboard / $25 excess / $1,500 cutoff / `CLM-24005` |
| Live: single-send email | ✅ `send_claim_email` fired in **exactly one** turn |
| Deployment re-read after repoint | ✅ serving `160dc3b2`; `WEB_UI`; `CHAT_ONLY`; `webWidgetConfig` unchanged |
| Voice app `6e01e4a5` / fork `9ae7a0c3` | **No API call of any kind issued.** Every mutating URL was gated on the literal `a2f621e4` in-script, with `6e01e4a5`/`9ae7a0c3` explicitly refused. |

**Secret handling:** `resolve_claim`'s source was never read, edited, echoed or copied. Its
`explanation` was pinned from an existing conversation record's `toolResponse` — claim data,
not source. Tool listings were filtered by `displayName` in-script and never printed whole. No
key appears in any file under `.planning/`.

## ⚠ The exported package on disk is STALE

This task used **direct `apps.tools.patch` and `apps.agents.patch`** — `importApp` was never
called. **Every exported zip in every scratchpad predates these edits, including
`chat-fresh-k5x.zip`**, which was taken as the pre-edit gate *before* the patches landed.

**Any future export→edit→import task MUST take a fresh `exportApp` first**, or it will
silently revert the `cover_offer_actions` schema, the `claim_decision_card`
`textResponseConfig` and `dataMapping`, and the `claim_intake` cross-sell **and**
decision-announcement instructions. Current live version: **chat v9
`160dc3b2-571c-480f-b901-e4dbe8947f70`**.

## What the user should check at http://localhost:3000

The rig is already running — **do not rebuild it**. **Reload the page** so the widget picks up
chat v9.

Run policy **PDP100294**, *Jordan Rivera*, a **keyboard** fault. Be explicit: *"some of the
physical keys have stopped working, the screen is completely fine and undamaged, there's no
liquid, it works normally otherwise."* **No photo needed** — that phrasing keeps it off the
screen-crack path, which would ask for one.

**Expected at the moment of approval:**

1. The agent **says the decision out loud** — naming the amount, the on-the-spot limit it is
   under, and why it qualifies. Roughly: *"Good news - that keyboard replacement comes to $420,
   which is under the $1,500 I can approve on the spot, so I can approve that for you right
   now. Your excess is $25, and your reference is CLM-24xxx."*
2. **The card draws below it** with the same facts in its small grey subtitle.
3. **That overlap is deliberate — the card is reinforcement, not a duplicate.**

### What duplication actually looks like — the thing to look for specifically

**Duplication = the same full sentence printed twice, one after the other, as two separate
agent messages** — one from the model and one attached to the card. That is the known trap
from `260810-ifr` and it is the single most likely cosmetic problem.

**It is NOT duplication** when one spoken sentence is followed by the card's small grey
subtitle repeating the same facts in a different, shorter form. That is the intended design
and you should expect to see it.

The conversation record shows exactly **one** text chunk server-side, so the server side is
clean — but the record cannot tell us what the browser draws.

### Other failure signals worth reporting

- **The agent still says nothing** and the reasoning survives only as subtitle text — the fix
  did not take on screen.
- **The agent speaks but names a figure that differs from the card** — a different amount, a
  rounded number, or a limit other than $1,500. **This is the most serious possible outcome**:
  it would mean the model composed the line instead of copying the rule engine's, which the
  field mapping is supposed to make impossible.
- **The card changed** — a different subtitle, a missing price, `$4,200.00`, `$NaN`, a "Sales
  tax" row, or a broken-image glyph instead of the plain grey tile. The card is the regression
  canary and its definition was deliberately preserved, so any change there is a genuine
  surprise. Expect the empty grey image tile and the two trailing divider rules — those are
  known and fine.

**While you are there**, the cross-sell from `260810-ifr` is still unwatched. If you have the
patience for two more turns — confirm the email, then say *"that's everything, thanks"* — the
`Add it` / `Not now` buttons should appear, and the offer sentence's own possible doubling is
still an open question.

## Artifacts

Session scratchpad:

- `k5x-claim_intake-live.txt` / `k5x-claim_intake-new.txt` — 18,310 → 18,496 chars
- `k5x-replacement.txt` — the authored replacement block; `k5x-region-removed.txt` — what it replaced
- `k5x_splice.py` — the anchor + six-assertion gate; `k5x_patch_agent.py` — the PATCH + byte-for-byte read-back
- `k5x_version.py` / `k5x-ver9.json` — the v9 cut and snapshot audit
- `run-k5x.py`, `k5x-turn-{1,2,3}.json` — the turn driver and raw per-turn responses
- `k5x-conv.json` — the full 3-turn conversation record (the proof)
- `k5x_assert.py` / `k5x_assert7.py` — the assertion battery; `k5x-live-explanation.txt` — the pinned string
- `k5x_repoint.py`, `k5x-dep-before.json` / `k5x-dep-after.json` — the conditional repoint
- `k5x-card-before.json` / `k5x-cover-before.json` — the regression baselines (Task 1)

## Self-Check: PASSED

- `.planning/quick/260810-k5x-…/260810-k5x-SUMMARY.md` — created
- `05-02-SUMMARY.md`, `STATE.md` and `DEMO-RUNBOOK.md` modified and dirty in `git status`
  (uncommitted by instruction; the orchestrator owns the docs commit)
- Every server assertion re-read from the API after the write, never inferred from the request
- No git commit made by this task; ROADMAP.md untouched; `05-02-SUMMARY.md` remains
  `status: incomplete` — outstanding: the spoken line's on-screen render and whether it
  doubles; the `cover_offer_actions` buttons' render and its own possible duplicate offer
  sentence; the photo CONTRADICTION path
