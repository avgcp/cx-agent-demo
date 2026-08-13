---
phase: 06
plan: 03
subsystem: cx-agent-chat-app, shared-claim-store
tags: [claim-store, record_claim, lookup_claim, chat, decision-turn, widget-terminates-turn, verbatim-relay, shipped]
version_cut: 129f8b31-f06e-48cc-90fb-15dcf8611db1
rollback_target: cdca14e3-5b0e-4675-9861-f5f22736362f
repoint_timestamp: "2026-08-13T17:56:20.954538Z"
verbatim_relay: "instruction-only relay HELD - the dataMapping widget fallback was NOT built and NOT needed"
conversations_spent: 7
covers_success_criteria: [1, 3, 5, 6]
requires: ["06-02 STORE-CONTRACT (TOOL_CONTRACT, STORE_SURFACE)"]
provides: ["chat writes every resolved claim to the store", "chat reads any claim back", "06-03-CHAT-EVIDENCE.md"]
affects: ["06-04 (wires voice)", "06-05 (cross-channel assertion)"]
date: 2026-08-13
---

# Phase 6 Plan 03: Wire the claim store into chat — Summary

**Version cut:** chat v15 `129f8b31-f06e-48cc-90fb-15dcf8611db1`.
**Rollback target, re-read LIVE immediately before the PATCH:** v14 `cdca14e3-5b0e-4675-9861-f5f22736362f`, `updateTime 2026-08-13T01:35:50.254056Z`, unmoved since plan start.
**Repoint:** `d7bfbb93` → v15 at **`2026-08-13T17:56:20.954538Z`**, read back not inferred; `channelProfile` byte-identical across it.
**Verbatim relay:** the **instruction-only relay HELD** — the named `dataMapping` widget fallback was **not built and not needed**.

**Phase 5 canaries, one line each — all PASS, all cited to a `toolCall`/`toolResponse`:**
deterministic tariff **PASS** · spoken decision byte-identical to `resolve_claim.explanation`, exactly once **PASS** · exactly one customer email (structural, via `resolve_claim`) **PASS** · **send-away line still SPOKEN PASS** · send-away text before every tool call on the escalated turn **PASS** · assessor packet on the email-confirmation turn **PASS** · six headings each once **PASS** · zero `{placeholder}` braces **PASS** · `packet_text` byte-identical **PASS** · `send_case_record_email` 200 with `[ASSESSOR] [CHAT]` **PASS** · photo attachment path **PASS** · cross-sell `{actions:[…]}` **PASS** · decision card `productItem` byte-identical, `units` int **PASS** · `DIAGNOSTIC_INCOMPLETE` present **PASS** · packet tools zero on approve **PASS** · cross-sell absent on escalation **PASS** · **zero store vocabulary in any customer-facing text PASS**.

## Headline

**A claim resolved on chat is now in the store before the customer has finished reading the decision, and a chat customer can hear the status of a claim they did not file in that session — in a sentence the model did not compose.**

Three claims were filed live by chat and read back out of GCS independently of the tool that wrote them: `CLM-24599` (approved, $840, photo assessed), `CLM-24536` (human review, $3,000, `DL-3`+`DL-2`), `CLM-24841` (approved, $420). The stored figures equal the figures the customer was told, asserted per run as an argument-versus-result comparison.

Then, in a separate session, a verified customer asked where their claim had got to and was read `CLM-24841` back — a claim filed by chat twelve minutes earlier in a different conversation.

## The finding that cost two conversations

**A widget tool call that carries a `textResponse` terminates the model turn. Anything instructed after it is never called.**

The plan's design — and the first thing built — put `record_claim` *after* `claim_decision_card`, on the reasoning that the decision text must reach the customer before any further tool call. That is the right instinct and it is the message-first rule the escalated path already obeys. But on chat the decision text **is** the card: since v9 the card's `dataMapping` `FIELD_MAPPING` `"textResponse": "explanation"` is what delivers the sentence, which is exactly why it is deterministic. So the card is not a step before the text — it *is* the text, and it closes the turn.

Consequences, all observed:

| Session | Branch | `record_claim` calls | What happened |
|---|---|---|---|
| `c1appr-9087deae` | AUTO_APPROVE | 1 | fired — **by luck**: the model issued card + `record_claim` in one parallel batch |
| `c2esc-6851866c` | HUMAN_REVIEW | **0** | `resolve_claim` → `claim_decision_card`, turn over |
| `c3esc-f1a4ad37` | HUMAN_REVIEW | **0** | same, after strengthening the instruction with an explicit "on BOTH branches" three-tool sequence and a "never end this turn without calling it" clause |

Two rounds of stronger wording changed nothing, because the problem was never instruction strength. Reordering to `resolve_claim` → `record_claim` → `claim_decision_card` fixed it on the first attempt, on both branches, and the card stayed exactly where it has always been: last, delivering the decision text, byte-identically, once.

**Carry forward:** this generalises past this plan. Any future edit that adds work to a turn ending in a widget must put that work *before* the widget. 06-04 inherits it directly — voice's decision turn has the same shape.

## Where `record_claim` fires, and why

**On the decision turn, between `resolve_claim` and `claim_decision_card`, on both branches.**

The decision turn is the only turn that definitionally happens on every resolved claim, on both branches, and it is where every field already exists in session state. The alternative — the email-confirmation turn, where `260812-hhi` put the assessor packet — is one the customer only reaches by speaking again. That is precisely the fragility `260812-hhi` existed to fix for the packet, and there is no reason to re-introduce it for the store write.

The customer-facing text of that turn is **unchanged**. Asserted on both shipped runs: the spoken decision is byte-identical to `resolve_claim.explanation`, emitted as exactly one agent chunk, delivered last on the turn, with no agent-composed figure anywhere near it.

**Cost to the customer: none measurable.** A full `record_claim` is 1 GET + 2 PUTs, measured at 1.52 s end-to-end from the executor and 0.31 s between the card's and `record_claim`'s responses when the platform ran them in parallel.

## `ASYNCHRONOUS` was considered and deliberately not used

06-01 established that `ExecutionType` accepts `ASYNCHRONOUS` (and rejects `"ASYNC"` with a 400). It was not used here, for three reasons in order of weight:

1. **The repoint is gated on evidence an async tool would put out of reach.** The gate requires `recorded: true` and a 2xx read from a `toolResponse` on the decision turn. Asynchronous execution removes that observation from the turn record, and the whole point of this plan's gate is that the repoint is conditional on code, not on a checklist.
2. **Port parity.** 06-02 locked `executionType: SYNCHRONOUS` on all four store tools and 06-05 re-checks parity. Flipping chat alone manufactures the drift 06-05 is built to catch.
3. **Chat has no latency problem to solve.** There was nothing to buy.

**06-04 should re-open this, and the reason is sharper than "voice is slower".** Under the shipped ordering `record_claim` sits *between the caller's last word and the spoken decision*. On a web widget that is invisible; on a phone call it is the one place a second of silence is audible, immediately before the demo's best moment. 06-04 has three options — `ASYNCHRONOUS`, `ces_requests.async_*`, or accepting ~1 s — and now has the measured numbers to choose with.

## What happens when the write fails

`record_claim` reports it, and nothing else changes.

Proven by invocation on the **shipped chat tool** (not a copy):

```json
{"recorded": false, "status_code": 0, "store_calls": 0,
 "error_stage": "validate", "error": "claim_ref and policy_id are both required",
 "claims_on_policy_after": 0}
```

The instruction does not paper over this. It states in as many words that a `false` is not a failure of the claim: the decision stands, the email still goes out, the case record is still filed, and the agent carries on to the next step exactly as it otherwise would. It must not retry, must not call the tool twice, and must not tell the customer anything either way. The single prohibition is against contradicting the tool — the agent never says or implies the claim was stored, on any branch, whatever it returns.

So the customer conversation completes normally and the failure is **visible in the record** rather than swallowed: `recorded`, `status_code`, `read_status`, `by_policy_status`, `by_ref_status`, `store_calls`, `error` and `error_stage` are all in the conversation's `toolResponse`, where 06-05 can find them. This is the deliberate inverse of `resolve_claim`'s unconditional `email_queued: true`, which `260812-trc` proved cannot report a failure at all.

## The read path, and why it lives on the root agent

`lookup_claim` is attached to **`claims_concierge`**, not to `claim_intake`. Two load-bearing reasons: `claims_concierge` already owns `verify_identity`, so the caller is authenticated before any lookup can run; and `claim_intake` is the instruction whose editing has repeatedly broken this project, so the read path is kept out of it entirely. The write lives in one agent, the read in the other, and neither edit can break the other.

Session `c7status-d9684db8`, `PDP100294`, four turns:

| Turn | `lookup_claim` | Relay |
|---|---|---|
| 0 | `found: true`, `match_count: 7` | `disambiguation_line` byte-identical, once |
| 1 | `selector: "the older one"` → `CLM-60202`, `match_count: 1` | `status_line` byte-identical, once |
| 2 | `claim_ref: "CLM-60203"` (a real claim on **another** policy) | the not-found line, verbatim |

**T-06-03 is mitigated in the strong form and was proven, not assumed:** `policy_source: "session"` on all three lookups. The policy came from the verified session, never from the model, and `lookup_claim` never reads `claims/by-ref/` at all — another customer's claim is not hidden from the response, it is never fetched.

The customer was never asked to read a claim reference aloud; identification was by name and policy id, and "the older one" was passed as `selector`. New-loss routing was untouched: no `claim_intake` tool fired at any point in that conversation.

## The edits, and the delta

| Agent | Before | After | Delta | Regions |
|---|---|---|---|---|
| `claim_intake` | 24,630 | **28,215** | **+3,585** | two insertions, then two in-place rewrites of those same two insertions |
| `claims_concierge` | 5,126 | **8,075** | **+2,949** | one insertion |

Both lengths were **read live from the API at plan start**, not taken from a planning document — Phase 5's quoted figures are stale, which is why the baseline was captured first. (A figure of 22,980 was quoted to me for `claim_intake`; the live read at plan start was 24,630, and every assertion in this plan is anchored on the live value.)

**What the `claim_intake` delta bought,** since it is roughly three times what the packet wiring needed:

- 718 chars — the opening three-tool sequence, stating the order and that it applies on **both** branches. This is the part the escalated branch actually obeys.
- ~2,900 chars — the recording block: the ordering rule and why the card is last; the fifteen arguments named individually against `06-02-STORE-CONTRACT.md ## TOOL_CONTRACT` verbatim, with "invent nothing" and an instruction to fall back to the tool's default rather than guess; the silence clause; and the failed-write behaviour spelled out branch by branch.

The argument list is the bulk of it and it is not padding: `record_claim` takes fifteen arguments, a drift in any one of them surfaces only at runtime, and this project has already lost a plan to a tool argument nobody checked (`timeout=20`, 05-07).

`claims_concierge` grew by a whole new subtask — trigger, identify-first, the verbatim-relay rule with its sentinel, disambiguation, not-found, and the ban on store vocabulary.

Every edit was a scripted `str.replace()` from a **file**, with anchors sliced out of each agent's own bytes, uniqueness asserted, a delta bound stated before running, prefix/suffix measurement of the changed span, and a byte-for-byte read-back. `claims_concierge` on chat is **CRLF** (63 CRLF, 0 bare LF) while `claim_intake` is pure LF — so the region was authored in LF and converted in-process, never through a text-mode round trip.

**Byte-unchanged, asserted against the plan-start capture:** the `- AUTO_APPROVE:` line, the Cross-sell step body, the Case record step body, the "Ask for photos by email" step body, the "Look at it and say what you see" step body, the whole Diagnose subtask, the `<constraints>` block, the `DIAGNOSTIC_INCOMPLETE` paragraph, the v9 card-text sentinel, and on `claims_concierge` the entire "Verify the caller" subtask, the `<constraints>` block and the `authenticated = true` → `claim_intake` routing sentence. **Every one of the twelve chat tools is byte-unchanged by whole-object sha256, `resolve_claim` included.**

## Deviations from plan

**[Rule 1 — bug, and the substantive one] `record_claim` moved BEFORE `claim_decision_card`.** The plan specifies "the decision text reaches the customer before any further tool call in that turn", with `record_claim` called after. That premise does not hold on chat: since v9 the decision text is delivered *by* the card, so the card is a tool call and it closes the turn. Following the plan literally produced `record_claim` call count **0** on the escalated branch, twice. Recorded in full in `06-03-CHAT-EVIDENCE.md ## CHAT_CANARIES`. The substance of the plan's rule is preserved — the decision text is byte-identical, emitted once, and is still the last thing on the turn.

**[deviation] Two contiguous regions in `claim_intake`, not one.** The plan asks for one. After the escalated branch ignored the single block, the step's *opening call line* was amended as well, because that is the line the model reads when deciding which tools this turn calls. Both regions are individually anchored, individually bounded, individually read back, and the protected slabs are asserted byte-identical — the one-region rule's purpose (T-06-13, no silent reversion of v1→v11 hardening) is met by measurement rather than by count. Placing the text at the end of the step instead was rejected outright: `260812-o5l` proved that adding a paragraph after a step's closing rule is how the v11 silencing was created.

**[deviation] Seven conversations against a budget of five.** Two were spent diagnosing the widget-terminates-the-turn defect (`c2esc`, `c3esc`) — neither was a retry of the same thing, the second ran a genuinely different instruction. One was lost to a transport stall that killed the runner mid-conversation (`c4esc`); its resumed turns re-entered verification and carried no evidence, so it was re-run clean as `c5esc`. The seventh was the approve-path battery re-run, which was **not optional**: `c1appr` had been driven against the pre-reorder instruction, and gating a customer-facing repoint on canaries measured against a superseded build is exactly the shape of mistake this project keeps paying for. No 429 was ever returned and no retry loop was ever run; conversations were paced at 75 s per turn and ≥90 s apart.

**[method] Two proofs taken by `apps.executeTool` rather than by conversation, to stay near budget.** The empty-policy graceful failure (`PDP-100746` → `found: false`, the EMPTY line, five keys, zero claim fields) and the photo-attachment path (`send_case_record_email` → `sent: true`, 200, `[ASSESSOR] [CHAT]`) were proven by invocation on the shipped, byte-unchanged source. The live half of the photo path — capture into session state — is evidenced on `c1appr` (`photo_b64_len: 1530736`, `photo_check: confirmed`, `photo_assessed: true` in the stored record).

**[scope] The plan's Conversation 3 (single-match status inquiry on `PDP100583`) was folded into the multi-match run.** After disambiguation the second `lookup_claim` returns `match_count: 1` and a single-record `status_line`, which is the same assertion against the same code path.

## Known gaps

1. **The photo path's end-to-end run predates the reorder by one revision.** `CLM-24599` carries `photo_assessed: true` and was filed live, but on the instruction where `record_claim` sat after the card. The argument flow is untouched by the reorder — only the call's position in the turn changed, and both shipped runs prove `record_claim` fires with correct arguments on both branches. A photo run on v15 would close it completely and was not affordable within budget.
2. **Nobody has watched v15 in a browser.** Everything here is server-side evidence. The decision card, the cross-sell buttons and the spoken line were all confirmed by eye on v13 and are byte-unchanged.
3. **The customer email remains unmonitored** — unchanged by this plan and untouched by it. `resolve_claim`'s `email_queued: true` is a single-send assertion only, never delivery evidence (`260812-trc`). The pending Resend key rotation and truthfulness fix are uncollided: no Resend code was read, echoed or modified.
4. **`policy_id_display` in chat-written records is unhyphenated** (`PDP100294`) because that is the form the customer typed, whereas 06-02's fixtures carry `PDP-100294`. Both key to the same object and both render correctly in `status_line`; it is a cosmetic difference in the display field, worth knowing before 06-05 compares records across channels.

## Store contents this plan changed

**Added:** `CLM-24599`, `CLM-24536`, `CLM-24841` — all on `PDP100294`, all `channel: CHAT`, all filed by live chat conversations.
**Deleted:** `by-ref/CLM60390.json` and `by-policy/PDP100390.json`, the throwaway objects written by the latency measurement. No test-only debris remains.
**Deliberately retained:** 06-02's fixtures `CLM-60201`, `CLM-60202`, `CLM-60203` and the cross-channel pair `CLM-60210` (chat) / `CLM-60211` (voice). 06-05 reads them.

## Self-Check: PASSED

- `06-03-CHAT-EVIDENCE.md` — created, **304 lines**; `## CHAT_WRITE`, `## CHAT_LOOKUP`, `## CHAT_GRACEFUL_FAILURE`, `## CHAT_CANARIES`, `## CHAT_SHIP` each present exactly once
- `06-03-SUMMARY.md` — created
- Chat `d7bfbb93` re-read after the PATCH → **`129f8b31-f06e-48cc-90fb-15dcf8611db1`**; `channelProfile` sha256 byte-identical across the repoint
- Rollback target **re-read live immediately before the PATCH** → v14 `cdca14e3`, `updateTime` unchanged since plan start
- Voice `d28bbcb0` → `5d9df25c`, `updateTime 2026-08-12T20:51:46.574949Z` — **unchanged**; every voice agent instruction sha unchanged; **zero non-GET calls to `6e01e4a5`**
- Fork `9ae7a0c3` — **zero calls of any method**
- All twelve chat tools byte-unchanged by whole-object sha256, `resolve_claim` included
- App list back to / still **five** — no probe app was created by this plan
- Gated batteries: `c5esc` 34/34, `c6appr` 31/31, `c7status` 24/24, zero-conversation battery ALL PASS; edit batteries 33/33, 45/45, 29/29, 30/30
- `grep -rE 're_[A-Za-z0-9_]{20,}|ya29\.|AKIA|GOOG1[A-Z0-9]{20,}' .planning/` → **no match**
- **No git commit, by instruction** — the tree is deliberately left dirty

- **Method note, recorded because it looked like drift and was not:** the plan-close `channelProfile` check first reported a mismatch. Cause was the assertion, not the platform — the baseline hashed with `separators=(",", ":")` and the closing script did not. Compared under one serialisation, `channelProfile` is byte-identical at plan start, immediately before the PATCH, immediately after it, and live now: sha256 `76f98fed6482c930…`, `channelType: WEB_UI`. **Hash both sides with the same serialiser or the check tests your own formatting.**
