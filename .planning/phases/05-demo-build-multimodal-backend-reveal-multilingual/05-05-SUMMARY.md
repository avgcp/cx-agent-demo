---
phase: 05-demo-build-multimodal-backend-reveal-multilingual
plan: 05
subsystem: chat-agent-wiring
tags: [ces, assessor-packet, resend, version-cut, deployment-repoint, regression-canaries]
status: awaiting-human-verification
requires:
  - "05-04 components (case_summary, generate_case_summary, send_case_record_email)"
  - "05-03 PACKET_RECIPIENT = same-mailbox"
provides:
  - "app:a2f621e4/agents/claim_intake — the Case record step on the HUMAN_REVIEW branch"
  - "app:a2f621e4/versions chat v10 658472a0 — serving on d7bfbb93"
  - "the assessor briefing packet delivered as a real email, Resend HTTP 200"
affects:
  - "05-07 (assessor packet on voice — reuse this wiring shape)"
  - "05-08 / 05-09 (Spanish work now builds on v10)"
tech-stack:
  added: []
  patterns: ["scripted str.replace() instruction edit", "conditional repoint gated on a re-run assertion battery"]
key-files:
  created:
    - ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-05-SUMMARY.md"
  modified:
    - "app:a2f621e4/agents/claim_intake (87551704) — 19,003 → 20,505 chars, 8 → 10 tools"
    - "app:a2f621e4/deployments/d7bfbb93 — 160dc3b2 (v9) → 658472a0 (v10)"
    - ".planning/spec/DEMO-RUNBOOK.md"
    - ".planning/STATE.md"
decisions:
  - "Attached generate_case_summary and send_case_record_email to claim_intake in the same PATCH (updateMask=instruction,tools) — the plan named instruction only, but an unattached tool cannot be called"
  - "Ran a second conversation for the English auto-approve canary, and gated the repoint on BOTH batteries"
  - "Did NOT weaken the email step's own-turn rule to make the packet fire a turn earlier — recorded as a presenter instruction instead"
metrics:
  conversations_spent: 2
  turns_spent: 8
  version_cut: 658472a0-05be-4a28-9d9f-9774ebe0dd05
  deployment_repointed: "d7bfbb93 → 658472a0"
  completed: 2026-08-11
---

# Phase 5 Plan 05: Wire, Prove and Ship the Assessor Briefing Packet Summary

An escalated chat claim now composes a six-section assessor briefing packet from its own
session state and delivers it as a real `[ASSESSOR]`-prefixed email — proven live end to end
with Resend returning HTTP 200, byte-identical between composer and mailer, invisible to the
customer, and with the single customer confirmation and every v7→v9 hardening item unregressed
on the same runs. Shipped as **chat v10 `658472a0-05be-4a28-9d9f-9774ebe0dd05`**, now served by
`d7bfbb93`.

## The specific answers this plan was asked for

| | |
|---|---|
| **New version ID** | **`658472a0-05be-4a28-9d9f-9774ebe0dd05`** — *"chat v10 - assessor briefing packet composed by case_summary and delivered by send_case_record_email on the HUMAN_REVIEW branch"* |
| **Was `d7bfbb93` repointed, or held at v9?** | **REPOINTED**, and read back rather than inferred. `160dc3b2` (v9) → `658472a0` (v10) at `2026-08-12T02:45:46Z`. `channelProfile` — which is where `channelType: WEB_UI` and `webWidgetConfig` actually live — is **byte-identical** before and after. |
| **The packet's six sections** | All six present, exactly once each, with substituted values and **no `{placeholder}` braces** — reproduced in full below |
| **Assessor email subject** | **`[ASSESSOR] CLM-24413 - Jordan Rivera`** |
| **Assessor email recipient** | **`akash.vinayak@nerdery.com`**, `from` unchanged at `onboarding@resend.dev` |
| **Delivery result** | `sent: true`, `status_code: 200` |
| **Customer-send count, asserted via `resolve_claim`** | **Exactly one.** `resolve_claim` fired **once**, returning `email_queued: true`, subject `Claim CLM-24413 - please reply with photos`. (`send_claim_email`, the keyless reporter, also appeared in exactly one turn — recorded for completeness, not used as the proof.) |
| **Did the repeated-question fix and the `es-US` config ship in v10?** | **Yes, both**, asserted inside the v10 snapshot — and English did **not** regress on either run |

## The packet that was actually emitted

Captured from the conversation record's `toolResponse`, session `esc-25e925f8`, claim
`CLM-24413`. 520 characters. The customer name reads `[REDACTED]` **in the conversation record
only** — the CES API redacts PII in stored transcripts; the real name went to the mailer.

```
SUMMARY: Review liquid damage claim for [REDACTED] Apple MacBook Pro 16" which has no power following water contact.
ACTION: Review claim details, rules fired, and photo evidence when received to make a final determination.
CLAIM: Customer: [REDACTED]; Policy: PDP100294; Device: Apple MacBook Pro 16"; Issue: liquid_damage; Amount: 3000.0; Excess: 25.0; Total-loss flag: true.
DIAGNOSTIC: Customer reported no power and water contact.
RULES FIRED: DL-3 (Total loss reported), DL-2 (Liquid damage reported).
FLAGS: none.
```

`packet_text` handed to `send_case_record_email` is **byte-identical** to
`generate_case_summary`'s `response` — 520 chars each, exact string equality, not normalised
and not truncated. T-05-08 is closed by evidence, not by posture.

## ⚠ The one behavioural finding a presenter must know

**The packet is filed on the turn AFTER the email confirmation, and the customer has to say
one more thing to get there.**

The escalated chat turn sequence is:

| Turn | Customer says | Agent does |
|---|---|---|
| 1 | policy + name | `verify_identity`, hands to `claim_intake` |
| 2 | the liquid damage | `run_diagnostic` ×2, `resolve_claim`, `claim_decision_card`, speaks the decision + specialist handoff |
| 3 | *"Okay, thanks"* | `send_claim_email`, the reply-with-photos line |
| **4** | ***anything*** | **`generate_case_summary` → `send_case_record_email` → `escalate_to_human` → `end_session`, all silent** |

**On the first attempt the run stopped at turn 3 and no packet was filed.** This is not a
defect introduced by this plan and it is not a wiring failure — `claim_intake`'s email step has
carried the clause *"This is its own turn and ends with the last word of it"* since long before
this change, and the pre-existing `escalate_to_human` / `end_session` pair was already stranded
behind the same boundary (05-04 found a HUMAN_REVIEW conversation on file where
`send_claim_email` was never called at all). The new step simply inherits that boundary.

**It was fixed by adding a fourth turn, not by weakening the instruction.** Sending
*"Thanks, that's everything."* produced the whole silent sequence and a clean
`end_session` with `session_escalated: true`, `reason: "escalated to specialist"`. Collapsing
the Case record step into the email turn was considered and rejected: it would mean editing the
one clause that keeps the email beat from being bundled with the decision beat, which is a
visible, verified demo property, and it would have cost another version and another
verification conversation to re-prove. The cost of the alternative is one sentence in the
runbook, which is where it now is, in bold, under Scenario D.

## Task 1 — the instruction edit

`claim_intake` (`87551704-9bc1-4fe0-bde2-57d377cb8963`), edited by scripted `str.replace()`
against the API. The instruction body was never printed in full and never composed inline in a
tool-call argument.

| Gate | Result |
|---|---|
| Pre-edit length | **19,003** chars, sha256[:16] `acf62e78877c2ea7` — **not** the 18,496 the plan predicted; see deviation 1 |
| Anchor uniqueness | START and END anchors each counted **exactly 1** before replacement |
| `<step name="Case record">` absent before, `generate_case_summary` / `send_case_record_email` absent before | ✅ |
| Exactly one contiguous region differs | ✅ common prefix 15,403 chars, common suffix 2,531 chars |
| Length delta | **+1,502** (bound +600…+1,800) |
| Decision sentinel survives | ✅ `THE CARD TEXT RESPONSE IS THE ONLY PLACE YOU SAY THE DECISION` ×1 |
| Ten preserved literals | ✅ all present — `<step name="Cross-sell">` ×1, `Close after the offer`, `Skip this step entirely on the HUMAN_REVIEW path`, `Do NOT end the session in this turn`, `WORD FOR WORD`, `assess_screen_crack`, `claim_decision_card`, `cover_offer_actions`, `send_claim_email`, `end_session`, `</taskflow>` |
| New step | ✅ `<step name="Case record">` ×1; packet sentinel ×1 |
| Step count | 10 → **11**; `Case record` sits after `Ask for photos by email` and before `Cross-sell` |
| Brace-token vocabulary | unchanged plus the two new `{@TOOL: …}` references — no bare/undeclared token introduced |
| Post-PATCH read-back | **byte-identical** to the file sent; 20,505 chars, sha256[:16] `f24cd195e306073e` |
| `validationErrors` | absent |
| `childAgents` / `transferRules` | byte-identical to pre-edit |
| app `variableDeclarations` | **37**, unchanged |
| Deployment during Task 1 | still `160dc3b2` ✅ |

**The packet sentinel, carried verbatim so it can be grepped:**

> `THE PACKET IS REPRODUCED WORD FOR WORD - if you change a figure, a heading or a line break, the case record is wrong.`

## Task 2 — the version, the live proof, the repoint

### v10 snapshot audit — all pass

| Check | Result |
|---|---|
| `POST /versions` returns the version resource, not an LRO | ✅ id read straight off the response, no `done` field, no polling |
| Tools in snapshot | **10**, including `generate_case_summary` and `send_case_record_email` |
| Agents in snapshot | `case_summary`, `claim_intake`, `claims_concierge` |
| `claim_decision_card.widgetTool` vs v9 | **byte-identical**, sha256[:16] `0f5a8404649265fa` |
| `cover_offer_actions.widgetTool` vs v9 | **byte-identical**, sha256[:16] `46e9a7e1b64567f2` |
| `resolve_claim`, `send_claim_email`, `run_diagnostic`, `assess_screen_crack`, `verify_identity`, `escalate_to_human` vs v9 | whole-object **unchanged**, all six |
| `variableDeclarations` | 37 |
| snapshotted `claim_intake` | 20,505 chars, 10 tools, `<step name="Case record">` ×1 |
| **the 260811-suy repeated-question fix shipped** | ✅ `DIAGNOSTIC_INCOMPLETE` present in the snapshot |
| **the k5x decision sentinel shipped** | ✅ present in the snapshot |
| **the 05-03 multilingual config shipped** | ✅ `{"defaultLanguageCode":"en-US","supportedLanguageCodes":["es-US"],"enableMultilingualSupport":true}` |
| Deployment before the first conversation | still `160dc3b2` ✅ |

### The nine live assertions — ALL PASS

Session `esc-25e925f8`, chat **DRAFT** (`runSession` with no `config.deployment`), 4 turns.
Every value below comes from a `toolCall` or `toolResponse` object, never from transcript prose.

| # | Assertion | Evidence |
|---|---|---|
| 1 | `resolve_claim` → `HUMAN_REVIEW`, `claim_amount` 3000 (JSON int), rules include `DL-3` and `DL-2` | `["DL-3","DL-2"]`, `CLM-24413` ✅ |
| 2 | `generate_case_summary` fired **exactly once**; response carries all six headings | 1 call / 1 response; `SUMMARY` `ACTION` `CLAIM` `DIAGNOSTIC` `RULES FIRED` `FLAGS` each ×1 ✅ |
| 3 | No `{ }` brace pair in the packet | zero matches — no unresolved variable token leaked ✅ |
| 4 | `send_case_record_email` fired **exactly once**; `packet_text` **byte-identical** to the composer's response | 520 == 520, exact equality ✅ |
| 5 | `sent == true`, 2xx `status_code`, subject starts with `[ASSESSOR]` | `true`, **200**, `[ASSESSOR] CLM-24413 - Jordan Rivera` ✅ |
| 6 | Single customer send | `resolve_claim` ×1 with `email_queued: true`; `send_claim_email` in exactly one turn ✅ |
| 7 | **No customer-facing agent text** contains a packet heading, `assessor`, `case record`, `briefing packet` or `packet` | 3 agent text chunks, **zero** hits ✅ |
| 8 | `claim_decision_card` `productItem` byte-identical to `resolve_claim.card_product_item` | ✅ `units` int `3000`, `nanos` 0, `USD` |
| 9 | `cover_offer_actions` did **not** fire on the escalation path | 0 calls ✅ |

The customer heard exactly three things across the whole claim, and none of them mention the
packet:

> *"I've got your Apple MacBook Pro 16" here, what's happened to it?"*
> *"So that comes to $3,000, and it looks like the device is a total loss. I'm going to pass this to one of our specialists. They'll already have everything we've talked about, so you won't need to go over it again. Your reference is CLM-24413."*
> *"An email is on its way to the address on your policy. Please reply to it with photos of the damage attached, and allow 3 to 7 business days for a representative to confirm the outcome."*

Turn 4 produced **no customer-facing text at all** — the packet, the escalation and the session
close were entirely silent. T-05-02 is closed server-side; the human check in Task 3 closes it
on screen.

### The English auto-approve canary — ALL PASS

Session `appr-0c70bb34`, chat DRAFT, 4 turns, run **before** the repoint so the repoint was
gated on it. This is the run that proves nothing from v7→v9 regressed and that enabling `es-US`
did not disturb English.

| Canary | Baseline | This run | |
|---|---|---|---|
| `resolve_claim` decision | `AUTO_APPROVE` | `AUTO_APPROVE` | ✅ |
| Deterministic tariff | $420 keyboard / $25 excess / $1,500 limit | $420 / $25 / `$1,500` named in the explanation, `CLM-24774` | ✅ |
| `claim_decision_card` fired | once | once | ✅ |
| `productItem` vs `card_product_item` | byte-identical | **byte-identical** | ✅ |
| `price` shape | int units, nanos 0, USD | `{"units":420,"nanos":0,"currencyCode":"USD"}` | ✅ |
| Spoken decision line vs `resolve_claim.explanation` | byte-identical | **byte-identical**, 199 chars | ✅ |
| Spoken decision line count (the k5x canary) | exactly 1 | **exactly 1** — no doubling | ✅ |
| `cover_offer_actions` fired | once | once | ✅ |
| Cross-sell payload | `{actions:[{content,utterance}×2]}`, no `description` | `[{"content":"Add it","utterance":"Yes please, add it to my cover."},{"content":"Not now","utterance":"Not right now, thanks."}]` | ✅ |
| **`generate_case_summary` on the approve path** | must not fire | **0 calls** | ✅ |
| **`send_case_record_email` on the approve path** | must not fire | **0 calls** | ✅ |
| `NO_PACKET_REQUIRED` anywhere in the record | absent | **absent** | ✅ |
| Single customer send | `resolve_claim` ×1, `email_queued: true` | ✅; `send_claim_email` in exactly one turn | ✅ |
| **English unaffected by `es-US`** | — | **0 Spanish markers**, 0 Spanish stopwords, 24 English stopwords, `Conversation.languageCode` `en-US` | ✅ |

**The double gate held in practice, not just on paper.** The packet tools were attached to
`claim_intake` and reachable on this run, and they were called **zero** times, because the
decision was `AUTO_APPROVE`. `case_summary`'s own `NO_PACKET_REQUIRED` fallback never had to
fire — the instruction-level trigger stopped it first. That is the stronger of the two results:
a misfire would have been visible as a `NO_PACKET_REQUIRED` string in the record, and there
isn't one.

### The repoint

Gated in code: `repoint.py` re-executed **both** assertion scripts and refused to PATCH unless
both exited 0. They did.

| | Before | After |
|---|---|---|
| `appVersion` | `…/versions/160dc3b2-571c-480f-b901-e4dbe8947f70` | `…/versions/658472a0-05be-4a28-9d9f-9774ebe0dd05` |
| `channelProfile` (holds `channelType` + `webWidgetConfig`) | `{"channelType":"WEB_UI","webWidgetConfig":{"modality":"CHAT_ONLY","theme":"LIGHT","webWidgetTitle":"Meridian Device Protection","securitySettings":{"enablePublicAccess":true}}}` | **byte-identical** |
| `updateTime` | — | `2026-08-12T02:45:46.995886Z` |

**Rollback, as an explicit step.** If anything is wrong on screen, point `d7bfbb93` back at
chat v9 `160dc3b2-571c-480f-b901-e4dbe8947f70` and reload `http://localhost:3000`. The exact
`curl` is now in DEMO-RUNBOOK.md under *If something goes wrong (chat)*, together with a chat
version table. Rolling back costs the packet beat and nothing else — the card, the spoken
decision, the cross-sell, the tariff and the customer email are byte-identical in v9 and v10.

### Apps not touched

**No API call of any kind was issued to voice `6e01e4a5-42a8-5213-b3da-c9053ff8ea52` or fork
`9ae7a0c3-6511-413c-8cdb-0efe9e90d2b9`.** The shared client refused any non-GET whose URL
either named a forbidden app or failed to name `a2f621e4`.

## Deviations from Plan

### 1. [Rule 3 — blocking] The pre-edit instruction is 19,003 chars, not the 18,496 the plan asserts

**Found during:** Task 1, the very first gate.
**Issue:** The plan tells the executor to *"assert its length is exactly 18,496 chars … A
different length means something changed underneath this plan; stop and report."* The live
value is **19,003**. Stopping would have been wrong: the difference is **known, intended and
already recorded** — quick task `260811-suy` (2026-08-11) added a **+507**-char
`DIAGNOSTIC_INCOMPLETE` recovery paragraph to both apps' `claim_intake` drafts, taking chat
from 18,496 to 19,003 with sha256[:16] `acf62e78877c2ea7`. STATE.md names both numbers.
**Fix:** the gate was re-pinned to 19,003 **and** to the `acf62e78877c2ea7` hash, which is the
stronger check — a length match alone could not have distinguished the suy edit from an
unrelated one of the same size. Both passed.
**Consequence:** cutting v10 ships the suy fix to the live chat demo. That was flagged as
intended by the orchestrator, and it is asserted present in the v10 snapshot above. The live
chat demo no longer carries the repeated-diagnostic-question defect; **the live phone demo
still does**, since voice `d28bbcb0` is untouched and still serves v11 `b17c9a26`.

### 2. [Rule 3 — blocking] The tools had to be attached, not just named in the instruction

**Found during:** Task 1.
**Issue:** The plan specifies `apps.agents.patch` with **`updateMask=instruction`** only — but
its own acceptance criterion requires `claim_intake` to report **10 tools** afterwards, and it
carried 8. A `{@TOOL: generate_case_summary}` reference in an instruction does nothing if the
tool is not in the agent's `tools` list; the step would have been unreachable and assertion 2
would have failed with zero calls.
**Fix:** one PATCH with `updateMask=instruction,tools`, sending the original 8 tool paths plus
the two new ones. Because a `tools` mask replaces the whole list, the pre-edit list was
captured, re-read from a fresh GET immediately before writing to confirm nothing had moved
underneath, and each of the 8 originals was asserted still present in the read-back.
**Verification:** 10 tools after; all 8 originals present; both new tools present;
`childAgents` and `transferRules` byte-identical.

### 3. [Rule 3 — blocking] The escalated path needed a fourth turn

Covered in full above. The plan's three-turn script reaches the email and stops; the packet is
filed on the next turn. One extra turn was spent (4 total, inside the plan's ≤5 budget).
**This is a pre-existing turn-boundary property of the email step, not a fault in the wiring**
— `escalate_to_human` and `end_session` were already behind the same boundary. Recorded as a
bolded presenter instruction in the runbook rather than fixed by weakening a verified clause.

### 4. [scope — orchestrator directive] Two conversations, not one

The plan budgets **one** conversation; the orchestrator additionally required an English
auto-approve canary proving the card, the cross-sell, the tariff and the `es-US` config. That
cannot be observed on an escalated claim, which deliberately has no cross-sell and no
auto-approve tariff. **Two conversations, 8 turns total**, paced ~62 s apart. No 429 occurred,
so the authorised single spaced retry was never used, and no retry loop ever ran. The canary
was run **before** the repoint and the repoint was gated on it, which is strictly safer than
the plan's ordering.

### 5. [Rule 1 — bug in this task's own tooling] Two wrong read paths, both in assertion scripts

Neither touched server state; both were found and corrected before any conclusion was drawn.

- **v10 snapshot audit:** `variableDeclarations` and `languageSettings` live under
  `snapshot.app`, not `snapshot`. The first run reported two spurious FAILs. Re-read at the
  correct path: 37 and the full `es-US` config. **No second version was cut** — the same
  snapshot was re-audited from disk.
- **Tool payloads are nested one level:** a Python tool's result is at
  `toolResponse.response.result`, an agentTool's at `toolResponse.response.response`. The first
  assertion run read `toolResponse.response` and reported `decision: None`. This is the same
  class of mistake `260810-k5x` recorded for `productItem`, and it is worth writing down twice:
  **on this platform, always unwrap the tool response before asserting on it.**

### 6. [documentation] `channelType` and `webWidgetConfig` are not top-level on the deployment

The plan asks the repoint to assert `channelType` is `WEB_UI` and `webWidgetConfig` is
byte-unchanged as top-level fields. On both `v1beta` and `v1` they are **nested inside
`channelProfile`**, and a naive top-level read returns `None` for both — which would have made
the assertion pass vacuously. Caught and re-asserted against `channelProfile` as a whole:
`channelType: WEB_UI`, `webWidgetConfig` byte-identical.

### 7. [documentation] The conversation record redacts the customer name

`generate_case_summary`'s response and `send_case_record_email`'s `customer_name` argument both
read `[REDACTED]` in the stored conversation record — CES redacts PII in transcripts. The value
that reached the mailer was the real one; the subject line assertion still fired on the
`[ASSESSOR]` prefix, which is not redacted. **This is why the inbox check in Task 3 is not
optional:** the record cannot show that the packet named Jordan Rivera, only that it named
somebody.

## Authentication gates

None. A fresh `gcloud auth print-access-token` was taken immediately before every single API
call (the shared client fetches one per request), and no token expired mid-run. Every call was
bounded — 120 s request timeout, and the two conversations ran detached so no foreground call
could approach a watchdog.

## Threat register outcomes

| Threat | Outcome |
|---|---|
| T-05-08 `packet_text` altered across the model turn | **Mitigated and proven.** Word-for-word sentinel in the step; live assertion 4 is an exact 520-byte equality |
| T-05-02 packet leaked to the customer | **Mitigated server-side.** Assertion 7 scanned every agent text chunk for six headings plus `assessor`, `case record`, `briefing packet` and `packet` — zero hits; turn 4 emitted no text at all. On-screen confirmation is Task 3 |
| T-05-10 double customer send | **Mitigated.** `resolve_claim` fired once with `email_queued: true` on both runs; `send_claim_email` in exactly one turn on both. See the STATE.md update closing the older open question |
| T-05-11 the edit silently reverts k5x/ifr/suy | **Mitigated.** Anchor uniqueness, one contiguous region, +1,502 bounded delta, ten preserved literals, byte-identical read-back, and all three fixes asserted present inside the v10 snapshot |
| T-05-05 rig serves an unverified build | **Mitigated.** Both conversations ran against the DRAFT; the repoint re-ran both batteries in code and would have refused on any failure; rollback version named in the plan, the summary and the runbook |
| T-05-01 Resend key | **Mitigated.** No tool source was read, echoed or copied. `grep -rE 're_[A-Za-z0-9_]{20,}' .planning/` returns no match. The runbook's key-revocation note now names `send_case_record_email` |
| T-05-04 quota | **Mitigated.** 2 conversations, 8 turns, ~62 s pacing, no 429, no retry loop |
| T-05-09 fork spoofing | **Mitigated.** Client-level write gate on the literal `a2f621e4`; zero calls to `6e01e4a5` / `9ae7a0c3` |
| T-05-SC package installs | n/a — none |

## Threat Flags

None. The only new network surface is the `POST https://api.resend.com/emails` inside
`send_case_record_email`, which 05-04 already created and covered; this plan added a call site,
not a surface. No new auth path, no schema change at a trust boundary.

## Known Stubs

None. No hardcoded empty value, placeholder string or unwired component was introduced. The
packet's `FLAGS: none.` line is a real computed value from the session state, not a placeholder.

## Still open after this plan

1. **Nobody has seen either email.** Resend returned 200 and the customer send is queued by
   `resolve_claim`, but a status code is not an inbox. **Task 3 is a blocking human checkpoint.**
2. **Nobody has watched the browser** on v10. Three on-screen items 05-02 left open are still
   open: the spoken decision line printing once vs twice, the decision card drawing as it did on
   2026-08-09, and the `Add it` / `Not now` buttons drawing in a single row. All three are
   proven byte-correct server-side on the canary run and unchanged since v9.
3. **The photo CONTRADICTION path is still unverified live** — unchanged by this plan.
4. **Voice has no packet yet.** 05-07 owns it, and should reuse this wiring shape — including
   the turn-boundary finding, which will bite harder on a phone call where the caller may simply
   stop talking.

## Self-Check: PASSED

- `.planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-05-SUMMARY.md` — created
- `.planning/spec/DEMO-RUNBOOK.md` — modified; `grep -c 'briefing packet'` = 4,
  `grep -c 'send_case_record_email'` = 2, `grep -c '160dc3b2'` = 7
- `.planning/STATE.md` — modified
- Version `658472a0-05be-4a28-9d9f-9774ebe0dd05` — `GET` returns it, snapshot audited
- Deployment `d7bfbb93` — re-`GET` after the PATCH returns `appVersion` ending `658472a0`
- Agent `claim_intake` — re-`GET` after the PATCH returns 20,505 chars byte-identical to what
  was sent, 10 tools, no `validationErrors`
- **Commits: none by instruction.** The tree is deliberately left dirty; the orchestrator owns
  the docs commit. ROADMAP.md untouched.
