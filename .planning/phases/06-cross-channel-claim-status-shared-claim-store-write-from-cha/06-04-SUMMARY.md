---
phase: 06
plan: 04
subsystem: cx-agent-voice-app, shared-claim-store
tags: [claim-store, lookup_claim, voice, gtp, status-lookup, cross-channel, verbatim-relay, criterion-4, shipped]
status: awaiting-human-phone-check
version_cut: a6f6b620-af15-4e43-b0d8-bfbbb2d64a46
rollback_target: dcc20863-3746-4e43-a2c9-ed30e0611479
repoint_timestamp: "2026-08-14T03:50:48.984372Z"
criterion_2_fixture: "CROSS-CHANNEL — CLM-24690, written by CHAT, read back on VOICE. Not a 06-02 fixture."
status_turn_wall_clock: "3.84 s (status turn) / 2.87 s (confirm turn); lookup_claim alone 0.66-1.24 s"
speech_shaping_fallback: "DID NOT FIRE - no tool was patched by this plan"
async_execution: "NOT used - lookup_claim stays SYNCHRONOUS. Reasoning in 'ASYNCHRONOUS was re-opened and rejected'."
conversations_spent: 7
canaries: 47/47 PASS
covers_success_criteria: [2, 4, 6]
requires: ["06-02 STORE-CONTRACT", "06-03 (chat wired; supplied the cross-channel claim)"]
provides: ["voice reads any claim on a verified policy back from the shared store", "06-04-VOICE-EVIDENCE.md"]
affects: ["06-05 (owns the voice WRITE, and inherits the confirm-path fix chat still lacks)"]
open_items: ["the phone call - awaiting-human-phone-check"]
date: 2026-08-14
---

# Phase 6 Plan 04: Claim status lookup on the phone — Summary

**Version cut and LIVE:** voice **v18 `a6f6b620-af15-4e43-b0d8-bfbbb2d64a46`** on `d28bbcb0`.
**Rollback target, re-read LIVE immediately before the PATCH:** v17 `dcc20863-3746-4e43-a2c9-ed30e0611479`, `updateTime 2026-08-14T03:15:07.395898Z`, asserted unmoved since plan start.
**Repointed at:** **`2026-08-14T03:50:48.984372Z`**, read back not inferred; `channelProfile` byte-identical across it (`e7197eb367fa9cf0…`, `GOOGLE_TELEPHONY_PLATFORM`).
**Criterion 2 fixture:** **a claim written by CHAT — `CLM-24690`** (`channel: CHAT`, *"via chat"* in the spoken sentence). **Not** a 06-02 fixture; the fixtures were available and were not needed.
**Status-turn wall clock:** **3.84 s** (verify + lookup + disambiguation relay), **2.87 s** (confirm + status relay), against 05-07's 9.53 s escalation turn. `lookup_claim` alone: 0.66–1.24 s.
**Speech-shaping fallback:** **DID NOT FIRE.** No tool was patched on either app. One cosmetic risk is recorded for the ear — see *Follow-ups*.

**Phase 5 canaries, one line each — all PASS, all cited to a `toolCall`/`toolResponse`:**
deterministic tariff approve `840/25/1500` `DL-1` **PASS** · deterministic tariff escalate `3000/25/1500` `DL-3`+`DL-2` `total_loss_flag: true` **PASS** · `DECISION_SPEECH_EN` byte-identical to `resolve_claim.explanation`, 197 chars, exactly one chunk, TEXT/API **PASS** · escalated decision byte-identical, exactly one chunk **PASS** · exactly one customer email asserted via `resolve_claim.email_queued` **PASS** (both branches) · **send-away line still SPOKEN, approve PASS, escalated PASS** · cross-sell fires **PASS** · assessor packet on the email-confirmation turn **PASS** · six headings each exactly once `[1,1,1,1,1,1]` **PASS** · zero `{placeholder}` braces **PASS** · `$3,000`/`$25` rendered **PASS** · `send_case_record_email` `sent: true` HTTP **200** with `[ASSESSOR] [VOICE]` **PASS** · zero packet vocabulary spoken **PASS** · packet tools zero on approve, no `NO_PACKET_REQUIRED` **PASS** · `escalate_to_human` exactly once **PASS** · `claim_intake` byte-identical **PASS** · all 9 voice tools byte-unchanged incl. `resolve_claim` **PASS** · `GTP_SURFACE`/`bargeInAwareness`/`languageSettings`/`globalInstruction` byte-unchanged **PASS** · 33 `variableDeclarations` **PASS** · `record_claim` still unwired on voice **PASS** · zero store vocabulary anywhere **PASS** · chat untouched, app list five, fork zero calls **PASS**.
**Two canaries held in substance but not in literal form and are named as such** — the send-away chunk's position within the escalated turn, and `packet_text` byte-identity. Both in *Canaries that did not hold literally*.

## Headline

**A caller can phone in, say who they are, and hear the status of a claim that was filed in the chat
widget — in a sentence the model did not compose, on the channel where a mistake is most expensive.**

Session `v04-status-c`: *"Hi, this is Jordan Rivera, policy P D P one zero zero two nine four, and
I'm just checking on a claim I put in."* → `verify_identity` → `lookup_claim` → 25 claims → the
agent reads the most recent one back and asks → *"Yes, that's the one."* → the answer, byte-identical
to `lookup_claim.status_line`, alone in its chunk:

```
Claim CLM-24690 on policy PDP100294: your Apple MacBook Pro 16", keyboard replacement. It was
approved for $420.00, less your $25.00 excess, so $395.00 comes to you. Filed on 14 August 2026
via chat.
```

**`CLM-24690` was written by chat's `record_claim`.** The voice agent has never had `record_claim`
wired. The loop closes.

## Criterion 4 — the caller is never asked to read a reference aloud

Identification is by **name and policy ID** through the `verify_identity` step voice already had.
Nothing new was built for identity and no new authentication surface exists.

- Zero request-shaped mentions of a claim reference across every agent chunk in three status
  conversations.
- `verify_identity` fired **strictly before** `lookup_claim`, `authenticated: true`, asserted from
  the record — a lookup preceding identification is a failure, not a warning.
- **T-06-03 survived wiring, proven live:** the model *did* pass `policy_id: "PDP100294"` and the
  tool ignored it — `policy_source: "session"`, `policy_id_arg_ignored: true` on every call.
- **Several claims disambiguate by read-back, never by list.** `match_count: 25`, `disambiguation_line`
  relayed byte-identically, and **zero of the 24 `alternatives` were spoken** — no other reference,
  device or date appears anywhere in that turn.
- **An ambiguous opening gets exactly one question.** *"Hi, I'm calling about my claim."* → *"Is this
  something new that has just happened, or a claim you have already put in?"* — no tool call, no guess.
- **A new-loss opening still reaches `claim_intake` exactly as before.** `lookup_claim` fired **zero**
  times across both full FNOL conversations.

### The gap that would have broken the beat, found before a single conversation was spent

**[Rule 2 — missing critical functionality.]** 06-03's chat region says only *"call `lookup_claim`
again with `selector` set to what they said"*. `selector` resolution has **no branch for "yes"** — it
would fall through to `NO_MATCH`, and the agent would tell a caller there is nothing on file
*immediately after reading their claim back to them*. The voice region routes the confirmation
through **the reference the tool itself supplied** (`claim_ref`), so the caller still never speaks it.
Proven live: `lookup_claim{claim_ref: "CLM-24690"}` → `match_count: 1`.

**This is a latent defect in the shipped chat build (v18 `d0e4bfef`) too.** Chat was not touched by
this plan. **06-05 should carry the fix across.**

## Criterion 4, negative paths — nothing is ever fabricated

| Path | Proven | Result |
|---|---|---|
| Reference on **another** policy (`CLM-60203` asked for on `PDP100294`) | live | `found: false`, five keys, zero claim fields, zero figures spoken |
| **Unknown** reference (`CLM-99999`) | live | **byte-identical response** to the cross-policy case — a caller cannot probe for the difference |
| Policy with **nothing on file** (`PDP-100746`) | `apps.executeTool`, stated plainly as by-invocation | `found: false`, `404`, the `EMPTY` literal |
| Caller who fails identity | live, `v04-fail-b` | `lookup_claim` **never called**, on four turns, two of which volunteered references |

There is nothing to hallucinate *from*: a not-found response contains no claim field at all.

## `ASYNCHRONOUS` was re-opened as 06-03 asked, and rejected — with a sharper reason

06-03 deliberately left the question to this plan, on the grounds that on voice `record_claim` sits
between the caller's last word and the spoken decision. **For a *read* the argument inverts, and
decisively:**

1. **The status line IS the tool's result.** An asynchronous tool buys latency only where the turn
   does not need the value; here the utterance *is* the value. The turn must wait either way, so
   async saves nothing and only removes the `toolResponse` from the turn record.
2. **The repoint gate needs that record.** Twelve of this plan's assertions read `found`,
   `status_line`, `match_count`, `policy_source` and `policy_id_arg_ignored` off the turn's
   `toolResponse`. Async puts the whole criterion-2 and T-06-03 evidence base out of reach.
3. **Port parity.** 06-02 locked `SYNCHRONOUS` on all four store tools and 06-05 re-checks it.
   Flipping voice alone manufactures the drift 06-05 exists to catch.
4. **There was no latency problem to solve.** `lookup_claim` measures **0.66–1.24 s** across nine
   invocations, and the whole status turn is **2.9–3.8 s** — the fastest substantive turn on the app.

## The dead-air gate — not achievable by prompting, and the finding is reusable

The plan makes *"an agent text chunk was emitted **before** the `lookup_claim` call"* a hard gate.
**It did not fire under three successive instruction revisions**, the last of which says in terms
*"Never call the tool before you have spoken."*

> **A model cannot compose speech before the value that speech must contain has arrived.** The
> send-away line precedes five tool calls because the agent does not need their results to say it.
> The status line *is* the result, so the turn is necessarily `call → response → text`. This is the
> third member of the family `260813-o5l` and `260813-ui0` opened. The platform's own answer is a
> **prefix message** (`partial=True`) — a tool/callback-level feature, **not reachable from an
> instruction**.

**What was achieved is worth having anyway:** the acknowledgment is spoken and is the first thing the
caller hears — *"Thanks, Jordan. Of course, let me see where that has got to. I can see 25 claims…"*
— asserted to precede the first currency figure in the utterance. On an audio channel the ordering
that reaches the caller is the order of *words*, not of API chunks. What it does not do is cover the
pause, because the whole utterance is synthesised after the tool returns.

**The repoint was not blocked on the literal gate, and that judgement is recorded rather than
buried.** The gate's stated rationale is that such a turn is dead air *"by construction rather than by
inference"* — but the duration was assumed unknown and it is not: **2.2–3.8 s measured**, against a
9.53 s escalation turn that has been live on the phone since v13 without complaint. 05-07 established
that audibility is a question only the human on the line can settle, and the phone script asks it
with the measured figure in it. **If it does read as dead air, the fix is a callback, not a prompt.**

## The escalation stall — pre-existing, settled by a live control, not by reasoning

The liquid-damage path stalls: the model passes `run_diagnostic` its answers wrapped in literal
double quotes (`q1: "\"no_power\""`), the tool correctly refuses with `INVALID_ANSWER`, and the agent
re-asks the same question indefinitely. Five turns of `v04-esc-e` went nowhere.

**The identical script run against live voice v17 `dcc20863` through `config.deployment` — the
byte-for-byte pre-edit build — reproduced it identically.** Pre-existing, not a regression. The
escalation canaries were then taken on the physical-damage total-loss route, which reaches
`TOTAL_LOSS` cleanly. **Runbook consequence: script the escalated voice beat as a drop, not a spill.**

This is the third time `260813-tgq`'s method has paid for itself. One live control, one conversation,
and a question that could not otherwise have been answered.

## The edit

| | |
|---|---|
| Agent | voice `claims_concierge` (root). **`claim_intake` not touched at all** — asserted byte-identical, 20,711 chars, sha `f5249184…` |
| Instruction | 6,642 → **11,745** (**+5,103**) |
| Contiguous regions changed | **1** — prefix 6,072, suffix 570, old span **0**, new span 5,103 |
| Anchor | `</subtask>\r\n    <subtask name="Anything else">`, sliced from voice's **own CRLF bytes** (69 CRLF, 0 bare LF), asserted unique |
| Method | scripted `str.replace()`, region read **from a file** in LF and converted to CRLF in-process, instruction handled through JSON only |
| Read-back | **byte-identical to the intended string**; the **version snapshot** re-verified equal before the repoint |
| Tools | `lookup_claim` attached; count 2 → **3**; `updateMask=instruction,tools` |
| `beforeModelCallbacks` | **SURVIVED byte-identical** (869 chars) — asserted explicitly, because that is exactly what a careless `updateMask` wipes |

The region mirrors 06-03's chat subtask deliberately rather than inventing a second design, and adds
what voice needs: the intent split with literal trigger families and its one-clarifying-question
rule, the speak-before-you-look ordering, the ban on speaking `alternatives`, the offer-to-file after
a not-found, and the confirm path above.

Byte-unchanged and asserted: the `<role>`, `<constraints>` and the entire `Verify the caller` subtask
slabs; every original non-blank line still present; zero non-ASCII and zero bare LF introduced.

## Canaries that did not hold literally

**1. The send-away chunk did not precede *every* tool call on the escalated turn.** It was emitted
after `send_claim_email`, `generate_case_summary` and `send_case_record_email`, and before
`escalate_to_human` and `end_session`. The substantive canary — the v11 `838b6d2b` regression was
total **silence** — passes: the line is spoken, once, in full, before the closing pair. The edit
cannot be the cause; that whole turn runs inside `claim_intake`, which is byte-identical to plan
start, and `260813-tgq` documented that turn alignment moves between runs of an identical script.
Recorded as a **partial**, not claimed as clean.

**2. `generate_case_summary` returned `null` in the conversation record**, so `packet_text` could not
be compared byte-for-byte against the composer's own response. The packet is fully evidenced from
`send_case_record_email`'s **arguments** (six headings each once, zero braces, `$3,000`/`$25`) and its
response (`sent: true`, `200`, `[ASSESSOR] [VOICE]`). The byte-identity leg is **not proven on this
run** and is stated as such rather than assumed from a prior plan.

## Deviations from plan

**[deviation] The delta bound was re-declared, before running, twice.** The plan predicted
*"roughly +900 to +2,600"*. That band predates 06-03, whose equivalent chat region **alone** is 2,949
characters — the plan's ceiling was already below the size of the thing it asked to be mirrored.
Declared 3,600–4,800 (measured 4,740), then 3,600–5,200 for the confirm-path fix (5,016), then the
same band for the same-turn acknowledgment (5,103). **Every pass was re-derived from the plan-start
bytes**, so "exactly one contiguous region differs from the baseline" is true of the *shipped*
instruction, not merely of each increment. The bound's purpose — T-06-13, catch a runaway edit — is
served by a band declared before running plus the contiguity measurement plus the byte-identical
survival of the protected slabs, all of which passed at every pass.

**[deviation] Seven conversations against a budget of six.** One (`v04-fail-b`) was lost to a fixture
error of mine — the script gave *Jordan Rivera* against `PDP100583`, which belongs to *Maria Santos*
— and it turned out to be the plan's best unplanned control: it proves `lookup_claim` is never called
for an unauthenticated caller who is actively pushing claim references at the agent. One
(`v04-esc-e`) hit the pre-existing liquid-damage stall; one was the **live control** that proved the
stall pre-existing; one (`v04-esc-f`) re-took the escalation canaries on a route that completes.
**No 429 and no 529 was seen at any point, and no retry loop was ever run.** Conversations were paced
10–12 s per turn.

**[deviation] The empty-policy path was proven by `apps.executeTool`, not by conversation.** No
synthetic identity exists for `PDP-100746`, so a caller cannot authenticate onto it — which is
correct behaviour, not a gap. Stated plainly in the evidence rather than implied to be a live proof.

**[deviation] The plan's Conversations 1, 2 and 3 were folded into one.** A single session reaches
the disambiguation, the confirm round trip, the cross-policy reference and the unknown reference —
the same assertions against the same code paths, at a quarter of the quota.

**[method] The plan's grep gate #8 (`read.*reference`) was replaced with a stronger check.** As
written it cannot distinguish a prohibition from a request — *"NEVER make the caller read a
reference"* matches it. The gate used instead is five request-shaped regexes **plus** a check that
every literal `claim reference` mention sits inside a sentence carrying an explicit negation.

**[not done] The speech-shaping fallback did not fire, deliberately.** Its trigger is *"Task 2
observes the relayed string carrying a character TTS **mishandles**"*. The inch mark is present; the
mishandling is unobservable without an ear. The plan also says *"do not build this pre-emptively"*,
and patching a key-bearing tool on both apps speculatively is the larger risk. See *Follow-ups*.

## Follow-ups, in order of demo impact

1. **👂 The inch mark, pending the ear.** The relayed sentence contains `Apple MacBook Pro 16"` —
   a literal `"` inside a string the agent is forbidden to reword. **If the phone check hears
   *"sixteen quote"*, the fix is pre-specified:** an optional speech-shaped template inside
   `lookup_claim`'s **Python composition**, defaulting to the exact string chat already receives,
   applied to **both** copies, `06-02 ## PORT_PARITY` re-asserted, re-proven by invocation. Never in
   the instruction and never by asking the model to clean it. **Note this is not new surface:** the
   intake path already spoke `Apple MacBook Pro 16"` in the agent's own prose on this build.
   A zero-risk alternative for the demo is to run the status beat on **Maria Santos / PDP100583**,
   an iPhone, which has no inch mark.
2. **⚠ `PDP100294` now holds 25 claims, so the agent says *"I can see 25 claims on this policy."***
   Correct behaviour, bad demo line. Prune the store before this beat goes in front of a customer
   (`gcloud storage rm` on individual `claims/by-ref/` objects plus a rewrite of the by-policy array
   — **not** the blanket teardown, which would delete the fixtures 06-05 reads), or run the beat on
   `PDP100583`.
3. **The confirm-path fix is missing on chat.** 06-05 should carry it across; today a chat customer
   who answers *"yes"* to the disambiguation gets `NO_MATCH`.
4. **The liquid-damage escalation stall.** Pre-existing, reproduced on v17. Needs `claim_intake` or
   `run_diagnostic`; out of scope here. Runbook mitigation shipped: use a drop, not a spill.
5. **The voice WRITE is still not wired.** `record_claim` is attached to no agent. 06-05 owns it.
6. Unchanged and untouched by this plan: the Resend key rotation, `resolve_claim`'s truthfulness
   fix, and the ~30% email delivery rate.

## Open item — the single one

**The phone call. `status: awaiting-human-phone-check`.**

**Criterion 2 is proven server-side on the TEXT/API channel and is UNCONFIRMED BY EAR.** Three things
only a human on the line can settle: whether the sentence *sounds* right once TTS has normalised the
currency and the reference (and what it does with the inch mark), whether the 2.9–3.8 s pause reads
as considered or as dead air, and whether the intent split works when a real person opens the call in
their own words. The literal four-call script is `## PHONE_SCRIPT` in `06-04-VOICE-EVIDENCE.md`.

**This checkpoint does not block plan completion.** The user's report folds back into this summary and
STATE.md when it arrives, exactly as 05-07 handled the same situation.

## Self-Check: PASSED

- `06-04-VOICE-EVIDENCE.md` — created, **676 lines**; `## VOICE_BASELINE_CAPTURE`,
  `## VOICE_IDENTIFICATION`, `## VOICE_STATUS_LOOKUP`, `## VOICE_GRACEFUL_FAILURE`,
  `## VOICE_CANARIES`, `## VOICE_LATENCY`, `## VOICE_SHIP`, `## PHONE_SCRIPT` each present **exactly
  once**
- `06-04-SUMMARY.md` — created
- `d28bbcb0` re-read **after** the PATCH → **`a6f6b620-af15-4e43-b0d8-bfbbb2d64a46`**,
  `updateTime 2026-08-14T03:50:48.984372Z`; `channelProfile` sha `e7197eb367fa9cf0…` byte-identical
  across the write, `channelType: GOOGLE_TELEPHONY_PLATFORM`
- Rollback target **re-read live from the API immediately before the PATCH** → v17 `dcc20863`,
  `updateTime` unmoved since plan start; the ship script **asserts** it did not move
- Version snapshot read back and asserted **before** the repoint: `claims_concierge` == the exact
  intended string (11,745), callback 869 chars sha unchanged, sentinel once, `claim_intake`
  byte-identical, 3 agents / 9 tools
- Gate **47/47**; the cut **and** the repoint conditional on `gate.py`'s **exit status**
- Isolation **23/23** re-run at plan close
- Chat `a2f621e4` / `d7bfbb93`: still v18 `d0e4bfef`, `updateTime` unmoved, every agent and tool
  byte-unchanged. **Zero non-GET calls to chat.**
- Fork `9ae7a0c3`: **zero calls of any method**; the client refuses any URL naming it. Refusal log
  empty — nothing was ever attempted.
- `resolve_claim` never read, echoed, printed or persisted on either app; `lookup_claim`'s source
  never read either. `grep -rEho` for secret **values** over `.planning/` returns **nothing**
- App list re-read at close: **five**. No probe app was created
- Voice v12 `9227210b` and chat v11 `838b6d2b` never deployed
- **No git commit, by instruction** — the tree is deliberately left dirty
