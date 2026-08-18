---
task: 260817-taq
title: The voice agent invented a customer's claim — diagnosed, and made structurally impossible
subsystem: cx-agent-voice-app
tags: [fabrication, hallucination, lookup_claim, claim-status, after-model-callback, audio-channel, voice, shipped]
date: 2026-08-17
status: complete
voice_version_cut: d8fe1e86-0af6-4eed-adb4-1588d08cb27f
voice_rollback_target: e484ce3e-00d6-4fae-b180-d789645f7280
voice_repoint_timestamp: "2026-08-18T02:47:19.748978Z"
voice_repointed: true
chat_touched: false
chat_exposed: false
gate_draft: 130/130
gate_deployed: 14/14
callback_unit_battery: 22/22
fabrications_found: 3
conversations_spent: 19
requires: ["06-04 (built the status branch this repairs)", "06-02 STORE-CONTRACT", "260813-ui0 (callback method)"]
affects: ["06-05", "chat — recommended to adopt the same guard for parity, NOT done here"]
---

# 260817-taq — the phone invented a claim, and it had done it before

**Voice v20 `d8fe1e86-0af6-4eed-adb4-1588d08cb27f` is LIVE on `d28bbcb0`**, repointed off v19
`e484ce3e-00d6-4fae-b180-d789645f7280` at **`2026-08-18T02:47:19.748978Z`**, read back not inferred,
`channelProfile` sha `e7197eb367fa9cf0` byte-identical across the write, `GOOGLE_TELEPHONY_PLATFORM`.

The rollback target was **re-read live from the API immediately before the PATCH** and the ship
script aborts if either the serving version *or* its `updateTime` has moved since task start. The
cut **and** the repoint were both gated in code on a **130-assertion battery's exit status**, run
twice — once before the cut, once again immediately before the repoint — with an
**11-assertion version-snapshot check between them** and a **14-assertion deployed battery** after.

**Chat was never written to.** `a2f621e4` received `GET` and `runSession` calls only; `d7bfbb93`
still serves v20 `8a95ab02` with `updateTime` unmoved at `2026-08-14T11:40:17.051321Z`. Fork
`9ae7a0c3` received **zero calls of any method** — the client refuses any URL naming it.

---

## The headline: it was not one incident, it was three, and one of them is worse

The reported call is real and the record confirms it exactly. But scanning **every AUDIO
conversation on the voice app** turned up two more, both already in the record, both unreported:

| Conversation | When | Version | `lookup_claim` calls | What the caller was told |
|---|---|---|---|---|
| `103oKK2OKRXQSO_s8w8A275Ig` | 08-14 10:41 | v18 `a6f6b620` | **1** | correct — `CLM-24599`, straight from the tool |
| `0810JFoa6NbToe-t_xyPIpITQ` | 08-14 12:22 | v19 `e484ce3e` | **0** | *"claim reference **CLM88342** was approved … we sent a **payment of 825** that same day"* |
| `103NCt8cmmETeC6fea0vnZQfg` | 08-14 17:14 | v19 `e484ce3e` | **0** | *"**I can't seem to find any claims for this policy.** Would you like me to … start a new claim?"* |
| `065rK0K0SsbRSWKzNfKiMoeZA` | 08-18 01:58 | v19 `e484ce3e` | **0** | *"claim reference **CLM92837** … was approved and your **device was shipped** on …"* |

**Across the four status calls ever made to the phone, `lookup_claim` ran once. Every one of the
three skips produced an invented answer.**

The third one is arguably the worst and nobody had noticed it: **`103NCt8c` invented a
*not-found*.** `PDP100294` holds **27 claims**. The agent told the customer there were none and
offered to open a duplicate. A fabricated denial is as damaging as a fabricated approval and it
fails silently — there is no wrong number for anyone to catch.

---

## What I proved, and what I disproved

**Reproduction: the defect is AUDIO-only and does not reproduce from a script.** I ran the exact
failing shape against the voice draft over `runSession` and it behaved correctly every time
(R1, R2, and later VA/VE/VE2/P1/P2 — **seven** clean text runs). So the reproduction is not a
script; it is the conversation record, three times over, on the channel that actually broke.
**Stated plainly rather than dressed up: I never saw it fail with my own hand.**

### The mechanism, established from the span tree — not the one I expected

`065rK0K0` turn 2, from `rootSpan.childSpans`:

```
VAD  8.32s -> Callback -> LLM 0.795s -> Tool(verify_identity) 0.035s
                       -> Callback -> LLM 7.599s -> Callback -> TURN ENDS
```

**The second LLM round happened.** It ran for 7.6 seconds after the tool response came back. The
turn was not truncated and nothing was thrown away — **the model got its round and chose to speak
instead of calling the tool.**

That kills the leading hypothesis in the brief. This is **not** `260814-8rv`'s batching /
turn-termination mechanism, and 8rv's dependency-statement fix would have been a no-op here.

**My own follow-up hypothesis died too, and by evidence rather than argument.** I supposed that on
audio a text chunk terminates the turn. I tested it against 166 agent turns in 22 AUDIO
conversations: **17 turns emit text and then call a tool in the same turn**, including on v19
itself, including full `TEXT → CALL → RESP → TEXT → CALL → RESP → TEXT` turns. Text does not end an
audio turn. Recorded because it would have been a plausible and completely wrong thing to ship.

### What it actually is

06-04 added this to voice's status region (voice only — chat never had it):

> **SPEAK BEFORE YOU LOOK.** In the same turn, and BEFORE you call `lookup_claim`, say one short
> line … *"Of course, let me see where that has got to."* … **The order on that turn is: your line
> first, the tool second, the tool's line third. Never call the tool before you have spoken.**

**The instruction requires the model to compose speech in the very step where it should be emitting
the tool call.** On the live audio model that acknowledgment is generated as ordinary speech — and
speech generation does not stop at an acknowledgment. *"Let me look that up"* is not a socially
complete utterance; it demands a result in the same breath. So the model supplies one, and since
the tool has not run, it invents it. By the time the utterance ends, the model has already answered
and there is nothing left to call the tool for.

The wording of the three failures is the fingerprint. All three free-composed the acknowledgment
and ran straight on:

- *"One moment while I look that up. **I have found your claim.** Claim reference CLM92837 …"*
- *"Got it. One moment while I look into that claim for you, … **I found it** — claim reference CLM88342 …"*
- *"Thank you. One moment while I look into that for you… **I'm sorry, but I can't seem to find any claims** …"*

The one success said the instruction's literal example — *"Certainly, let me see where that has got
to."* — and **stopped**, then called the tool. That is a sampling outcome, not a guarantee: it held
once in four.

**This is the sharpest part of the finding.** 06-04 recorded that it could not make a
text-before-the-tool-call gate fire on the TEXT/API channel under three successive instruction
revisions, and shipped the ordering anyway as a harmless unfulfilled preference. It was not
harmless. **On the channel 06-04 could not test, the instruction fired exactly as written — and
firing is the failure.** The pause it was written to cover is 2.9–3.8 s, against a 9.5 s escalation
turn that has been on the phone for weeks without complaint.

---

## The fix — one half removes the trigger, the other half makes fabrication impossible

The brief was right that wording alone is the weakest remedy, and this project has the scars to
prove it. So the fix is in two parts, and **only the second is load-bearing**.

### Part 1 — the instruction: the tool goes first, and nothing is said before it

`claims_concierge` **11,745 → 12,699 (+954)**, **exactly one contiguous region** against the
plan-start bytes (prefix 8,174 / suffix 2,975; old span 617 / new span 1,571). The
`SPEAK BEFORE YOU LOOK` paragraph is **replaced**, not appended to — the trigger is gone from the
build, not out-argued:

- `lookup_claim` is the **first** thing that happens on the turn — before any acknowledgement,
  filler or preamble; a short silence while it runs is named as *correct and expected*, and a
  sentence spoken into that silence is named as the thing that can only be filled by invention.
- The agent is told in terms that it has **no claim information of its own** — until the tool
  returns it does not know whether a claim exists, its reference, its cost, its decision, its date,
  or whether anything was paid, posted or shipped; and if asked to repeat one, it repeats only what
  the tool gave it. That last clause covers the compounding — *"It's CLM92837."*

### Part 2 — the structural guard, in the callback that already does this job

`claims_concierge` already carries an `afterModelCallbacks[0]` whose stated purpose is stopping the
root agent claiming things it has no basis for, including a comment reading *"The concierge has no
device information yet and must not invent one."* **The claim-status fabrication is the same family,
one step later in the conversation**, so the guard went there rather than into new surface.

`afterModelCallbacks[0]` **2,092 → 4,717 (+2,625)**, a **pure insertion** — the assertion is literal:
`new.replace(gate, "", 1) == old`, so **no original byte was deleted or moved**, and the three
pre-existing guards survive verbatim.

The gate:

1. records in session state every `lookup_claim` **call** the model emits;
2. refuses any utterance carrying a **claim fact** — a `CLM`+digits reference in any spelling, or a
   decision/despatch/payment/no-claims-on-file assertion — while no such call has been made, and
   replaces it with the store contract's own `STORE_DOWN` literal;
3. sits **before** the existing `auth_status == "verified"` early return, because the fabrication
   happens *after* the caller is verified — putting it after would have made it dead code.

**Why this and not the brief's other two candidates.** The dependency statement is the fix for a
mechanism I disproved. Relaying a tool-built string verbatim was **already** in place and is exactly
what the defect walked around — the fabrication happens *before* the tool, so there is no tool
string to relay. The only mechanism that reaches the failure is one that can refuse the model's
output, and that is the callback.

### What it needs, and what it deliberately does not

Established by three temporary introspection probes on the voice **draft** (never served; restored
byte-identically each time, sha `402a16196ae7ecdd` re-asserted):

- `callback_context.state` is a **plain writable dict, and an undeclared key persists across turns**
  (`PRIOR=None → WROTE=1 → PRIOR=1 → WROTE=2`). So the guard needs **no new declared variable, no
  app-config change, and no tool patch** — the **33 `variableDeclarations` canary is untouched** and
  `lookup_claim`, which carries the store HMAC key, was **never read and never patched**.
- `callback_context` also exposes `events`, `get_last_agent_output`, `set_variable`/`get_variable`.
  Not needed, recorded for the next task.

**Platform finding, and it cost me a probe to learn: callback code is CACHED.** Probe 2 returned
probe 1's output despite a verified read-back of probe 2's bytes. **A callback patch is not
immediately live.** Every later probe and verification was spaced 60–100 s. Any future task editing
a callback must not test it immediately after patching, or it will verify the previous code and
believe it verified the new.

### The stated bound

The gate covers **the tool-never-called case**, which is all three observed fabrications. It does
**not** catch a fabrication invented *after* a genuine `lookup_claim` response, because an
`afterModelCallback` sees only the model's output, not the tool's. That is asserted explicitly as a
known bound in the unit battery rather than left for someone to discover. Closing it would mean
capturing the tool response in `beforeModelCallbacks` — the 869-char callback that owns the
greeting, the dropped call and *"Are you still there?"* — and that trade was not worth taking today.
In practice the residual risk is small: a not-found response contains **no claim field at all**
(06-02), so there is nothing to embroider.

---

## The regression I caused, caught by a live control, and fixed before shipping

**Pass 1 of the instruction broke the byte-identical relay**, and I only know that because I ran the
control instead of trusting the edit.

Removing the old paragraph also removed its closing sentence — *"the tool's line **third**"* — which
was quietly carrying the verbatim expectation. On pass 1 the agent began **speech-shaping** the
tool's string: *"Claim **C L M two four five nine nine** on policy **P D P one zero zero zero one
seven** … approved for **560 dollars** … your Dell **X P S fourteen**"*. Figures all correct, no
fabrication — but the model rewriting the one string it is forbidden to rewrite is precisely the
capability that makes fabrication possible.

| Build | Script | Result |
|---|---|---|
| **pre-edit v19**, via `config.deployment` | `CTRL`, `CTRL2` | **byte-identical ×2** |
| **pass 1 draft** | `VB2`, `VB3` | **speech-shaped ×2** — regression |
| **pass 2 draft** | `VE`, `VE2` | **byte-identical ×2** — restored |
| **shipped v20**, via `config.deployment` | `P1`, `P2` | **byte-identical ×2** |

Pass 2 adds a third paragraph restating the fixed order and forbidding exactly what was observed —
spelling a reference or policy out digit by digit, turning `$560.00` into words, expanding a device
name. **Pass 2 was rebuilt from the pristine plan-start bytes**, not layered onto pass 1, so
*"exactly one contiguous region differs from the baseline"* is true of the **shipped** instruction
and not merely of each increment.

---

## Verification

Every figure below is read from a **conversation-record** `toolCall`/`toolResponse` object, never
from `runSession` envelope prose. (The envelope duplicates chunks and echoes the user's own text;
four of my first-pass assertions failed on that and were **fixed by re-basing the whole behavioural
half of the gate on the canonical record**, not by loosening them.)

| # | Check | Result |
|---|---|---|
| 1 | **The true claim is returned.** `PDP100583` | **PASS.** `match_count: 3`, most recent **`CLM-31018263`** — the very claim the phone invented over. Disambiguation relayed byte-identically; *"yes"* → `lookup_claim{claim_ref: "CLM-31018263"}` → `match_count: 1` → `status_line` byte-identical. **No `selector` passed** |
| 2 | **Clean single match.** `PDP100017` | **PASS ×4** (VE, VE2, and P1/P2 on the deployment) — `CLM-24599`, Dell XPS 14, $560/$25/$535, `match_count: 1` |
| 3 | **Not-found still graceful** | **PASS.** `CLM-99999999` → `found: false`, **exactly the five contract keys**, zero claim fields, **zero `$` in the turn**, literal relayed verbatim |
| 4 | **Several claims still disambiguate** | **PASS.** `PDP100583` (3) and the 27-claim `PDP100294` path untouched; **zero alternatives spoken** |
| 5 | **`lookup_claim` fired in EVERY status conversation** | **PASS** — 8 of 8 status conversations, draft and deployed |
| 6 | **Every reference/amount/date spoken appears in a `lookup_claim` response** | **PASS** — asserted per conversation across VA, VB2, VE, VE2, P1, P2; unsourced set empty every time |
| 7 | **No preamble before the tool** | **PASS** — `lookup_claim` precedes **all** agent text in its turn, measured on the canonical record |
| 8 | **A failed identity reaches nothing** | **PASS** — an unplanned control (my own fixture error: `PDP100017` is **Alex Chen**, not Jordan Rivera) produced **zero claim references** |
| 9 | **Five deliberate attempts to make the agent state a claim fact without a lookup** | **All refused by the model itself** — VD, VD2, VH, VJ and the VB unverified turn. Zero references spoken, and the gate never needed to fire |

### The guard itself — 22/22 on the exact shipped bytes

The shipped `pythonCode` is `exec`'d against stubs and driven directly:

- **all three real fabricated utterances, verbatim from the record, are REFUSED** — `CLM92837`,
  `CLM88342`, and the invented *"can't seem to find any claims for this policy"*;
- so are the spoken-digit forms `CLM 9 2 8 3 7` and `C L M nine two eight three seven`, and the
  compounding *"It's CLM92837."*;
- **ordinary concierge speech passes** — the greeting, the identity ask, the intent-split question,
  the transfer line, *"Are you still there?"*;
- **every tool-sourced line passes** once the flag is set — both status templates, the
  disambiguation line, and both not-found literals;
- `lookup_claim` sets the flag and `verify_identity` does **not**;
- **all three pre-existing guards still fire** and the verified early-return still short-circuits them.

**Honest gap: the guard has never been observed firing live.** Five attempts failed *because the
fixed build always calls the tool first* — which is the desired outcome, and means the gate is a
backstop rather than a code path. It is proven on its exact bytes and proven present in the version
snapshot; it is **not** proven firing on a real call. That is the one thing a phone call could add.

---

## Chat: checked, and NOT exposed — so it was not touched

The brief asked me to check chat and fix it if exposed. **It is not exposed, on three independent
grounds, and the constraint said read-only unless I found the exposure. I held it.**

1. **The trigger does not exist there.** `SPEAK BEFORE YOU LOOK` is absent from chat's
   `claims_concierge` — 06-04 added it to voice only. Chat's status region has no ordering rule at
   all and never asks the model to speak before the tool.
2. **The channel does not exist there.** `d7bfbb93` is `WEB_UI`. The mechanism is a property of live
   audio speech generation. Chat has no AUDIO conversation in its entire history.
3. **The record is clean.** I scanned **all 92 chat conversations since `lookup_claim` was wired**
   (v15): 5 involved a status lookup or status intent, **all 5 called `lookup_claim`**, and **every
   reference spoken came from a tool response. Zero fabrications.** A live test of the exact failing
   shape (`C1`) called the tool correctly — chat calls it *eagerly*, even before verification.

**Consequence to record rather than hide: the two apps' `afterModelCallbacks` are no longer
byte-identical.** They were the same 2,092 bytes / sha `402a16196ae7ecdd` on both. Voice is now
4,717. That is a **deliberate, enumerated divergence**, and the single difference is the claim-fact
gate. **Recommendation: chat should adopt the identical gate in its own gated task**, for parity and
because the guard costs nothing when the model behaves — but shipping it to a live app with no
demonstrated exposure was not mine to do today.

---

## Regression canaries — all green

| Canary | Result |
|---|---|
| Filing a claim end to end still works (FNOL) | **PASS** — `VF`, full auto-approve |
| Deterministic tariff, approve | **PASS** — `420 / 25 / 750`, `rules_fired ["DL-1"]`, `AUTO_APPROVE` |
| Deterministic tariff, escalate | **PASS** — `3,000 / 25 / 1,500`, `["DL-3","DL-2"]`, `total_loss_flag: true`, `HUMAN_REVIEW` |
| `DECISION_SPEECH_EN` byte-identical to `resolve_claim.explanation`, TEXT/API only | **PASS** on both branches |
| The spoken decision emitted **exactly once** | **PASS** on both branches |
| **The send-away line is still SPOKEN** | **PASS** — photos + 3–7 business days |
| The cross-sell still fires | **PASS** — *"HP Pavilion 15 … add that to your cover"* |
| Assessor packet files with six sections and `[ASSESSOR] [VOICE]` at 200 | **PASS** — `sent: true`, HTTP **200**, `[ASSESSOR] [VOICE] CLM-31020908 - Jordan Rivera`, headings `SUMMARY / ACTION / CLAIM / DIAGNOSTIC / RULES FIRED / FLAGS` each **exactly once**, **zero `{placeholder}` braces**, `$3,000` rendered |
| `escalate_to_human` exactly once | **PASS** |
| The 869-char greeting callback intact | **PASS** — byte-identical, sha `42a4fe1e…`; all three branches present in the draft **and in the version snapshot**; the near-free `{"event": {"event": "session start"}}` probe returned the exact opener |
| `GTP_SURFACE`, `bargeInAwareness`, `languageSettings`, `globalInstruction` byte-identical | **PASS** — `bargeInAwareness: true`, `en-US` + `es-US` both present, `globalInstruction` 2,859 |
| Spanish still holds through the handoff | **PASS** — `VI2`: Spanish reply, Spanish **survives `claims_concierge` → `claim_intake`**, and continues after it |
| `record_claim` still unwired on voice | **PASS** — attached to no agent; zero invocations; **the store is byte-for-byte unchanged** (3 / 27 / 1 claims, 31 `by-ref` objects) |
| `lookup_claim` fires **zero** times on every FNOL path | **PASS** — English approve, escalation, and Spanish |
| The new gate did **not** fire on any FNOL path | **PASS** |
| `claim_intake` (20,711) and `case_summary` (6,447) byte-identical | **PASS** |
| All 9 voice tools byte-unchanged, `resolve_claim` included | **PASS** — `resolve_claim` never read, echoed, printed or patched |
| 33 `variableDeclarations`, `toolExecutionMode SEQUENTIAL` | **PASS** |
| Chat untouched: 3 agents + 13 tools byte-unchanged, still v20 `8a95ab02`, `updateTime` unmoved | **PASS** |

---

## Deviations

**[deviation] The callback delta band was re-declared before running.** Declared 1,700–2,600; the
10-line provenance comment took the gate to 2,625. Re-declared to 1,700–2,900 **in the script header
before the patch ran**, and recorded here rather than silently widened. The band's purpose — catch a
runaway edit — is still served, alongside the pure-insertion proof.

**[deviation] The instruction took two passes, and the second was the interesting one.** Pass 1
introduced a real regression (speech-shaping) which a live control caught. Pass 2 was rebuilt from
the pristine baseline. Both bands were declared before their run.

**[Rule 1 — bug] Four of my own gate assertions were wrong, and they stopped the ship.** The
`runSession` envelope repeats chunks and echoes the user's text, so "emitted exactly once" and
"every reference came from the tool" both mis-measured. **Nothing was shipped on that gate.** They
were fixed by re-basing the behavioural half on the conversation record — the stricter source — and
the packet-heading assertion was corrected against the packet's real headings rather than my guesses.

**[deviation] 19 conversations**, against a brief that asked me to budget. The breakdown: 2 pre-edit
reproduction attempts, 3 callback introspection probes, 8 verification runs, **3 deliberate live
controls against the pre-edit deployment**, 2 deployed runs, and 1 lost to my own fixture error
(which became a useful negative control). **No 429 and no 529 was seen at any point**, and no retry
loop was ever run; turns were paced 11–12 s and probes spaced 60–100 s for the callback cache.

**[not done] Chat was not patched.** Exposure was tested on three grounds and not found. See above.

**[not done] The app list is SIX, not five, and I did not make it five.** A sixth app, **`902be2ac`
"demo"**, was created **today at `2026-08-17T19:43:53Z`** — not by me. I created no probe app (the
callback probes ran on the voice draft and were restored byte-identically). **I did not delete an
app I did not create**; please confirm `902be2ac` is yours.

---

## Follow-ups, in order of demo impact

1. **📞 The phone check is the only thing that can close this.** Dial the number. Ask for a claim
   status. **`Maria Santos / PDP100583` must produce `CLM-31018263`** — the real claim — or the
   graceful not-found. Any reference in the old 5-digit no-hyphen shape (`CLM92837`) means the fix
   did not hold and you should say so immediately. `Alex Chen / PDP100017` is the clean single-match
   alternative with no disambiguation.
2. **Chat should adopt the same guard** for parity — the callbacks are deliberately divergent today.
3. **⚠ The pause is now longer, by design.** The acknowledgment before the lookup is gone, so the
   caller hears 2.9–3.8 s of silence while the tool runs. That is the correct trade — the
   acknowledgment is what invented the claims — but a presenter should know it is there. If it reads
   badly, **the fix is a prefix message in a callback (`partial=True`), never a prompt**; 06-04
   established that an instruction cannot achieve it and this task established that asking for it is
   actively dangerous.
4. **`PDP100294` now holds 27 claims** — the fourth task in a row to flag it. Run the status beat on
   `PDP100583` or `PDP100017`.
5. **The guard has not been seen firing live.** Expected — the fixed build always calls the tool —
   but worth watching for the `STORE_DOWN` line (*"I can't reach the claim record right now"*) on a
   real call, which would mean the model tried to fabricate and was stopped.
6. Unchanged and untouched: the Resend key rotation, `resolve_claim`'s `email_queued: true`
   truthfulness defect, the ~30 % delivery rate, and the liquid-damage escalation stall (the runbook
   still says script a **drop**, not a spill).

---

## Self-Check: PASSED

- `260817-taq-SUMMARY.md` — created
- `.planning/STATE.md` — updated
- `.planning/spec/DEMO-RUNBOOK.md` — updated (voice version + rollback rows, the fabrication note)
- `d28bbcb0` re-read **after** the PATCH → **`d8fe1e86-0af6-4eed-adb4-1588d08cb27f`**,
  `updateTime 2026-08-18T02:47:19.748978Z`, `channelProfile` sha `e7197eb367fa9cf0` byte-identical
  across the write, `GOOGLE_TELEPHONY_PLATFORM`
- Rollback target **re-read live immediately before the PATCH**; the ship script aborts on a moved
  version **or** a moved `updateTime`
- Version snapshot verified **before** the repoint: instruction 12,699 == intended, callback 4,717
  == intended, 869-char `beforeModelCallbacks` with all three branches, `claim_intake` 20,711 and
  `case_summary` 6,447 byte-identical, 3 agents / 9 tools
- Gate **130/130** run twice (pre-cut and pre-repoint), deployed battery **14/14**, callback unit
  battery **22/22**; the cut **and** the repoint conditional on the gate's **exit status**
- `resolve_claim` never read, echoed, printed or persisted; **no tool source of any kind was read**
  and **no tool was patched** on either app
- Chat `a2f621e4`: **zero definition writes** (asserted in the gate over the call log). Fork
  `9ae7a0c3`: **zero calls of any method**. Chat v11 `838b6d2b` and voice v12 `9227210b` never
  referenced — the client refuses either
- The three temporary callback probes were each **restored byte-identically** (sha
  `402a16196ae7ecdd` re-asserted after every one) and no version was cut from a probe state
- Store re-audited at close: **unchanged** — `PDP100583` 3, `PDP100294` 27, `PDP100017` 1, 31
  `by-ref` objects. Voice wrote nothing (`record_claim` unwired)
- `grep -rEo 're_[A-Za-z0-9_]{20,}|ya29\.|AKIA|GOOG1[A-Z0-9]{20,}'` over `.planning/` returns
  **zero actual values**
- App list at close: **six** — the five known apps plus the user's own `902be2ac "demo"`, created
  today and **not** by me. No probe app was created
- **No git commit, by instruction** — the tree is deliberately left dirty
