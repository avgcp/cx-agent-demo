---
quick_id: 260809-n1b
type: quick
status: complete
date: 2026-08-09
subsystem: cx-agent-chat-app
tags: [widget, rich-response, order-summary, chat-app, ces, sdk-contract]
server_changes:
  app: a2f621e4-9faf-505a-b804-22471f022366 (Meridian Claim - Chat (hardened))
  new_version: bb14cdcc-d723-4be1-85af-9f4451e22ed5 (chat v7 - decision card reshaped to SDK order_summary contract)
  previous_version: 56a8b22a-2baf-4b18-9540-cdc6185acbee (chat v6)
  deployment_repointed: d7bfbb93-8cee-43fe-9095-bc5775f353bd -> bb14cdcc
  field_mapping_fallback_used: 1 (object-valued FIELD_MAPPING - accepted and working)
  untouched: [6e01e4a5-42a8-5213-b3da-c9053ff8ea52 (voice, v11 b17c9a26), 9ae7a0c3-6511-413c-8cdb-0efe9e90d2b9 (fork)]
files_modified:
  - .planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md
  - .planning/STATE.md
---

# 260809-n1b — Reshape `claim_decision_card` to the SDK `order_summary` contract

The decision card printed raw JSON because **its declared `parameters` — and therefore its
emitted payload — matched none of the fields the deployed web-widget SDK's `order_summary`
renderer reads.** A schema mismatch in the tool's own definition. Not an SDK limitation, not an
agent defect, and not the console-Preview limitation it had been attributed to.

Fixed and deployed as **chat v7 `bb14cdcc-d723-4be1-85af-9f4451e22ed5`**, now served by
`d7bfbb93`. The payload is verified on a live conversation. **Only the on-screen render is
unconfirmed** — that needs a human at `http://localhost:3000`.

## Answers to the specific questions asked

| | |
|---|---|
| **New version ID** | `bb14cdcc-d723-4be1-85af-9f4451e22ed5` — *"chat v7 - decision card reshaped to SDK order_summary contract"* |
| **Was `d7bfbb93` repointed?** | **Yes.** `56a8b22a` (v6) → `bb14cdcc` (v7), confirmed by re-reading the deployment: `appVersion` ends `bb14cdcc`, `updateTime` `2026-08-09T21:55:04Z`, still `WEB_UI` / `CHAT_ONLY` |
| **Which `FIELD_MAPPING` fallback?** | **Fallback 1 — object-valued mapping.** `{"productItem": "card_product_item"}` was accepted by `importApp` and works at runtime. Fallbacks 2 and 3 were never needed. **The model supplies no figure on this card.** |
| **How was `salesTax` resolved?** | **`costBreakdown` was dropped entirely** — the pre-decided contingency. An absent `salesTax` does *not* skip the row; `DF_MEz(undefined)` returns a formatted zero, so it prints a hardcoded `Sales tax $0.00`. The label is a string literal with no override. The excess arithmetic moved into `productItem.subtitle`. |
| **Widget-tool survival check** | `claim_decision_card` ✅ and `cover_offer_actions` ✅ present in the fresh export **before** editing, in the live tool list **after** `importApp`, and in the **v7 version snapshot**. All 8 tools intact; `variableDeclarations` unchanged at 37. |

### The exact payload emitted on the live run

Captured from the `runSession` output, session `cd73b68d-34d4-4d3e-8914-fb9f1e27324e`,
policy PDP100294, keyboard fault:

```json
{
  "productItem": {
    "title": "Apple MacBook Pro 16\"",
    "subtitle": "Keyboard replacement · CLM-24806 · $420 less $25 excess = $395 to you · Approved on the spot (on-the-spot limit $1,500)",
    "price": { "units": 420, "nanos": 0, "currencyCode": "USD" },
    "imageUri": "https://www.gstatic.com/psa/static/1.gif"
  },
  "type": "order_summary"
}
```

All assertions pass: `type` is the platform's own tag; no old flat key survives; `units` arrived
as a Python `int` (not a string); `CLM-`, the decision and the `$1,500` cutoff are all present;
`420 - 25 = 395`.

**Determinism proved, not assumed:** the conversation record shows the emitted `productItem` is
**byte-identical** to `resolve_claim`'s `card_product_item`, including the `·` separators.
The tariff, not the model, produced every figure.

## What was wrong, precisely

The tool declared seven flat claim fields. The SDK does this:

```js
case "order_summary":
  c = new DF_Mzz(...); c.productItem=b.productItem; c.costBreakdown=b.costBreakdown;
  c.paymentMethod=b.paymentMethod; c.actions=b.actions;
```

Four keys copied, everything else discarded silently. All four were `undefined`, so nothing
usable was drawn and the raw payload showed instead.

**The lesson worth keeping:** the payload was *well-formed* — the platform tagged it
`"type": "order_summary"` — and that was previously read as "the payload is correct".
Well-formed is not the same as matching the renderer's contract. That inference is what kept
the real cause hidden.

## The contract, and the four contingencies

Read from the deployed bundle `chat-messenger.js` v1.16 (1.36 MB, disassembled), not inferred.
**The full reusable contract is written up in `05-02-SUMMARY.md`**; the working notes with
quoted source fragments for all eight questions are in `<scratchpad>/sdk-contract.md`.

| Contingency | What the source actually says | Branch taken |
|---|---|---|
| `salesTax` absent renders cleanly → omit it | **No.** Rendered unconditionally; absent ⇒ hardcoded `Sales tax $0.00`. Labels (`Subtotal`/`Sales tax`/`Discount`/`Shipping`/`Total`) are string literals with no override. | **Dropped `costBreakdown`.** Plan's standing rule: *do not ship a zero sales-tax row.* |
| `discount` label wrong for an excess | Hardcoded `Discount`, amount prefixed `-` | Moot — no `costBreakdown` |
| `actions` labels interactive | **Confirmed.** Click ⇒ `presenter.sendQuery(label)` — the label is echoed into the transcript as the user's own message and sent to the agent. | **Dropped `actions`.** `decision` and `auto_approval_cutoff` moved to the subtitle — **not** dropped. |
| Absent section draws an empty box | **No.** All four empty-states are `""`. | **Dropped `paymentMethod`.** No fabricated card brand. |

## Deviations

**[Rule 3 — blocking] `imageUri` is mandatory; omitting it destroys the card.** The plan's
contingency assumed an absent `imageUri` would draw a broken-image icon and could be left out.
In fact `DF_MXp` calls `a.startsWith(...)` **unguarded on its first line**, and `DF_MAz` calls it
whenever `productItem` exists — so `imageUri: undefined` throws a `TypeError` and `render()`
fails entirely, which is *worse* than the JSON blob being replaced. Resolved by emitting
`https://www.gstatic.com/psa/static/1.gif` — verified live (HTTP 200, `image/gif`, 53 bytes,
`GIF89a`, 1×1 with the transparency flag). The sanitiser **hard-trusts the
`https://www.gstatic.com` prefix**, so it needs no `url-allowlist` on the embed (the deployment
has none configured, so any other URL would have been rewritten to `about:invalid#zClosurez`).
It draws as a neutral empty 64×64 tile.

**[Rule 1 — bug] `units` must be a JSON number; a string silently 10×'s the figure.** The plan's
contingency assumed `Number(x.units)` and that "a JSON number or a proto3 int64 string works
either way". There is **no `Number()` call**: `var b=a.units+a.nanos/1E9`. With `units` as the
string `"420"`, JS concatenates — `"420"+0` = `"4200"` → **`$4,200.00`**. Absent `nanos` gives
`$NaN`; absent `currencyCode` makes `Intl` throw. The schema now declares `units`/`nanos` as
`NUMBER` and all three as required, with the reason in each field description.

**[Rule 3 — blocking] `importApp` is on the collection, not the app resource.** The plan's
`${APP}:importApp` returns a **404 HTML page**. The correct path is
`.../locations/us/apps:importApp` with `appId` in the body (per the discovery document).
`exportApp` *is* on the app resource and additionally requires `{"exportFormat":"JSON"}` —
omitting it fails with `Unsupported export format: EXPORT_FORMAT_UNSPECIFIED`.

**[Rule 2 — missing correctness] The card must not undo the photo-contradiction hardening.**
`resolve_claim` deliberately withholds a price when the photo could not be verified (*"naming a
figure here reads as though the claim has been valued when it plainly has not"*). A card that
printed `$420 less $25 excess = $395 to you` in that branch would have undone it. The subtitle
is therefore branch-aware, and in the `photo_blocked` case reads *"$840 indicative cost, not yet
valued · Passed to a specialist"* with no payout figure. Proven offline.

**Wording adjustment (cosmetic).** `$420 repair less $25 excess` reads wrong for a total-loss
*replacement*, so the noun was dropped: `$420 less $25 excess = $395 to you`.

**Task 3's automated check asserted `actions` in the payload.** It conflicts with the plan's own
contingency 4, which pre-authorised omitting `actions`. The contingency was followed — shipping
buttons that inject *"Approved on the spot"* into a live demo as a user turn is a real failure
mode — and the assertion was adapted to assert `actions` is **absent**. The must-have that
matters (*decision and cutoff both visible, neither dropped*) is met via the subtitle.

## Quota

The live run hit **HTTP 429 `RESOURCE_EXHAUSTED`** on the decisive third turn. Per instruction I
did not retry-loop; I made **one** spaced attempt (~95 s) to finish the *same* conversation,
since the limit is per-minute, and it succeeded. Total: **one conversation, four API turns**
(three distinct user turns plus the one repeat of the quota-failed turn). `RunSession LLM tokens`
still wants raising in IAM → Quotas before any demo.

## New defect found, NOT fixed

**The decision turn draws the card but the agent says nothing.** The conversation record shows
the agent put its sentence — *"Good news, I can approve that right now."* — into the widget
tool's **`summary` parameter** rather than emitting text.

Root cause candidate, and a genuinely misleading API: per the CES discovery document,
`textResponseConfig.type: NONE` means **"the LLM dynamically decides whether to generate a text
response"** — *not* "no text". So the model was free to stay silent, and did.

This is **pre-existing** (that config is unchanged since the widget was created) and it
contradicts `claim_intake`'s own *"Never send the card without a sentence."* That instruction
block is itself self-contradictory — lines 159-164 say read the `explanation` **WORD FOR WORD**,
lines 170-173 say do **not** repeat the figures and say only the short human part — which is the
likelier root cause.

Not fixed here: it is outside "payload shape only", and the remaining quota could not verify a
behaviour change. **Suggested fix:** `textResponseConfig: {type: "LLM_GENERATED",
textResponseInstruction: ...}` *and* resolve the instruction contradiction, then test live.

## `cover_offer_actions` — flagged, contract captured, NOT fixed

It declares `{prompt, options[{label,value}]}` with **no `dataMapping`**. The SDK reads
`{actions: [{content, description?, utterance?}]}` — `prompt` and `options` are read by nothing.
**Worse failure mode than the decision card's:** `b.actions.map(...)` **throws** on `undefined`
rather than degrading to a JSON blob. Still has never fired. Full contract recorded in
`05-02-SUMMARY.md`; left for a follow-up.

## Verification evidence

| Check | Result |
|---|---|
| Fresh `exportApp` before editing | Yes — new zip `chat-fresh-n1b.zip`; the cached `chat-fresh.zip` was not reused |
| Widget tools in the fresh export, before editing | `claim_decision_card` ✅ `cover_offer_actions` ✅ |
| Dry-run `importApp` (`validateOnly`, `REPLACE`) | Clean, no warnings, correct app resource returned |
| Widget tools server-side after the push | Both ✅ (8 tools, unchanged set) |
| `retrieveToolSchema` after | Nested `productItem` with `price{units,nanos,currencyCode}`; no `issue_label`/`auto_approval_cutoff`/`claim_amount`/`deductible` |
| `dataMapping` after | `{"productItem": "card_product_item"}`, `mode: FIELD_MAPPING`, same `sourceToolName` |
| `widgetType` / `textResponseConfig` | `ORDER_SUMMARY` / `{type: NONE}` — both preserved |
| Session variables | 37 before, 37 after |
| v7 snapshot audit | Both widget tools present; card declares only `productItem`; no flat params |
| Deployment `d7bfbb93` re-read | Serving `bb14cdcc` |
| Offline suite (`cardtest.py`) | ALL PASS — auto-approve, total loss, photo contradiction; every pre-existing return key intact; exactly one email per claim |
| Live payload assertions | ALL PASS |
| Voice app `6e01e4a5` / fork `9ae7a0c3` | **No API call of any kind issued.** Every mutating URL names `a2f621e4` literally. |

**No regression:** decision logic, thresholds, the tariff, the rules, the email block and every
v1→v11 hardening item are untouched. The `resolve_claim` diff is **+45 lines, −1** (the one
removal is the old final line, re-emitted with the new key appended).

**Secret handling:** the live `RESEND_API_KEY` in `resolve_claim` was never echoed, never copied
into `.planning/`, and appears in no summary or commit. The file was edited by script, never
printed.

## What the user should see at http://localhost:3000

The rig is already running — do not rebuild it. Run policy **PDP100294**, *Jordan Rivera*, an
**Apple MacBook Pro 16"**, **keyboard** fault, "works normally otherwise". No photo needed.

**Expected — a drawn card, no JSON:**

1. A small **empty grey 64×64 tile** on the left. Expected, not a bug — there is no product
   image; it is a 1×1 transparent GIF, required because omitting it stops the card rendering.
2. Title **`Apple MacBook Pro 16"`**, price **`$420.00`** on the same row, right-aligned.
3. Subtitle: **`Keyboard replacement · CLM-…· $420 less $25 excess = $395 to you · Approved on
   the spot (on-the-spot limit $1,500)`**.
4. **Two horizontal divider rules below the row, with nothing after them.** Expected — the SDK's
   card wrapper emits them unconditionally and we deliberately send no cost breakdown, payment
   method or action buttons.
5. **No "Sales tax" line, no "Shipping" line, no "Discount" line, no payment-method row, and no
   buttons on the card.** All deliberate.
6. **Known and expected: the agent will probably say nothing at all in this turn** — see the open
   defect above. If it *does* say something, note it; that is new information.

Anything else — a stray label, a visible empty box, a broken-image glyph instead of a plain
tile, or `$4,200.00` / `$NaN` / `$0.00` instead of `$420.00` — is worth reporting.

## Artifacts

Session scratchpad:

- `sdk-contract.md` — the eight questions answered with quoted source, plus "Mapping as built"
- `chat-messenger-v1.16.js` — the disassembled bundle
- `payload-live.json` — the payload captured from the live run
- `conv.json` — the conversation record (proves determinism and the `summary` behaviour)
- `chat-fresh-n1b.zip` / `pkg-n1b/` — the fresh export and the edited source pushed
- `cardtest.py` — the offline suite (ALL PASS)
- `schema-before.json` / `schema-after.json`, `tools-before-n1b.json` / `tools-after-n1b.json`,
  `ver7.json`, `dep-before-n1b.json` / `dep-after.json` — evidence for every assertion above

## Self-Check: PASSED

- `.planning/quick/260809-n1b-…/260809-n1b-SUMMARY.md` — created
- `05-02-SUMMARY.md` and `STATE.md` modified and dirty in `git status` (uncommitted by
  instruction; the orchestrator owns the docs commit)
- Every server assertion re-read from the API after the write, never inferred from the request
- No git commit made by this task; ROADMAP.md untouched
