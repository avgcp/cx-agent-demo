---
phase: 05-demo-build-multimodal-backend-reveal-multilingual
plan: 04
subsystem: chat-agent-build
tags: [ces, agent-as-tool, assessor-packet, resend, chat-app]
requires:
  - "05-03 PACKET_RECIPIENT decision (same-mailbox, 2026-08-11)"
  - "chat app a2f621e4 at v9 160dc3b2 baseline"
provides:
  - "app:a2f621e4/agents/case_summary — six-section assessor briefing packet composer"
  - "app:a2f621e4/tools/generate_case_summary — SYNCHRONOUS agentTool wrapper"
  - "app:a2f621e4/tools/send_case_record_email — Resend delivery of a supplied packet body"
affects:
  - "05-05 (wiring, version cut, repoint)"
  - "05-07 (assessor packet on voice)"
tech-stack:
  added: []
  patterns: ["agent-as-tool via agentTool", "scripted key transplant, value never printed"]
key-files:
  created:
    - "app:a2f621e4/agents/case_summary (2d870068-104a-4f22-a962-a7936b251a75)"
    - "app:a2f621e4/tools/generate_case_summary (b7904d0d-2fbb-4c84-a6e2-1605e8725a09)"
    - "app:a2f621e4/tools/send_case_record_email (a08623e6-c3a1-4c64-b345-5fe83cea22dc)"
  modified: []
decisions:
  - "Key sourced from resolve_claim, not send_claim_email — send_claim_email holds no key and makes no HTTP call"
  - "Packet pinned to English in the instruction; no {preferred_language} token exists on the chat app"
metrics:
  conversations_spent: 0
  version_cut: none
  deployment_repointed: none
  completed: 2026-08-11
---

# Phase 5 Plan 04: Build the Assessor Briefing Packet Components Summary

Built the three chat-app components for the backend reveal — a `case_summary` agent that composes the six-section assessor packet on the HUMAN_REVIEW branch only, a `generate_case_summary` agentTool that exposes it synchronously, and a `send_case_record_email` Python tool that delivers a supplied packet body to `akash.vinayak@nerdery.com` under an `[ASSESSOR]` subject prefix — with every pre-existing resource byte-identical, nothing wired, no version cut and zero conversation quota spent.

## What already sends mail on the chat escalated path — and how a double-send was avoided

**`resolve_claim` sends the customer email. `send_claim_email` does not.** The wave-1 voice
baseline recorded this for voice; it is now confirmed independently for the **chat** app, from
historical conversation records only (no new conversations):

| Evidence | Value |
|---|---|
| `resolve_claim` result fields | `email_queued: true`, `email_subject: "Claim CLM-… - please reply with photos"` |
| Present on the **HUMAN_REVIEW** branch | **Yes** — session `wid-45e6`, `decision: HUMAN_REVIEW`, `email_queued: true`, and `send_claim_email` was **never called** in that conversation |
| Present on AUTO_APPROVE | Yes — 14 of 15 conversations |
| `send_claim_email` result `message` | `"Email already sent when the claim was decided. Customer should reply with photos attached."` |
| `send_claim_email` source probe (booleans only) | contains **no** Resend key (`0` regex matches), **no** `ces_requests.post(`, **no** `api.resend.com`, **no** `urllib` — it is a pure reporter, 1,705 chars |
| `resolve_claim` source probe (booleans only) | contains `ces_requests.post(`, `api.resend.com`, `onboarding@resend.dev`, `Authorization`/`Bearer`, and **exactly one** key match |

So the customer email is a side effect of the decision call, on **both** branches. That is the
real shape of the single-send guarantee: it is one send per `resolve_claim` invocation, not one
`send_claim_email` call.

**How the double-send is avoided:**

1. `resolve_claim` was **not modified** — byte-identical before and after (whole-object sha256,
   volatile fields stripped). Its source was never read into the transcript; only booleans and
   counts were emitted, and only inside a script.
2. `send_case_record_email` is a **separate tool** that never composes or sends a customer
   message. It sends one message, to the assessor prefix, from a body handed to it.
3. The new tool makes **exactly one** `ces_requests.post` call per invocation and **never
   retries internally** (asserted: `rsrc.count("ces_requests.post(") == 1`). A failure returns
   `{"sent": false, …}` rather than raising, so a mail failure cannot break the conversation.
4. Its description tells the model in the first two sentences that it is *not* the customer
   confirmation, that the confirmation is sent elsewhere at decision time, and that it must be
   called **at most once per claim, only on the human-review branch**.
5. Nothing was wired. `claim_intake` still carries exactly its original 8 tools; `case_summary`
   is an orphan agent with no parent and no tools. The single-send invariant therefore cannot
   have moved in this plan — there is no new call site yet.

**Carry into 05-05:** the invariant to assert on the live run is *one customer email
(`resolve_claim` fires once) and one assessor email (`send_case_record_email` fires once)*.
Do **not** assert on `send_claim_email` call count as a proxy for customer sends — it is
decorative on this path and was absent entirely from the one HUMAN_REVIEW record on file.

## Components created

| Component | Resource id | Notes |
|---|---|---|
| agent `case_summary` | `2d870068-104a-4f22-a962-a7936b251a75` | instruction 5,273 chars, read back byte-identical; `tools: []` |
| tool `generate_case_summary` | `b7904d0d-2fbb-4c84-a6e2-1605e8725a09` | `executionType: SYNCHRONOUS`; `agentTool.agent` → `apps/a2f621e4-…/agents/2d870068-…`; contains no `9ae7a0c3` |
| tool `send_case_record_email` | `a08623e6-c3a1-4c64-b345-5fe83cea22dc` | Python tool, 2,428 chars, one POST to `https://api.resend.com/emails` |

### `generate_case_summary` schema — 05-05 reads this

`{app}:retrieveToolSchema` reports, and this is **not** obvious from the tool definition:

```json
{"inputSchema": {"type":"OBJECT","properties":{"request":{"type":"STRING"}},"required":["request"]},
 "outputSchema": {"type":"OBJECT","properties":{"response":{"type":"STRING"}}}}
```

An agentTool takes a **single free-text `request` string** and returns a single `response`
string. It does **not** take the claim fields as parameters — `case_summary` reads the eleven
session variables itself by `{token}` interpolation. So `claim_intake` supplies a short
instruction sentence as `request`, and the packet arrives as `response`, which is then what
must be handed verbatim to `send_case_record_email(packet_text=…)`. Saved at
`scratchpad/s04/task2-toolschema.json`.

### `send_case_record_email` schema

`properties` and `required` are both exactly `["claim_ref", "customer_name", "packet_text"]` —
no extra declared parameter. Delivery values, all pinned from `## PACKET_RECIPIENT`:

| Field | Value |
|---|---|
| `from` | `onboarding@resend.dev` (unchanged, untouched) |
| `to` | `akash.vinayak@nerdery.com` |
| `subject` | `[ASSESSOR] {claim_ref} - {customer_name}` |
| `text` | `packet_text`, unmodified |
| returns | `{"sent": bool, "status_code": int, "subject": str}` |

## Version / deployment

**Neither.** No version was cut and the deployment was not repointed, exactly as the plan and
the orchestrator required. `d7bfbb93-8cee-43fe-9095-bc5775f353bd` was re-read after the last
write and still serves `160dc3b2-571c-480f-b901-e4dbe8947f70`. **Zero conversations** were
spent — no `runSession` call was made at any point. Every read of live behaviour came from
conversation records already on the app.

## Regression canaries

| Canary | Baseline | After | |
|---|---|---|---|
| tool count | 8 | 10 (the 8 + 2 new) | ✅ |
| 8 pre-existing tools, whole-object sha256 | — | **all identical** | ✅ |
| `claim_decision_card` `widgetTool` | `0f5a8404649265fa` | identical | ✅ |
| `cover_offer_actions` `widgetTool` | `46e9a7e1b64567f2` | identical | ✅ |
| `send_claim_email` whole object | — | identical | ✅ |
| `resolve_claim` whole object (deterministic tariff pricing, photo confirm path, single-send) | — | identical | ✅ |
| `variableDeclarations` | 37 | 37 | ✅ |
| `claim_intake` instruction | 18,496 chars | 18,496 chars | ✅ |
| `claim_intake` attached tools | 8 | same 8, unchanged | ✅ |
| `rootAgent` / `claims_concierge.childAgents` | `claim_intake` | unchanged | ✅ |
| `languageSettings` (05-03's multilingual config) | `es-US` + `enableMultilingualSupport` | untouched, still present | ✅ |
| deployment `d7bfbb93` | `160dc3b2` | `160dc3b2` | ✅ |

Note the canaries are **structural**, asserted by byte-comparison of the resource objects. The
behavioural canaries (card renders, buttons fire, tariff prices, photo confirm) were not
re-exercised because nothing they depend on changed by a byte and no conversation was spent.

## Deviations from Plan

### 1. [Rule 3 — blocking issue] The key is in `resolve_claim`, not `send_claim_email`

**Found during:** Task 3.
**Issue:** The plan's transplant script was specified to grep `send_claim_email`'s source for
`re_[A-Za-z0-9_]{20,}` and abort unless there was **exactly one** match. There are **zero**.
`send_claim_email` holds no key, imports nothing, and makes no HTTP call — the wave-1 finding
that `resolve_claim` owns the send is the reason, and it holds for chat as well as voice.
Following the plan literally would have failed the task on a false premise.
**Fix:** sourced the key from `resolve_claim` instead, under the same single-process,
never-printed discipline the plan specified — exactly one unique match found, substituted into
the `__RESEND_KEY__` placeholder in memory, tool POSTed, key variable dropped. The transplant
script emitted only `key_found: true` and `key_len: 36`. The keyed source was **never written
to disk**: the on-disk template still carries the placeholder.
**Verification:** `__RESEND_KEY__` absent from the created tool (boolean check in-script);
exactly one key match in the created tool; `grep -rE 're_[A-Za-z0-9_]{20,}' .planning/` returns
**no match**; the only two scratchpad hits are a false positive — an assertion variable name
in the executor's own script whose middle happens to satisfy the pattern, not a key.
`resolve_claim` itself is byte-identical.

### 2. [Rule 1 — wrong assumption] The plan's justification for a separate tool was right for the wrong reason

The plan says *"the chat app already sends the customer confirmation email via
`send_claim_email`, which is part of the single-send guarantee"*. It does not. The conclusion —
build a separate tool, leave the existing one alone — survives intact and is if anything
better supported: the true send site is `resolve_claim`, a far more dangerous tool to touch,
and it was not touched.

### 3. [Rule 3] `preferred_language` does not exist

Task 1 change 5 says to keep the packet English *"regardless of `preferred_language`"*. There
is no `preferred_language` in the chat app's 37 `variableDeclarations`, so interpolating it
would have printed a raw brace. The English-only constraint is stated in prose instead, with
no `{token}`: *"The packet is always written in English, whatever language the customer used or
prefers … Never translate this packet."*

### 4. [documentation] The fork's closing directive was not where the plan said

Task 1 change 4 instructs deleting *"Do not read it out to the customer — it is for the file.
Carry on and close warmly."* from the source agent. That sentence is **not in the fork's
`case_summary` instruction** (5,293 chars, verified) — it must live in the fork's `claim_intake`.
Nothing to delete; the replacement sentence was added regardless, as constraint 9 and in the
Output step: *"The packet is returned as this tool's response and is never spoken to the
customer."* The literal string `for the file` is absent from the adapted instruction.

### 5. [scope] The fork's `case_summary` carried an `end_session` tool

The source agent declares `tools: [.../tools/end_session]`. The new agent was created with
`tools: []` per the plan. A packet composer that can end the session is a live hazard —
an agentTool invocation could terminate the conversation mid-claim.

## Token audit (Task 1 gate)

All eleven `{token}`s interpolated by the adapted instruction are declared session variables on
the chat app — `claim_amount`, `covered_device`, `customer_name`, `decision`, `deductible`,
`dx_answers`, `issue_category`, `photo_status`, `policy_id`, `rules_fired`, `total_loss_flag`.
**Unmatched-token list is empty**, so no renaming was needed and no declaration was added. The
`photo_status` vocabulary matches what the chat app actually writes today (05-02).

Instruction assertions, all passing before the write: each of `SUMMARY`, `ACTION`, `CLAIM`,
`DIAGNOSTIC`, `RULES FIRED`, `FLAGS` appears **exactly once**; `NO_PACKET_REQUIRED` and
`HUMAN_REVIEW` both present; `AUTO_APPROVE` and `for the file` both absent.

## Known Stubs

None. All three components are complete as specified. They are, by design, **unreached** —
nothing calls them until 05-05 wires `claim_intake`. That is the plan's intent, not a stub.

## Threat Flags

None. No new network surface beyond the one `POST https://api.resend.com/emails` already
covered by T-05-01/T-05-02, no new auth path, no schema change at a trust boundary.

### Threat register outcomes

| Threat | Outcome |
|---|---|
| T-05-01 key disclosure | Mitigated — value never printed, never on disk, `.planning/` grep clean. **Carry to 05-05:** the runbook's post-demo key-revocation step now covers a third tool. |
| T-05-02 packet content | Mitigated — token audit proves every field is a declared session variable; recipient fixed. |
| T-05-08 `packet_text` paraphrase | Accepted, unverified — the word-for-word posture is pinned in both the docstring and the tool description. Byte-identity must be asserted on 05-05's live run. |
| T-05-03 resource reversion | Mitigated — `create` only, no `patch` of any existing resource, no `importApp`, all pre-existing objects byte-identical. |
| T-05-05 unverified build served | Mitigated — no version cut, no repoint, `d7bfbb93` re-read at `160dc3b2`. |
| T-05-04 quota | Mitigated — zero conversations. |
| T-05-09 fork spoofing | Mitigated — `agentTool.agent` asserted to contain `apps/a2f621e4-…/agents/` and not `9ae7a0c3`; a write gate in the client refused any non-GET whose URL lacked the chat app id. |

## Notes for 05-05

1. Call shape is `generate_case_summary(request="…")` → `response` string. Hand that string to
   `send_case_record_email(packet_text=…, claim_ref=…, customer_name=…)` unchanged.
2. `case_summary` returns the literal `NO_PACKET_REQUIRED` when `{decision}` is not
   `HUMAN_REVIEW`, so a wrong-branch call is visible in the conversation record. Assert its
   **absence** on an escalated run and its presence if the tool is ever called unconditionally.
3. Assert customer-send count via `resolve_claim`, not `send_claim_email`.
4. Any version cut from the draft will include 05-03's `supportedLanguageCodes: ["es-US"]` and
   `enableMultilingualSupport: true`. Deliberate, left in place.
5. Live delivery of the assessor packet is unblocked and untested — the plan deferred it here
   and `same-mailbox` removed the external dependency, so 05-05 should verify it in the inbox.

## Self-Check: PASSED

- `.planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-04-SUMMARY.md` — created.
- agent `case_summary` `2d870068-…` — read back from the API, instruction byte-identical.
- tool `generate_case_summary` `b7904d0d-…` — read back, schema retrieved.
- tool `send_case_record_email` `a08623e6-…` — read back, schema retrieved, placeholder absent.
- Commits: **none by design** — the orchestrator directed that this plan leave the tree dirty
  and not update STATE.md or ROADMAP.md. All artifacts of record are live on app `a2f621e4`.
