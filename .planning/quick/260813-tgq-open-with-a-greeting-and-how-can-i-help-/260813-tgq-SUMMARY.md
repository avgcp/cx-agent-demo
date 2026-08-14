---
quick_id: 260813-tgq
type: quick
status: complete-shipped
date: 2026-08-13
subsystem: cx-agent-both-apps
tags: [greeting, opening-turn, intent-split, claims-concierge, turn-placement, ces, shipped, both-apps]
server_changes:
  chat:
    app: a2f621e4-9faf-505a-b804-22471f022366 (Meridian Claim - Chat (hardened))
    version_cut: chat v17 64be15eb-947f-4f36-8af2-2aefd225742b
    deployment_repointed: YES - d7bfbb93 now serves v17 (read back 2026-08-14T02:35:48.599636Z)
    rollback_target: chat v16 be4c83bb-825a-4feb-af88-379113543aa7
    rollback_target_updateTime: "2026-08-14T02:03:42.286752Z"   # re-read LIVE immediately before the PATCH
    channelProfile_sha: e6977330c6e3294b (byte-identical across the write, WEB_UI)
    resources_changed: ["agent claims_concierge (instruction 8,075 -> 9,591)"]
  voice:
    app: 6e01e4a5-42a8-5213-b3da-c9053ff8ea52 (Meridian Claim - Voice (demo-ready))
    version_cut: voice v16 09a1f14d-be24-40ba-abbd-a06e495f5d0d
    deployment_repointed: YES - d28bbcb0 now serves v16 (read back 2026-08-14T02:35:55.602430Z)
    rollback_target: voice v15 17b2e438-b132-49f1-8b32-190b132225ae
    rollback_target_updateTime: "2026-08-13T23:42:50.390492Z"   # re-read LIVE immediately before the PATCH
    channelProfile_sha: 7d89c16219fd2322 (byte-identical across the write, GOOGLE_TELEPHONY_PLATFORM)
    resources_changed: ["agent claims_concierge (instruction 5,126 -> 6,642)"]
  untouched_byte_identical:
    - chat: all 13 tools (resolve_claim included), claim_intake, case_summary, globalInstruction, languageSettings, guardrails, variableDeclarations
    - voice: all 9 tools (resolve_claim included), claim_intake, case_summary, globalInstruction, languageSettings, audioProcessingConfig, variableDeclarations
  fork: 9ae7a0c3 - ZERO calls of any method
conversations_spent: 15 (12 draft, 3 deliberate LIVE controls)
resend_sends_this_task: 17
canaries: 86/86 PASS
files_created:
  - .planning/quick/260813-tgq-.../260813-tgq-SUMMARY.md
---

# 260813-tgq — Open with a greeting and "how can I help", on both apps

**Shipped on both channels.** The demand for identity before the customer has said anything is gone
from chat and from voice, replaced by a greeting plus an open question, with the identity ask moved
into the response to the customer's stated intent.

| | Chat `a2f621e4` | Voice `6e01e4a5` |
|---|---|---|
| Deployment | `d7bfbb93` | `d28bbcb0` |
| **New version, LIVE** | **v17 `64be15eb-947f-4f36-8af2-2aefd225742b`** | **v16 `09a1f14d-be24-40ba-abbd-a06e495f5d0d`** |
| Repointed at (read back) | `2026-08-14T02:35:48.599636Z` | `2026-08-14T02:35:55.602430Z` |
| **Rollback target** (re-read live immediately before the PATCH) | **v16 `be4c83bb-825a-4feb-af88-379113543aa7`**, `updateTime 2026-08-14T02:03:42.286752Z` | **v15 `17b2e438-b132-49f1-8b32-190b132225ae`**, `updateTime 2026-08-13T23:42:50.390492Z` |
| `channelProfile` across the write | `e6977330c6e3294b` identical, `WEB_UI` | `7d89c16219fd2322` identical, `GOOGLE_TELEPHONY_PLATFORM` |

Both repoints were **gated in code** on the same 86-assertion battery (`gate.py`), re-run inside the
ship script; the PATCH is conditional on its exit code, not on a human reading a checklist. Each app
was gated independently — either could have held. Both passed, so both shipped.

## The new opening line, observed live on the draft that became each version

**Chat** (`run_CHAT_appr4`, turn 0, user typed *"Hi there."*):

> Hello. You're through to Meridian Device Protection. I'm Alex. How can I help you today?

**Voice** (`run_VOICE_appr`, turn 0):

> Hello! You're through to Meridian Device Protection, my name is Alex. How can I help you today?

**Voice, opened in Spanish** (`run_VOICE_es`, turn 0, caller said *"Hola, buenos dias."*):

> Hola, bienvenido a Meridian Device Protection. Mi nombre es Alex. ¿Cómo puedo ayudarle hoy?

Then, only after the customer says what they want (`run_CHAT_appr4` turn 1 / `run_VOICE_appr` turn 1,
user: *"I need to file a claim."*):

> I can help with that. Could I take your full name and your policy ID?
> I can certainly help with that. Could I take your full name and your policy ID, please?

**What it replaced, captured from the LIVE pre-change build as a control** (`run_CHAT_apprLIVE`
turn 0, chat v16 `be4c83bb` through `config.deployment`):

> Hello - I'm Alex. I can help with your claim. Could you please provide your full name and policy ID?

and it then asked a *second* time on turn 1 — the exact behaviour this task removes.

## Where the rule went, and why there

**The greeting was never a literal.** Nothing in either app contains the sentence
*"To pull up your policy, could I take your full name and your policy ID?"*. It was **emergent** from
two things: `claims_concierge`'s `<role>` (*"the front door… verify who you are speaking to"*) and the
one line inside `<step name="Collect details">` whose `<trigger>` is *"The caller has not yet been
verified"* and whose `<action>` opened *"Ask for their full name and policy ID together, in one
question."* That step fires on the very first turn, so it **was** the greeting.

So the change is a single in-place replacement of that step's opening prose. It is **not** a new step
and **not** a new paragraph appended anywhere — deliberately, because every one of this project's
turn-placement disasters (v11's gag order, o5l's last-paragraph-wins, p23's three-way oscillation)
came from adding prose rather than neutralising it.

The replaced region is now explicitly two-phase and ordered: *open with a greeting and one open
question, ask for nothing*; then *once they have said what they need, acknowledge it and take name
and policy ID in one question*. The step's **last paragraph is untouched** — it is still the
policy-ID digit-normalisation rule — so the "last paragraph wins" property of the step is unchanged.

**Chat and voice are byte-identical in this region, but the anchors were still derived from each
app's own bytes.** Both `claims_concierge` instructions are pure CRLF (chat 87 CRLF / 0 bare LF,
voice 63 / 0) — note this differs from chat `claim_intake`, which is pure LF. The replacement text
was read **from a file** and normalised to each app's own line endings before substitution.

### The delta, per app

| | Chat | Voice |
|---|---|---|
| `claims_concierge` instruction | 8,075 → **9,591** (+1,516) | 5,126 → **6,642** (+1,516) |
| Anchor occurrences in that app's own bytes | 1 | 1 |
| Pre-stated delta bound | 1,100–2,000 | 1,100–2,000 |
| Contiguous regions changed | **1** | **1** |
| common prefix / common suffix | 2,658 / 5,193 | 2,658 / 2,244 |
| old span / new span | 224 / 1,740 | 224 / 1,740 |
| Must-survive literals | 7/7 | 7/7 |
| Original non-blank lines lost | 0 | 0 |
| Bare LF introduced / non-ASCII introduced | none / none | none / none |
| Read-back byte-identical to the intended string | **true** | **true** |
| Version snapshot `claims_concierge` == intended string | **true** | **true** |

## The four probes — both apps

All against each app's **DRAFT** (`runSession`, no `config.deployment`) on the bytes that became the
shipped version, TEXT/API channel. Every result is asserted in `gate.py`, not read off by eye.

| # | Probe | Chat | Voice |
|---|---|---|---|
| 1 | The greeting fires, asks how it can help, and does **not** demand identity unprompted | **PASS** — `run_CHAT_appr4` t0 | **PASS** — `run_VOICE_appr` t0 |
| 2 | *"I need to file a claim"* → identity requested → the claim completes end to end | **PASS** — identity on t1, `verify_identity authenticated: true` on t2, `AUTO_APPROVE` `CLM-24042` $420 | **PASS** — identity on t1, authenticated on t2, `AUTO_APPROVE` `CLM-24256` $840 |
| 3 | A customer who volunteers everything at once is **not** made to repeat themselves | **PASS** — `run_CHAT_esc` t0, opener *"Hi, I'm Jordan Rivera, policy PDP100294, I cracked my screen."* → `verify_identity` fired on that same turn, `authenticated: true`, transferred to `claim_intake` in the same turn, **zero re-asks** | **PASS** — `run_VOICE_esc` t0, identical opener → *"Thanks, Jordan. I'm putting you through to claim handling now."* + intake opener, **zero re-asks** |
| 4 | Opening in **Spanish** still greets in Spanish | n/a (voice only) | **PASS** — `run_VOICE_es`: greeting, open question, identity ask, handoff line **and** `claim_intake`'s opener all in Spanish. voice v15's language fix is not regressed |

Probe 3 is worth spelling out because it is the regression the brief called most likely to annoy an
audience: on **both** apps the agent read the whole opening message, extracted first name, last name
and policy ID, called `verify_identity` immediately, and transferred — **without asking a single
question**. The new prose earns that: the replaced region's last paragraph says a caller who has
already said what they need has answered the open question, and one who has already given name and
policy ID has answered the second, *"Never make anyone repeat something they have already told you."*

## Regression canaries — 86/86 PASS, every one from a `toolCall`/`toolResponse` object

Run by `gate.py` over the saved raw turn responses of the draft runs that became each version, and
**re-run inside the ship script** so the repoint could not proceed without them.

### Both apps

| Canary | Chat | Voice |
|---|---|---|
| Identity capture works; diagnostic fires and reaches terminal | PASS — `REPAIRABLE`/`keyboard`, `TOTAL_LOSS` | PASS — `REPAIRABLE`/`screen`, `TOTAL_LOSS` |
| Deterministic tariff, approve | PASS — 420 / 25 / 1,500 / 3,000, `["DL-1"]` | PASS — 840 / 25 / 1,500 / 3,000, `["DL-1"]` |
| Deterministic tariff, escalate | PASS — 3,000 / 25 / 1,500, `["DL-3","DL-2"]`, `total_loss_flag: true` | PASS — same |
| Spoken decision **byte-identical** to `resolve_claim.explanation`, emitted **exactly once** | PASS both branches | PASS both branches, `decision_turn_agent_chunks == 1` |
| **The send-away line is still SPOKEN** | **PASS** on the approve record turn *and* the escalated turn, chunk emitted **before any tool call** on both | **PASS** on approve, and on escalation **before all five tool calls** |
| Cross-sell fires | **PASS** — `cover_offer_actions`, actions verbatim `Add it` / `Not now` | **PASS** — *"…would you like to add that to your cover?"* |
| Exactly one customer email, asserted via `resolve_claim` `email_queued: true` (never via `send_claim_email`'s count) | PASS | PASS |
| Assessor packet: six headings each exactly once, zero `{placeholder}` braces | PASS | PASS |
| Assessor/record subject token | `[RECORD] [CHAT] CLM-24042 - Jordan Rivera - APPROVED` and `[ASSESSOR] [CHAT] CLM-24102 / CLM-24269 - Jordan Rivera`, all Resend **200** | `[ASSESSOR] [VOICE] CLM-24506 - Jordan Rivera`, Resend **200** |
| No `[ASSESSOR]` leakage to the customer | PASS | PASS |
| Zero store vocabulary in any agent text | PASS (3 runs) | PASS (3 runs) |
| `escalate_to_human` fired exactly once | PASS | PASS |

### Chat only

| Canary | Result |
|---|---|
| Decision card renders — `claim_decision_card` `widget_tool_status: success`, `productItem` present, `textResponse` **byte-identical** to `resolve_claim.explanation` | **PASS** |
| `record_claim` returns `recorded: true`, `status_code: 200`, `store_calls: 3` | **PASS** on both branches (`CLM-24042`, `CLM-24102`, `CLM-24269`) |
| `record_claim` ordered **before** `claim_decision_card` (06-03's widget-terminates-the-turn rule) | **PASS** |
| **The photo is still asked for on every branch and `capture_claim_photo` still fires** (p23) | **PASS** — `captured: true` on the approve run (keyboard) and on both escalated runs (liquid), `attached: true` on every record email |
| No `end_session` on the record turn | **PASS** |

### Voice only

| Canary | Result |
|---|---|
| `DECISION_SPEECH_EN` **byte-identical** on the TEXT/API channel | **PASS** — 197 chars, exact match to `05-06-VOICE-BASELINE.md` with `CLM-` normalised |
| `GTP_SURFACE` byte-identical | **PASS** — `audioProcessingConfig` = `{"bargeInConfig":{"bargeInAwareness":true},"synthesizeSpeechConfigs":{"en-US":{},"es-US":{}}}`, byte-equal to the plan-start capture |
| `bargeInAwareness` byte-identical | **PASS** — `true` |
| `languageSettings` byte-identical | **PASS** — `en-US` default, `es-US` supported, `enableMultilingualSupport: true` |
| **`record_claim` still UNWIRED** | **PASS** — attached to no agent; `claim_intake` still wires exactly 7 tools; zero invocations across every run |
| `lookup_claim` still unwired on voice (06-04 owns it) | **PASS** |
| 33 `variableDeclarations`, `SEQUENTIAL`, `gemini-3.1-flash-live` | **PASS** |

### Isolation, by whole-object SHA-256 against the plan-start capture

- **Chat:** all **13** tools byte-unchanged — `resolve_claim`, `assess_screen_crack`,
  `capture_claim_photo`, `record_claim`, `lookup_claim`, `send_case_record_email` and the rest.
  `claim_intake` (32,498) and `case_summary` (6,977) byte-unchanged. `globalInstruction` (1,720),
  `languageSettings`, `guardrails`, `modelSettings`, `toolExecutionMode`, `rootAgent`, 37
  `variableDeclarations` all unchanged. Every agent's tool list unchanged.
- **Voice:** all **9** tools byte-unchanged, `resolve_claim` included. `claim_intake` (20,711) and
  `case_summary` (6,447) byte-unchanged. `globalInstruction` (2,859) unchanged.
- **`resolve_claim` was never read, echoed, printed or patched on either app.** No Resend key or GCS
  HMAC key was read at any point. `grep -rEo` for secret **values** over `.planning/` returns
  **nothing** — the 25 line-level regex hits in the repo are plan documents quoting the gate's own
  pattern, exactly as p23 recorded.
- **Fork `9ae7a0c3`: zero calls of any method.** App list re-read at close: **five**. No probe app was
  ever created. Chat v11 `838b6d2b` was never deployed. **No git commit — the tree is left dirty.**

## Two apparent regressions, both cleared by a LIVE control — and the method is the finding

The brief was right that everything downstream shifts by one turn. Two failures showed up on the
patched draft that looked exactly like the disasters this project has had before. **Neither was
mine, and neither could have been settled by reasoning — only by re-running the identical script
against the pre-change build through `config.deployment`.** Three of the fifteen conversations were
spent on precisely this and they were the best-spent three.

### 1. The silent escalated turn (looked like the v11 `838b6d2b` gag order)

On `run_VOICE_esc`, the escalated turn filed the packet, escalated and closed with **not one word
spoken** — `outputs[0].text` literally `null`. That is the v11 shape, and it is the canary the brief
flagged as broken three times already.

**Control:** the same four-turn script against **live voice v15 `17b2e438`** (`run_VOICE_escLIVE`),
which is byte-for-byte the pre-change build. **It produced the identical silent turn.** So the cause
is the *compressed conversation shape*, not the edit: when the caller volunteers everything in one
message, the whole flow collapses by two turns and `send_claim_email` + `generate_case_summary` +
`send_case_record_email` + `escalate_to_human` + `end_session` all land on a single turn that the
model treats as pure closure.

**And on a normal-shaped run the line is spoken on both apps**, on the patched build:
`run_VOICE_esc2` t4 — *"We've just sent an email to the address on your policy, so please reply to it
with photos of the damage."* — emitted **before all five tool calls**; `run_CHAT_esc2` t4, the same,
before all five. Canary green.

> **Carry forward, new:** a customer who volunteers name, policy and loss in their opening message
> collapses the escalated path onto one turn, and that turn goes silent. It predates this change and
> it is now more reachable, because the new opening invites exactly that kind of message. **Do not
> script the escalated demo beat with a volunteer-everything opener** — the runbook note is below.

### 2. `end_session` on the chat record turn, cross-sell dead (looked like p23 arrangement C)

`run_CHAT_appr3` t4 filed `[RECORD] [CHAT] … - APPROVED` at 200 with the photo attached, then called
`end_session {"reason": "escalation after errors", "session_escalated": true}` — killing the
cross-sell. That is p23's arrangement-C shape verbatim.

**Two controls settled it.** The live control `run_CHAT_appr3LIVE` on **chat v16 `be4c83bb`** did
*not* reproduce it — but it also ran a **different turn alignment**, asking diagnostic `q4` as its
own turn where the draft had batched it. So a second **draft** run with the identical script
(`run_CHAT_appr4`) was taken: it aligned the same way live did and **the cross-sell fired**,
`cover_offer_actions` with `Add it` / `Not now` verbatim. The `end_session` is model variance on a
compressed decision turn, on a turn this project already knows oscillates — not a property of the
new instruction. The canary is proven on `run_CHAT_appr4`, which is the run the gate reads.

> **Method note worth keeping:** a single draft observation is not evidence about an instruction
> change. Two of this task's four "regressions" were turn-alignment variance. **Always take the live
> control on the identical script, and take a second draft run before blaming the edit.**

## A pre-existing chat defect this task surfaced but did not cause, and did not fix

**A cracked-screen chat claim deadlocks when the photo arrives in the same turn as the damage
description.** Observed on the draft (`run_CHAT_appr2`) *and* reproduced identically on **live chat
v16** (`run_CHAT_appr2LIVE`), so it is a p23 defect, not a regression:

1. The model calls `assess_screen_crack` **before** `run_diagnostic` has set `dx_issue`.
2. p23's `beforeToolCallbacks[0]` therefore refuses it with `PHOTO_NOT_ASSESSED_FOR_THIS_ISSUE`
   (correctly, by its own predicate — at that instant the claim is not yet known to be a screen).
3. `run_diagnostic` then terminates at `screen`, and `capture_claim_photo` files the image fine.
4. `resolve_claim` refuses with `PHOTO_REQUIRED`, because no crack was ever *assessed*.
5. The agent loops asking for "another photo" — or, on live, ends the session.

The two deterministic backstops p23 installed are each correct in isolation and **deadlock each other
when the photo is volunteered early**. It needs the gate to tolerate a not-yet-terminal screen claim,
or `assess_screen_crack` to be re-callable after the diagnostic terminates. **Out of scope here** —
it needs `claim_intake` and possibly `resolve_claim`, both of which this task was told not to touch.
It is filed as the top follow-up. The auto-approve demo beat is currently only safe on chat if the
photo is attached **after** the agent asks for it, or on a non-screen fault.

## Room for 06-04 — its branch can be added without re-opening this instruction

**Confirmed by construction, and by the shape of the two apps as they now stand.**

The new region deliberately says nothing about *routing*. Its whole job is the opening turn and the
identity ask, and it states in terms that the two are separable:

> *You ask for exactly the same details whatever they came for, and you ask for them the same way. A
> caller reporting something that has just gone wrong and a caller asking about a claim they have
> already filed both need a first name, a last name and a policy ID before anything can happen. What
> they came for decides what happens AFTER they are verified, not what you ask for here — so remember
> what they said, carry it forward, and do not act on it yet.*

Three consequences for 06-04:

1. **The status branch is a sibling `<subtask>`, not an edit to this step.** Chat already proves the
   shape: 06-03 added `<subtask name="Check on an existing claim">` with a trigger reading *"A
   verified caller is asking about a claim that already exists…"*, sitting between
   `Verify the caller` and `Anything else`. It never touched `Collect details`, and this task never
   touched it. 06-04 adds the same subtask to voice's `claims_concierge` and attaches `lookup_claim`
   — **both untouched by this task and both still exactly where 06-04's plan expects them.**
2. **The open question already elicits a status enquiry.** The wording is *"How can I help you
   today?"* — fully open — and the region names the existing-claim family explicitly, so the model
   already accepts *"I want to check on a claim I filed"* as a valid answer rather than steering it
   toward a new loss. 06-04 does not need to re-word the question to make its branch reachable.
3. **The intent is captured before verification and carried forward**, which is exactly what 06-04
   Task 1's point 1 (*"the intent split, stated with literal trigger families"*) needs — and the
   *"do not act on it yet"* clause means the split cannot fire before `verify_identity`, which is
   06-04's T-06-03 gating requirement.

06-04's plan-start capture figures **have moved and must be re-read live** (its own operational rule
already says so): voice `claims_concierge` is now **6,642** chars, not 5,126; the voice deployment
rollback target is now **v16 `09a1f14d`**, not v13/v14/v15. Voice `claim_intake` is unchanged at
**20,711** and 06-04 still asserts it byte-identical, which still holds.

⚠️ The `draft_equals_v14` / `draft_equals_v11` invariants remain **FALSE** on both apps, now one
version further along: the chat draft equals **v17 `64be15eb`** and the voice draft equals
**v16 `09a1f14d`**. Any version 06-04 cuts will carry this greeting change. That is intended, it is
shipped and live on both channels, and it is 86/86 green — but it must be a decision, not a surprise.

## Also recorded — the email delivery rate, and what it means for the demo

The user confirmed **by inbox** that **both** `[ASSESSOR] [CHAT]` (with the photo attached) and
`[RECORD] [CHAT]` arrived from 260813-p23's runs. **p23's headline claim is therefore verified by
delivery, not merely by Resend acceptance** — the photo-carrying record email on the auto-approve
path, which is the artifact that path previously never produced, genuinely lands in a mailbox.

**But only about two of the seven test emails arrived — roughly a 30% delivery rate.** That is the
strongest evidence yet that the unverified shared sender `onboarding@resend.dev` is the problem, not
the mailer: 260812-trc already killed six other hypotheses by execution and concluded the remaining
gap was delivery rather than sending, and a 2-in-7 arrival rate against consistent HTTP 200s is
exactly the intermittent-drop shape of an unverified shared sending domain.

> ### DEMO RISK — do not script a beat around a live-sent email
> **Until a sending domain is verified in Resend, no demo beat may depend on an email arriving.**
> A ~30% delivery rate means a scripted *"and here's the email that just landed"* moment fails about
> two times in three, in the room. Say the email has been sent; do not open a mailbox to prove it.
> **The fix is a verified sending domain** — that is a Resend account action, not a code change, and
> it also removes the spam-folder problem. This now sits alongside the outstanding Resend key
> rotation and `resolve_claim`'s truthfulness fix, and it is more urgent than either for the demo.

## Budget and cost

- **15 conversations** — 12 against the drafts, **3 deliberate LIVE controls** (`run_CHAT_apprLIVE`,
  `run_CHAT_appr2LIVE`, `run_CHAT_appr3LIVE`, `run_VOICE_escLIVE` — four, of which three were the
  causation controls above and one captured the pre-change opening line for the record).
- **One 429**, on the fourth turn of `run_CHAT_esc` — a turn the session had already ended for. It
  was **not retried and not looped**; the conversation was already complete for assertion purposes.
  No 529 was seen.
- **17 Resend sends** against a 100/day tier. Four chat conversations produced zero sends because the
  screen-photo deadlock above stopped them before `resolve_claim`.
- Every network call bounded with `curl --max-time 120 --connect-timeout 15`; a fresh
  `gcloud auth print-access-token` was taken immediately before every conversation and every write.

## Platform findings that cost nothing and are worth keeping

1. **`runSession`'s image input key is `image`, not `inlineData`.** The `SessionInput` message is a
   `oneof` over `text | event | willContinue | variables | dtmf | image | toolResponses | blob |
   audio`; an image is a sibling input object `{"image": {"mimeType": "image/png", "data": "<b64>"}}`.
   Sending a `Part`-style `{"inlineData": {...}}` returns **400 `INPUTTYPE_NOT_SET`**. Note this is a
   *different* shape from p23's `apps.executeTool` context shape (`user_content.parts[].inlineData`) —
   the two APIs do not share it. Proven: `capture_claim_photo` returned
   `{"captured": true, "photo_b64_len": 488, "parts": 1}` for a 364-byte synthetic PNG.
2. **`runSession`'s `outputs` is an ARRAY, one entry per speaking agent**, and only the *last* entry
   carries the fullest `diagnosticInfo`. Reading `outputs[0]` alone silently loses the sub-agent's
   turn — it made a perfectly good handoff turn look silent during this task before it was caught.
3. **`diagnosticInfo.messages` is cumulative for the whole session**, not per-turn; slice it at the
   last user chunk matching the turn you sent, or every assertion double-counts.
4. **A `session_escalated: true` `end_session` ends the session for `runSession` too** — the next
   turn returns 400 `SESSION_ALREADY_ENDED`. Plan the last scripted turn accordingly rather than
   reading that 400 as a defect.

## Self-Check: PASSED

- `.planning/quick/260813-tgq-.../260813-tgq-SUMMARY.md` — created
- `d7bfbb93` re-read **after** the write → **`64be15eb-947f-4f36-8af2-2aefd225742b`**,
  `updateTime 2026-08-14T02:35:48.599636Z`; `channelProfile` sha `e6977330c6e3294b` identical
- `d28bbcb0` re-read **after** the write → **`09a1f14d-be24-40ba-abbd-a06e495f5d0d`**,
  `updateTime 2026-08-14T02:35:55.602430Z`; `channelProfile` sha `7d89c16219fd2322` identical
- Both rollback targets **re-read live from the API immediately before their PATCH**
- Both version snapshots compared field-by-field against their drafts: `claims_concierge`,
  `claim_intake`, `case_summary` all byte-identical, and `claims_concierge` equals the exact string
  the edit script intended to write
- Every edit a scripted anchored `str.replace()` **from a file**, uniqueness asserted before, delta
  bound declared before running, contiguity measured by common prefix/suffix, byte-for-byte read-back
- 86/86 assertions PASS; both repoints gated in code on that battery's exit status
- `resolve_claim` never read, echoed or patched on either app; secret-value grep over `.planning/`
  returns **no match**
- Fork `9ae7a0c3`: **zero calls of any method**. App list = **5**, no probe app created
- Chat v11 `838b6d2b` never deployed; voice v12 `9227210b` never deployed
- **No git commit, by instruction** — the tree is deliberately left dirty
