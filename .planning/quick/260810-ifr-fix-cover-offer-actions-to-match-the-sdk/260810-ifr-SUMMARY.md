---
quick_id: 260810-ifr
type: quick
status: complete
date: 2026-08-10
subsystem: cx-agent-chat-app
tags: [widget, rich-response, quick-actions, cross-sell, chat-app, ces, sdk-contract]
server_changes:
  app: a2f621e4-9faf-505a-b804-22471f022366 (Meridian Claim - Chat (hardened))
  new_version: 3f85b1d8-4810-44eb-85e6-39adc42593c9 (chat v8 - cover_offer_actions reshaped to SDK quick_actions contract; cross-sell step says the offer aloud and defers end_session)
  previous_version: bb14cdcc-d723-4be1-85af-9f4451e22ed5 (chat v7)
  deployment_repointed: d7bfbb93-8cee-43fe-9095-bc5775f353bd -> 3f85b1d8 (read back, not inferred)
  mechanism: direct apps.tools.patch + apps.agents.patch (NO importApp)
  untouched: [6e01e4a5-42a8-5213-b3da-c9053ff8ea52 (voice, v11 b17c9a26), 9ae7a0c3-6511-413c-8cdb-0efe9e90d2b9 (fork), claim_decision_card (byte-identical)]
files_modified:
  - .planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md
  - .planning/STATE.md
  - .planning/spec/DEMO-RUNBOOK.md
---

# 260810-ifr — Fix `cover_offer_actions` to the SDK `quick_actions` contract

**The cross-sell fired for the first time ever.** Zero of the 14 conversations on record
before today contained a `cover_offer_actions` call. It now emits exactly the shape the
deployed widget SDK reads, alongside a spoken offer naming the uninsured device, and the
session no longer ends before the buttons can be tapped.

Shipped as **chat v8 `3f85b1d8-4810-44eb-85e6-39adc42593c9`**, served by `d7bfbb93`.
**Only the on-screen render is unconfirmed** — that needs a human at `http://localhost:3000`.

## Answers to the specific questions asked

| | |
|---|---|
| **New version ID** | `3f85b1d8-4810-44eb-85e6-39adc42593c9` — *"chat v8 - cover_offer_actions reshaped to SDK quick_actions contract; cross-sell step says the offer aloud and defers end_session"* |
| **Was `d7bfbb93` repointed?** | **Yes, and read back rather than inferred.** `bb14cdcc` (v7) → `3f85b1d8` (v8); the re-GET shows `appVersion` ending `3f85b1d8`, `updateTime` `2026-08-10T18:45:01Z`, still `WEB_UI` / `CHAT_ONLY`, `webWidgetConfig` unchanged. The live conversation was routed **through the deployment**, which proves the repoint a second way — the conversation record's own `appVersion` field reads `3f85b1d8`. |
| **Did the conversation get past the email?** | **Yes — two turns past it.** Turn 4 was the email (`send_claim_email`), turn 5 was the cross-sell (`cover_offer_actions`), turn 6 was the close (`end_session`). This was the whole point; stopping at the email would have proven nothing. |
| **Did the decision card still render correctly?** | **Yes, on the same run.** `claim_decision_card` fired with a `productItem` carrying `title`, `subtitle`, `price.units` as a JSON **int** `420` (not a string), `nanos: 0`, `currencyCode: USD` and the gstatic `imageUri`. Its tool definition is **byte-identical** to the pre-edit snapshot. |
| **Widget-tool survival — check 1 (pre-edit)** | Fresh `exportApp` taken before any mutation: `claim_decision_card` ✅ and `cover_offer_actions` ✅ both present, 8 tools total. |
| **Widget-tool survival — check 2 (post-edit)** | Both ✅ in the live tool list after both patches, both ✅ in the v8 version snapshot, 8 tools throughout, `variableDeclarations` 37 before and after. |

### The exact emitted `cover_offer_actions` payload

Captured from the conversation record, session `f9c3999f-2c0d-4908-8a1d-c22971547bb4`,
turn 5, policy PDP100294, keyboard fault:

```json
{
  "type": "quick_actions",
  "actions": [
    { "content": "Add it",  "utterance": "Yes please, add it to my cover." },
    { "content": "Not now", "utterance": "Not right now, thanks." }
  ]
}
```

`type` is the platform's own tag. Every assertion passes: the argument keys are exactly
`{actions}` — no `prompt`, no `options`; the array is length 2; every entry has a non-empty
`content` and `utterance`; **no entry carries a `description`** (which would have flipped the
whole set to a grid layout); and the utterances are human sentences, not the bare
`accept`/`decline` tokens the old schema produced.

### The agent's offer sentence, verbatim

> *"One more thing while I have you: your Apple iPhone 16 Pro Max isn't on this policy. Would
> you like me to add it to your cover?"*

Emitted **in the same turn as the buttons**, naming the uninsured device. `end_session` was
**not** called in that turn.

### The close, proven too

Sending `"Yes please, add it to my cover."` — exactly what tapping **Add it** sends — produced:

> *"I'll have someone send the options over. Thanks for chatting, Jordan - have a good day."*

followed by `end_session` (`session_escalated: false`). **No price, premium or monthly cost
was quoted at any point**, which is what the instruction demands since there isn't one.

## Why it had never fired — the finding that matters

**It was not a wiring defect, and that assumption had stood for a week.**

All three prerequisites were already correct: the tool **was** attached to `claim_intake` (the
agent every conversation reaches), `uninsured_device` **was** a declared session variable
written by `verify_identity` before the claim is even priced, and `claim_intake`'s instruction
**did** contain an explicit `<step name="Cross-sell">` naming the tool.

**It had never fired because no conversation had ever taken a turn past the email
confirmation on the auto-approve path.** The instruction forbids bundling the offer onto the
email turn, so the cross-sell *structurally requires one more customer turn after the email*.
Exactly one conversation ever got close — the user's live 2026-08-09 run — and it ended on the
email turn (last customer message *"ok great"*; closed two minutes later). Every other run
stopped earlier: simulator scripts, an API run that hit quota, or the escalation path, which
deliberately has no cross-sell.

So the payload defect was real but entirely **latent** — it would have thrown the instant it
fired, and fixing the payload alone would have changed nothing observable. That is why this
task's live run had to drive past the email, not merely reach it.

## What was actually wrong, and what shipped

**The payload.** The tool declared `{prompt, options[{label, value}]}`. The SDK's builder
reads `{actions: [{content, description?, utterance?}]}` and calls `b.actions.map(...)`
**unguarded** — `actions` absent throws a `TypeError`, `render()` never runs, and **nothing
appears at all**. A *harder* failure than the decision card's, which at least degraded to a
visible JSON blob. Now declares exactly one property, `actions`, required, with `items`
requiring `content` and `utterance` and **no `description` declared at all** (declaring it
would invite the model to fill it and silently switch the layout to a grid).

**Three instruction defects that would have kept it broken anyway:**

1. **`end_session` was instructed in the same turn as the widget** — *"Accept a decline
   gracefully. Then {@TOOL: end_session}."* Obeyed literally, the session ends before the
   customer can tap. **The buttons would have been dead even with a perfect payload.** A new
   `<step name="Close after the offer">` now owns `end_session` and runs only after the answer.
2. **"the buttons ARE the question" was provably false.** The old step forbade asking in text
   and put the sentence in the `prompt` parameter — which renders nowhere. Following the
   instruction produced two naked buttons with no question above them. Inverted: the agent
   says the offer aloud, and the step now states plainly that the buttons cannot ask for it.
3. **`textResponseConfig: NONE`** means *"the LLM dynamically decides whether to generate a
   text response"*, not "no text" — leaving the sentence free to vanish. Set to
   `LLM_GENERATED` **on `cover_offer_actions` only**; the decision card's `NONE` was
   deliberately left untouched as the visually-verified known-good baseline.

**Mechanism worth keeping:** `retrieveToolSchema` confirms `LLM_GENERATED` adds a **required
`textResponse` string parameter** to the tool's input schema, described by the
`textResponseInstruction` verbatim. The model cannot call the tool without composing a
sentence — that is the enforcement, not a hint.

## Deviations

**[Rule 3 — blocking] `POST /versions` returns the version resource directly, not an LRO.**
The n1b precedent polled an operation. Here the POST response *is* the version
(`.../versions/3f85b1d8-...`) with no `done` field, so the poll loop span twelve times against
a resource that would never report `done`. Resolved by reading the version id straight off the
POST response and confirming with a `GET` on the version, which returned the full snapshot
(`displayName`, `createTime`, `snapshot`). No retry or duplicate version was created.

**[Rule 3 — blocking] The live run needed four extra turns to reach the priced claim.** The
plan's turn table assumed *"the keyboard is broken…"* would price immediately. In practice the
agent asked a confirming diagnostic question, then **drifted onto the screen-crack path and
demanded a photo** — which the executor cannot supply, and which is exactly the gate the
keyboard fault was chosen to avoid. The tool record shows why: the second `run_diagnostic`
call was made with `q1: "screen"` despite the customer having said keyboard, returning
`terminal: {outcome: REPAIRABLE, issue: "screen"}`. Two explicit corrections put it back on
`q1: "keyboard"` → `terminal: {outcome: REPAIRABLE, issue: "keyboard"}` and the claim priced
correctly at $420. **This is a demo-relevant fragility, not a defect introduced here** — the
instruction and `run_diagnostic` are untouched by this task — but a presenter who says
"broken" ambiguously can be asked for a photo of an undamaged screen. Recorded here rather
than fixed: it is outside "fix `cover_offer_actions`" and fixing it would need another version
and another verification conversation.

**No 429 occurred, so the authorised single retry was never used.** Turns were paced 45–55 s
apart deliberately. One conversation, ten API turns.

**Task 3's turn table was extended by one turn beyond the plan.** After the assertions passed
I spent one extra turn sending the exact utterance a button tap emits, to prove the new
"Close after the offer" step and confirm `end_session` fires *there* and not in the button
turn. Not required by the plan; it converts assertion 8 from "absent in that turn" into
"present in the right turn", which is the stronger claim.

## New finding, NOT fixed — the offer sentence is emitted twice

The cross-sell turn contains **two identical text chunks**:

| Message | Source |
|---|---|
| `msg1` — text + `toolCall` | the model's own output |
| `msg3` — text + `payload` | the delivered widget message, sourced from the required `textResponse` parameter |

The decision-card turn, on `textResponseConfig: NONE`, emits only one text chunk and the
delivered widget message carries none. So this is a direct consequence of `LLM_GENERATED`
combined with an instruction that *also* tells the agent to say the offer aloud — both produced
the same sentence.

**Whether the browser renders it once or twice is not determinable from the conversation
record**, which is a server-side log, and no prior run provides a comparison (the card turn was
silent on the verified run). It is therefore the **first thing to check on screen**.

**If it does double up, the fix is already known:** drop the "say the offer out loud as your own
message" clause from the Cross-sell step and let the widget's `textResponse` be the sole
source. Because `LLM_GENERATED` makes that parameter **required**, the question cannot go
missing — which is the property the old `prompt`-based design lacked. Not applied here because
it is speculative against a working state, would need another version, another deployment
repoint and another full verification conversation, and one glance at the screen settles it.

## Also observed: the silent decision turn is unreliable, not reliably broken

On this v8 run the agent **did** speak alongside the card — *"I've looked into that, and I can
approve this claim right now."* — with `textResponseConfig` still `NONE` and the instruction
contradiction still unresolved. On the 2026-08-09 run it said nothing. That is consistent with
`NONE` meaning "the LLM decides each time", and downgrades the open defect from *always
silent* to *non-deterministic*. A presenter still cannot depend on it. The runbook's presenter
warning was updated to say "may say nothing" and to keep the read-the-card-aloud instruction.

## Verification evidence

| Check | Result |
|---|---|
| Fresh `exportApp` before any edit (mandatory gate) | Yes — 27-entry package; both widget tools ✅, 8 tools |
| Pre-edit state captured | `claim_decision_card` widgetTool, `cover_offer_actions` etag, `claim_intake` instruction **17,057 chars** (as predicted), 37 variables, `uninsured_device` present |
| `retrieveToolSchema` after the tool patch | Declared params exactly `{actions}`; `prompt`/`options` absent; items require `content`+`utterance`; `description` not declared; `textResponseConfig` `LLM_GENERATED` with non-empty instruction |
| `claim_decision_card` byte-identical | ✅ before vs after, and again inside the v8 snapshot |
| Instruction pre-PATCH string assertions | All 8 passed + length delta **+1,253** (bounded +500…+1,500); single `<step name="Cross-sell">`, replaced block exactly 1,217 chars as predicted |
| Agent after the patch | Instruction read back matches byte-for-byte; `validationErrors` absent; 8 tools, same set; `childAgents`/`transferRules` unchanged; both widget tools still attached |
| v8 snapshot audit | Both widget tools present; `cover_offer_actions` declares only `actions`; no `prompt`/`options`; `LLM_GENERATED`; card identical; instruction contains `Close after the offer` and `Do NOT end the session in this turn` |
| Deployment `d7bfbb93` re-read | Serving `3f85b1d8`; `WEB_UI` / `CHAT_ONLY` / `webWidgetConfig` unchanged |
| Live payload assertions (15) | **ALL PASS** |
| Live regression: decision card | Present, `units` int `420`, `nanos` 0, `USD`, gstatic `imageUri`, subtitle `Keyboard replacement · CLM-24167 · $420 less $25 excess = $395 to you · Approved on the spot (on-the-spot limit $1,500)` |
| Live regression: single-send email | `send_claim_email` fired in **exactly one** turn |
| Live regression: deterministic tariff | $420 keyboard / $25 excess / $1,500 cutoff — unchanged |
| `end_session` timing | Fired in turn 6 only, never in turn 5 (the button turn) |
| Voice app `6e01e4a5` / fork `9ae7a0c3` | **No API call of any kind issued.** Every mutating URL was grepped for the literal `a2f621e4` before sending. |

**Secret handling:** `resolve_claim`'s source was never read, edited, echoed or copied. Tool
listings were filtered by `displayName` in-script and never printed whole; the export package
was inspected by `namelist()` only. No key appears in any file under `.planning/`.

## ⚠ The exported package on disk is now STALE

This task used **direct `apps.tools.patch` and `apps.agents.patch`** — `importApp` was never
called. Every exported zip in every scratchpad predates these edits.

**Any future export→edit→import task MUST take a fresh `exportApp` first**, or it will
silently revert the `cover_offer_actions` schema and the `claim_intake` cross-sell
instruction. This warning is recorded at the top of `05-02-SUMMARY.md` and in `STATE.md`.

## What the user should check at http://localhost:3000

The rig is already running — **do not rebuild it**. If the widget was open before this change,
reload the page so it picks up v8.

Run policy **PDP100294**, *Jordan Rivera*, a **keyboard** fault, *"works normally otherwise"*.
**No photo needed.** Be explicit that it is the keyboard and that the screen is fine — see the
drift deviation above. Accept the decision, then say **"that's everything, thanks"**.

**Expected:**

1. A sentence naming the **Apple iPhone 16 Pro Max** and offering to add it to cover —
   *"One more thing while I have you: your Apple iPhone 16 Pro Max isn't on this policy. Would
   you like me to add it to your cover?"*
2. Directly below it, **two buttons side by side in a single row**, labelled **`Add it`** and
   **`Not now`**.
3. Tapping one **types its sentence into the chat as the customer's own message** — `Add it`
   sends *"Yes please, add it to my cover."* — and the agent closes warmly, offering to have
   the options sent over, **without quoting any price**.

**Failure signals worth reporting:**

- **Nothing at all appears where the buttons should be.** That is the `.map()` throw — the
  whole message dies; it does **not** degrade to a visible JSON blob the way the old card did.
  This is the failure the fix targets, so it should not happen.
- **The offer sentence appears twice**, once above the buttons and once again. This is the new
  finding above and is the single most likely cosmetic problem. **Please look for it
  specifically** — the fix is known and one line.
- **Buttons appear but no question is asked above them** — the `LLM_GENERATED` text did not land.
- **The buttons are stacked in a grid with subtitles** rather than a plain row — a stray
  `description` crept into the payload.
- **The transcript shows a bare `accept` / `decline`** instead of a sentence when a button is
  tapped — the utterances were not honoured.
- **The conversation ends before the buttons can be tapped** — the deferred `end_session` did
  not hold.

Also worth a glance while you are there: the decision card should draw exactly as it did on
2026-08-09 (grey tile, `Apple MacBook Pro 16"`, right-aligned price, two divider rules below).
It is the regression canary and its definition is byte-identical, so any change there is a
genuine surprise.

## Artifacts

Session scratchpad:

- `chat-fresh-ifr.zip` — the mandatory fresh pre-edit export (27 entries, both widget tools)
- `card-before.json` — the decision-card regression baseline (byte-compared three times)
- `claim_intake-before.txt` / `claim_intake-after.txt` — 17,057 → 18,310 chars
- `cover-before.json` / `cover-patch.json` / `schema-after-ifr.json` — the tool before, the patch, the resulting schema
- `tools-before-ifr.json` / `tools-after-ifr.json`, `agent-before-ifr.json` / `agent-after-ifr.json`
- `ver8.json` — the v8 snapshot; `dep-before-ifr.json` / `dep-after-ifr.json` — the repoint
- `conv-ifr-final.json` — the full 7-turn conversation record (the proof)
- `run-ifr.py` — the turn driver; `ifr-turn-*.json` — raw per-turn responses

## Self-Check: PASSED

- `.planning/quick/260810-ifr-…/260810-ifr-SUMMARY.md` — created
- `05-02-SUMMARY.md`, `STATE.md` and `DEMO-RUNBOOK.md` modified and dirty in `git status`
  (uncommitted by instruction; the orchestrator owns the docs commit)
- Every server assertion re-read from the API after the write, never inferred from the request
- No git commit made by this task; ROADMAP.md untouched; `05-02-SUMMARY.md` remains
  `status: incomplete`
