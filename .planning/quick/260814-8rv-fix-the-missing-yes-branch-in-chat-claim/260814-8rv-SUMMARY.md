---
task: 260814-8rv
title: Fix the missing "yes" branch in chat claim status — and the record email the cross-sell was eating
subsystem: cx-agent-chat-app
tags: [claim-status, lookup_claim, confirm-path, cross-channel-parity, widget-terminates-turn, send_case_record_email, cover_offer_actions, chat, shipped]
date: 2026-08-14
status: complete
chat_version_cut: 8a95ab02-e653-4d1e-baee-eb500c52710c
chat_rollback_target: 619e13a1-2a31-4627-aca3-2ff3a32e336b
chat_repoint_timestamp: "2026-08-14T11:40:17.051321Z"
repointed: true
voice_touched: false
defects_fixed: 2
gate_draft: 116/116
gate_deployed: 44/44
conversations_spent: 9
requires: ["06-04 (the voice confirm-path design this mirrors)", "06-02 STORE-CONTRACT", "260814-80f"]
affects: ["06-05 (the chat/voice confirm-path divergence it inherited is now closed)"]
---

# Quick task 260814-8rv — the missing "yes" branch, and the record email the cross-sell was eating

**Chat v20 `8a95ab02-e653-4d1e-baee-eb500c52710c` is LIVE on `d7bfbb93`**, repointed off v19
`619e13a1-2a31-4627-aca3-2ff3a32e336b` at **`2026-08-14T11:40:17.051321Z`**, read back not inferred,
`channelProfile` sha `76f98fed6482c930` byte-identical across the write, `WEB_UI`.

The rollback target was **re-read live from the API immediately before the PATCH** and the ship
script aborts if either the serving version *or* its `updateTime` has moved since plan start. The
cut and the repoint were both **gated in code on a 116-assertion battery's exit status**, run twice —
once before the cut and again immediately before the repoint. A **44-assertion deployed battery**
then re-proved everything against `d7bfbb93` itself.

**Voice was never written to.** `6e01e4a5` received GET calls only; `d28bbcb0` still serves v19
`e484ce3e` with `updateTime` unmoved at `2026-08-14T11:10:40.558076Z`. Fork `9ae7a0c3` received zero
calls of any method — the client refuses any URL naming it.

---

## 1. Defect one — the gap was one sentence, and voice had already solved it

**Chat's `claims_concierge`, live on v19, said this and nothing more:**

> When the tool has several matches it returns disambiguation_line instead. Say that one, word for
> word, exactly once, and wait for their answer. **Then call {@TOOL: lookup_claim} again with
> selector set to what they said**, and read back the status_line it returns the same way.

The customer's answer to *"Is that the one you mean?"* is *"yes"*. That went into `selector`.
`lookup_claim`'s selector resolution is ordinal → reference → date → device/issue words, and **none
of the four has a branch for a bare agreement** — so `"yes"` fell through to `NO_MATCH`, `found:
false`, and the agent read out *"I can't match that to a claim on this policy, so there is nothing
for me to read back"* **immediately after reading the customer's claim back to them.** That is Beat 2
of the demo script.

**Voice fixed this in 06-04 by routing the confirmation through the reference the tool itself had
already supplied** — `claim_ref`, not `selector` — so the customer never has to repeat it. That
design was mirrored, not re-invented.

### The anchor and the delta

| | |
|---|---|
| Agent | chat `claims_concierge` (root) |
| Anchor | the 289-char `When the tool has several matches…` line, sliced from chat's **own CRLF bytes**, asserted to occur **exactly once** |
| Replacement | 1,173 chars, authored in a **file**, applied by scripted `str.replace()`, instruction handled through JSON only |
| Delta | 9,591 → **10,475** (**+884**), against a band of **+700…+1,300 declared before running** |
| Contiguity | prefix 7,917 / old span 148 / new span 1,032 / suffix 1,526 — **exactly one contiguous region** |
| Read-back | **byte-identical to the intended string**, sha `1737036cb4f3ab46…`; re-verified in the version snapshot **before** the repoint |
| `beforeModelCallbacks` | **SURVIVED byte-identical** — 869 chars, sha `42a4fe1e…`, all three branches present. Asserted explicitly, because that is exactly what a careless `updateMask` wipes |

Byte-unchanged and asserted: the `<role>` slab, the entire **Verify the caller** subtask, the
**Anything else** subtask, the `<constraints>` block, the `authenticated = true` routing sentence,
93 CRLF / 0 bare LF, zero non-ASCII, brace count unchanged at 6 (no `{placeholder}` introduced).

### Chat and voice now behave identically on confirmation — and the enumerated differences

The behavioural contract is the same on both channels: **an affirmative resolves through `claim_ref`
taken from the disambiguation line; anything that points at a different claim resolves through
`selector`; the alternatives list is never read out.** Proven identical by execution on both.

The prose differs in three enumerated, deliberate ways, all channel register or reliability:

1. **Register.** Voice says *"say"*, *"out loud"*, *"never make them repeat it back"*; chat says
   *"read back"*, *"never make them type it back"*. Voice's rationale sentence (*"a spoken list is
   impossible to hold in your head on a phone call"*) is replaced by chat's (*"they are not for the
   customer"*).
2. **Chat lists more affirmatives.** Voice names *"yes"*, *"that's it"*, *"that's the one"*; chat
   adds *"yep"*, *"correct"*, *"right"*, **and the catch-all *"or any other plain agreement"*** —
   because a fix that matches only literal tokens is not a fix. Verified against three different
   affirmatives on two builds.
3. **Chat carries an explicit negative that voice does not:** *"A bare agreement is NOT a selector:
   never pass 'yes' or anything like it as selector, because nothing matches it and you would be
   telling the customer there is nothing on file immediately after reading their claim back to
   them."* This is the failure mode named in the instruction, not a second design. It is the one
   place chat is now **stronger** than voice, and 06-05 may want to carry it back.

---

## 2. Defect two, folded in mid-task — the cross-sell widget was eating the record email

The user filed a real claim and got no `[RECORD]` email. Reported live conversation
`dfMessenger-1e6f363e…`: `generate_case_summary` fired, `cover_offer_actions` fired,
**`send_case_record_email` never did.**

### It was reproduced before it was fixed — that mattered, because the reconstruction was wrong

The coordinator's proposed fix was *"order the packet send before the cross-sell widget."*
**The instruction already said exactly that.** In document order the Case record step (offset 27,669)
precedes the Cross-sell step (29,976), and the Case record step already reads *"First call
{@TOOL: generate_case_summary} … Then call {@TOOL: send_case_record_email}"*. Re-ordering text that
is already in the right order would have shipped a no-op with a plausible story attached.

Conversation **D2**, the identical script run against `claim_intake` **byte-identical to live v19**,
reproduced the defect exactly:

```
[3] TEXT  the send-away / email line
[3] call  send_claim_email
[3] call  generate_case_summary
[3] call  cover_offer_actions      <- widget, issued in the SAME batch as the composer
[3] resp  send_claim_email
[3] resp  generate_case_summary
[3] resp  cover_offer_actions
                                   <- turn over. send_case_record_email never issued.
```

**The mechanism is 06-03's finding taken one step further.** 06-03 established that *a widget tool
call carrying a `textResponse` terminates the model turn*, which is why `record_claim` must precede
`claim_decision_card`. The extra step is this:

> **`send_case_record_email` cannot be issued in the same batch as `generate_case_summary`, because
> it takes that composer's response as its `packet_text`.** It needs a *second round of tool calls
> inside the turn*. A widget issued in the first batch ends the turn before that second round can
> happen. So document ordering is not enough — **a two-round dependency and a turn-ending widget
> cannot share a batch**, and the model batches by default even with `toolExecutionMode: SEQUENTIAL`.

`record_claim` → `claim_decision_card` survives only because `record_claim` needs nothing after it.
The packet chain does. That is the whole difference, and it is why the same instruction shape worked
in one place and failed in the other.

### The fix: state the dependency, not the order

Two anchored in-place amendments to `claim_intake`, both at the **opening of a step's `<action>`** —
06-03's carried-forward lesson that the opening lines are what the model reads when deciding which
tools a turn calls, and `260812-o5l`'s that appending after a step's closing rule is how the v11
silencing was created.

- **Case record step** — names the two rounds explicitly, states the permitted call order for the
  turn, and bans issuing `cover_offer_actions` alongside `generate_case_summary`, with the reason.
- **Cross-sell step** — makes the offer wait: it does not begin until `send_case_record_email` has
  returned, and `cover_offer_actions` is the **last** tool call of the turn.

| | |
|---|---|
| Anchors | 558 chars and 189 chars, each asserted to occur **exactly once**, each **preserved verbatim as the prefix** of its replacement |
| Delta | `claim_intake` 32,498 → **34,311** (**+1,813**), band **+900…+2,200 declared before running** |
| Regions | **exactly two**, measured by opcode diff: two pure insertions of 1,228 and 585 chars — **no original byte was deleted or moved** |
| Read-back | byte-identical, sha `9a8f95b1293f6c55…` |
| Line endings | `claim_intake` is **pure LF** where `claims_concierge` is CRLF; regions authored in LF, 0 CR introduced |

Byte-unchanged and asserted: `- AUTO_APPROVE:`, the `DIAGNOSTIC_INCOMPLETE` paragraph, the whole
Diagnose subtask, the send-away step, the "Look at it and say what you see" step, the "Ask for photos
by email" step, the "Price and decide" step, `<constraints>`, *"THE PACKET IS REPRODUCED WORD FOR
WORD"*, *"Use those four strings verbatim"*, *"Do NOT end the session in this turn."*, and
**every original non-blank line still present**. `tools`, `beforeToolCallbacks` (p23's photo gate)
and `afterModelCallbacks` all byte-identical.

### Both fire. Same turn. Proven twice, including on the deployment

Deployed run **P2** on chat v20, AUTO_APPROVE with a photo uploaded:

```
[2] TEXT  send-away line, BEFORE every tool call
[2] call  send_claim_email
[2] call  generate_case_summary
[2] resp  send_claim_email
[2] resp  generate_case_summary
[2] call  send_case_record_email   -> sent:true  status_code:200  attached:TRUE  photo_b64_len:208
                                      subject "[RECORD] [CHAT] CLM-30707672 - Jordan Rivera - APPROVED"
[2] resp  send_case_record_email
[2] TEXT  "One more thing while I have you: your Apple iPhone 16 Pro Max isn't on this policy..."
[2] call  cover_offer_actions      -> Add it / Not now, verbatim, LAST tool call of the turn
```

**The turn did not need splitting.** The coordinator asked to report if it did; it did not.

---

## 3. This also settles 80f's cross-sell canary — and 80f's conclusion was half right

80f recorded the cross-sell as *pre-existing-broken*, having failed across three runs on two builds
with a bogus `end_session {"reason":"escalation after errors"}`. The user's live conversation showed
the opposite: `cover_offer_actions` fired and the record email did not.

**Both observations are the same defect seen from two ends.** The two tools were competing for the
last slot of one turn, and **whichever the model batched first ended it**. 80f watched the mailer
win; the user watched the widget win. Neither tool was broken.

**My conclusion, stated as a conclusion and not an attribution:** the cross-sell was never
independently defective. With the dependency stated, **both fire, in order, on every run of this
task's fixed build** — draft D3 and deployed P2. I saw no `end_session {"reason":"escalation after
errors"}` on any of the nine conversations. I did not need a live control against v19 to attribute
it, because I had something stronger: **D2 is a live reproduction of the defect on a byte-identical
`claim_intake`, and D3 is the same script on the fixed one.**

---

## 4. The five verification results the brief asked for

All five taken on the **draft** first, then the confirm path and a full AUTO_APPROVE re-taken on the
**deployment** after the repoint.

| # | Check | Result |
|---|---|---|
| **1** | Look up a policy, get a read-back, answer **"yes"** | **PASS.** D1: `lookup_claim{}` → `match_count: 25` → disambiguation → bare *"yes"* → `lookup_claim{claim_ref: "CLM-24690"}` → `found: true`, `match_count: 1`, `status_line` relayed byte-identically exactly once. **No `selector` was passed.** |
| **2** | At least one other natural affirmative | **PASS ×2.** D6 *"that's the one"* → `claim_ref`, `found: true`. **P1 on the deployment,** *"correct"* → `claim_ref`, `found: true`. Three distinct affirmatives, two builds. |
| **3** | A negative still works, and no list is read aloud | **PASS.** D5 *"no, the older one"* → `lookup_claim{selector: "the older one"}` (**not** `claim_ref`) → `CLM-60202`, a *different* claim, `match_count: 1`. Zero of the alternatives named in any agent chunk across both turns. |
| **4** | Not-found still graceful, fabricates nothing | **PASS.** D1 `CLM-99999999` → `found: false`, **exactly the five contract keys**, zero claim fields, zero `$` in the turn, the literal relayed verbatim. |
| **5** | Filing a claim end to end still works | **PASS.** D3 (approve), D4 (escalation, total loss), D7 (approve→escalation with photo), **P2 on the deployment** (approve + photo). Intake is undisturbed: `lookup_claim` fired **zero** times in every one. |

---

## 5. Canaries

**Draft battery 116/116. Deployed battery 44/44.** Every one cited to a `toolCall`/`toolResponse` or
to a live API read — never to transcript prose.

| Canary | Result |
|---|---|
| Deterministic tariff, approve `420/25/1500` `DL-1` | **PASS** (D3, P2) |
| Deterministic tariff, escalate `3000/25/1500` `DL-3`+`DL-2` `total_loss_flag: true` | **PASS** (D4) |
| Spoken decision **byte-identical** to `resolve_claim.explanation`, emitted **exactly once** | **PASS** on both branches, draft and deployed |
| **The send-away line is still SPOKEN** | **PASS** on every run, both branches — and on all three it was emitted **before the first tool call of the turn** |
| `record_claim` `recorded: true`, **`store_calls: 4`** (not 3), `precheck_status: 404`, `ref_collision: false` | **PASS** on every resolved claim |
| `record_claim` ordered **before** `claim_decision_card` | **PASS** on every branch (06-03's rule) |
| `lookup_claim` reads by policy, `policy_source: "session"` | **PASS**; the model passed no `policy_id`, and the argument-ignoring path is unchanged |
| Decision card renders, `productItem.price.units` an **int** | **PASS** |
| `capture_claim_photo` fires | **PASS**; `captured: true`, `photo_b64_len: 208` on the photo runs |
| Packet files with the right subject token at 200 | **PASS**: `[RECORD] [CHAT] … - APPROVED` and `[ASSESSOR] [CHAT] …`, both `sent: true`, HTTP **200** |
| Packet **attached: true** when a photo was uploaded | **PASS** (D7 escalation, **P2 deployed auto-approve**) |
| Six packet headings each **exactly once**, zero `{placeholder}` braces | **PASS** on all four shipped-build packets |
| Cross-sell fires, buttons verbatim | **PASS**, and now **after** the mailer, as the turn's last call |
| Cross-sell correctly **absent** on escalation | **PASS** (D4) |
| `escalate_to_human` exactly once | **PASS** |
| `claims_concierge`'s **869-char `beforeModelCallbacks` intact**, all three branches | **PASS** — live, and re-asserted in the **version snapshot before the repoint** |
| Greeting still opens with *"How can I help you today?"* | **PASS**, via the near-free `{"event": {"event": "session start"}}` probe |
| Zero packet / store vocabulary spoken to the customer | **PASS** |
| `case_summary` byte-unchanged; **all 13 chat tools byte-unchanged**, `resolve_claim` included | **PASS** |
| Voice untouched: three instructions byte-unchanged, `d28bbcb0` on v19 `e484ce3e`, `updateTime` unmoved | **PASS** |
| App list **five**, no probe app, fork zero calls | **PASS** |

**No canary was skipped, and none held only "in substance".**

---

## 6. Deviations

**[Rule 1 — bug, and the substantive one] The coordinator's stated fix would have been a no-op.**
The instruction already ordered the mailer before the widget. The real cause is the two-round
dependency, and the fix states the dependency rather than the order. Recorded in §2 rather than
quietly substituted.

**[deviation] Nine conversations.** Budgeted loosely for one defect; a second arrived mid-task. The
nine: D1 (yes-branch + not-found), **D2 (the deliberate live reproduction of defect two)**, D3
(approve, fixed), D4 (escalation, fixed), D5 (negative), D6 (second affirmative), D7 (photo path),
**P1 + P2 on the deployment**. None was a retry of another; D2 in particular is the control that
kept me from shipping a no-op. **No 429 and no 529 was seen at any point and no retry loop was ever
run.** Turns were paced 10–12 s apart.

**[deviation] Two assertion failures were mine, not the edit's, and both stopped the script.** The
pre-patch gate refused twice on brace/mention arithmetic I had miscounted. Nothing was written on
either refusal; the assertions were corrected against measured values and re-derived from the
region files, not loosened.

**[method] The photo path was proven by conversation, not by `executeTool`.** `SessionInput.image`
(`{"image": {"data": "<b64>", "mimeType": "image/png"}}`) pushes an image into a live conversation
over the API exactly as the widget does — o5l found this and no later plan used it. A 156-byte
synthetic PNG went in on the acknowledgement turn and came out the other end as `attached: true`,
`photo_b64_len: 208`, on the real mailer. **`apps.executeTool` was not needed and was not used.**

**[not done] The runbook was not fully reconciled.** It is stale from two prior tasks (it still
names chat v18 `d0e4bfef` as live). Only the rows this task changes were corrected — version, roll
back, the Beat 2 confirm line, and the `[RECORD]` note. A full reconciliation is a task of its own.

---

## 7. Store hygiene

Five claims were filed by this task's conversations and **all five were removed**: `CLM-30706882`,
`CLM-30707088`, `CLM-30707233`, `CLM-30707434`, `CLM-30707672` — each `by-ref` object deleted and
each entry pruned from `by-policy/PDP100294.json`, rewritten with
`json.dumps(obj, sort_keys=True, separators=(",", ":"))`, 06-01's load-bearing serialisation.

Re-audited at close, and the store is **byte-for-byte back at 80f's closing state**:

```
30 objects | 27 by-ref | 27 by-policy entries | 3 policies
PDP100017: 1   PDP100294: 25   PDP100583: 1
Class A (ref under >1 policy): 0    orphaned entries: 0    debris: none
```

No fixture was touched. No HMAC key was read, minted or handled — the writes used the executor's own
credentials via `gcloud storage`.

---

## 8. Follow-ups

1. **⚠ `PDP100294` still holds 25 claims**, so Beat 2 opens with *"I can see 25 claims on this
   policy."* Correct behaviour, bad demo line. Third task in a row to flag it. Prune it, or run the
   status beat on **Maria Santos / `PDP100583`** (one claim, resolves without disambiguation — but
   note that then exercises no confirm path at all, so rehearse both).
2. **`generate_case_summary` leaked a literal `{dx_answers}` into a packet, once.** Seen on the D2
   control (*"reported via {dx_answers} as having several keys not working"*). It did **not** recur
   on any of the four shipped-build packets, all of which pass the zero-braces canary. The
   `case_summary` agent is byte-unchanged by this task, so this is pre-existing and intermittent.
   Worth a look before a customer demo, because that string would appear in an assessor's inbox.
3. **The confirm-path negative clause is stronger on chat than on voice.** 06-05 may want to carry
   *"a bare agreement is NOT a selector"* back to voice for exact parity.
4. **Nobody has watched v20 in a browser.** Everything here is server-side, including the deployed
   run. The decision card, the cross-sell buttons and the status read-back are all byte-unchanged in
   definition from builds verified by eye, but v20 itself is unseen.
5. Unchanged and untouched: the Resend key rotation, `resolve_claim`'s `email_queued: true`
   truthfulness defect, the ~30 % delivery rate, the cracked-screen same-turn-photo deadlock, and
   the voice write (`record_claim` still attached to no agent on voice — 06-05).

---

## Self-Check: PASSED

- `260814-8rv-SUMMARY.md` — created
- `.planning/STATE.md` — updated
- `.planning/spec/DEMO-RUNBOOK.md` — updated (chat version + rollback rows, Beat 2 confirm line,
  the `[RECORD]` note)
- `d7bfbb93` re-read **after** the PATCH → **`8a95ab02-e653-4d1e-baee-eb500c52710c`**,
  `updateTime 2026-08-14T11:40:17.051321Z`, `channelProfile` sha `76f98fed6482c930` byte-identical
  across the write, `WEB_UI`
- Rollback target **re-read live immediately before the PATCH**; the ship script aborts on either a
  moved version or a moved `updateTime`
- Version snapshot verified byte-identical to both intended instructions **before** the repoint,
  with the 869-char callback and all three branches asserted present
- Draft gate **116/116**, deployed gate **44/44**; the cut **and** the repoint conditional on the
  gate's **exit status**, re-run between them
- `resolve_claim` never read, echoed, printed or persisted; no tool source of any kind was read
- Voice `6e01e4a5`: **zero non-GET calls**. Fork `9ae7a0c3`: **zero calls of any method** — the
  client refuses any URL naming it. Chat v11 `838b6d2b` and voice v12 `9227210b` never deployed
- Store re-audited at close: **30 objects, 27 entries, 27 `by-ref`, 0/0/0 collisions, no debris**
- App list re-read at close: **five**. No probe app was created
- `grep -rE 're_[A-Za-z0-9_]{20,}|ya29\.|AKIA|GOOG1[A-Z0-9]{20,}'` over `.planning/` → **no match**
- **No git commit, by instruction** — the tree is deliberately left dirty
