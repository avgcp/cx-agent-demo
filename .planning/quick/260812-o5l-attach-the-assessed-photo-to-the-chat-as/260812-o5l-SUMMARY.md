---
quick_id: 260812-o5l
type: quick
status: complete-with-gate-held
date: 2026-08-12
subsystem: cx-agent-chat-app
tags: [multimodal, photo-attachment, assessor-packet, resend, spike, ces, chat-app, gate-held]
spike_verdict: FEASIBLE
server_changes:
  app: a2f621e4-9faf-505a-b804-22471f022366 (Meridian Claim - Chat (hardened))
  draft_modified: yes - 4 resources
  version_cut: NONE
  deployment_repointed: NO - d7bfbb93 still serves chat v13 1eb3fd5c-5aff-46c8-b572-e3fe18bf966f
  untouched: [6e01e4a5 (voice, live on 5d9df25c), 9ae7a0c3 (fork)]
files_created:
  - .planning/quick/260812-o5l-.../260812-o5l-SPIKE.md
  - .planning/quick/260812-o5l-.../260812-o5l-SUMMARY.md
---

# 260812-o5l — Attach the assessed photo to the chat assessor packet

**Spike verdict: FEASIBLE.** The uploaded image reaches a Python tool as
`context.user_content.parts[].inline_data` — a `Blob` whose `.data` is *already base64*, which
is exactly what Resend wants — **but only on the turn the image arrives**; it is stripped from
conversation history on every later turn. So the photo is captured on the upload turn and
carried to the packet turn in sliced session variables.

All four code changes are **live on the chat DRAFT and proven**. **No version was cut and
`d7bfbb93` was NOT repointed** — one required canary did not pass, and the brief's own rule is
that any failure leaves v13 `1eb3fd5c` serving. **The live chat demo is completely unchanged.**

## What was built

| # | Resource | Change | Size |
|---|---|---|---|
| 1 | `assess_screen_crack` | Captures the uploaded photo's base64 on the turn it arrives and slices it into ≤250,000-char session variables `photo_b64_0..N` + `photo_b64_parts` / `_mime` / `_len` | 4,796 → **6,934** (+2,138) |
| 2 | `send_case_record_email` | Reassembles the slices and sends them as Resend `attachments:[{filename, content}]`; returns `attached` and `photo_b64_len` | 2,435 → **4,155** (+1,720) |
| 3 | `resolve_claim` | Task 3b — customer email subject and body branch on whether a photo was assessed | 14,368 → **15,648** (+1,280) |
| 4 | `claim_intake` | Task 3a — spoken send-away line branches on the same thing (+832); plus a same-turn photo rule (+491) | 22,148 → **23,471** (+1,323) |

Every edit was a scripted anchored `str.replace()`, asserted unique before and byte-for-byte
read-back after, each producing **exactly one contiguous changed region**. Chat's instruction
and tool sources are **pure LF** (0 CRLF) — the CRLF trap is voice-only, confirmed by
measurement, not assumed. `resolve_claim`'s Resend key count was **1 before and 1 after**; its
source was never echoed beyond the 20 structural lines needed to place the anchor, none of
which carry the key. `grep -rE 're_[A-Za-z0-9_]{20,}' .planning/` returns no match.

**Nothing else moved.** Whole-object sha256 over all 10 tools and 3 agents: the other 7 tools
and 2 agents are **byte-identical**, tool count 10, `variableDeclarations` 37, `rootAgent`
unchanged, `globalInstruction` unchanged.

## How "no photo" is handled

Three independent guards, because the failure mode here is severe:

1. **A session variable may not exceed 262,144 bytes, and an oversize write fails the
   CUSTOMER'S ENTIRE TURN** with `HTTP 400 FAILED_PRECONDITION` — not a soft error the tool can
   swallow. Found by doing it. So `assess_screen_crack` slices at 250,000 and refuses outright
   above 4,000,000 base64 chars (~3 MB image), writing nothing.
2. `send_case_record_email` attaches only when `photo_b64_parts > 0` and the reassembled string
   is non-empty and within the cap. Anything else — no photo, a short read, an unexpected type —
   is caught and degrades to a packet with **no `attachments` key at all**, not an empty one.
3. Both blocks are wrapped so that any exception leaves the surrounding tool untouched.

**Proven, not asserted.** `executeTool` was used to invoke the real chat tool directly, twice:

| Call | Result |
|---|---|
| No photo variables | `{"sent": true, "status_code": 200, "subject": "[ASSESSOR] [CHAT] CLM-O5LNOPHOTO - Jordan Rivera", "attached": false, "photo_b64_len": 0}` |
| 7 slices of the real 1.1 MB demo photo | `{"sent": true, "status_code": 200, "subject": "[ASSESSOR] [CHAT] CLM-O5LPHOTO - Jordan Rivera", "attached": true, "photo_b64_len": 1530736}` |

The packet body and the `[ASSESSOR] [CHAT]` subject are **byte-unchanged** in both.

## Task 3 — the photo redundancy fix. This works, and it is the nicest thing here

Anchored on `claim_intake`'s own `"Ask for photos by email"` step: the 261-char first paragraph
was replaced with a 1,093-char branched version (**+832**, one contiguous region). `resolve_claim`
was branched the same way on `photo_check in ("confirmed", "contradicted")` — `retry` and
`unclear` deliberately do **not** count, because in those branches the photo was unusable and
asking for another one is correct. **Voice was not touched.**

Observed live on both paths:

| | Escalated `o5l-esc-b1` (CLM-24366) | Approved `o5l-appr-e1` (CLM-24537) |
|---|---|---|
| Customer email subject | `Claim CLM-24366 - photo received, nothing needed` | `Claim CLM-24537 - photo received, nothing needed` |
| Email body | *"The photo you sent has already been assessed and attached to your claim - there is nothing further you need to send us."* | same, and the opening is now *"your claim has been approved."* with the *"subject to receipt of photos"* clause correctly dropped |
| Spoken send-away line | 170 chars, *"The photo you sent is already attached to the claim, so there is nothing further you need to send us."* | 246 chars, *"Your photo is attached to the claim, so there is nothing further you need to send."* |

The old wording asked a customer to email in a photo they had just uploaded. It no longer does.

## Canary results — and the one that held the gate

Two live conversations against the **DRAFT** (the deployment was never exposed).

| Canary | Result |
|---|---|
| Send-away line still **SPOKEN** (the chat v11 `838b6d2b` regression) | ✅ **170 chars escalated, 246 approved**, and the text chunk is emitted **before any tool call** on that turn. This was the canary most at risk — Task 3 rewrote exactly that sentence |
| Packet files on the **email-confirmation turn**, six sections | ✅ `send_claim_email` → `generate_case_summary` → `send_case_record_email` → `escalate_to_human` → `end_session`, **all on turn 4**, Resend **200**, subject `[ASSESSOR] [CHAT] CLM-24366 - Jordan Rivera` |
| Exactly ONE customer email, asserted via `resolve_claim` | ✅ `email_queued: true`, once, on both runs. Never asserted via `send_claim_email` |
| Deterministic tariff | ✅ escalated **$3,000 / $25 / cutoff $1,500**, `TOTAL_LOSS`; approved **$840 / $25 / $1,500**, `REPAIRABLE`, `AUTO_APPROVE` |
| Decision card | ✅ fired on both runs; spoken decision delivered as the widget's `textResponse` (197 chars), the deterministic `dataMapping` path |
| Packet tools on the approve path | ✅ **zero** calls, no `NO_PACKET_REQUIRED` anywhere |
| Photo captured on the same turn as the upload | ✅ on `o5l-appr-e1` — `assess_screen_crack` fired on the image turn |
| **`cover_offer_actions` fires** | ❌ **DID NOT FIRE.** The agent went straight to `end_session` on the send-away turn |
| Attachment present on a live escalated packet | ❌ **`attached: false`** on `o5l-esc-b1` — see below |

**The gate is held. `d7bfbb93` still serves v13 `1eb3fd5c`, `updateTime` still `2026-08-12T20:51:48Z`, read back after the last write, not inferred.**

## The two failures, described honestly

**1. `cover_offer_actions` did not fire on the approve run.** The Cross-sell step is
**byte-unchanged** by this task (both it and `Close after the offer` are present, at the same
relative positions, with `{@TOOL: cover_offer_actions}` count 1 and `{@TOOL: end_session}` count
4 before and after). The customer's turn was *"Yes, that is right."*, which the agent read as
closure — the same "never reached, not miswired" failure mode 260810-ifr diagnosed and
260811-tgm confirmed on voice. **But byte-unchanged is not the same as unaffected**, and this
project has already been burned once by exactly that inference (05-02: a well-formed payload
that did not match the renderer's contract). The edit immediately precedes this step in the
instruction. **It needs one more approve run with a neutral closing turn before anyone repoints.**

Also observed and **not explained**: that `end_session` carried `reason: "escalation after
errors", session_escalated: true` on a cleanly **approved** claim. Whether that predates this
task is not established.

**2. The live escalated packet went out without the photo — and the reason matters more than
the miss.** On `o5l-esc-b1` the image arrived on turn 1 in a turn with **no tool call at all**,
and `assess_screen_crack` did not run until turn 3. By then the image was gone from context, so
capture correctly wrote nothing and the packet shipped with `attached: false` — **which is the
designed degradation working exactly as intended: Resend 200, six sections, packet delivered.**

The structural point: **the capture only exists if some Python tool runs on the upload turn.**
A `+491`-char rule was added to `claim_intake` telling the agent to call `assess_screen_crack`
**in the same turn** the photo arrives, before continuing the diagnostic, and it worked on the
next run (`o5l-appr-e1`, tool fired on the image turn). That fix is **one observation old**.

There is a second, deeper limitation worth recording: on the **total-loss** branch the agent
never asks for a photo at all, because the "See the damage" subtask triggers only on
`REPAIRABLE` + cracked screen. So an escalated claim carries a photo only when the customer
volunteers one or on the contradiction path. **The demo scenario that shows this feature off is
the photo-contradiction path** — reported crack, no damage visible, `DL-5`, human review, photo
attached for the assessor. That scenario has never been run live and needs a photo of an
undamaged device.

## What the user must do

1. **Check `akash.vinayak@nerdery.com` for two test emails** — `[ASSESSOR] [CHAT] CLM-O5LPHOTO`
   should carry an attachment named **`CLM-O5LPHOTO-damage.png`** (the smashed-MacBook photo);
   `[ASSESSOR] [CHAT] CLM-O5LNOPHOTO` should have none. **A Resend 200 proves acceptance, not
   delivery — only the inbox proves the attachment arrived and opens.** Nothing else in this
   task can settle it.
2. Two more real packets are in that inbox from the live runs — `CLM-24366` (escalated, no
   attachment, for the reason above) and the customer emails for `CLM-24366` / `CLM-24537`,
   which are the ones to read for the Task 3 wording.
3. **Decide on the repoint.** The draft is ready; the two canaries above are what stand in the
   way. Rollback target if anyone does repoint later: **chat v13 `1eb3fd5c-5aff-46c8-b572-e3fe18bf966f`**.

## Deviations

**[Rule 2 — missing critical functionality] The same-turn photo rule (+491 chars).** Not in the
brief. Without it the capture is a coin flip: the first live escalation put the image in a turn
with no tool call and lost it. The feature does not work reliably without this, so it was added.

**[Rule 1 — safety] The size guard is mandatory, not optional.** The brief framed it as "skip
rather than fail the send". The real hazard is upstream: an oversize *session variable write*
returns HTTP 400 and **kills the customer's turn**. The guard therefore sits in
`assess_screen_crack` before any write, not only in the mailer.

**[scope] `photo_status` was deliberately left writing `"requested"`.** It feeds `{photo_status}`
in the packet's FLAGS section. Changing it to `"received"` would alter packet text and risk a
canary for no benefit this task needs. Recorded so it is a decision, not an oversight.

**[method] The spike spent zero demo-app conversation quota.** `apps.executeTool` and a
throwaway app `o5lspikeprobe` (created, used, **deleted** — the app list is back to its original
five) carried all of it.

## Platform findings worth carrying forward

1. **`apps.executeTool` exists** — `POST {app}:executeTool` with `tool`, `args`, `variables` and
   `context`. It runs a Python tool with **no conversation and no quota**. This project has been
   rationing conversations for weeks; a large class of tool verification never needed one. It is
   how the attachment was proven.
2. **`SessionInput.image`** lets `runSession` push an image exactly as the widget does — the
   photo path is testable from the API, no browser required.
3. **Tool code changes are not effective immediately.** A conversation started seconds after a
   `tools.patch` ran the **previous** code, while the same tool ~40 s later ran the new one — and
   the byte-for-byte read-back said the new code was live the whole time. **Wait ~25 s after
   patching a tool.** This burned one conversation.
4. Session variables hold **5 MB across 20 keys**, survive turn boundaries exactly, and cost
   nothing on later turns (the read-back turn's whole response was 2,744 bytes, 1.8 s). Only the
   writing turn echoes the payload back.

## Self-Check: PASSED

- `260812-o5l-SPIKE.md` and `260812-o5l-SUMMARY.md` — created
- 4 chat resources changed, re-read from the API after each write; the other 7 tools and 2
  agents byte-identical by whole-object sha256
- `d7bfbb93` re-read after the last write → **`1eb3fd5c`**, unchanged
- Voice `6e01e4a5` and fork `9ae7a0c3`: **zero API calls of any method** — refused at the client gate
- Throwaway spike app deleted; app list back to its original five
- No git commit, by instruction
