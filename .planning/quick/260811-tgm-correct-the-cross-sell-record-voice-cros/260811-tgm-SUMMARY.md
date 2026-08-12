---
task: 260811-tgm
title: Correct the record — the voice cross-sell fires (reverses 260811-suy)
date: 2026-08-11
type: documentation-only
apps_touched: none
versions_cut: none
deployments_repointed: none
api_calls: read-only GETs only (3 conversations, 1 deployment)
evidence: "conversation 103r6XO3ZXjS3S97D-AtLK1ag, 2026-08-12T02:06:37Z → 02:08:35Z, AUDIO/LIVE, v11 b17c9a26"
key-files:
  modified:
    - ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-06-VOICE-BASELINE.md"
    - ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-06-SUMMARY.md"
    - ".planning/quick/260811-suy-record-phone-check-and-fix-the-repeated-/260811-suy-SUMMARY.md"
    - ".planning/STATE.md"
  created:
    - ".planning/quick/260811-tgm-correct-the-cross-sell-record-voice-cros/260811-tgm-SUMMARY.md"
---

# 260811-tgm — The voice cross-sell fires; 260811-suy's conclusion is reversed

Quick task `260811-suy` recorded the voice cross-sell as a *"probable genuine gap, two
observations, one scripted retest outstanding."* **That was wrong.** A third live call disproves it.
The record is now corrected across four files, in every case by **appending a superseded block that
leaves the original reasoning visible** rather than rewriting history — so the reversal is auditable.

**Nothing was changed on any app.** Every CES call issued by this task was a `GET`.

## Verification of the five claims in the brief

I fetched the record myself before writing anything. All five hold, but **three carry corrections**
to the brief's own description — noted below and reflected in what I wrote into the docs.

| # | Claim | Verdict |
|---|---|---|
| 1 | The cross-sell fired | **CONFIRMED** — with a turn-number correction |
| 2 | Barge-in confirmed working | **CONFIRMED** — with a turn-number correction |
| 3 | Repeated diagnostic question did not reproduce | **CONFIRMED**, exactly as described |
| 4 | Audio-channel text normalisation differs from text/API | **CONFIRMED**, exactly as described |
| 5 | Possible double email | **PARTIALLY CONFIRMED** — the two calls are real, but the brief's characterisation of the responses is wrong, and the record points *toward* single-send |

### Conversation metadata — all verified

`103r6XO3ZXjS3S97D-AtLK1ag`: `startTime 2026-08-12T02:06:37.872937Z` → `endTime
2026-08-12T02:08:35.956310Z` (1m58s), `channelType: AUDIO`, `source: LIVE`, `turnCount: 9`,
`languageCode: en-US`, `deployment d28bbcb0-066e-4127-a894-fbf9ba39789f`, `appVersion
b17c9a26-3485-4658-9259-dfa4839a7977` (v11, i.e. **without** the 260811-suy draft fix), policy
`PDP100583`. Every field matches the brief.

### 1. Cross-sell — CONFIRMED, but it is **turn 8 of 9, not turn 7**

The agent text is verbatim as the brief quotes it, including the HP Pavilion 15 offer appended to
the send-away instructions in the same turn, the user's *"Uh sure."*, the lead-in reply and
`end_session`. **Correction:** in the conversation resource's own `turns` array (`turnCount: 9`),
this is index 7 → **turn 8**. The brief says turn 7; it is off by one throughout. I used the
API's numbering in all four documents so the citations are checkable against the record.

**I also verified the false-negative explanation rather than assuming it.** I fetched
`081cCNZtVwgSGqmfMpFSpbxMQ` and read its tail: its turn 8 carried the send-away line, the caller
said **"¿Qué?"** at turn 9, and the agent went straight to *"Thanks for calling … have a good day."*
+ `end_session`. The call **ended before the offer's slot**, it did not skip it. That is direct
evidence for the false-negative reading, not an inference.

### 2. Barge-in — CONFIRMED, marker is on **turn 8**, not turn 7

The marker exists exactly once in the whole record, verbatim:
`<context>agent speaking was interrupted. user only heard 'Since we've' in the last agent
response.</context>`. Turn 7's agent text was *"Since we've approved that, an email is already on
its way to the address we have on…"* — truncated precisely where the marker says. The recovery
claim also holds: turn 8 re-delivers the reply-with-photos instruction and the 3–7 day expectation
before appending the cross-sell.

### 3. Repeated diagnostic question — CONFIRMED, no correction

On live v11 `b17c9a26` (no suy fix): `q2=no_liquid` supplied at T4 → tool asks `q1` → `q1=screen`
at T5 → tool asks `q3` → `q3=works_normally` at T6 → `terminal REPAIRABLE`, `questions_asked: 3`.
`repeat_of_previous_question: false` on every response. No `DIAGNOSTIC_INCOMPLETE`. Both
consequences the brief draws are sound and are now written into the docs: demo risk is lower than
suy implied, **and** the draft fix cannot be validated by absence of the symptom.

### 4. Audio-channel normalisation — CONFIRMED, no correction. This is the 05-09 item.

| Source | Channel | Amount as recorded |
|---|---|---|
| `resolve_claim.explanation` (tool string) | — | `$420` |
| `081cCNZtVwgSGqmfMpFSpbxMQ` (`2026-08-12T01:37:41Z`) | AUDIO/LIVE v11 | `420 dollars` |
| `103r6XO3ZXjS3S97D-AtLK1ag` (`02:06:37Z`) | AUDIO/LIVE v11 | `420` |

Same app, same deployment, same version, 29 minutes apart — different spoken forms. The `$` and the
`CLM-` hyphen are both dropped on audio. Recorded prominently in `## DECISION_SPEECH_EN` as a
must-read for 05-09: **Spanish byte-identity assertions must run on the text/API channel**; an
audio-channel byte comparison will fail spuriously and look like a paraphrase regression. Worth
noting the underlying determinism is intact — `relay_mismatch: false` on the tool response.

### 5. Possible double email — ⚠️ THE BRIEF'S CHARACTERISATION IS WRONG

The two calls are real, but they are on **turns 7 and 8** (brief says 6 and 7), and more
importantly **both returned identical payloads**:

| Turn | `sent` | `delivery` | `message` |
|---|---|---|---|
| 7 | `true` | `live` | `Email already sent when the claim was decided. Customer should reply with photos attached.` |
| 8 | `true` | `live` | *(identical string)* |

The brief asserts the first call *"returned `sent: true`, which is ambiguous"* while the second
returned the already-sent message. **Not so — both carry `sent: true` AND the already-sent
message.** `resolve_claim` at turn 6 had already returned `email_queued: true`. So on the record,
both `send_claim_email` calls are idempotent *reports* of the one decision-time send: the invariant
holding, not breaking. I recorded it as an open question anyway, per instruction and correctly —
`sent: true` is ambiguous on its face and the tool source cannot be read to settle it — but I wrote
it up as **weaker than the brief framed it**, with the evidence pointing toward single-send and an
inbox check for exactly one `CLM-24464` email as the only thing that closes it.

## What I corrected, file by file

**1. `05-06-VOICE-BASELINE.md`** — five edits:
- **Preamble table:** `## AUTO_APPROVE_PATH` row struck through and corrected to "cross-sell FIRES
  ✅"; `## PHONE_CHECK` row rewritten (barge-in PASS, cross-sell fires, repeated question
  intermittent); `## DECISION_SPEECH_EN` row now flags that byte-identity is TEXT/API-only. A note
  under the table records that the table has now been corrected **twice**, and why.
- **`## DECISION_SPEECH_EN`:** new ⚠️ "05-09 MUST READ" block with the three-row normalisation table.
- **`## AUTO_APPROVE_PATH`:** the "Cross-sell — DID NOT FIRE ❌" heading corrected to "FIRES ✅",
  the `cross_sell_fired: false` block labelled a false negative in place, and the "one substantive
  defect" claim withdrawn. The full ⛔ SUPERSEDED block sits at the end of the hedge that the brief
  flagged near line ~445.
- **`## PHONE_CHECK` preamble:** a "second live call" block with a was/now table for items 3, 4 and
  open item 4, so a reader hitting the section top cannot act on the stale items below.
- **`## PHONE_CHECK` body:** item 3 gains the intermittency nuance; item 4 gains the full ⛔
  SUPERSEDED block; open item 4 (barge-in) struck through and closed; **new item 8** (barge-in PASS,
  with the content-aware recovery finding) and **new item 9** (possible double email) added.

**2. `05-06-SUMMARY.md`** — the one-liner now carries a correction note, and **BLOCKER 1 is
withdrawn** via a ⛔ block appended after the 2026-08-12 hedge. Includes the runbook-accuracy
correction that the offer arrives inside the send-away turn, not as one of "three separate turns".

**3. `260811-suy-SUMMARY.md`** — a ⛔ PARTIALLY SUPERSEDED block at the top. **Its history is
untouched.** The block states the two reversed conclusions (cross-sell, intermittency) and
explicitly lists what is *unaffected*: the draft patch itself, the +507-char delta, the
byte-identical anchors, and that no version was cut or deployment moved.

**4. `.planning/STATE.md`** — the suy cross-sell entry is struck through with a "SUPERSEDED, this
conclusion is WRONG" preface and left readable. Four new entries added above it: cross-sell
RESOLVED/NOT A DEFECT, barge-in VERIFIED, the 05-09 audio-normalisation warning, the double-email
open question, and the intermittency nuance. Quick-tasks table row added for this task.

Every correction cites conversation `103r6XO3ZXjS3S97D-AtLK1ag` and its `2026-08-12T02:06:37Z`
timestamp as the evidence, as required.

## No app or deployment was modified — confirmed

- **Zero writes.** Every CES call was a `GET`: three conversations
  (`103r6XO3ZXjS3S97D-AtLK1ag`, `081cCNZtVwgSGqmfMpFSpbxMQ`, plus one 404 on a truncated ID from
  the brief) and one deployment.
- **Verified after the fact:** deployment `d28bbcb0-066e-4127-a894-fbf9ba39789f` still serves
  `b17c9a26-3485-4658-9259-dfa4839a7977` (v11) with `updateTime 2026-08-05T17:00:57Z` — unchanged
  and still pre-dating today.
- No version cut, no repoint, no patch, no draft edit, on any app.
- Fork `9ae7a0c3-6511-413c-8cdb-0efe9e90d2b9` was never touched. `resolve_claim`'s source was never
  read or echoed — only its `toolResponse` output as recorded in the conversation, which contains no
  credentials.
- All network calls bounded with `--max-time 120 --connect-timeout 15`, fresh token per call.
- **Not committed.** Working tree left dirty as instructed.

## One thing worth flagging for whoever reads next

The brief's turn numbers are consistently **one lower** than the conversation resource's own `turns`
array. I used the API's numbering. If anyone cross-references the brief against the record, expect
the offset — the content matches exactly, only the indices differ.

## Self-Check: PASSED

- `05-06-VOICE-BASELINE.md`, `05-06-SUMMARY.md`, `260811-suy-SUMMARY.md`, `STATE.md` — all four
  edits applied (Edit tool would have errored on a failed match).
- `260811-tgm-SUMMARY.md` created at
  `.planning/quick/260811-tgm-correct-the-cross-sell-record-voice-cros/`.
- Deployment state re-verified unchanged after all edits.
- No commits made, per instruction.
