---
phase: 06
plan: 03
artifact: CHAT-EVIDENCE
date: 2026-08-13
status: shipped
shipped_version: 129f8b31-f06e-48cc-90fb-15dcf8611db1
rollback_target: cdca14e3-5b0e-4675-9861-f5f22736362f
purpose: >
  The chat-side evidence 06-05 re-checks: session ids, assertion results, the shipped version
  id, and the rollback target. Every result below is read from a saved raw toolCall /
  toolResponse object, never from transcript prose.
---

# 06-03 — Chat evidence

## CHAT_WRITE

`record_claim` fires on the **decision turn**, on **both** branches, positioned
`resolve_claim` → `record_claim` → `claim_decision_card`.

### Why the decision turn, and why record_claim goes BEFORE the card

The decision turn is the only turn that definitionally happens on every resolved claim, on
both branches, and it is where every field already exists in session state. The email turn —
where 06-02's sibling work put the assessor packet — only happens if the customer speaks
again, which is exactly the fragility `260812-hhi` was created to fix.

**The card goes last, and that is load-bearing.** `claim_decision_card` is what delivers the
decision text (the v9 `dataMapping` `FIELD_MAPPING` `"textResponse": "explanation"`
mechanism), and **calling it terminates the model turn**. Anything instructed after it is
simply never called. Proven by execution, not inference — see `## CHAT_CANARIES`, "the two
runs that failed".

### Live evidence

| Run | Session | Branch | `record_claim` calls | Decision-turn tool order |
|---|---|---|---|---|
| c5esc | `c5esc-ab2d04ff` | HUMAN_REVIEW | **1** | `run_diagnostic` → `resolve_claim` → **`record_claim`** → `claim_decision_card` |
| c6appr | `c6appr-afa234bc` | AUTO_APPROVE | **1** | `run_diagnostic` ×2 → `resolve_claim` → **`record_claim`** → `claim_decision_card` |
| c1appr | `c1appr-9087deae` | AUTO_APPROVE + photo | **1** | `assess_screen_crack` → `resolve_claim` → `claim_decision_card` → `record_claim` *(superseded ordering — see note)* |

`record_claim` returned on every one of them:

```
recorded: true   status_code: 200   store_calls: 3
by_policy_status: 200   by_ref_status: 200   error_stage: ""
```

### The three claims chat filed, read back from GCS independently of the tool

Read with `gcloud storage cat`, not through `lookup_claim` — so neither the agent nor the
tool is vouching for itself.

| Claim | Run | `status` | amount / excess | `photo_assessed` | `rules_fired` | `channel` | `created_at` |
|---|---|---|---|---|---|---|---|
| `CLM-24599` | c1appr | `APPROVED` | 840.0 / 25.0 | **true** | `["DL-1"]` | `CHAT` | `2026-08-13T12:25:04Z` |
| `CLM-24536` | c5esc | `HUMAN_REVIEW` | 3000.0 / 25.0 | false | `["DL-3","DL-2"]` | `CHAT` | `2026-08-13T17:38:58Z` |
| `CLM-24841` | c6appr | `APPROVED` | 420.0 / 25.0 | false | `["DL-1"]` | `CHAT` | `2026-08-13T17:44:03Z` |

Every record also carries `claim_ref`, `policy_id`, `customer_name`, `device`, `issue_label`,
`coverage_limit`, `auto_approval_cutoff`, `total_loss_flag`, `photo_status` and
`schema_version: 1`.

**The stored figures equal the figures the customer was told.** Asserted per run as
`record_claim.args.claim_amount == resolve_claim.claim_amount` and
`record_claim.args.deductible == resolve_claim.deductible` — 840/25, 3000/25, 420/25.

### Latency — the number 06-04 needs

A full `record_claim` is 1 GET + 2 PUTs. Measured end-to-end from the executor via
`apps.executeTool`: **1.52 s / 1.55 s**. Inside a live conversation the added cost is smaller
still: on c1appr the platform issued `claim_decision_card` and `record_claim` in **one
parallel batch** and both responses landed 0.31 s apart.

**`ASYNCHRONOUS` was NOT used, and that was a judgement, not an oversight.** Reasons, in
order of weight: (1) the repoint is gated on observing `recorded: true` in a `toolResponse` on
the decision turn, and an asynchronous tool puts that observation out of reach of the gate;
(2) 06-02 locked `executionType: SYNCHRONOUS` on all four store tools and 06-05 re-checks port
parity — flipping chat alone is drift; (3) chat has no latency problem to solve. **06-04 will
care and should re-open it**: on voice, `record_claim` now sits between the caller's last word
and the spoken decision, which is the one place a second is audible.

## CHAT_LOOKUP

The read path lives on the **root agent `claims_concierge`**, not on `claim_intake`.
`claims_concierge` already owns `verify_identity`, so the caller is authenticated before any
lookup; and `claim_intake` is the instruction whose editing has repeatedly broken this
project, so the read path is kept out of it entirely.

Session `c7status-d9684db8`, 4 turns, `PDP100294` / Jordan Rivera, run against the DRAFT.

| Turn | Customer | `lookup_claim` result | What the agent said |
|---|---|---|---|
| 0 | "check on a claim I already have… Jordan Rivera, policy PDP100294" | `found: true`, `match_count: 7` | `disambiguation_line`, **byte-identical, once** |
| 1 | "The older one, please." | `found: true`, `match_count: 1`, `claim_ref: CLM-60202` | `status_line`, **byte-identical, once** |
| 2 | "And what about CLM-60203?" | `found: false` | the not-found line, **verbatim** |

**The instruction-only verbatim relay HELD. The named `dataMapping` widget fallback was NOT
built and was NOT needed.**

The two lines the customer actually received, unedited:

```
I can see 7 claims on this policy. The most recent is CLM-24841, your Apple MacBook Pro 16"
- keyboard replacement, filed on 13 August 2026. Is that the one you mean?
```

```
Claim CLM-60202 on policy PDP-100294: your Apple MacBook Pro 16", Liquid damage to keyboard.
It was approved for $310.00, less your $25.00 excess, so $285.00 comes to you.
Filed on 2 June 2026 via chat.
```

Notice the first line names `CLM-24841` — a claim **filed by chat 12 minutes earlier in a
different session**. The write half and the read half already meet.

Asserted, all from `toolResponse` objects:

- `verify_identity` ran **before** any lookup, exactly once.
- No `claim_intake` tool fired at all — `run_diagnostic`, `resolve_claim`,
  `claim_decision_card` and `record_claim` all zero. New-loss routing was not disturbed.
- **T-06-03, strong form:** `policy_source: "session"` on all three lookups. The policy came
  from the verified session, never from the model.
- "the older one" was passed as **`selector`**, not as a reference the customer read out, and
  resolved to `CLM-60202` — the genuinely oldest claim on the policy.
- Zero agent-composed figures on any relay turn: every `$` in every agent chunk came from the
  tool's string.
- The customer was **never asked to read a claim reference aloud**.

## CHAT_GRACEFUL_FAILURE

Three negative paths. Two live, one by invocation.

### 1. A reference belonging to another policy — live, c7status turn 2

`CLM-60203` is a real claim, on `PDP-100583`. Queried under `PDP-100294` it returned:

```json
{"found": false,
 "status_line": "I can't find a claim with that reference on this policy, so there is nothing for me to read back.",
 "match_count": 0, "policy_source": "session", "lookup_status_code": 200}
```

Exactly five keys. **No claim field of any kind** — asserted by whole-key-set equality, so
there is literally nothing for the model to hallucinate from. The control is structural as
well as textual: `lookup_claim` never reads `claims/by-ref/` at all, so another customer's
claim is not hidden from the response, it is never fetched.

Gated on the agent's own text for that turn:

- **zero** currency figures (regex `\$\s?\d` over every agent chunk) — 0 hits
- **zero** `CLM-` strings (regex `CLM[- ]?\d`) — 0 hits
- no claim detail from earlier in the conversation restated — no `310`, no `MacBook`
- the tool's line relayed verbatim, exactly once, and nothing else

### 2. A verified policy with nothing on file — by invocation on the shipped tool

`PDP-100746` (David Okafor) is a real policy that carries no claims. Not exercised live
because the conversation budget was spent; proven on the shipped source via
`apps.executeTool`, which costs no conversation:

```json
{"found": false,
 "status_line": "There are no claims on file for this policy at the moment.",
 "match_count": 0, "policy_source": "session", "lookup_status_code": 404}
```

Same five keys. An outage says so and does not invent an empty claim.

### 3. A write that fails — how it reports itself, on the SHIPPED chat tool

Invoked with arguments it cannot proceed on:

```json
{"recorded": false, "status_code": 0, "store_calls": 0,
 "error_stage": "validate", "error": "claim_ref and policy_id are both required",
 "claims_on_policy_after": 0}
```

**A failed write reports itself as a failed write.** `recorded` is derived at the end of the
function from status codes already observed, never pre-set; `claims_on_policy_after` is forced
to 0; the stage it stopped at is named. This is the deliberate inverse of `resolve_claim`'s
`email_queued: true` literal, which `260812-trc` proved cannot report a failure.

**What the customer gets when the write fails: exactly what they would have got anyway.** The
instruction says so in as many words — the decision stands, the email still goes out, the case
record is still filed, the agent does not retry, does not apologise, and tells the customer
nothing either way. The one thing it may never do is contradict the tool: it never says or
implies the claim was stored. The failure lives in the tool's own return fields, in the
conversation record, where 06-05 can find it.

## CHAT_CANARIES

Every canary below cites a specific `toolCall` / `toolResponse`. A canary with no cited
evidence would count as a FAIL.

### The two runs that failed, and what they proved

This is the finding of the plan, and it cost two conversations.

| Session | Result |
|---|---|
| `c2esc-6851866c` | escalated cleanly, packet filed, **`record_claim` call count 0** |
| `c3esc-f1a4ad37` | same script after strengthening the instruction — **`record_claim` call count 0 again** |

Both ran the decision turn as `resolve_claim` → `claim_decision_card` and stopped. The
instruction said, at the top of the step and again in its own paragraph, to call
`record_claim`. It was ignored — **only on the escalated branch**, while the approve branch
(c1appr) fired it.

The cause is not instruction strength. **`claim_decision_card` terminates the model turn**,
because the card is what delivers the decision text. On c1appr `record_claim` survived only
because the model happened to issue card + `record_claim` in one parallel batch — luck, not
design. Reordering the instruction to `resolve_claim` → `record_claim` → `claim_decision_card`
fixed it on the first attempt, on both branches.

**Carry forward to 06-04 and to every future turn edit on this project: nothing may be
instructed after a widget tool call that carries a `textResponse`. The widget is the end of
the turn.**

### The shipped battery

Run against the DRAFT before any repoint, on the exact instruction that shipped.

| Canary | Result | Evidence |
|---|---|---|
| `record_claim` fires exactly once per resolved claim, `recorded: true`, 2xx | **PASS** | c5esc, c6appr — `store_calls: 3`, `by_policy_status: 200`, `by_ref_status: 200` |
| Stored amount and excess byte-equal to `resolve_claim`'s figures | **PASS** | 3000/25 and 420/25, arg-vs-result comparison |
| `channel` is the literal `CHAT` | **PASS** | `record_claim.args.channel == "CHAT"` both runs |
| Store record carries every criterion-1 field | **PASS** | GCS read-back of all three refs |
| Relayed status sentence byte-identical to `lookup_claim.status_line`, once | **PASS** | c7status turn 1 |
| Multi-match relays `disambiguation_line`; `selector` resolves | **PASS** | c7status turns 0–1, `CLM-60202` |
| Unknown / mismatched reference fails gracefully, no figure spoken | **PASS** | c7status turn 2 |
| Deterministic tariff, `rules_fired` matching the branch | **PASS** | 420/25/1500 `["DL-1"]`; 3000/25/1500 `["DL-3","DL-2"]` |
| Spoken decision **byte-identical** to `resolve_claim.explanation`, exactly one agent chunk | **PASS** | c6appr 199 chars, c5esc 240 chars — TEXT/API channel |
| Exactly ONE customer email (structural: `resolve_claim` once, `email_queued: true`) | **PASS** | both runs — **not delivery evidence, `260812-trc` voided that** |
| **The send-away line is still SPOKEN** (v11 `838b6d2b`) | **PASS** | c5esc 184 chars, c6appr 187 chars |
| Send-away text chunk precedes every tool call on the escalated turn | **PASS** | c5esc: `text` → `send_claim_email` → `generate_case_summary` → `send_case_record_email` → `escalate_to_human` → `end_session` |
| Assessor packet files on the email-confirmation turn | **PASS** | c5esc, all five tools on turn 2 |
| Six packet headings, each exactly once | **PASS** | `{SUMMARY:1, ACTION:1, CLAIM:1, DIAGNOSTIC:1, RULES FIRED:1, FLAGS:1}` |
| Zero `{placeholder}` braces | **PASS** | regex `\{[a-z_]+\}` → 0 hits |
| `packet_text` byte-identical to the composer's response | **PASS** | 429 == 429 |
| `send_case_record_email` `sent: true`, HTTP 200, `[ASSESSOR] [CHAT]` | **PASS** | `[ASSESSOR] [CHAT] CLM-24536 - Jordan Rivera` |
| Photo attachment path still works | **PASS** | mailer re-invoked on byte-unchanged source, `sent: true`, 200; capture proven live on c1appr, `photo_b64_len: 1530736`, `photo_check: confirmed` |
| Cross-sell fires with the correct `{actions:[…]}` shape | **PASS** | c6appr — `[{content:"Add it",utterance:"Yes please, add it to my cover."},{content:"Not now",…}]` |
| Decision card `productItem` byte-identical to `card_product_item`, `units` an int | **PASS** | c6appr 420, c5esc 3000 |
| `DIAGNOSTIC_INCOMPLETE` guard present in the snapshot | **PASS** | v15 snapshot assertion W6 |
| Packet tools fire **zero** times on the approve path, no `NO_PACKET_REQUIRED` | **PASS** | c6appr |
| Cross-sell does **not** fire on the escalation path | **PASS** | c5esc |
| **Zero store vocabulary in any customer-facing text** | **PASS** | word-boundary regex over every agent chunk of c5esc, c6appr, c7status: store / record_claim / lookup_claim / saved your claim / database / lookup / our records / claim store — **0 hits** |
| Zero packet vocabulary in any agent chunk | **PASS** | c5esc |

### Conversations spent

**Seven**, against a plan budget of five. Recorded as a deviation, not hidden.
`c1appr-9087deae`, `c2esc-6851866c`, `c3esc-f1a4ad37`, `c4esc-04593740`,
`c5esc-ab2d04ff`, `c6appr-afa234bc`, `c7status-d9684db8`. Two were spent diagnosing the
widget-terminates-the-turn defect (c2esc, c3esc); one was lost to a transport stall that
killed the runner mid-conversation (c4esc) — its resumed turns re-entered verification and
carried no evidence. No 429 was ever returned and no retry loop was ever run.

## CHAT_SHIP

| Thing | Value |
|---|---|
| **Version cut** | **`129f8b31-f06e-48cc-90fb-15dcf8611db1`** (chat v15) |
| **Rollback target** | **`cdca14e3-5b0e-4675-9861-f5f22736362f`** (chat v14) — **re-read LIVE immediately before the PATCH**, `updateTime 2026-08-13T01:35:50.254056Z` unchanged since plan start |
| Deployment | `d7bfbb93-8cee-43fe-9095-bc5775f353bd` (`WEB_UI` / `CHAT_ONLY`) |
| **Repoint timestamp** | **`2026-08-13T17:56:20.954538Z`** |
| `channelProfile` across the repoint | **BYTE-IDENTICAL**, sha256 `e6977330c6e3294b…` before and after; `channelType` still `WEB_UI` |
| Read back, not inferred | `d7bfbb93` re-read after the PATCH serves `129f8b31` |

**The repoint was gated in code.** `t3_ship.py` runs the three shipped-instruction assertion
batteries plus the zero-conversation battery as subprocesses and `sys.exit`s before touching
`/versions` if any returns non-zero. It then re-reads the rollback target live, asserts voice
and the app list are untouched, cuts the version, audits the snapshot, and only then PATCHes.

Version snapshot audit, all passing: 12 tools including `record_claim` and `lookup_claim`;
`claim_intake` 28,215 chars / 11 tools / sha16 `766116e2693590b5`; `claims_concierge` 8,075
chars / 3 tools / sha16 `df9c9dd0e2c27453`; both new sentinels present exactly once; every
pre-existing hardening string present; no v11-shaped gag clause; 37 `variableDeclarations`.

### Rollback

```
gcloud auth print-access-token | { read T; curl --max-time 120 --connect-timeout 15 -s -X PATCH \
  -H "Authorization: Bearer $T" -H "Content-Type: application/json" \
  --url "https://ces.googleapis.com/v1beta/projects/insurance-agent-demo-500614/locations/us/apps/a2f621e4-9faf-505a-b804-22471f022366/deployments/d7bfbb93-8cee-43fe-9095-bc5775f353bd?updateMask=appVersion" \
  -d '{"appVersion":"projects/insurance-agent-demo-500614/locations/us/apps/a2f621e4-9faf-505a-b804-22471f022366/versions/cdca14e3-5b0e-4675-9861-f5f22736362f"}'; }
```

### Isolation at plan close

| Assertion | Result |
|---|---|
| Voice app `6e01e4a5` — every agent instruction sha | **unchanged** |
| Voice deployment `d28bbcb0` | **`5d9df25c`**, `updateTime 2026-08-12T20:51:46.574949Z` — identical at plan start and close |
| Fork `9ae7a0c3` | **zero calls of any method**; client gate logged zero refusals because nothing was attempted |
| Every chat tool, whole-object sha256 | **unchanged**, `resolve_claim` included — no Resend code touched |
| App list | **five**, same ids as at plan start; no probe app was created |
| Store debris | latency-probe objects `by-ref/CLM60390.json` and `by-policy/PDP100390.json` **deleted** |
| Retained seed data | 06-02's `CLM-60201/60202/60203` fixtures and `CLM-60210`/`CLM-60211` cross-channel pair — **deliberately retained**, 06-05 reads them |
| New store contents from this plan | `CLM-24599`, `CLM-24536`, `CLM-24841`, all on `PDP100294`, all `channel: CHAT` |
