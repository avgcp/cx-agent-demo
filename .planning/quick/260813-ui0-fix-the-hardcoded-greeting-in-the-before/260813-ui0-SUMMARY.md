---
quick_id: 260813-ui0
type: quick
status: complete-shipped
date: 2026-08-13
subsystem: cx-agent-both-apps
tags: [greeting, before-model-callback, llmresponse-short-circuit, session-start, opening-turn, ces, shipped, both-apps, platform-finding]
server_changes:
  chat:
    app: a2f621e4-9faf-505a-b804-22471f022366 (Meridian Claim - Chat (hardened))
    version_cut: chat v18 d0e4bfef-6d5f-43b3-b490-3d9036d030e2
    deployment_repointed: YES - d7bfbb93 now serves v18 (read back 2026-08-14T03:15:20.124453Z)
    rollback_target: chat v17 64be15eb-947f-4f36-8af2-2aefd225742b
    rollback_target_updateTime: "2026-08-14T02:35:48.599636Z"   # RE-READ LIVE immediately before the PATCH
    channelProfile_sha: e6977330c6e3294b (byte-identical across the write, WEB_UI)
    resources_changed: ["agent claims_concierge beforeModelCallbacks[0].pythonCode (915 -> 869)"]
  voice:
    app: 6e01e4a5-42a8-5213-b3da-c9053ff8ea52 (Meridian Claim - Voice (demo-ready))
    version_cut: voice v17 dcc20863-3746-4e43-a2c9-ed30e0611479
    deployment_repointed: YES - d28bbcb0 now serves v17 (read back 2026-08-14T03:15:07.395898Z)
    rollback_target: voice v16 09a1f14d-be24-40ba-abbd-a06e495f5d0d
    rollback_target_updateTime: "2026-08-14T02:35:55.602430Z"   # RE-READ LIVE immediately before the PATCH
    channelProfile_sha: 7d89c16219fd2322 (byte-identical across the write, GOOGLE_TELEPHONY_PLATFORM)
    resources_changed: ["agent claims_concierge beforeModelCallbacks[0].pythonCode (915 -> 869)"]
  untouched_byte_identical:
    - chat: all 13 tools (resolve_claim included), claims_concierge INSTRUCTION, claim_intake,
        case_summary, globalInstruction, languageSettings, afterModelCallbacks, 37 variableDeclarations
    - voice: all 9 tools (resolve_claim included), claims_concierge INSTRUCTION, claim_intake,
        case_summary, globalInstruction, languageSettings, audioProcessingConfig, afterModelCallbacks,
        33 variableDeclarations
  fork: 9ae7a0c3 - ZERO calls of any method
conversations_spent: 9 draft + 1 deliberate LIVE control + 12 single-turn event probes
resend_sends_this_task: 6
canaries: 93/93 PASS
files_created:
  - .planning/quick/260813-ui0-.../260813-ui0-SUMMARY.md
---

# 260813-ui0 — Fix the hardcoded greeting in the `beforeModelCallback`

**Shipped on both channels.** The phone and the chat widget now open with the line the
instruction has said since `260813-tgq`, because the line is no longer contradicted by a
hardcoded literal that runs *instead of* the model.

| | Chat `a2f621e4` | Voice `6e01e4a5` |
|---|---|---|
| Deployment | `d7bfbb93` | `d28bbcb0` |
| **New version, LIVE** | **v18 `d0e4bfef-6d5f-43b3-b490-3d9036d030e2`** | **v17 `dcc20863-3746-4e43-a2c9-ed30e0611479`** |
| Repointed at (read back) | `2026-08-14T03:15:20.124453Z` | `2026-08-14T03:15:07.395898Z` |
| **Rollback target** (re-read live immediately before the PATCH) | **v17 `64be15eb-947f-4f36-8af2-2aefd225742b`**, `updateTime 2026-08-14T02:35:48.599636Z` | **v16 `09a1f14d-be24-40ba-abbd-a06e495f5d0d`**, `updateTime 2026-08-14T02:35:55.602430Z` |
| `channelProfile` across the write | `e6977330c6e3294b` identical, `WEB_UI` | `7d89c16219fd2322` identical, `GOOGLE_TELEPHONY_PLATFORM` |

Both repoints were **gated in code** on the same 93-assertion battery (`gate.py`), re-run inside
the ship script with the PATCH conditional on its **exit status**. Each app was gated
independently — either could have held. Both passed, so both shipped.

## The headline: CHAT HAD THE IDENTICAL DEFECT, and its probes missed it identically

The brief asked me to check. The answer is unambiguous, and it is worse than "an equivalent
callback": **chat's `claims_concierge.beforeModelCallbacks[0].pythonCode` was byte-identical to
voice's** — same 915 characters, same SHA-256 `10cf58b137b2b4fa…`, same three branches, same
hardcoded greeting.

Proven on the **serving version**, not the draft, by running the session-start event through
`config.deployment` against **live chat v17 `64be15eb`**:

> **t0 (session start):** *"Hello, you're connected to Meridian Device Protection. I'm Alex. **To pull up your policy, could I take your full name and your policy ID?**"*
> **t1 (customer: "Hi, I need to file a claim — my laptop keyboard has stopped working."):** *"I understand. **To pull up your policy, could I take your full name and your policy ID?**"*

So the live chat widget did not merely open by demanding identity — it demanded identity
**twice**, which is precisely the double-ask `260813-tgq` was written to remove. tgq's chat
result was true of the instruction and false of the product.

The two apps are now byte-identical again, on the fixed code: SHA-256
`42a4fe1ea5a12cf05711b483aa410b1079221cdd532e42a42c9c90e3c1991164`, 869 chars, on both.

## The new literal, and why it is exactly this

One anchored `str.replace()` inside the callback, on each app:

| | |
|---|---|
| Anchor (removed) | `To pull up your policy, could I take your full name and your policy ID?` |
| Replacement | `How can I help you today?` |
| **Assembled greeting, both apps** | **`Hello, you're connected to Meridian Device Protection. I'm Alex. How can I help you today?`** |

The first two sentences are **byte-untouched**. Only the third changed. The wording is not
invented: `claims_concierge`'s `<step name="Collect details">` — which this task did **not**
edit — reads *"Greet them, say they are through to Meridian Device Protection, give your name,
and ask ONE open question about what they need. **"How can I help you today?"** is the whole of
it."* The replacement is that sentence verbatim, so the callback and the instruction now agree
literally rather than approximately.

### The delta, per app — identical on both

| | Chat | Voice |
|---|---|---|
| `beforeModelCallbacks[0].pythonCode` | 915 → **869** (−46) | 915 → **869** (−46) |
| Anchor occurrences in that app's own bytes | 1 | 1 |
| Replacement text already present before | 0 | 0 |
| Pre-declared delta / pre-declared length | −46 / 869 | −46 / 869 |
| Contiguous regions changed | **1** | **1** |
| common prefix / common suffix | 451 / 394 | 451 / 394 |
| old span / new span | 70 / 24 | 70 / 24 |
| Line endings | 20 CRLF, **0 bare LF**, unchanged | 20 CRLF, **0 bare LF**, unchanged |
| Non-blank lines before / after | 18 / 18 | 18 / 18 |
| Non-ASCII introduced | none | none |
| Must-survive literals | 9/9 | 9/9 |
| Read-back byte-identical to the intended string | **true** | **true** |
| **Version snapshot** callback == intended string | **true** | **true** |

> **Line-ending correction for future tasks.** The brief carried forward "chat is CRLF, voice is
> LF". For `beforeModelCallbacks[0].pythonCode` that is **wrong in both directions — both apps
> are pure CRLF (20 / 0)**, and the two sources are byte-identical, so the anchor *was* in fact
> transferable here. It was still derived from each app's own bytes and asserted unique per app
> before substitution. Measure; do not inherit.

## The three branches — all preserved, and all three PROVEN on the live deployments

The brief was right that breaking the other two branches would be worse than the greeting bug.
They are intact **structurally** (byte-for-byte literals asserted present in the draft, in the
read-back, and in each cut version's snapshot) **and functionally** — each was fired through
`config.deployment` against the newly-serving version:

| Branch | Trigger | Live voice v17 | Live chat v18 |
|---|---|---|---|
| 1 — greeting | `<event>session start</event>` | **`Hello, you're connected to Meridian Device Protection. I'm Alex. How can I help you today?`** byte-identical | same, byte-identical |
| 2 — dropped call | `<event>sys.remote-call-disconnected</event>` | **no text, session ended, `reason: 'caller disconnected'` present** | same |
| 3 — silence | text containing `no user activity detected` | **`Are you still there?`** | same |

Branches 2 and 3 returned **byte-identical output before and after the patch** on both apps
(gate assertions `B5 voice` / `B5 chat`). The 11-second call closes cleanly exactly as before.

## The verification problem — and the finding that dissolves it

The brief's premise was that **`runSession` structurally cannot verify this fix**, and that only
a phone call could. **That premise is false, and correcting it is the most valuable thing this
task produced.**

### `runSession` DOES deliver `<event>session start</event>` — via an input shape nobody had used

`SessionInput` is a `oneof` over `text | event | willContinue | variables | dtmf | image |
toolResponses | blob | audio` (tgq). The `event` arm takes a **message**, not a string, and the
field inside it is itself called `event`:

| Input sent | Result |
|---|---|
| `{"event": "session start"}` | **400 `INVALID_ARGUMENT`** — `Invalid value at 'inputs[0].event' (…ces.v1beta.Event), "session start"` |
| **`{"event": {"event": "session start"}}`** | **200 — fires the callback**, returns the hardcoded literal, model never runs |
| `{"event": {"name": "session start"}}` | 200 — a *different* event; the callback does not match, the **model** runs and produces the instruction-driven greeting |
| `{"text": "<event>session start</event>"}` | 200 — also fires the callback (the branch is a literal `part.text ==` comparison) |

So the honest diagnosis of why tgq's probes missed the bug is **not** that `runSession` cannot
see turn one. It is that **tgq's probes opened with `{"text": "Hi there."}`** — an ordinary user
turn. The callback's `part.text == "<event>session start</event>"` never matched, the model ran,
and the probes truthfully measured the instruction. The phone opens with the event; the probe
opened with a sentence. **The channel was never the problem; the first input was.**

That third row is the sharp edge: `{"event": {"name": …}}` and `{"event": {"event": …}}` both
return 200 and both produce a plausible greeting, but **only one of them is what the phone
sends**. A probe using the wrong one looks like it verified the opening turn and did not.

### What that bought

A genuine before/after control on both apps, on the real code path:

| | Voice | Chat |
|---|---|---|
| **PRE-patch**, draft, session-start event | *"…**To pull up your policy, could I take your full name and your policy ID?**"* | identical |
| **POST-patch**, draft, session-start event | *"…**How can I help you today?**"* | identical |
| **POST-ship**, through `config.deployment` on the newly-serving version | *"…**How can I help you today?**"* | identical |

The fix is therefore verified **structurally** (draft read-back, version snapshot) **and
behaviourally on the deployed build** — not merely inferred. What remains unverified is only
what an ear can settle: TTS delivery over the GTP leg. See *What the user must do*.

## Proof the new text is in the cut version, not just the draft

Read back from `GET …/versions/{id}` after each cut, from the snapshot itself:

| | Voice v17 `dcc20863` | Chat v18 `d0e4bfef` |
|---|---|---|
| `snapshot.agents[N]` index of `claims_concierge` | **0** | **0** |
| `snapshot.agents[0].beforeModelCallbacks[0].pythonCode` length | **869** | **869** |
| …its SHA-256 | **`42a4fe1e…`** (the new code) | **`42a4fe1e…`** |
| new literal present / old literal present | **true / false** | **true / false** |
| branch 2 (`from_end_session(reason='caller disconnected')`) in snapshot | **true** | **true** |
| branch 3 (`Are you still there?`) in snapshot | **true** | **true** |
| `snapshot` `claims_concierge` **instruction** byte-unchanged | **true** (6,642) | **true** (9,591) |
| snapshot agents / tools | 3 / 9 | 3 / 13 |

## Regression canaries — 93/93 PASS, and the deployment repointed on the exit status

Every figure below is read from a `toolCall` / `toolResponse` object, never from agent prose.

### The callback and the write itself (A, B)

| Canary | Chat | Voice |
|---|---|---|
| `beforeModelCallbacks` still exists, exactly one, new sha, 869 chars | PASS | PASS |
| All three branches present byte-for-byte; `return None` fallthrough intact | PASS | PASS |
| Branches 2 and 3 byte-identical output PRE vs POST | PASS | PASS |
| **`claims_concierge` INSTRUCTION byte-unchanged** | **PASS** (9,591) | **PASS** (6,642) |
| `afterModelCallbacks` byte-unchanged | PASS | PASS |
| **Agent key set unchanged — no `updateMask` wipe** | **PASS** (11 keys) | **PASS** (11 keys) |
| Whole agent minus the one field byte-identical to plan-start | PASS | PASS |

### Configuration surface (C)

| Canary | Chat | Voice |
|---|---|---|
| `languageSettings` byte-identical | PASS (`en-US` + `es-US`, multilingual true) | PASS (same) |
| `GTP_SURFACE` / `audioProcessingConfig` byte-identical | PASS | **PASS** — `{"synthesizeSpeechConfigs":{"en-US":{},"es-US":{}},"bargeInConfig":{"bargeInAwareness":true}}` |
| `bargeInAwareness` still `true` | PASS | PASS |
| `globalInstruction`, `rootAgent`, `modelSettings`, `toolExecutionMode` unchanged | PASS | PASS |
| `variableDeclarations` count | PASS (37) | PASS (33) |
| **All tools byte-unchanged, `resolve_claim` included** | **PASS (13/13, none differ)** | **PASS (9/9, none differ)** |
| `claim_intake` and `case_summary` byte-unchanged | PASS | PASS |
| **`record_claim` still UNWIRED on voice** | — | **PASS** (attached to no agent) |

### Voice, approve path (D) — `ui0-VOICE-appr`, `CLM-24316`

| Canary | Result |
|---|---|
| Session-start greeting is the new literal inside a real conversation, and turn 0 does **not** demand identity | **PASS** |
| Deterministic tariff | **PASS** — 840 / 25 / 1,500 / 3,000, `["DL-1"]`, `AUTO_APPROVE` |
| **`DECISION_SPEECH_EN` byte-identical on the TEXT/API channel** | **PASS** — 197 chars, spoken == `resolve_claim.explanation` exactly |
| Decision emitted **exactly once** (`decision_turn_agent_chunks == 1`) | **PASS** |
| `verify_identity authenticated: true`; `run_diagnostic` terminal `REPAIRABLE` | PASS |
| Exactly one customer email, asserted via `resolve_claim.email_queued` | PASS |
| **Send-away line SPOKEN** | **PASS** — *"…please reply to it with photos of the damage attached, and allow 3 to 7 business days…"* |
| Cross-sell fires | **PASS** — *"…I see you also have an Apple iPhone 16 Pro Max that isn't covered yet. Would you like to add that to your policy?"* |
| `record_claim` zero invocations | PASS |

### Voice, escalation path (E) — `ui0-VOICE-esc`, `CLM-24404`

| Canary | Result |
|---|---|
| Deterministic tariff, escalate | **PASS** — 3,000 / 25 / 1,500, `["DL-3","DL-2"]`, `total_loss_flag: true`, `HUMAN_REVIEW` |
| Escalated decision byte-identical to `explanation`, exactly once | **PASS** |
| **Packet subject token** | **PASS** — `[ASSESSOR] [VOICE] CLM-24404 - Jordan Rivera`, `sent: true`, Resend **200** |
| Six packet headings each exactly once, zero `{placeholder}` braces | **PASS** — `[1,1,1,1,1,1]` |
| `escalate_to_human` fired exactly once | PASS |
| **Send-away line SPOKEN, and emitted BEFORE all tool calls** | **PASS** |
| No `[ASSESSOR]` leakage to the customer | PASS |
| `record_claim` zero invocations | PASS |

### Voice, Spanish (F) — `ui0-VOICE-es`

| Canary | Result |
|---|---|
| Spanish opener → agent replies in Spanish from the first user turn | **PASS** — *"Siento mucho escuchar eso, pero puedo ayudarle. ¿Me podría decir su nombre completo y su número de póliza?"* |
| Spanish survives the `claims_concierge` → `claim_intake` handoff (olv's fix) | **PASS** — *"Gracias, Jordan. Le paso ahora mismo." / "Tengo aquí su Apple MacBook Pro de 16 pulgadas, ¿qué le ha pasado?"* |

### Chat (G) — `ui0-CHAT-appr2` `CLM-24578` and `ui0-CHAT-appr` `CLM-24482`

| Canary | Result |
|---|---|
| Session-start greeting is the new literal inside a real conversation | **PASS** |
| Deterministic tariff | **PASS** — 420 / 25 / 1,500 / 3,000, `["DL-1"]`, `AUTO_APPROVE` |
| Decision byte-identical to `explanation`, emitted exactly once | **PASS** |
| **`record_claim` `recorded: true`, `status_code: 200`, `store_calls: 3`** | **PASS** |
| **`record_claim` ordered BEFORE `claim_decision_card`** (06-03's widget-terminates-the-turn rule) | **PASS** (call indices 8 → 10) |
| **Decision card renders** — fires with a `productItem` payload | **PASS** |
| **`capture_claim_photo` still fires**, on the photo's arrival turn | **PASS** — `captured: true`, `parts: 1` |
| **`[RECORD] [CHAT]` record email with the photo attached** | **PASS** — `[RECORD] [CHAT] CLM-24578 - Jordan Rivera - APPROVED`, `attached: true`, Resend **200** |
| **The send-away line is still SPOKEN** | **PASS** — and it correctly used the photo-redundancy wording (*"I have got that photo on the file too, so there is nothing further you need to send us"*) |
| **`cover_offer_actions` fires** | **PASS** — actions verbatim `Add it` / `Not now` |
| `end_session` reason sane on a clean approve close | **PASS** — `session_escalated: false`, *"claim auto-approved and email sent, cross-sell accepted"* |

## One apparent regression, cleared by the LIVE control — the method again

On the second chat draft run (`ui0-CHAT-appr2`) the record turn filed
`[RECORD] [CHAT] … - APPROVED` at 200 with the photo attached and then called
`end_session {"reason": "escalation after errors", "session_escalated": true}` on a cleanly
**approved** claim, killing the cross-sell (the next turn returned 400 `SESSION_ALREADY_ENDED`).
That is o5l pass-1 / p23 arrangement-C / tgq `run_CHAT_appr3` **verbatim**, down to the bogus
reason string.

**It is not this edit, and it was not settled by reasoning.** Two controls:

1. **The live control** — the identical six-turn script against **chat v17 `64be15eb` through
   `config.deployment`** (`ui0-CHAT-LIVEctl2`). It ran a **different turn alignment** (the photo
   turn consumed one diagnostic question, pushing the decision to t4) and filed the record email
   **and** the cross-sell on the same turn with **no `end_session` at all**.
2. **The first draft run** (`ui0-CHAT-appr`) aligned differently again and produced a firing
   cross-sell with a **sane** `end_session` reason.

So across three runs of near-identical scripts the turn alignment moved twice and the
`end_session` shape moved with it. Every canary is green on at least one draft run, and the
bogus-`end_session` shape is documented pre-existing three times over. Consistent with tgq's
rule, which held again: **a single draft observation is not evidence about a change.**

> **New, small, and useful:** `[RECORD] [CHAT]` files on the **send-away turn**, not the decision
> turn. If the customer's photo arrives *on* the send-away turn, that turn is consumed by
> `capture_claim_photo` and the record email is skipped for the rest of the conversation
> (observed on `ui0-CHAT-appr`). **Attach the photo when the agent asks for it, not later** —
> which is already the runbook's advice for a different reason (the p23 screen deadlock).

## Recorded, not a regression: the session-start greeting is ALWAYS English

Because branch 1 returns a hardcoded literal, **the model never runs on turn 0** — so a Spanish
caller hears the English opener, then the agent switches from their first real utterance
onward. Confirmed on `ui0-VOICE-es`: turn 0 English literal, turn 1 fully Spanish, and Spanish
survives the sub-agent handoff.

This predates the task (the literal was already English) and tgq never saw it, because tgq's
Spanish probe opened with a Spanish *sentence* and so got a model-composed Spanish greeting.
**On the phone, the first sentence is English every time.** It is not worth fixing by prompting —
by construction, no instruction can affect a turn the model does not run — and localising it
would mean the callback guessing the caller's language before they have spoken. Left as is,
recorded, and noted in the runbook so the Spanish beat is scripted as *"the agent switches"*
rather than *"the agent opens in Spanish."*

## Deviations from plan

**None.** The brief's diagnosis was correct in every particular except the verification premise:
`runSession` *can* reach the callback, which made the fix provable rather than merely arguable.
That is recorded above and in STATE.md rather than treated as a deviation. Nothing outside
`beforeModelCallbacks[0].pythonCode` was written on either app.

## What the user must do — the phone check

**Only an ear can close this.** Dial the GTP number for the voice demo. The build now serving is
**voice v17 `dcc20863`**.

1. **The first sentence you hear must be, word for word:**
   > *"Hello, you're connected to Meridian Device Protection. I'm Alex. How can I help you today?"*

   If you hear *"To pull up your policy, could I take your full name and your policy ID?"*, the
   old build is still answering — **stop and say so**; do not work around it.
2. **Do not answer it with your name.** Say what you want (*"I need to file a claim, I dropped my
   laptop"*). Identity should be asked for **once**, in the reply to that — never before it and
   never twice.
3. **Spot-check branch 2 — hang up mid-call.** Ring off partway through, then confirm the call
   closes cleanly (no hanging session, no stuck line). This is the branch that closed the
   11-second call and it must still work.
4. **Spot-check branch 3 — stay silent.** After the greeting, say nothing for ~10–15 seconds.
   You should hear *"Are you still there?"*
5. **Chat is checkable in a browser in ten seconds**, on **chat v18 `d0e4bfef`**: open the widget
   and read its **very first message**, before typing anything. It must be the same sentence. Until
   now it has been the identity demand — twice.

**Nothing else in this task needs you.** The fix is verified structurally and on the deployed
builds over the API; only TTS delivery of the new sentence is unheard.

## Budget, cost and hygiene

- **9 draft conversations, 1 deliberate LIVE control, 12 single-turn event probes.** The event
  probes short-circuit the model entirely (the callback returns before the LLM call), so they
  cost effectively nothing — a large class of turn-zero verification is now free.
- **6 Resend sends** against a 100/day tier. **No 429 and no 529 was seen at any point.**
- Every network call bounded with `curl --max-time 120 --connect-timeout 15`; a fresh
  `gcloud auth print-access-token` taken immediately before every call, read and write.
- A client-side gate refused, by construction, any call touching fork `9ae7a0c3` or any app
  outside the two demo apps. **Fork `9ae7a0c3`: zero calls of any method.**
- **App list re-read at close: five.** No probe app was created. Chat v11 `838b6d2b` was never
  deployed. `resolve_claim` was never read, echoed, printed or patched on either app; no Resend
  key or GCS HMAC key was read at any point.
- **No git commit, by instruction** — the tree is deliberately left dirty.

## Platform findings worth carrying forward

1. **A `beforeModelCallback` that returns an `LlmResponse` short-circuits the model**, exactly as
   o5l's `beforeToolCallback` returning a dict short-circuits the tool. **An instruction change
   cannot affect a turn the model does not run on.** This is the second member of a family; assume
   there is a third.
2. **`runSession` delivers session start** — `{"event": {"event": "session start"}}`. The `event`
   arm of `SessionInput` is a message whose own field is `event`; a bare string is a 400, and
   `{"event": {"name": …}}` is a *different, silently-plausible* event that lets the model run. Use
   the `event`-in-`event` form or you will verify the wrong code path and not know it.
3. **`config.deployment` must be a resource name, not a URL.** Passing the full
   `https://ces.googleapis.com/v1beta/projects/…` form returns **400 `Invalid deployment`**; the
   accepted form is `projects/{p}/locations/{l}/apps/{a}/deployments/{d}`.
4. **`diagnosticInfo.messages` was per-turn on these builds, not cumulative.** tgq recorded it as
   cumulative. It varies — slice defensively rather than assuming either.
5. **`agents.patch` with `updateMask=beforeModelCallbacks` is safe and surgical here**: the agent's
   key set, instruction, `afterModelCallbacks`, `childAgents` and `tools` were all byte-identical
   after the write. Assert the key set anyway — that is the assertion that would have caught a wipe.

## Self-Check: PASSED

- `.planning/quick/260813-ui0-.../260813-ui0-SUMMARY.md` — created
- `d7bfbb93` re-read **after** the write → **`d0e4bfef-6d5f-43b3-b490-3d9036d030e2`**,
  `updateTime 2026-08-14T03:15:20.124453Z`; `channelProfile` sha `e6977330c6e3294b` identical
- `d28bbcb0` re-read **after** the write → **`dcc20863-3746-4e43-a2c9-ed30e0611479`**,
  `updateTime 2026-08-14T03:15:07.395898Z`; `channelProfile` sha `7d89c16219fd2322` identical
- Both rollback targets **re-read live from the API immediately before their PATCH**, never hardcoded
- Both version snapshots read back and asserted: `beforeModelCallbacks[0].pythonCode` == the new
  code (sha `42a4fe1e…`, 869 chars), all three branches present, instruction byte-unchanged
- All three callback branches fired through `config.deployment` on the **newly-serving** version
  of each app, post-repoint
- 93/93 assertions PASS; both repoints gated in code on that battery's **exit status**
- `resolve_claim` never read, echoed or patched on either app
- Fork `9ae7a0c3`: **zero calls of any method**. App list = **5**, no probe app created
- **No git commit, by instruction** — the tree is deliberately left dirty
