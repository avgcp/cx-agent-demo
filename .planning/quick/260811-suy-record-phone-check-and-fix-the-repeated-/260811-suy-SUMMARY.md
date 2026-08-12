---
task: 260811-suy
title: Record the phone check; fix the repeated diagnostic question (draft only)
date: 2026-08-12
apps_touched:
  - "6e01e4a5-42a8-5213-b3da-c9053ff8ea52 (voice, demo-ready) — DRAFT agent 87551704 patched"
  - "a2f621e4-9faf-505a-b804-22471f022366 (chat, hardened) — DRAFT agent 87551704 patched"
versions_cut: none
deployments_repointed: none
key-files:
  modified:
    - ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-06-VOICE-BASELINE.md"
    - ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-06-SUMMARY.md"
    - ".planning/STATE.md"
---

# 260811-suy — Phone check recorded; repeated-question defect fixed in the DRAFT

The real GTP call was fetched and transcribed into `05-06-VOICE-BASELINE.md ## PHONE_CHECK`, and the
one defect it exposed — the agent re-asking a question the customer had already answered — is fixed
in **both** apps' drafts. **No version was cut. Neither deployment moved.**

## The four things you asked to be stated explicitly

| Question | Answer |
|---|---|
| **Did chat have the same defect?** | **Yes — identical.** Neither `claim_intake` contained the string `DIAGNOSTIC_INCOMPLETE` at all, on either app, so neither had any instruction for what to do when `resolve_claim` rejects a premature call. Chat's instruction has diverged from voice's overall (18,496 vs 14,140 chars, 8 tools vs 5), but the diagnostic block is **byte-identical in the region that matters** — the anchor and its surrounding 180 characters match exactly on both. Fixed on both. |
| **Anchor text replaced, and the length delta** | Anchor: `stop asking and let the tool escalate.` — verified **unique** (count == 1) in both instructions before patching. Replaced with itself + `\n\n` + a six-line paragraph. **Delta +507 chars on both apps**, identical because the inserted text is identical: voice **14,140 → 14,647**, chat **18,496 → 19,003**. |
| **Did the verification conversation run?** | **Yes — one conversation, against the voice DRAFT, and it passed.** Session `suy-verify-563ed61c`, 5 turns. Detail below. |
| **Was a version cut / did `d28bbcb0` move?** | **No and no.** Voice `d28bbcb0` still serves **v11 `b17c9a26`** and chat `d7bfbb93` still serves **v9 `160dc3b2`**; both deployment `updateTime`s still pre-date today (2026-08-05 and 2026-08-10). Voice still has 19 versions, latest `b17c9a26` (2026-08-05); chat still has 13, latest `160dc3b2` (2026-08-10). **The live phone demo still exhibits the repeated-question defect.** |

## Part 1 — the phone check

Fetched the authoritative record rather than working from the summary: conversation
`081cCNZtVwgSGqmfMpFSpbxMQ`, `channelType: AUDIO`, `source: LIVE`, `deployment d28bbcb0`,
`appVersion b17c9a26` (v11), 9 turns, `en-US`, `2026-08-12T01:37:41.859Z → 01:39:54.102Z`. Policy
**PDP100583** (iPhone 16 Pro Max, coverage 1500, cutoff **750**, excess 25) — **not** the scripted
`PDP100294`, which turned out to be a bonus.

Seven items recorded in `## PHONE_CHECK`, each with its evidence:

1. **PASS — decision line.** Same template as `DECISION_SPEECH_EN`, correctly re-priced: `$420 /
   $750 / $25`, `CLM-24552`, `rules_fired ["DL-1"]`, relayed verbatim in one chunk. Because the
   caller used a *different* policy, this proves something the baseline could not: the template
   **re-prices off the policy tariff** and still relays byte-for-byte. TTS normalisation
   (`$420` → "420 dollars", `CLM-24552` → "CLM 24552") is recorded as **expected, not a defect** —
   with the warning that future assertions must compare against `toolResponse.explanation`, never
   the transcript.
2. **PASS — single-send email, live.** `resolve_claim` returned `email_queued: true` /
   `email_delivery: live`; `send_claim_email` fired in exactly one turn and returned *"Email already
   sent when the claim was decided."* Confirms on a real call what 05-04 established from historical
   records. (The mailbox itself was not reported back — one send *call* proven, one *delivery* not.)
3. **FAIL — repeated diagnostic question.** The phone check's one failure. See Part 2.
4. **Cross-sell did NOT fire — second independent observation.** `uninsured_device` was populated
   (`HP Pavilion 15`), the claim auto-approved, and the customer's post-email turn was a neutral
   **"Okay."** — the natural opening, and not the `"No thanks"` refusal that explanation (1) blamed.
   Zero cross-sell prose hits. Written up as **probable genuine gap, two observations, one scripted
   retest still outstanding** — explicitly *not* as a proven defect, since the exact beat
   (*"That's everything, thanks"* + 10 s silence) was not run. The hedge in `05-06-SUMMARY.md` was
   updated to say the "artifact of how the API drives turns" explanation is now substantially
   weaker.
5. **Spanish — "¿Qué?" got an English reply**, `languageCode` stayed `en-US`. Consistent with
   `LOCKS_AT_FIRST_UTTERANCE` but caveated honestly: two syllables, during the close sequence, with
   `end_session` in the same turn. **Suggestive, not conclusive** — 05-09 must not cite it as a
   settled negative.
6. **2m12s** (132.2 s wall clock, 9 turns) recorded as a demo-timing data point, with per-turn span
   durations. The whole auto-approve story fits in ~2 minutes *including* the ~11 s the item-3
   defect wasted. Decision turn (12.3 s) is the longest agent-side beat.
7. **The email asks the customer to reply with photos** (`Claim CLM-24552 - please reply with
   photos`, body says so in capitals, agent voices it) — voice compensating for having no photo
   upload, and a genuinely good answer to *"how do you see the damage over the phone?"*. **Flagged
   as missing from `DEMO-RUNBOOK.md`** and worth two lines there.

Four of the original eight asks (the phone number, barge-in, the voiced quote marks, the mailbox
count) are **not answerable from the record** and are listed as still open. Note that BLOCKER 2 is
**untested, not cleared**: this call closed on *"Thanks for calling, &lt;name&gt; - have a good
day."*, which carries no quote marks — a different sentence from the one that does.

## Part 2 — the fix

**The defect.** After the customer answered *"does it still switch on and respond normally?"* with
**"It does."**, the agent called `resolve_claim` (guessing `diagnostic_outcome: "REPAIRABLE"`)
instead of `run_diagnostic`. The tool refused with `DIAGNOSTIC_INCOMPLETE` — **correct behaviour,
working hardening, untouched.** The agent then recovered by **re-asking a question the customer had
already answered**, costing one customer turn and ~8% of the call immediately before the moment the
demo is meant to shine.

**The fix is in the recovery path.** Inserted one paragraph telling the agent that
`DIAGNOSTIC_INCOMPLETE` is a mistake in its own sequencing, to call `run_diagnostic` again
immediately carrying every answer it already has *including the one just given*, and **never to
re-ask a question the caller has already answered in order to recover.** No change to the guard, no
restructuring of the diagnostic flow, no other region touched.

**How it was applied** — scripted `str.replace()` against the API, per the standing warning that
inline `claim_intake` edits have killed multiple agents here:

- Read-verify-patch-verify via direct `apps.agents.patch?updateMask=instruction`. No `importApp`,
  no export package used.
- Pre-assertions: exact char length **and** SHA-256 prefix match the 05-06 baseline
  (voice `628548c1e582e34d`, 14,140 — i.e. the draft had not moved since the baseline); anchor count
  == 1; `DIAGNOSTIC_INCOMPLETE` absent (not already patched).
- Build assertions: delta bounded `300 ≤ Δ ≤ 700` (actual **+507**); **reversibility check** —
  deleting the inserted block from the new string reproduces the original byte-for-byte, proving
  exactly one region changed; `run_diagnostic` mention count +1.
- Post-assertions: read back and compared **byte-for-byte** against the intended string; tool count
  and `childAgents` unchanged. New SHA-256: voice `02b1e6ae…`, chat `acf62e78…`.
- Nothing was printed but lengths, booleans, counts and ≤200-char excerpts. The instruction bodies
  were never echoed; `resolve_claim`'s source was never read.

## Verification — one conversation, passed

Session `suy-verify-563ed61c`, `runSession` against the voice **DRAFT** (no `config.deployment`),
five turns, driven one diagnostic answer per turn to reproduce the original trigger. Used
`Jordan Rivera / PDP100294` because `PDP100583`'s customer name is redacted in the record and a
name mismatch would have failed `verify_identity`.

| Assertion | Result |
|---|---|
| Agent re-asked a question already answered | **No.** Three questions asked, three distinct; `repeat_of_previous_question: false` on every `run_diagnostic` response |
| Diagnostic reached terminal | **Yes** — `terminal {outcome: REPAIRABLE, issue: screen}`, `questions_asked: 3` |
| Claim resolved with correct tariff | **Yes** — `AUTO_APPROVE`, `claim_amount 840`, `deductible 25`, `auto_approval_cutoff 1500`, `coverage_limit 3000`, `rules_fired ["DL-1"]`, `CLM-24437` |
| `DECISION_SPEECH_EN` canary | **Holds** — decision turn carries exactly **one** agent text chunk, byte-identical to `resolve_claim.explanation`, 197 chars, ASCII, template-identical to the baseline with only the `CLM-` digits differing |
| `DIAGNOSTIC_INCOMPLETE` guard | Still present and unweakened — but **did not fire this run** |

**Honest caveat: the recovery path itself was not exercised.** The agent took the correct path at
turn 4 (`run_diagnostic` → terminal → `resolve_claim`), so `resolve_claim` was never called
prematurely and the guard never triggered. What this run proves is that the +507-char insertion is
**non-regressive** — the diagnostic flow, tariff, verbatim decision line and single-chunk relay all
survive it — and that the failure mode did not reproduce. It does **not** prove the recovery branch
behaves correctly when the guard does fire, because the model's premature call is non-deterministic
and cannot be forced from the client. Under the one-conversation quota rule no second attempt was
made. The residual risk is small (the branch is a strict improvement on having no instruction at
all) but it is real, and the next live call is the natural place it gets observed.

## Deployment status — read this before demoing

**The fix is in the DRAFT of both apps only.** No version was cut and no deployment was repointed,
as instructed — plan 05-07 will cut voice v12 shortly and can carry this fix.

- Voice `d28bbcb0` → **v11 `b17c9a26`** (unchanged). **The live phone demo still re-asks the
  question** if the model makes the same premature call. Until v12 is cut and the deployment
  repointed, the defect is live.
- Chat `d7bfbb93` → **v9 `160dc3b2`** (unchanged), same caveat.

The repoint decision is left to the user.

## Constraint compliance

- Fork `9ae7a0c3` — **never touched**; asserted against in the patch script.
- `resolve_claim` source — **never read or echoed**. All figures from `toolResponse` objects.
  Secret scan for `re_[A-Za-z0-9]{8,}` over the updated baseline: **no match**.
- No `importApp`; no exported package on disk was read or written. Direct `apps.agents.patch` only.
- Every network call bounded (`--max-time 120 --connect-timeout 15` / `timeout=120`), fresh
  `gcloud auth print-access-token` immediately before each.
- Quota: **one** conversation (5 turns). No retries, no loops.
- Regression canaries all survive: deterministic tariff ✅, `DIAGNOSTIC_INCOMPLETE` guard ✅,
  single-send email ✅, verbatim one-chunk decision line ✅, tool counts 5 / 8 ✅.
- **Not committed** — tree left dirty, as instructed.

## Self-check

- `05-06-VOICE-BASELINE.md` — 781 lines; all six named headings present exactly once; zero
  `phone_check: PENDING` and zero `PENDING — Task` placeholders remain.
- Both patched instructions read back byte-identical to intent; both deployments and both version
  lists re-read **after** the patches and unchanged.
