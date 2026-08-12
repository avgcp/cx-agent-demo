---
phase: 05-demo-build-multimodal-backend-reveal-multilingual
plan: 06
subsystem: voice-channel-baseline
tags: [voice, gtp, baseline, regression-anchor, read-only, spanish-input, cross-sell]
requires:
  - "05-03-SPIKE-FINDINGS.md ## VOICE_BASELINE (escalation evidence, reused under the quota rule)"
provides:
  - "05-06-VOICE-BASELINE.md ## DECISION_SPEECH_EN — the byte-pinned English decision string 05-09 must localise"
  - "05-06-VOICE-BASELINE.md ## AUTO_APPROVE_PATH — tariff, single-send email mechanic, cross-sell gap"
  - "05-06-VOICE-BASELINE.md ## VOICE_INVENTORY / ## GTP_SURFACE — the pre-edit surface later voice plans assert against"
  - "05-06-VOICE-BASELINE.md ## PHONE_CHECK — EXECUTED 2026-08-12 (quick task 260811-suy); results recorded in the section"
affects:
  - "05-07 (assessor packet on voice) — must not double-send the customer email"
  - "05-09 (Spanish on voice) — has a deterministic English source string to mirror"
tech-stack:
  added: []
  patterns:
    - "runSession against the DRAFT app (no config.deployment) as the read-only way to drive live turns without touching the phone line"
    - "assert from toolCall/toolResponse objects, never from agent prose"
key-files:
  created:
    - ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-06-SUMMARY.md"
  modified:
    - ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-06-VOICE-BASELINE.md"
decisions:
  - "Reused the completed Scenario A conversation already on disk rather than spending a second one — zero new CES quota consumed"
  - "Corrected two preamble rows that Task 1 had written from expectation rather than evidence"
  - "Recorded the cross-sell miss as two unresolved candidate explanations rather than asserting a v11 defect the API cannot prove"
metrics:
  conversations_run: 0
  network_calls: 3 (all GET, all bounded)
  duration: ~25 min
  completed: 2026-08-11
---

# Phase 5 Plan 06: Voice Baseline Summary

Established the voice channel's regression anchor from live evidence — and found that the decision
sentence is **relayed verbatim from the pricing tool** (better than expected) while the **cross-sell
beat never fired** (worse than expected).

> **CORRECTED 2026-08-11 (quick task `260811-tgm`): the cross-sell DOES fire.** A third live call,
> conversation `103r6XO3ZXjS3S97D-AtLK1ag` (`2026-08-12T02:06:37Z`, AUDIO/LIVE, v11 `b17c9a26`),
> shows the offer landing verbatim on the auto-approve path. The two observations behind the
> "never fired" claim were both **false negatives** — those conversations ended before the beat.
> **BLOCKER 1 below is withdrawn.** See the ⛔ block in `## Blockers`.

## What Happened

Two prior executors stalled on a 600s watchdog at the Scenario A `runSession` call. **Neither stall
lost the work.** On inspection, the second executor had already driven all four Scenario A turns to
completion and saved every response, then died during a follow-up conversation-record fetch.

So **this run spent zero conversations and zero CES quota.** The remaining work was: refresh the
record with bounded GETs, assert over the saved evidence offline, and write three sections.

Every network call was bounded (`timeout=60` per request, fresh token forced immediately before).
All three returned in seconds.

## Answers to the Questions Asked

| Question | Answer |
|---|---|
| Did the Scenario A conversation succeed or time out? | **Succeeded** — completed by a prior executor, session `962bc958-8dd0-4232-ae6d-7ec1c23e00cf`, 4 user turns, all responses on disk. No `TIMED_OUT` anywhere. |
| Verbatim decision sentence | `Good news - that screen replacement comes to $840, which is under the $1,500 I can approve on the spot, so I can approve that for you right now. Your excess is $25, and your reference is CLM-24868.` (197 chars, ASCII, sha256 `9dc9b347…`) |
| Conversations spent by me | **0** |
| Does voice still serve v11? | **Yes** — `d28bbcb0` → `b17c9a26-3485-4658-9259-dfa4839a7977`, re-read `2026-08-11T21:05Z`. Chat `d7bfbb93` → `160dc3b2…` also unmoved. |
| Phone-check instructions | Written into `## PHONE_CHECK`; the ask is reproduced below. |

## Findings

### 1. The English decision line is deterministic — `spoken_equals_explanation: true`

The plan expected a paraphrase. The reality is better: v11 (*"no self-narration"*) emits
`resolve_claim.explanation` **byte-for-byte, in exactly one text chunk**, with no narration around
it. The doubled-sentence trap `260810-k5x` caught on chat does not occur here.

**This materially de-risks 05-09.** A Spanish line emitted from the tool on the `es` branch will be
spoken verbatim exactly as the English one is. The Spanish work is a template change inside
`resolve_claim`, not a prompt-engineering problem — and the regression check is exact (one chunk,
byte-equal, only the `CLM-` digits varying), not fuzzy.

### 2. Tariff PASS, and pricing is provably not model-generated

`AUTO_APPROVE`, `claim_amount 840`, `deductible 25`, `auto_approval_cutoff 1500`, `rules_fired
["DL-1"]`, `claim_ref CLM-24868`, `coverage_limit 3000`, `total_loss_flag false`. All from
`toolResponse`. Since the spoken sentence is a byte-copy of the tool string, the model contributes
none of the numbers.

### 3. `send_claim_email` fires exactly once — but it is not the send site

Confirmed on the auto-approve path what `## ESCALATION_PATH` recorded on the escalation path:
`resolve_claim` returns `email_queued: true` and `send_claim_email`'s own message reads *"Email
already sent when the claim was decided."* **The customer email is sent by `resolve_claim`.**
`send_claim_email` reports on a send that already happened. This is the app's general behaviour on
both branches, not a branch quirk — 05-07 must account for it or it will double-send.

### 4. `email_on_file` in the response is not the real recipient

The `toolResponse` reports a synthetic `…@example.com` address while `delivery: "live"` means Resend
delivered to the hardcoded mailbox. **Do not use the `toolResponse` recipient to assert where mail
landed** — only the mailbox check in `## PHONE_CHECK` can.

### 5. Conversation records omit the closing turn

`GET …/conversations/{session}` returned **3** turns for a 4-turn session, even ~17 hours later. The
`end_session` close is absent from the persisted record. Later plans asserting only against the
conversation record **will not see the close**; assert against the per-turn `runSession` responses.

## Blockers

### BLOCKER 1 — the cross-sell beat did not fire on the voice auto-approve path

`cross_sell_fired: false`. No cross-sell tool, and zero prose hits across every agent chunk for
`iphone | 16 pro | uninsured | add it to your cover | also insure | another device | bundle`. The
runbook promises *"Then it offers to add the uninsured iPhone 16 Pro Max"* as one of *"three
separate turns"*. It did not come. Turn 4's `No thanks` got `"Understood. Have a good day."` and
`end_session`.

**Recorded, not fixed** (hard constraint 8). The API cannot distinguish two explanations, and both
are written into the baseline:

1. **Runbook script defect** — a text `runSession` gives the agent one turn per user turn, so the
   offer needed a *neutral* utterance to hang off. The runbook's chat table uses *"That's
   everything, thanks"* for this beat; the voice script jumps to *"No thanks"*, which reads as a
   refusal. The runbook's own note *"Don't say 'no thanks' early"* points here.
2. **v11 capability gap** — the "three separate turns" phrasing implies the agent volunteers the
   offer unprompted, which a live call can do and a text `runSession` cannot.

Under (1) the fix is one line of the runbook; under (2) it is a later voice plan. **`## PHONE_CHECK`
is scripted to distinguish them** — the caller says *"That's everything, thanks"* and waits 10
seconds before *"No thanks"*.

> **UPDATE 2026-08-12 (quick task `260811-suy`) — explanation (1) is now substantially weaker.**
> The phone check happened (conversation `081cCNZtVwgSGqmfMpFSpbxMQ`, `channelType: AUDIO`,
> `source: LIVE`, deployment `d28bbcb0` on v11 `b17c9a26`) and **reproduced
> `cross_sell_fired: false` on a live audio call** — the setting explanation (1) said the API test
> could not simulate. `uninsured_device` was populated (`HP Pavilion 15`), the claim auto-approved,
> and the customer's post-email turn was a neutral **"Okay."** rather than a refusal. No offer came;
> the only prose hit in the whole call was the *covered* device at intake. That is **two
> independent observations**, one of them immune to the "a text `runSession` cannot volunteer a
> turn" argument.
>
> **Do not write this up as a proven defect yet.** The caller said *"Okay."*, not the scripted
> *"That's everything, thanks"*, and did not hold the 10 seconds of silence. **Current status:
> probable genuine gap (explanation 2) — two observations, one scripted retest still
> outstanding.** Detail in `05-06-VOICE-BASELINE.md ## PHONE_CHECK` item 4.

> ### ⛔ BLOCKER 1 WITHDRAWN 2026-08-11 (quick task `260811-tgm`) — THE CROSS-SELL FIRES
>
> **This blocker is not real.** Everything above it in BLOCKER 1, including the 2026-08-12 update,
> is superseded. It is retained verbatim so the reversal is auditable rather than silently
> rewritten. `cross_sell_fired: **true**`.
>
> **Evidence — a third live call:** conversation **`103r6XO3ZXjS3S97D-AtLK1ag`**, voice app
> `6e01e4a5-42a8-5213-b3da-c9053ff8ea52`, `2026-08-12T02:06:37Z` → `02:08:35Z` (1m58s),
> `channelType: AUDIO`, `source: LIVE`, deployment `d28bbcb0`, `appVersion` **`b17c9a26`** (v11),
> policy `PDP100583`, 9 turns — same configuration as `081cCNZtVwgSGqmfMpFSpbxMQ`, placed 29
> minutes later. Verified read-only 2026-08-11.
>
> **Turn 8 of 9, agent, verbatim:**
>
> > "Please reply to that email with photos of the damage. Once those are in, allow three to seven
> > business days for a representative to confirm everything. **Also, I see you have an HP Pavilion
> > 15 that isn't covered yet – would you like to add that to your policy?**"
>
> The customer said *"Uh sure."* and the agent took the lead-in — *"Great, I'll have someone send
> over the options for that."* — then closed with `end_session`.
>
> **Both earlier negatives were false negatives: neither conversation reached the beat.**
> `081cCNZtVwgSGqmfMpFSpbxMQ` ended at its turn 9 after the caller said *"¿Qué?"* and the agent went
> to the sign-off; the `runSession` observation ended on an early *"No thanks"*. Exactly the failure
> the runbook's own presenter note warns about. **Explanation (1) was correct** — and only as
> presenter guidance, since the offer arrives unprompted off a neutral reply with no 10 s pause
> needed. **Explanation (2), the v11 capability gap, is disproven on v11 itself.**
>
> **Runbook accuracy note:** the offer is appended to the send-away instructions **within the same
> agent turn**, not delivered as a turn of its own — the runbook's *"three separate turns"* phrasing
> is wrong on this point. It also fired on the turn immediately after a **barge-in**, so
> interrupting the agent does not suppress it.
>
> **No retest outstanding. The `"cost centre → profit centre"` wow moment works on live voice v11.**

This matters commercially: the cross-sell is the *"cost centre → profit centre"* wow moment.

### BLOCKER 2 — cosmetic: literal quotation marks in the closing line

The close ships as `"Understood. Have a good day."` **including the literal `"` characters**. Same
family as the 👂 artifacts the runbook already lists. The phone check asks whether TTS voices them.

### Corrected — two preamble rows were written from expectation, not evidence

Task 1's summary table predicted `spoken_equals_explanation: false` and *"cross-sell fires"*. Both
were wrong, in opposite directions. Corrected in place, with a note that the section bodies are the
authority.

## What the User Needs to Do — the phone check

Full script is in `## PHONE_CHECK`. In short:

1. **Get the number** from the console (deployment `d28bbcb0` → integration panel) — it is not
   API-retrievable — and write it into `DEMO-RUNBOOK.md`'s blank `☎ ____` line (~line 20).
2. **Call and run Scenario A out loud**, with one deliberate change: at beat 4 say **"That's
   everything, thanks"** and then **wait 10 seconds in silence** before saying "No thanks". That
   silence is the whole point — it is what settles Blocker 1.
3. **Report eight things:** the number and whether it answered; the decision sentence as heard and
   whether the figures were $840 / $1,500 / $25; **whether the iPhone cross-sell came, and at which
   beat**; barge-in worked or ignored; longest silence in seconds (>8 s is new); whether the close
   was clean and whether the quote marks were voiced; how many emails arrived at
   `akash.vinayak@nerdery.com` (expect exactly one) and the claim reference on it; anything else odd.

Nothing downstream is blocked on this — 05-07 and 05-09 can proceed on the API evidence.

## Constraint Compliance

- **Read-only on voice** — 3 network calls this run, all `GET`, all through the scripted method/URL
  gate (log: `scratchpad/s06-gate-log.txt`). No `PATCH`, no version, no deployment change.
- **Fork `9ae7a0c3`** — never touched; refused by the gate for every method.
- **`resolve_claim` source** — never read or echoed. Figures come from `toolResponse` only. Secret
  scan over the baseline for `re_[A-Za-z0-9]{8,}` returns no match.
- **Quota** — 0 conversations, 0 `runSession` calls.
- **Deployment pins** — voice `b17c9a26…` ✅, chat `160dc3b2…` ✅, both re-read after the fact.
- **Not committed** — tree left dirty as instructed; `STATE.md` and `ROADMAP.md` untouched.

## Self-Check: PASSED

- `05-06-VOICE-BASELINE.md` exists, **657 lines** (min 60), all **6** named headings present.
- **0** `PENDING — Task N` placeholders remain.
- Required literals present: `draft_equals_v11: true`, `spoken_equals_explanation: true`,
  `auto_approve_source: fresh …`, `escalation_source: reused …`, `send_claim_email_turn_count: 1`,
  `cross_sell_fired: false`, `phone_check: PENDING`.
- Secret scan clean — no API-key-shaped token, no tool source, no instruction body, no raw
  conversation record in `.planning/`.
- Both deployment pins verified by re-read, not assumption.
