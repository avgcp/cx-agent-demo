---
phase: 05-demo-build-multimodal-backend-reveal-multilingual
plan: 02
status: incomplete
date: 2026-08-06
covers_success_criteria: []
---

# 05-02 — Decision card + quick actions (IN PROGRESS)

> ⚠ **ANY EXPORTED PACKAGE ON DISK IS STALE.** Quick task `260810-ifr` (2026-08-10) changed
> the chat app `a2f621e4` by **direct `apps.tools.patch` / `apps.agents.patch`**, not by
> export→edit→import. Every zip in any scratchpad predates those edits. **A future
> export→edit→import task MUST take a fresh `exportApp` first, or it will silently revert the
> `cover_offer_actions` schema and the `claim_intake` cross-sell instruction.** Current live
> version: chat v8 `3f85b1d8-4810-44eb-85e6-39adc42593c9`.

## Built

Two `widgetTool`s on the chat app `a2f621e4`, deployed as chat v3 `657ecb18`:

| Tool | Type | Wiring |
|---|---|---|
| `claim_decision_card` | `ORDER_SUMMARY` | `FIELD_MAPPING` from `resolve_claim`, `textResponseConfig: NONE` |
| `cover_offer_actions` | `QUICK_ACTIONS` | Model-supplied prompt + Add it / Not now |

## Discovery notes (no docs, no examples)

`uiConfig` is an undocumented bare object in the API schema and no widget tool existed
anywhere in the project to copy. `retrieveToolSchema` (v1 and v1beta, POST to
`{app}:retrieveToolSchema` with a tool resource name) was the way in: a widget returns
`{summary, widget_tool_status, payload}`, and the `payload` shape is whatever is declared in
the tool's own `parameters`. Reusable for any future widget.

Also learned the hard way: widget tools created directly via the API exist **only**
server-side. The local source had to be refreshed via `exportApp` before the next
`importApp`, which would otherwise have silently deleted them.

## Confirmed working

- `claim_decision_card` is invoked and receives the correct payload, verified on a live run
  (`CLM-24689`, keyboard replacement, $420, $25 excess, cutoff $1,500).
- ~~The platform tags it `"type": "order_summary"`, so the widget is correctly formed.~~
  **Corrected 2026-08-09.** The tag proves the payload is *well-formed*, which is not the same
  as matching the renderer's contract — and it did not. See *"Rendering — ANSWERED"* above.
  This inference is what kept the real cause hidden.

## Rendering — ANSWERED 2026-08-09 (quick task `260809-n1b`, chat v7 `bb14cdcc`)

The card did not draw because **its declared `parameters` — and therefore its emitted payload —
matched none of the fields the deployed web-widget SDK's `order_summary` renderer reads.**

This was **not** an SDK limitation, **not** an agent defect, and **not** the Preview-panel
limitation it was originally attributed to below. It was a schema mismatch in the tool's own
definition, found by disassembling the deployed bundle
`https://www.gstatic.com/chat-messenger/sdk/prod/v1.16/chat-messenger.js`.

The original diagnosis in this section was wrong in an instructive way: the payload was
**well-formed** — the platform tagged it `"type": "order_summary"` — and that was read as
"the payload is correct". *Well-formed is not the same as matching the renderer's contract.*
The SDK copied four keys off the payload, found all four undefined, and drew nothing, so the
raw payload is what appeared on screen.

- ~~**`cover_offer_actions` has still never fired**, and now also looks defective.~~
  **Fixed and fired 2026-08-10** — see *"`cover_offer_actions` — FIXED AND FIRED"* below.

---

## The SDK `order_summary` contract (reusable — read from source, not inferred)

Everything here is read from `chat-messenger.js` v1.16. **Reuse this for every future widget.**

### How a widget payload reaches the renderer

```js
case "order_summary":
  c = new DF_Mzz(a.utterance.utteranceId, b.id);
  c.productItem   = b.productItem;
  c.costBreakdown = b.costBreakdown;
  c.paymentMethod = b.paymentMethod;
  c.actions       = b.actions;
```

`b` is the widget payload. **Only these four keys are read; everything else is discarded
silently.** A widget tool's declared `parameters` *are* its emitted payload, so getting a card
to draw is a schema-authoring job, not an agent-prompting job.

| Section | Sub-fields read |
|---|---|
| `productItem` | `title`, `subtitle`, `price`, `imageUri` |
| `costBreakdown` | `subtotal`, `salesTax`, `discount`, `shipping`, `total` |
| `paymentMethod` | `brand`, `lastFour` |
| `actions` | `primaryLabel`, `secondaryLabel` |

The platform adds `"type": "order_summary"` itself — never declare a `type` parameter.

### Money is `google.type.Money`, and `units` must be a NUMBER

```js
function DF_MEz(a){if(!a)return (...USD...).format(0);
var b=a.units+a.nanos/1E9;
return (new Intl.NumberFormat("en-US",{style:"currency",currency:a.currencyCode,...})).format(b)}
```

There is **no `Number()` coercion**. `a.units` goes straight into a `+` expression:

| Emitted | Renders |
|---|---|
| `{units: 420, nanos: 0, currencyCode:"USD"}` | `$420.00` ✅ |
| `units` as a proto3 int64 **string** `"420"` | `"420"+0` = `"4200"` → **`$4,200.00`**, a silent 10× error |
| `nanos` omitted | `420 + undefined/1e9` = NaN → **`$NaN`** |
| `currencyCode` omitted | `Intl` **throws** |
| whole Money absent | `$0.00` (hardcoded USD) |

**Always emit `units` as a JSON number, with `nanos: 0` and `currencyCode` present.**

### Zero-suppression, and the `salesTax` trap

`discount` and `shipping` are suppressed when `units===0 && nanos===0`. **`subtotal`,
`salesTax` and `total` are rendered unconditionally** — and an absent `salesTax` does not skip
the row, it prints `$0.00`, because `DF_MEz(undefined)` returns a formatted zero.

All row labels are **hardcoded en-US string literals** — `Subtotal`, `Sales tax`, `Discount`
(with a literal `-` prefix on the amount), `Shipping`, `Total`. There is no way to relabel a
row. **Consequence: you cannot show a cost breakdown on an insurance claim without also
showing a "Sales tax" line.**

### Absent sections draw nothing; two fields are load-bearing

All four empty-state templates are the empty string, so an omitted section draws nothing.
**But the card wrapper emits two `<div class="divider">` rules unconditionally**, so a card
using only `productItem` shows the product row followed by two horizontal rules.

Two fields are accessed **unguarded** and will throw, destroying the whole card:

- `imageUri` — `DF_MXp` calls `a.startsWith(...)` on its first line. **`imageUri` is mandatory
  whenever `productItem` is present.** The sanitiser hard-trusts the `https://www.gstatic.com`
  prefix, so `https://www.gstatic.com/psa/static/1.gif` (a real 1×1 transparent GIF) is a
  demo-safe asset needing no `url-allowlist` on the embed. Any other URL, with no allowlist
  configured, is rewritten to `about:invalid#zClosurez` and fails to load.
- `quick_actions`' `actions` — `b.actions.map(...)`, same class of failure.

### `actions` labels are buttons wired to `sendQuery`

```js
else { a.clicked=!0; a.v.renderCustomText(b,!1); a.v.presenter.sendQuery(b) }
```

Any label other than the magic string `"Place order"` is, on click, **echoed into the
transcript as the user's own message and sent to the agent as a query**. Never use an `actions`
label decoratively — a presenter tapping one would inject it into the conversation.

### `textResponseConfig: NONE` does not mean "no text"

From the CES discovery document, `WidgetToolTextResponseConfig.type`:

| Value | Actual meaning |
|---|---|
| `NONE` | **"The LLM dynamically decides whether to generate a text response alongside the widget."** |
| `LLM_GENERATED` | "The LLM is explicitly required to generate a text response." |
| `STATIC` | A fixed `staticText` is always used. |

The name is misleading and it matters — see the open defect below.

---

## Rendering — CONFIRMED BY EYE 2026-08-09 (quick task 260809-nt7)

The user drove a full chat claim through the real, console-generated chat-messenger SDK
**v1.16** widget embed, served locally at `http://localhost:3000` with a token broker,
against deployment `d7bfbb93` serving chat v7 `bb14cdcc`.

**The card drew.** Title `Apple MacBook Pro 16"` with right-aligned price **`$840.00`** on
the same row. Subtitle: `Screen replacement · CLM-24442 · $840 less $25 excess = $815 to
you · Approved on the spot (on-the-spot limit $1,500)`.

The two predicted cosmetic artifacts appeared: a small empty grey image tile at the left,
and two divider rules below the row with nothing after them — both expected consequences of
deliberately dropping `costBreakdown` and `actions`, not defects.

**Load-bearing part: none of the five failure modes the SDK contract predicted occurred.**
No `$4,200.00` from a units-as-string, no `$NaN` from an absent `nanos`, no hardcoded `Sales
tax $0.00` row, no broken-image glyph in place of the 1×1 GIF, and no raw JSON. This makes
the `order_summary` contract documented above **verified by observation**, not only read
from source, and therefore safe to reuse for future widgets.

This run also closed two things beyond rendering, neither of which this plan had ever had
evidence for: **photo upload through the real browser widget** — it needed no Cloud Storage
configuration, `enable-file-upload` alone sufficed — and **the end-to-end chat claim through
the real embed**. The vision read was accurate on a matching device/policy pair: *"I can see
the cracks on the screen there."*

---

## What was built instead (chat v7 `bb14cdcc-d723-4be1-85af-9f4451e22ed5`)

Given the contract, `costBreakdown`, `paymentMethod` and `actions` were all **deliberately
omitted**, and the card is a single `productItem` row:

```json
{"productItem": {
  "title": "Apple MacBook Pro 16\"",
  "subtitle": "Keyboard replacement · CLM-24806 · $420 less $25 excess = $395 to you · Approved on the spot (on-the-spot limit $1,500)",
  "price": {"units": 420, "nanos": 0, "currencyCode": "USD"},
  "imageUri": "https://www.gstatic.com/psa/static/1.gif"},
 "type": "order_summary"}
```

- **`costBreakdown` dropped** because it would have printed `Sales tax $0.00` on an insurance
  claim, and the label is unchangeable. The arithmetic moved into the subtitle instead.
- **`actions` dropped** because its labels are live buttons. `decision` and
  `auto_approval_cutoff` — the demo's headline beat — were **not** dropped; they moved into the
  subtitle, which is inert.
- Every figure is computed in Python inside `resolve_claim` from the tariff values it already
  had. Verified on the live run: the emitted `productItem` is **byte-identical** to
  `resolve_claim`'s `card_product_item`, and `units` arrived as an `int`. The model supplies no
  number on this card.

## `cover_offer_actions` — FIXED AND FIRED 2026-08-10 (quick task `260810-ifr`, chat v8 `3f85b1d8`)

**It fired for the first time ever on 2026-08-10.** Zero of the 14 conversations on record
before that date contained a `cover_offer_actions` call.

### Why it had never fired — and it was *not* a wiring defect

All three prerequisites were already in place: the tool **was** attached to `claim_intake`
(the agent every conversation reaches), `uninsured_device` **was** a declared session variable
written by `verify_identity` before the claim is even priced, and `claim_intake`'s instruction
**did** contain an explicit `<step name="Cross-sell">` naming the tool.

**It had never fired because no conversation had ever taken a turn past the email
confirmation on the auto-approve path.** The instruction forbids bundling the offer onto the
email turn, so the cross-sell *structurally requires one more customer turn after the email*.
Exactly one conversation ever got close — the user's live 2026-08-09 run — and it ended on the
email turn (last customer message *"ok great"*, conversation closed two minutes later). Every
other run stopped earlier: simulator scripts, an API run that hit quota, or the escalation
path, which deliberately has no cross-sell.

**Lesson worth keeping:** "the tool has never fired" was read for a week as evidence of a
wiring defect. It was evidence of an unexercised path. The payload defect below was real but
entirely *latent* — it would have thrown the instant it fired, and fixing it alone would have
changed nothing observable.

### The SDK `quick_actions` contract (reusable — read from source, verified)

Read from the deployed bundle `chat-messenger.js` v1.16. Builder, dispatched from
`case "quick_actions":`:

```js
function DF_MQA(a, b) {
  a = new DF_MKs(a.utterance.utteranceId, b.id);
  a.quickActions = b.actions.map(function (c) {
    return { content: c.content, description: c.description, utterance: c.utterance };
  });
  return a;
}
```

**Payload contract: `{ actions: [ { content, description?, utterance? } ] }`**

| Field | Verified behaviour | Shipped decision |
|---|---|---|
| `actions` | `b.actions.map(...)` is **unguarded**. Absent ⇒ `TypeError` ⇒ `render()` never runs ⇒ **nothing appears at all**. A *harder* failure than the decision card's, which degraded to a JSON blob. | **Required** in the schema, with the reason in its description. |
| `content` | Button label. Interpolated as a Lit template value — no unguarded deref. | **Required.** `"Add it"` / `"Not now"`. |
| `utterance` | `e = d.utterance \|\| d.content`, then `renderCustomText(e,false)` **and** `presenter.sendQuery(e)` — echoed into the transcript **as the customer's own message** and sent to the agent as a query. | **Required, and must read like a human sentence.** A bare `"accept"` would appear as the customer typing `accept`. |
| `description` | Guarded, so omitting is safe — **but** `.some(d => d.description)` flips the *whole set* to `grid-layout columns-1` plus a left-align style injected into each button's shadow root. | **Not even declared** — declaring it invites the model to fill it and silently change the layout. |
| `id` | Consumed as `b.id` by the platform, not by us. | Not declared. |
| `prompt`, `options` (the old schema) | **Read by nothing.** There is no title/prompt slot anywhere in the builder or the renderer. | **Deleted.** |

The widget self-dismisses on the first user input (`connectedCallback` registers a `once`
listener on `chat-messenger-user-input-entered`).

### The three instruction defects — a correct payload alone would still have been broken

1. **`end_session` was instructed in the same turn as the widget** (*"Accept a decline
   gracefully. Then {@TOOL: end_session}"*). Obeyed literally, the session ends before the
   customer can tap: **the buttons would have been dead even with a perfect payload.**
2. **"the buttons ARE the question" was provably false.** The old step said *"Do not ask the
   question in plain text as well - the buttons ARE the question"* and put the sentence in the
   `prompt` parameter, which renders nowhere. Following it produced two naked buttons with no
   question above them. Inverted: the agent now says the offer aloud, and the step states
   plainly that the buttons cannot ask on its behalf.
3. **`textResponseConfig: {type: NONE}`** means *"the LLM dynamically decides whether to
   generate a text response"* — not "no text" — which left the offer sentence free to vanish.
   Set to `LLM_GENERATED` **on this widget only**; the decision card's `NONE` was left alone
   because that widget was visually verified on 2026-08-09 and is the known-good baseline.

The fix added a second step, `<step name="Close after the offer">`, which owns `end_session`,
so the close happens only after the customer has answered.

### What `LLM_GENERATED` actually does to the tool's input schema

Confirmed via `retrieveToolSchema`: setting `LLM_GENERATED` adds a **required `textResponse`
string parameter** to the tool's input schema, whose description is the
`textResponseInstruction` verbatim. That is the enforcement mechanism — the model cannot call
the tool without composing a sentence. `textResponseInstruction` is a real field
(`WidgetToolTextResponseConfig {type, textResponseInstruction, staticText}`).

### The payload emitted on the live run

Session `f9c3999f-2c0d-4908-8a1d-c22971547bb4`, chat v8, PDP100294, keyboard fault:

```json
{"type": "quick_actions",
 "actions": [
   {"content": "Add it",  "utterance": "Yes please, add it to my cover."},
   {"content": "Not now", "utterance": "Not right now, thanks."}]}
```

Agent text in the **same turn**, verbatim:

> *"One more thing while I have you: your Apple iPhone 16 Pro Max isn't on this policy. Would
> you like me to add it to your cover?"*

`end_session` was **not** called in that turn — it fired one turn later, after the answer.
Sending `"Yes please, add it to my cover."` (exactly what tapping **Add it** sends) produced
*"I'll have someone send the options over. Thanks for chatting, Jordan - have a good day."*
plus `end_session`. No price was quoted at any point.

### Open: the offer sentence is emitted twice in the record

The cross-sell turn contains **two identical text chunks** — the model's own output message
(alongside the `toolCall`) and the delivered widget message (sourced from the required
`textResponse` parameter). The decision-card turn, on `NONE`, emits only one. Whether the
browser renders the sentence once or twice is **not determinable from the record** and is the
first thing to check on screen. If it does double up, the fix is to stop instructing the agent
to say the offer as its own message and let the widget's `textResponse` be the only source —
`LLM_GENERATED` makes that parameter mandatory, so the question cannot go missing.

## Open defect: the decision turn draws the card but says nothing

On the live chat-v7 run the agent produced **no text at all** alongside the card. The
conversation record shows why: it put its sentence — *"Good news, I can approve that right
now."* — into the widget tool's **`summary` parameter**, and with `textResponseConfig: NONE`
the LLM then chose not to emit a separate text response.

This contradicts `claim_intake`'s own instruction, which says *"Never send the card without a
sentence."* It is **pre-existing** — `textResponseConfig: NONE` is unchanged since the widget
was created — and it is not caused by the payload reshape.

That instruction block is also self-contradictory and is the likely root cause: lines 159-164
say read the `explanation` **WORD FOR WORD**, while lines 170-173 say do **not** repeat the
figures and say only the short human part.

**Suggested follow-up (not applied, needs a live test):** set
`textResponseConfig: {type: "LLM_GENERATED", textResponseInstruction: "..."}` to *require* a
sentence, and resolve the instruction contradiction at the same time.

## Defect found during testing — TWO FIXES ATTEMPTED, BOTH FAILED

**The wrong-subject guard does not work.** A photo of a **laptop** was accepted and
auto-approved against **PDP100583** (Maria Santos, iPhone 16 Pro Max) — payload shows
`device: Apple iPhone 16 Pro Max`, `auto_approval_cutoff: 750`, `claim_amount: 420`, i.e.
28% of the iPhone's $1,500 value.

`assess_screen_crack` has a `WRONG_SUBJECT` branch for exactly this, but its
`image_shows_device` parameter asks the **model** to decide whether the photo matches the
policy. The model answered "yes" while looking at a laptop on a phone policy.

This is the same failure mode designed out everywhere else in this build during v1→v11: a
judgment call left with the model rather than enforced in code. It was inherited when the
tool was ported from `9ae7a0c3` and its interface was not re-examined.

**Fix applied** *(chat v4 — verified offline only; it failed live and no longer exists in the
build, see Resolution below)*: replaced `image_shows_device` (a comparison) with `device_in_photo`
(an observation — `laptop` | `phone` | `other` | `unclear`) compared against `device_category`
from the policy record inside the tool. Verified: laptop-on-phone and phone-on-laptop both
rejected; matching device still approves at $840 / $420; wrong object, unclear, contradiction
and bad-enum cases all still behave. Five stale assertions in phototest.py were migrated —
they passed the old yes/no values and would have started silently returning INVALID_VALUE. The model reports what it sees;
the tool decides whether it matches. It demonstrably describes images well — it produced
"several cracks branching across the display" unprompted — but is unreliable at holding two
facts in mind and comparing them.

## Verified separately

The vision read itself is good, on two real photos: *"I can see several cracks across the
screen"* and *"I can see several cracks branching across the display"*.


---

# Wrong-subject guard: removed by decision (resolved 2026-08-06, chat v6 `56a8b22a`)

*Sections below record the investigation as it stood at chat v5 `5ccd34a4`. The
**Resolution** section at the end records the decision taken and what the build does now.*

## The finding

**The photo assessment cannot verify *which device* is in the image, and this is not
fixable by prompting.** Two structural attempts both failed against live testing.

| Version | Approach | Outcome |
|---|---|---|
| v4 `7710088b` | Model reports the device category (`device_in_photo`), tool compares against `device_category` in code | Model reported `"phone"` while looking at a laptop |
| v5 `5ccd34a4` | Model reports **physical features** (`keyboard_visible`, `hinged_lid`) plus a named description; tool infers the device and compares | Model reported `keyboard_visible: "no"`, `hinged_lid: "no"`, and *"a black phone with several cracks on the screen"* |

## The evidence

`evidence-laptop-photo.png` in this directory is the exact image uploaded in
`simulator-da97c67a` against **PDP100746 (Apple iPhone 16)**. It is an unmistakable MacBook:
a full keyboard with individual keys fills the lower half of the frame, with a trackpad and
a hinged aluminium lid. It was approved at $280 as an iPhone screen repair.

Retrieved by decoding the base64 PNG stored inline in the conversation record
(`turns[n].messages[].chunks[].image.data`) — worth knowing, as it makes any past image
claim auditable after the fact.

## Root cause

The model already knows what the policy covers by the time it sees the photo, and its
report conforms to that rather than to the image. This is not a labelling problem: it
answered three independent, concrete, physical questions falsely against overwhelming
visual evidence.

Note also that the vision capability itself is **good** — the same model produced
"several cracks branching across the display" accurately on two occasions. It reports the
damage faithfully and the identity unfaithfully, because only the identity conflicts with
the surrounding context.

## Why the offline tests did not catch either attempt

Both suites supplied the observation values directly, which proved the comparison logic and
never the observation. A test that feeds the tool `device_in_photo="laptop"` cannot detect
that the model would have said `"phone"`. The suites now include the misleading inputs the
model actually produced, but that is a regression guard, not a fix.

## Options

1. **Isolate the vision call** in an agent-as-tool with no policy knowledge — it receives
   only the image and answers what it shows. Removing the context is the only structural
   fix. Unknown: whether CES gives such a sub-agent a clean context or passes conversation
   history through. If it inherits history, this fails identically. The pattern exists
   (`generate_case_summary` in `9ae7a0c3` is an agent-as-tool).
2. **Accept the limitation.** Remove the wrong-subject guard, document that photo
   verification confirms *damage* but not *device identity*, and keep demos on matching
   device/policy pairs. **A guard that does not work is worse than none** — it reads as
   protection in both the code and the runbook.

The contradiction path (crack reported, none visible) is **unaffected** and still works: it
only asks about damage, which the model reports honestly.

## Resolution — option 2, accept the limitation (user decision, 2026-08-06)

Executed as quick task `260806-u21`.

**The decision, in the user's framing:** *a guard that does not work is worse than none,
because it reads as protection in both the code and the runbook.* Option 1
(agent-as-tool vision isolation) was explicitly **not** attempted — it rests on an unverified
assumption about whether CES gives a sub-agent a clean context, and a third failed structural
attempt would buy nothing the documentation cannot state honestly today.

**What was removed** from `assess_screen_crack` (chat v6 `56a8b22a-2baf-4b18-9540-cdc6185acbee`):

| Removed | Note |
|---|---|
| `keyboard_visible`, `hinged_lid` parameters | Signature is now exactly `(crack_visible, what_you_see)` — confirmed server-side via `retrieveToolSchema` |
| The device-inference block | Laptop/phone signal counting and keyword matching, and the comparison against `device_category` |
| The `WRONG_SUBJECT` return and `photo_check="rejected"` | `"rejected"` is now unreachable; it is retained defensively in `resolve_claim`'s `photo_blocked` set with a comment saying so |
| The `photo_device_seen` session variable | Declaration removed from the app; nothing else referenced it |
| The instruction paragraph telling the agent to answer physical questions and ignore the policy | Replaced with one sentence: describe the object and the damage plainly in `what_you_see` |

A code comment at the top of the function records **why** the check is absent, so a future
reader does not "fix" the omission by reintroducing the same failure.

**What still holds** — all proven offline against the edited source in `phototest2.py`:

- The contradiction path: a reported crack absent from the photo sets `photo_contradiction`,
  returns `NO_DAMAGE_VISIBLE`, fires **DL-5**, forces `HUMAN_REVIEW`, and says nothing
  accusatory.
- The unusable-photo path: exactly one retry, then a human.
- Deterministic tariff pricing — $840 for a screen, $3,000 total loss — never from the vision read.
- The total-loss path, the bad-enum rejection, and all v1→v11 hardening.

**A junk photo now lands in `unclear`, not in a rejection.** A picture with no device or no
screen in it is caught by `crack_visible="unclear"` → one retry → human review, which is the
same end state the removed guard produced. That enum's description was deliberately
strengthened to name "no screen and no device visible in the picture at all" as one of its
cases, because it is now the only thing standing between a junk photo and an approval.

**The limitation, stated for reuse:**

> **Photo assessment confirms whether the reported damage is visible. It cannot verify that
> the photographed device is the insured device. Demos must use matching device/policy pairs.**

`phototest2.py` now asserts this as an explicit passing test — it feeds the exact false report
the model produced live (`"a black phone with several cracks on the screen"` against
PDP100294, a MacBook) and asserts the claim proceeds and is priced from the tariff at $840.
The limitation is a visible test result, not a silent absence.

**Version and deployment:** chat v6 `56a8b22a-2baf-4b18-9540-cdc6185acbee` was cut, and the
customer-facing chat deployment `d7bfbb93` was repointed to it (confirmed serving). Both
widget tools survived the export→edit→import round trip, verified in the fresh export before
editing and again server-side after the push. Voice app `6e01e4a5` untouched, still pinned to
v11 `b17c9a26`.

**Still open on 05-02** (why this plan stays `status: incomplete`), as of 2026-08-10. This
plan is *"Decision card + quick actions"*. The decision card is built, deployed and
**visually confirmed**. `cover_offer_actions` — the other half of the plan's own title — is
now built, deployed and **fires with a contract-shaped payload**, but only its *payload* is
proven; **nobody has yet watched the buttons draw in a browser**, and the plan does not flip
to `complete` on a headless-only result. Two of the plan's four original items are closed:

- ~~**The card's on-screen render is still unconfirmed by eye.** The payload now provably
  matches the SDK contract and was verified on a live chat-v7 run, but nobody has yet watched
  the card draw in a browser. Only a human at `http://localhost:3000` can close this.~~
  **CLOSED 2026-08-09** (quick task 260809-nt7) — see *"Rendering — CONFIRMED BY EYE
  2026-08-09"* above.
- ~~**`cover_offer_actions` has still never fired**, and is now also believed defective against
  the `quick_actions` schema — unfixed.~~ **RESOLVED WITH A CAVEAT 2026-08-10** (quick task
  `260810-ifr`, chat v8 `3f85b1d8`). It was defective, it is fixed, and it has now fired on a
  live conversation with a contract-shaped payload, a spoken offer naming the device, and no
  same-turn `end_session`. **Caveat — this is what keeps the plan incomplete:** the buttons'
  on-screen render is **not yet confirmed by eye**, and the record shows the offer sentence
  emitted **twice**, which may or may not double up in the browser. Only a human at
  `http://localhost:3000` can close both.
- **The decision turn says nothing alongside the card** — still open, and now known to be
  *non-deterministic* rather than always-silent: on the 2026-08-10 v8 run the agent *did* say
  *"I've looked into that, and I can approve this claim right now."* alongside the card, with
  `textResponseConfig` still `NONE` and the instruction contradiction still unresolved. That
  is consistent with `NONE` meaning "the LLM decides each time", and makes the beat unreliable
  rather than reliably broken — a presenter still cannot depend on it.
- The **photo CONFIRM path is now verified live** (2026-08-09, quick task 260809-nt7, chat v7
  `bb14cdcc`). The **photo CONTRADICTION path** remains not verified live — it needs a photo
  of an undamaged device through the widget, which no executing session has had; proven
  offline only, in `phototest2.py`.
