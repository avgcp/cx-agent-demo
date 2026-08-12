---
task: 260812-hhi
title: File the assessor packet on the email-confirmation turn, on both apps
date: 2026-08-12
subsystem: chat-agent-wiring, voice-agent-wiring
tags: [ces, assessor-packet, turn-boundary, version-cut, deployment-repoint, regression-canaries]
apps_modified:
  - "chat a2f621e4 — agents/claim_intake; versions v11 838b6d2b (cut, never deployed) + v12 26c3aebd; deployment d7bfbb93 repointed"
  - "voice 6e01e4a5 — agents/claim_intake (instruction + tools). NO version cut. NO repoint."
apps_untouched:
  - "fork 9ae7a0c3 — zero calls of any method"
key-files:
  created:
    - ".planning/quick/260812-hhi-file-the-assessor-packet-on-the-decision/260812-hhi-SUMMARY.md"
  modified:
    - ".planning/spec/DEMO-RUNBOOK.md"
    - ".planning/STATE.md"
metrics:
  conversations_spent: 4
  turns_spent: 13
  chat_version_cut: "26c3aebd-d72b-4ec5-861d-8a9fabb140cf (plus 838b6d2b, superseded and never deployed)"
  chat_deployment_repointed: "d7bfbb93: 658472a0 -> 26c3aebd"
  voice_version_cut: none
  voice_deployment_repointed: none
---

# 260812-hhi — File the assessor packet on the email-confirmation turn

An escalated chat claim now files the assessor briefing packet on the **same turn as the email
confirmation** — the turn that always happens — instead of the turn after it, which a real
conversation had already been observed never reaching. Proven live on a three-turn conversation
that ends exactly where the failing one did, with **no further customer message of any kind**.
The same wiring was added to the voice app, which had none; voice is left **wired but
undeployed** by instruction.

Shipped as **chat v12 `26c3aebd-d72b-4ec5-861d-8a9fabb140cf`**, now served by `d7bfbb93`.

## The answers this task was asked for

| | |
|---|---|
| **New chat version** | **`26c3aebd-d72b-4ec5-861d-8a9fabb140cf`** — *"chat v12 - assessor briefing packet filed on the email-confirmation turn, send-away line still spoken"* |
| **Was `d7bfbb93` repointed?** | **Yes**, and read back rather than inferred. `658472a0` (v10) → `26c3aebd` (v12) at `2026-08-12T18:38:25Z`. `channelProfile` — which holds `channelType: WEB_UI` and `webWidgetConfig` — is **byte-identical** before and after. |
| **A second version, `838b6d2b`, also exists** | Cut mid-task, **never deployed, and must never be**. See *The regression this nearly shipped*. |
| **Voice** | `claim_intake` wired: 5 → **7 tools**, instruction 14,647 → **18,215** chars. **No version cut. `d28bbcb0` still serves v11 `b17c9a26`**, `updateTime` still `2026-08-05T17:00:57Z`. |
| **Does the packet file with no further customer turn?** | **Yes.** Session `esc4-68e580e1`, 3 customer turns, packet filed on turn 3, conversation ends there. |
| **Assessor email** | `[ASSESSOR] CLM-24041 - Jordan Rivera`, Resend **HTTP 200**, `packet_text` byte-identical (477 == 477) to the composer's response |

## Why the email-confirmation turn, and what was rejected

Recorded because the reasoning is the durable part, not the diff.

- **Rejected — keep it on a later turn.** This was v10's design and it is proven fragile. Chat
  conversation `dfMessenger-ca0d435b` escalated correctly on v10 — `TOTAL_LOSS`, `DL-3`+`DL-2`,
  `CLM-24625`, `$1,500` — and filed **no packet at all**, because the customer said *"ok"*, got
  the send-away line, and stopped. Worse than fragile: **the turn the packet lands on was never
  fixed.** In 05-05's passing test the email landed on turn 2 and the packet on turn 3; in the
  wild run the email landed on turn 3 and the packet needed a turn 4. The design fired only when
  the conversation happened to continue, which for a demo is the *unlikely* case, and it failed
  silently with nothing on screen indicating a problem.
- **Rejected — file it on the decision turn.** This was 05-07's design for voice, chosen to keep
  latency off the decision. It puts two tool calls in front of the customer hearing their
  outcome; on a phone call that is dead air at the worst possible moment.
- **Chosen — the email-confirmation turn.** On an escalated claim that turn *always* happens, so
  the packet is guaranteed. And the agent is already speaking a send-away line there, so the two
  tool calls land while the customer is being talked to rather than waiting in silence.

## The bonus finding — the human handoff was never firing either

`escalate_to_human` and `end_session` sat **behind the same turn boundary** as the packet, at
the end of the same instruction chain (*"only once the case record is filed call
`escalate_to_human` and immediately `end_session`"*). So on v10, on the wild conversation and on
any escalated chat claim that ended at the email line:

> **the escalation was never actually signalled and the session was never closed.** The claim
> looked escalated to the customer — they heard the specialist handoff — but no
> `session_escalated` signal was ever emitted.

05-04 had already found a HUMAN_REVIEW conversation on file where `send_claim_email` was never
called at all, and 05-05 recorded the pair as "stranded", but neither treated it as a live
defect. It was one. **v12 fixes it as a side effect**: on session `esc4-68e580e1`,
`escalate_to_human` and `end_session` both fire on turn 3, and the conversation closes cleanly
with no further customer input. This is a real improvement to the escalated demo path that was
not in the task brief.

## The edits, per app — anchors, deltas, tool counts

Every anchor was **sliced out of that app's own instruction bytes** and asserted to occur exactly
once before any replacement. Anchors were never copied between apps, and could not have been:
**chat's instruction is pure LF; voice's is 176 CRLF plus 7 bare LF.** A chat anchor string will
never match in voice. (Corollary worth carrying: a text-mode file round-trip silently collapses
voice's CRLFs — all voice instruction handling here went through JSON, never a `.txt` file.)

### Chat `a2f621e4` / `claim_intake` `87551704` — 20,505 → **22,148** chars (**+1,643**), tools **10 → 10 (unchanged)**

| # | Anchor replaced (exact text, sliced from the live instruction) | Effect |
|---|---|---|
| 1 | `This is its own turn and ends with the last word of it.` … through to that step's `</action>` | Appended a paragraph carving out a HUMAN_REVIEW-only exception: the silent Case record step belongs **inside this same turn**. The clause itself was **not weakened** — it survives verbatim, as does `do not bundle the cover offer onto the end.` |
| 2 | `<trigger>The claim was decided HUMAN_REVIEW. On any other decision, skip this step entirely - an approved claim has no case record.</trigger>` | Retriggered on *"you have just given the email confirmation in this same turn"* + *"never on a later turn"* |
| 3 | `File the case record for the specialist. This is internal work and it is silent.` | Added *"It happens in the SAME turn as the email confirmation, immediately after `send_claim_email`, and never on a later turn."* |
| 4 | `The only thing they hear on this path is the specialist handoff you have already given them, unchanged.` | **Deleted** — this sentence caused the regression below. Replaced with *"This silence covers the case record and nothing else. You still say the email line for this turn out loud and in full…"* |
| 5 | `Never leave the case record for a later` … `record would never be filed at all.` | Appended the ordering rule: *"the message to the customer comes FIRST … before you call a single tool in this turn"* |
| 6 | `Still do the email step first, so they leave with a reference and know what happens next.` | → *"Still do the email step first **and actually say its sentence out loud**, so they leave with a reference…"* |

Step count unchanged at **11**; `Case record` still sits between `Ask for photos by email` and
`Cross-sell`; brace-token vocabulary unchanged; read-back byte-identical on every PATCH.

**Version cut: `26c3aebd-d72b-4ec5-861d-8a9fabb140cf` (v12). Deployment `d7bfbb93` REPOINTED**
`658472a0` → `26c3aebd` at `2026-08-12T18:38:25Z`, gated in code on both assertion batteries
re-passing, `channelProfile` byte-identical.

### Voice `6e01e4a5` / `claim_intake` `87551704` — 14,647 → **18,215** chars (**+3,568**), tools **5 → 7**

This was the **first** wiring, not a correction — voice had no `Case record` step at all.

| # | Anchor replaced (sliced from the voice instruction, CRLF) | Effect |
|---|---|---|
| 1 | `        <step name="Cross-sell">` | Inserted a whole new `<step name="Case record">` immediately before it — same-turn trigger, word-for-word packet sentinel, and a **stronger** silence clause than chat's: *"none of it may reach text to speech"*, plus *"Do not narrate the wait … do not add a holding phrase"* |
| 2 | `This is its own turn and ends with the last word of it.` … through to `</action>` | Same same-turn carve-out as chat edit 1 |
| 3 | `Then call {@TOOL: escalate_to_human} and` | → `Then do the Case record step, and only once the case record is filed call {@TOOL: escalate_to_human} and` — bringing voice's HUMAN_REVIEW bullet in line with chat's |
| 4 | `The only thing they hear on this path is the specialist handoff you have already given them, unchanged.` | Same deletion as chat edit 4 |
| 5 | `Never leave the case record for a later` … `record would never be filed at all.` | Same message-first ordering rule as chat edit 5 |

Step count **7 → 8**. `updateMask=instruction,**tools**` — attaching
`generate_case_summary` and `send_case_record_email` was **required**; `updateMask=instruction`
alone leaves the step unreachable (chat needed the same 8 → 10 in 05-05). All 5 baseline tools
retained; `childAgents`, `transferRules` and the 33 `variableDeclarations` byte-identical.

**No version was cut and `d28bbcb0` was not repointed** — see the voice section below.
