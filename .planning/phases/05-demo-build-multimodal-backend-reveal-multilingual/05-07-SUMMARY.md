---
phase: 05-demo-build-multimodal-backend-reveal-multilingual
plan: 07
subsystem: voice-agent-wiring, chat-agent-wiring
tags: [ces, assessor-packet, voice, gtp, version-cut, deployment-repoint, regression-canaries, currency-formatting, channel-token]
status: awaiting-human-phone-check
requires:
  - "05-06 voice baseline (VOICE_INVENTORY, DECISION_SPEECH_EN, ESCALATION_PATH, GTP_SURFACE)"
  - "260812-hhi voice claim_intake wiring (Case record step, same-turn design)"
  - "05-05 chat packet (the proven design being ported)"
provides:
  - "app:6e01e4a5/versions voice v13 5d9df25c - LIVE on d28bbcb0, the phone number"
  - "app:a2f621e4/versions chat v13 1eb3fd5c - LIVE on d7bfbb93"
  - "the assessor briefing packet delivered as a real email on BOTH channels, Resend HTTP 200"
  - "channel-discriminated packet subjects: [ASSESSOR] [VOICE] / [ASSESSOR] [CHAT]"
  - "packet money rendered as whole dollars ($3,000 / $25) on both channels"
affects:
  - "05-09 (Spanish on voice) - now builds on voice v13, not v11"
  - "the live phone demo and the live chat demo"
tech-stack:
  added: []
  patterns:
    - "anchored scripted str.replace() against key-bearing tool source, printing only booleans"
    - "contiguity measured by common prefix/suffix, not difflib opcodes"
    - "repoint gated in code on three assertion batteries re-passing"
    - "latency baseline taken by running the same script against the LIVE deployment"
key-files:
  created:
    - ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-07-SUMMARY.md"
  modified:
    - "app:6e01e4a5/agents/case_summary - 5,750 -> 6,447 chars (+697, MONEY FORMATTING rule)"
    - "app:6e01e4a5/tools/send_case_record_email - [PHONE] -> [VOICE] token; unsupported timeout kwarg removed"
    - "app:6e01e4a5/deployments/d28bbcb0 - b17c9a26 (v11) -> 5d9df25c (v13)"
    - "app:a2f621e4/agents/case_summary - 5,273 -> 5,970 chars (+697, MONEY FORMATTING rule)"
    - "app:a2f621e4/tools/send_case_record_email - [CHAT] token added"
    - "app:a2f621e4/deployments/d7bfbb93 - 26c3aebd (v12) -> 1eb3fd5c (v13)"
    - ".planning/spec/DEMO-RUNBOOK.md"
    - ".planning/STATE.md"
decisions:
  - "Channel token is [ASSESSOR] [VOICE] / [ASSESSOR] [CHAT] - a second bracketed word, ASCII, [ASSESSOR] prefix preserved byte-identical"
  - "Rule 1 auto-fix: voice send_case_record_email passed an unsupported timeout kwarg to ces_requests.post and had NEVER delivered - found only because this plan ran it live"
  - "voice v12 9227210b cut then superseded by v13 5d9df25c; 9227210b must never be deployed"
  - "No chat auto-approve canary: chat claim_intake is byte-identical to live v12, so the approve path is provably untouched"
  - "Latency baseline taken by running Scenario B against the LIVE v11 deployment, because 05-06 records no per-turn wall clock"
metrics:
  conversations_spent: 5
  turns_spent: 16
  voice_version_cut: "5d9df25c-3771-45bb-bd20-b28978cc5955 (plus 9227210b, superseded, never deployed)"
  chat_version_cut: "1eb3fd5c-5aff-46c8-b572-e3fe18bf966f"
  voice_deployment_repointed: "d28bbcb0: b17c9a26 -> 5d9df25c"
  chat_deployment_repointed: "d7bfbb93: 26c3aebd -> 1eb3fd5c"
  completed: 2026-08-12
---

# Phase 5 Plan 07: The Assessor Briefing Packet on the Phone Channel Summary

An escalated **phone** claim now composes the same six-section assessor briefing packet the
chat channel produces and delivers it as a real email — proven live with Resend returning
**HTTP 200**, byte-identical between composer and mailer, filed on the turn that always
happens, with the send-away line still spoken and nothing about the packet audible. Both
channels' packets now render money as whole dollars and stamp themselves with a channel
token. Shipped as **voice v13 `5d9df25c-3771-45bb-bd20-b28978cc5955`**, now served by the
phone deployment `d28bbcb0`, and **chat v13 `1eb3fd5c-5aff-46c8-b572-e3fe18bf966f`**, now
served by `d7bfbb93`.

**The finding that mattered most was not in the brief:** the voice packet mailer had never
worked. It threw on every send. Nothing in the plan, the quick task or the read-back
assertions could have caught it — only running it.

## The specific answers this plan was asked for

| | |
|---|---|
| **New voice version** | **`5d9df25c-3771-45bb-bd20-b28978cc5955`** — *"voice v13 - assessor briefing packet delivered on the email-confirmation turn (Resend 200), currency as whole dollars, [ASSESSOR] [VOICE] subject"* |
| **New chat version** | **`1eb3fd5c-5aff-46c8-b572-e3fe18bf966f`** — *"chat v13 - packet currency formatted as whole dollars, [CHAT] subject token"* |
| **Was `d28bbcb0` (the phone) repointed?** | **YES**, and read back rather than inferred. `b17c9a26` (v11) → `5d9df25c` (v13) at `2026-08-12T20:51:46Z`. `channelProfile` — which holds `channelType: GOOGLE_TELEPHONY_PLATFORM` — is **byte-identical** before and after. |
| **Was `d7bfbb93` (chat) repointed?** | **YES.** `26c3aebd` (v12) → `1eb3fd5c` (v13) at `2026-08-12T20:51:48Z`. `channelProfile` (`WEB_UI` + `webWidgetConfig`) **byte-identical**. |
| **A second voice version, `9227210b`, also exists** | Cut mid-plan, **never deployed, and must never be** — its mailer throws on every send. See *The defect this plan found by running it*. |
| **The packet's six sections** | `SUMMARY` / `ACTION` / `CLAIM` / `DIAGNOSTIC` / `RULES FIRED` / `FLAGS`, each exactly once, substituted values, **zero `{placeholder}` braces** — full text below |
| **New subject format** | **`[ASSESSOR] [VOICE] CLM-24xxx - Jordan Rivera`** and **`[ASSESSOR] [CHAT] CLM-24xxx - Jordan Rivera`** |
| **Customer-send count, asserted via `resolve_claim`** | **Exactly one**, on every run. `resolve_claim` fired once with `email_queued: true`. `send_claim_email` — the keyless reporter — is never used as the proof. |
| **Escalation-turn latency vs baseline** | **3.30 s on v11 → 9.53 s on v13** (+6.2 s), same script, same channel, 40 minutes apart. The send-away line is emitted **before** any tool call on that turn. |
| **If a caller hangs up after the email confirmation** | **Nothing is lost.** The packet, `escalate_to_human` and `end_session` all complete inside that same turn, server-side, before the caller could hang up on the silence — proven on a 3-turn conversation that ends exactly there. |

## The packet that was actually delivered

Voice, session `s07v-esc-a2`, claim `CLM-24257`, **434 characters**, Resend **HTTP 200**,
subject `[ASSESSOR] [VOICE] CLM-24257 - Jordan Rivera`. The customer name reads `[REDACTED]`
**in the conversation record only** — CES redacts PII in stored transcripts; the real name
went to the mailer.

```
SUMMARY: [REDACTED] Apple MacBook Pro 16" which is liquid damaged and does not turn on.
ACTION: Assess the claim details and photos to determine the outcome.
CLAIM: Customer: [REDACTED]; Policy: PDP100294; Device: Apple MacBook Pro 16"; Issue: liquid_damage; Amount: $3,000; Excess: $25; Total-loss flag: true.
DIAGNOSTIC: Customer reported: q1=no_power.
RULES FIRED: DL-3; DL-2.
FLAGS: Total loss indicated.
```

Chat, session `s07c-esc-d1`, claim `CLM-24858`, 518 characters, Resend **HTTP 200**, subject
`[ASSESSOR] [CHAT] CLM-24858 - Jordan Rivera`, with the same `Claim amount $3,000, Excess $25`.

`packet_text` handed to `send_case_record_email` is **byte-identical** to
`generate_case_summary`'s response on both channels — 434 == 434 and 518 == 518, exact string
equality, not normalised and not truncated. T-05-08 stays closed by evidence.

## The defect this plan found by running it

**The voice packet mailer had never delivered a single email.**

On the first live escalated run, every server-side assertion passed except delivery:

```
send_case_record_email -> { "sent": false, "status_code": 0,
  "error": "Requests.request() got an unexpected keyword argument 'timeout'",
  "subject": "[ASSESSOR] [VOICE] CLM-24850 - ..." }
```

The tool passed `timeout=20` to `ces_requests.post(...)`. The platform's HTTP shim does not
accept it, so the call raised on every invocation and the tool's own `except` branch returned
`sent: false` — **silently, without breaking the call**, which is exactly the behaviour it was
designed for and exactly what made this invisible. Chat's equivalent tool, written in 05-04,
has no `timeout` kwarg and has returned HTTP 200 since. The kwarg was introduced when the tool
was ported to voice.

**Why nothing before this caught it.** 05-07's own Task 1 says *"Send no test email in this
task"*. Quick task `260812-hhi` wired the voice step and verified it by **read-back only** — 30
assertions, all passing, none of which execute the tool. Every gate up to this point checked
that the right text was in the right place. **The first thing that could have found it was
calling it, and this plan is the first thing that did.**

Fixed as a Rule 1 auto-fix: one anchored `str.replace()` removing `\n            timeout=20,`,
asserted unique, −24 chars, one contiguous region, `compile()`-checked for syntax, key count
still 1, byte-identical read-back. Re-run: `sent: true`, `status_code: 200`.

> **Carry this forward:** a tool that swallows its own exceptions cannot be verified by
> reading it. `send_case_record_email` returns `sent: false` rather than raising precisely so
> a mail failure can never break a live phone call — a good design that also guarantees
> failure is silent. **Any future change to it must be proven by one live invocation, not by
> a read-back.**

## The regression canaries — every one held

Voice escalation `s07v-esc-a2` **28/28**, chat escalation `s07c-esc-d1` **28/28**, voice
auto-approve `s07v-appr-b1` **13/13**. All gated in code, all evidenced from a `toolCall` or
`toolResponse` object, never from transcript prose.

| Canary | Result |
|---|---|
| **`DECISION_SPEECH_EN` byte-identical, TEXT/API channel only** | ✅ one agent chunk, 197 chars, `spoken == resolve_claim.explanation` exactly, matching the 05-06 string modulo the per-run `CLM-` digits. Asserted on text, never on audio, per the 2026-08-11 normalisation warning |
| **The cross-sell still fires** | ✅ *"While I have you, I see you also have an Apple iPhone 16 Pro Max - would you like to add that to your cover?"* — **first observation ever on the voice text/API channel.** It came as its own turn after a neutral *"That's everything, thanks"*; live audio appends it to the send-away line instead. Both shapes are the beat working |
| **The send-away line is still spoken** (the v11 `838b6d2b` regression) | ✅ 250 chars on voice, 224 on chat, and the record shows the TEXT chunk emitted **before any tool call** on that turn |
| Deterministic tariff | ✅ `840 / 25 / 1500` with `rules_fired == ["DL-1"]` on approve; `3000 / 25 / 1500` with `DL-3`+`DL-2` on escalation |
| Exactly ONE customer email, via `resolve_claim` | ✅ `resolve_claim` fired once with `email_queued: true` on every run. `send_claim_email` never used as the proof |
| `DIAGNOSTIC_INCOMPLETE` guard | ✅ present in both v13 snapshots — this repoint also ships the 260811-suy fix to the phone for the first time |
| Barge-in / rest of `GTP_SURFACE` | ✅ untouched — no app-level setting, `audioProcessingConfig` or `languageSettings` was written by this plan |
| **No packet vocabulary spoken** | ✅ zero hits for the six headings (as headings), `assessor`, `case record`, `briefing packet`, `packet`, `[ASSESSOR]` across every agent text chunk on both channels |
| No cross-sell on the escalation path | ✅ zero |
| Packet files with NO further customer turn | ✅ 3-turn conversation ending exactly where a real caller stops; packet + `escalate_to_human` + `end_session` all on turn 3 |

**Escalation-turn latency: 3.30 s (v11) → 9.53 s (v13), +6.2 s.** `05-06-VOICE-BASELINE.md`
records no per-turn wall clock, so the baseline was taken by running the identical Scenario B
script against the **live v11 deployment** (`s07v-v11base-c1`, `appVersion b17c9a26`) 40
minutes before the repoint. Whether +6.2 s is *audible* is not settled by this number: the
agent speaks the 250-char send-away line before it calls any tool, which is roughly 16 s of
speech, so the work should run under the caller's ear — but that is an inference about the GTP
audio pipeline, not an observation. **The human on the phone is the judge.**

## Phone check — what the user needs to do

The number is in the CX Agent Studio console on deployment `d28bbcb0`; the runbook has the
blank. **Confirm the version first** — expect `5d9df25c…`, not `b17c9a26…`.

**One call, Scenario B.** Say, in order:
1. *"Hello, this is Jordan Rivera, policy P D P one zero zero two nine four"*
2. *"I spilled a full glass of water on my MacBook and now it won't turn on at all"*
3. *"Okay, thanks"* — then **stop talking and let it close.**

Listen for exactly three things:
- **The outcome is unchanged** — total loss, **$3,000**, routed to a specialist.
- **After the send-away line, is there a pause that feels wrong?** A second or two before the
  call closes is fine. Several seconds of dead air is not. Time it roughly.
- **Any leak** — it must not say "packet", "case record", "assessor", or read a heading aloud.

**Then check `akash.vinayak@nerdery.com`.** Two emails for the same claim:
`Claim CLM-24xxx - please reply with photos` and **`[ASSESSOR] [VOICE] CLM-24xxx - Jordan
Rivera`**. Open the packet: six headings, real values, **`$3,000`** and **`$25`** with dollar
signs and no `.0`, `DL-3` and `DL-2` under `RULES FIRED`, no `{placeholder}`. If you ran the
chat packet too, its subject reads `[ASSESSOR] [CHAT] …` — that is the whole point of the token.

**If anything is wrong, roll back to v11 `b17c9a26-3485-4658-9259-dfa4839a7977`** — one call,
in the runbook under *If something goes wrong (phone)*. Rolling back costs the packet beat on
the phone and nothing else.

## Deviations from Plan

1. **[Rule 1 — bug] The voice mailer's unsupported `timeout` kwarg.** Full account above. Not
   in the plan, not findable by read-back, and it meant the plan's headline deliverable did not
   work. Fixed inline, re-proven live.
2. **[Rule 3 — blocking] Voice v12 `9227210b` was cut before that fix and had to be
   superseded.** The plan says cut first, run second; the run then exposed a defect, so a second
   version `5d9df25c` was cut from the fixed draft and `9227210b` was left undeployed and named
   as never-deployable, mirroring how chat handled `838b6d2b`.
3. **[scope — orchestrator directive] Five conversations, not one.** The plan budgets one. The
   added chat scope, the auto-approve canary the cross-sell requires, the v11 latency baseline
   and the post-fix re-run needed five (16 turns, paced ~62 s, no 429, the authorised spaced
   retry never used, no retry loop).
4. **[deviation from the plan's design, inherited] The packet files on the email-confirmation
   turn, not the decision turn.** The plan's decision-turn design was rejected by quick task
   `260812-hhi` before this plan ran, on the grounds that it puts two tool calls in front of the
   customer hearing their outcome. This plan verified and shipped the same-turn design instead of
   re-authoring it.
5. **[measurement] The plan asks for the escalation turn's time "from the 05-06 baseline run".**
   That number does not exist — `05-06-VOICE-BASELINE.md` records no per-turn wall clock. Taken
   by re-running the script against the live v11 deployment instead, which is a stronger control.
6. **[Rule 1 — bug in this task's own tooling] Two assertion-script defects, both false
   failures.** (a) A photo-vocabulary check gated on any occurrence of "photo" flagged the
   packet's *"assess the claim details and photos"* — but **both** channels' claim emails ask the
   customer to reply with photos, so that prose is true, and the real threat (T-05-14) is an
   unresolved `{photo…}` token or a literal `photo_status`, which is what it now gates on.
   (b) A leak check on the bare word `CLAIM` flagged the send-away line *"Confirmation of your
   **claim**…"*; headings are now matched anchored (line-start + colon). Neither touched server
   state; both were corrected and every battery re-run before the repoint.
7. **[deferred, not fixed] The chat send-away line and email subject are voice-shaped.** On chat
   the agent still asks the customer to reply with photos — and the subject still reads *"please
   reply with photos"* — even when a photo was already uploaded and assessed in the conversation.
   Correct on voice, which has no image path; on chat it asks for what the customer just gave and
   undercuts the multimodal beat. Out of scope here; a separate task owns branching the sentence
   and the subject on whether a photo was already assessed. **Voice must keep the current wording.**

## Independent user evidence, chat v13

Conversation `dfMessenger-05b6446b-d6c4-4faf-9ac3-3e0b29d7a0cd` (`2026-08-12T22:12:34Z`,
MULTIMODAL, PDP100583, 9 turns) drove the full happy path through the real widget on v13:
photo uploaded → `assess_screen_crack` → `CRACK_CONFIRMED` → **$420** under **$750** → decision
card → single email → cross-sell. **The user tapped the `Add it` button** — turn 8 is
`"Yes please, add it to my cover."`, byte-identical to the configured `utterance`. A tap is
only possible if the button drew, so **05-02's last open on-screen item is closed**; the user
separately confirmed *"the card renders fine."* The send-away line was spoken on that run too,
so the v11 `838b6d2b` turn-silencing regression did not recur on v13.

## Threat register outcomes

| Threat | Outcome |
|---|---|
| T-05-12 packet spoken aloud via TTS | **Mitigated server-side.** Zero hits across every agent text chunk on both channels. Audio confirmation is the human phone check |
| T-05-13 audible dead air from two extra tool calls | **Measured, not asserted.** +6.2 s, landing behind a 250-char spoken line emitted before any tool call. Judged by ear, rollback named |
| T-05-08 `packet_text` paraphrased across the model turn | **Closed by evidence.** 434 == 434 and 518 == 518, exact equality |
| T-05-14 photo vocabulary on a channel with no photo | **Mitigated.** No `photo_status`, no `{photo…}` token, no photo literal in either `case_summary` instruction. Forward-looking prose about photos is correct on both channels |
| T-05-10 double customer send | **Mitigated.** `resolve_claim` once with `email_queued: true` on every run |
| T-05-11 the edit reverts v1→v11 hardening | **Mitigated.** `claim_intake` diffed against the v11 snapshot: three changed regions, all either the `DIAGNOSTIC_INCOMPLETE` paragraph, the `- HUMAN_REVIEW:` bullet, or the new step. The `- AUTO_APPROVE:` line and the whole Cross-sell step are byte-unchanged |
| T-05-01 Resend key | **Mitigated.** No tool source was read, echoed or copied; edits were anchored replacements printing only booleans. `grep -rE 're_[A-Za-z0-9_]{20,}' .planning/` returns no match |
| T-05-05 a deployment serves an unproven build | **Mitigated.** Every conversation ran against the DRAFT; the repoint re-executed all three batteries in code and re-checked snapshot-vs-draft byte equality before either PATCH |
| T-05-09 / T-05-07 cross-app writes | **Mitigated.** Client-level gate refused any URL naming the fork `9ae7a0c3`; zero calls of any method to it |
| T-05-04 quota | **Mitigated.** 5 conversations, 16 turns, ~62 s pacing, no 429, no retry loop |

## Known Stubs

None.

## Still open after this plan

1. **Nobody has made a phone call on v13.** That is the blocking human check above.
2. **Nobody has seen the voice packet in the inbox.** Resend returned 200; a status code is not
   an inbox, and the conversation record redacts the customer name, so only the mailbox can
   prove the packet named Jordan Rivera.
3. **The chat "reply with photos" wording** — deviation 7, owned by a separate task.
4. **Voice v12 `9227210b` exists and is poison.** Never deploy it.

## Self-Check: PASSED

- `05-07-SUMMARY.md` — created
- `.planning/spec/DEMO-RUNBOOK.md` — modified: phone version table now names v13 `5d9df25c`
  and the v11 `b17c9a26` rollback, the packet beat is under Scenario B, both subject tokens are
  documented, the key-revocation note names voice's `send_case_record_email`
- `.planning/STATE.md` — modified
- Voice deployment `d28bbcb0` re-read after the PATCH → `5d9df25c`, `GOOGLE_TELEPHONY_PLATFORM`
- Chat deployment `d7bfbb93` re-read after the PATCH → `1eb3fd5c`, `WEB_UI`
- `grep -rE 're_[A-Za-z0-9_]{20,}' .planning/` → no match
- **Commits: none by instruction.** The tree is deliberately left dirty.
