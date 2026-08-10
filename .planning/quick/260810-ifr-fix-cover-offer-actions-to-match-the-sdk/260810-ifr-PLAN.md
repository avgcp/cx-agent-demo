---
quick_id: 260810-ifr
phase: quick-260810-ifr
type: quick
plan: 01
wave: 1
depends_on: []
autonomous: true
requirements: [QUICK-260810-ifr]
files_modified:
  - .planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md
  - .planning/STATE.md
  - .planning/spec/DEMO-RUNBOOK.md

must_haves:
  truths:
    - "`cover_offer_actions` emits `{actions: [{content, utterance}, ...]}` and nothing else"
    - "The agent speaks the offer sentence as text in the same turn as the buttons"
    - "The agent does NOT end the session in the turn that shows the buttons"
    - "The cross-sell turn is actually reached in a live conversation, with the tool invoked"
    - "`claim_decision_card` is byte-identical before and after, and still draws"
  artifacts:
    - path: "server: tools/1a02f494-691e-40c8-8472-adfe4307c930 (cover_offer_actions)"
      provides: "QUICK_ACTIONS widget matching the SDK contract"
      contains: "actions"
    - path: "server: agents/87551704-9bc1-4fe0-bde2-57d377cb8963 (claim_intake)"
      provides: "Cross-sell step that says the question aloud and defers end_session"
      contains: "Close after the offer"
    - path: ".planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md"
      provides: "The quick_actions contract and the outcome"
      contains: "quick_actions"
  key_links:
    - from: "claim_intake Cross-sell step"
      to: "cover_offer_actions"
      via: "tool call, same turn as the offer sentence"
      pattern: "cover_offer_actions"
    - from: "deployment d7bfbb93"
      to: "the new chat version"
      via: "deployments.patch appVersion"
      pattern: "appVersion"
---

<objective>
Reshape the `cover_offer_actions` widget tool to the deployed SDK's `quick_actions` contract,
fix the three instruction defects that would keep the cross-sell beat broken even with a
correct payload, and prove the tool actually fires on a live conversation.

Purpose: the cross-sell is the demo's "cost centre → profit centre" headline moment and it has
never once fired. Its payload is also a hard-failure shape, not a cosmetic one.
Output: a new chat version served by `d7bfbb93`, a live conversation record containing a
contract-shaped `cover_offer_actions` call, and three planning docs updated.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/STATE.md
@.planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md
@.planning/quick/260809-n1b-reshape-claim-decision-card-to-the-sdk-o/260809-n1b-SUMMARY.md
@.planning/spec/DEMO-RUNBOOK.md
@./CLAUDE.md
</context>

---

# EVERYTHING YOU NEED IS IN THIS PLAN — do not go re-derive it

You have no memory of the investigation that produced this plan. All of the following was
read from source (the deployed SDK bundle, the CES v1beta discovery document, and the live
API) during planning. **Trust it and do not spend context re-confirming it.** Do not
re-download the SDK bundle and do not re-read the conversation records to re-derive the
findings below — the only things you must confirm at runtime are the ones the tasks tell you
to assert against the live API.

## Identifiers

| Thing | Value |
|---|---|
| GCP project / location | `insurance-agent-demo-500614` / `us` |
| API host | `https://ces.googleapis.com/v1beta` |
| `APP` (use this literal everywhere) | `projects/insurance-agent-demo-500614/locations/us/apps/a2f621e4-9faf-505a-b804-22471f022366` |
| App name | *Meridian Claim - Chat (hardened)* |
| Current version | chat v7 `bb14cdcc-d723-4be1-85af-9f4451e22ed5` |
| Deployment to repoint | `d7bfbb93-8cee-43fe-9095-bc5775f353bd` (*chat - meridian demo*, `WEB_UI` / `CHAT_ONLY`) — currently serving `bb14cdcc`, confirmed at planning time |
| `cover_offer_actions` tool id | `1a02f494-691e-40c8-8472-adfe4307c930` |
| `claim_decision_card` tool id | `5456c890-10ad-4784-aedf-2d46ec918437` |
| `claim_intake` agent id | `87551704-9bc1-4fe0-bde2-57d377cb8963` (holds both widget tools) |
| `claims_concierge` agent id | `2d066224-bea6-4c99-8bca-28b5f9f89d55` (root; do not touch) |
| **NEVER TOUCH** | voice app `6e01e4a5-42a8-5213-b3da-c9053ff8ea52` (pinned v11 `b17c9a26`), fork `9ae7a0c3-6511-413c-8cdb-0efe9e90d2b9` |

Auth: `TOK=$(gcloud auth print-access-token)`. Confirmed working at planning time.
**Every mutating URL must contain the literal string `a2f621e4`.** Grep your own commands for
it before sending. If a URL you are about to `PATCH` or `POST` does not contain `a2f621e4`,
stop.

## The `quick_actions` contract — read from the deployed bundle, VERIFIED

Source: `https://www.gstatic.com/chat-messenger/sdk/prod/v1.16/chat-messenger.js` (1,358,541
bytes). Builder, dispatched from `case "quick_actions":`:

```js
function DF_MQA(a, b) {
  a = new DF_MKs(a.utterance.utteranceId, b.id);
  a.quickActions = b.actions.map(function (c) {
    return { content: c.content, description: c.description, utterance: c.utterance };
  });
  return a;
}
```

Renderer (`DF_MKs.prototype.render`), the part that settles the field semantics:

```js
var b = (this.Kf = this.quickActions.some(function(d){return d.description}))
        ? "grid-layout columns-" + this.Ig : "";
var c = this.quickActions.map(function(d){
  return DF_Mx(DF_MIs, function(){
      ... var e = d.utterance || d.content;
      a.v.renderCustomText(e, false);
      a.v.presenter.sendQuery(e);
      ...
    }, d.content, d.description ? DF_Mx(DF_MHs, d.description) : null)
});
```

**Payload contract:**

```
{ actions: [ { content, description?, utterance? } ] }
```

| Field | Verified behaviour | Decision for this build |
|---|---|---|
| `actions` | `b.actions.map(...)` is **unguarded**. Absent ⇒ `TypeError` ⇒ `render()` never runs ⇒ **nothing appears at all**. This is a hard failure, not a JSON blob. | **Required.** Declared `required` in the schema, with the reason in its description. |
| `content` | Button label. Interpolated as a Lit template value — no method call, no unguarded deref. | **Required.** `"Add it"` / `"Not now"`. |
| `utterance` | `e = d.utterance || d.content`, then `renderCustomText(e,false)` **and** `presenter.sendQuery(e)`. So it is echoed into the transcript **as the customer's own message** and sent to the agent as a query. | **Required, and it must read like a human sentence** — the audience sees it in the transcript. A bare `"accept"` would appear as the customer typing `accept`. |
| `description` | **Guarded** (`d.description ? … : null`) — omitting is safe. BUT `.some(d => d.description)` means one description flips the *whole set* to `grid-layout columns-1` plus a left-align style injected into each button's shadow root. | **Omit, and do not even declare it** — declaring it invites the model to fill it and silently change the layout. |
| `id` | Consumed as `b.id` by the platform, not by us. The decision card works without declaring it. | **Do not declare.** |
| `prompt` (current) | **Read by nothing.** There is no title/prompt slot anywhere in `DF_MQA` or `render`. | **Delete.** The question has no home in the widget — see the instruction fix. |
| `options` (current) | Read by nothing. | **Delete.** |

No field other than `actions` is dereferenced unguarded. (Contrast the decision card, where
`imageUri` was.) The widget self-dismisses on the first user input
(`connectedCallback` registers a `once` listener on `chat-messenger-user-input-entered`).

## WHY IT HAS NEVER FIRED — established during planning, from the conversation records

**It is not a wiring defect.** All three prerequisites are in place:

- `cover_offer_actions` **is** attached to `claim_intake` — the agent that runs the entire
  claim, which every conversation reaches.
- `uninsured_device` **is** a declared session variable (one of 37) and **is** written by
  `verify_identity`, i.e. populated before the claim is even priced.
- `claim_intake`'s instruction **does** contain an explicit `<step name="Cross-sell">` that
  names `{@TOOL: cover_offer_actions}`.

**It has never fired because no conversation has ever taken a turn past the email
confirmation on the auto-approve path.** Of the **14 conversations on record** for this app,
`toolCall.displayName == "cover_offer_actions"` appears in **zero**. Exactly one conversation
ever reached `send_claim_email` alongside a decision card — `dfMessenger-7c8fef9a-bb20-45e1-b5ea-067ea1911854`,
the user's live 2026-08-09 run — and it **ended on the email turn**: the customer's last
message was *"ok great"*, the agent's last message was the email confirmation, and the
conversation closed 2 minutes later with no further input.

The instruction *forbids* bundling the offer onto the email turn ("This is its own turn and
ends with the last word of it… do not bundle the cover offer onto the end"), so the
cross-sell **structurally requires one more customer turn after the email**. Nobody has ever
given it one. Every other run stopped earlier — simulator scripts, the API run that hit quota,
or the escalation path (which deliberately has no cross-sell).

**Consequence for this task:** the payload defect is real and latent — it would have thrown the
instant it fired — but fixing the payload alone changes nothing observable. The verification
run in Task 3 **must drive one turn past the email**, or this task proves nothing.

## Three further defects found in the Cross-sell instruction — all must be fixed

1. **`end_session` is instructed in the same turn as the widget.** The step currently reads
   *"…Accept a decline gracefully. Then {@TOOL: end_session}."* If the agent obeys literally,
   the session ends before the customer can tap and the buttons are inert. **The buttons would
   be dead even with a perfect payload.**
2. **"the buttons ARE the question" is provably false.** The step says *"Do not ask the
   question in plain text as well - the buttons ARE the question"*, and puts the sentence in the
   `prompt` parameter. `prompt` renders nowhere. Following that instruction produces two naked
   buttons with no question above them. **This must be inverted.**
3. **`textResponseConfig: {type: NONE}`** means *"the LLM dynamically decides whether to
   generate a text response"* — not "no text". Combined with (2) that leaves the offer sentence
   free to vanish, which is exactly the already-observed silent-decision-turn failure. Fix by
   setting `LLM_GENERATED` **on this widget only**.

`textResponseInstruction` is a **real field**, confirmed in the CES v1beta discovery document:
`WidgetToolTextResponseConfig { type, textResponseInstruction, staticText }`, where
`textResponseInstruction` is *"Instruction for the LLM on how to generate the text response.
Used as the description for the text response parameter if type is LLM_GENERATED."* No
contingency ladder is needed.

## Mechanism: direct PATCH, NOT export/import

`ces.projects.locations.apps.tools.patch` and `…apps.agents.patch` both exist
(`PATCH v1beta/{+name}?updateMask=…`), confirmed in the discovery document.

**Do not call `importApp` anywhere in this task.** The `exportApp` in Task 1 is a mandatory
*snapshot and assertion gate* only. Patching directly avoids the import path that previously
risked silently deleting server-only widget tools, and is far cheaper.

⚠ **Flag this in the summary:** after this task the exported package on disk is stale. Any
future export→edit→import task must take a **fresh** export first or it will revert these
changes.

## Traps carried forward from the decision-card fix (260809-n1b)

- A widget tool's declared `parameters` **are** its emitted payload. There is no mapping layer.
- `exportApp` is on the app resource and **requires** `{"exportFormat":"JSON"}`.
- `importApp` is on the **collection** (`…/apps:importApp`), not `${APP}:importApp` (404 HTML).
  Not needed here.
- Numbers must be JSON numbers, never proto3 strings. (Not applicable — this payload has none.)
- CES quota (`RunSession LLM tokens`, 1,000/min) is the binding constraint. One conversation
  costs 120k–150k input tokens.

---

<tasks>

<task type="auto">
  <name>Task 1: Snapshot the app, then reshape cover_offer_actions to the actions contract</name>
  <files>server-side only: tools/1a02f494-691e-40c8-8472-adfe4307c930 on app a2f621e4</files>
  <action>
Work in the session scratchpad. Nothing from the app source goes into `.planning/`.

**A. Mandatory pre-edit gate.** Take a FRESH `exportApp`:
`POST {APP}:exportApp` with body `{"exportFormat":"JSON"}`. Do not reuse any cached zip.
Assert **both** `claim_decision_card` AND `cover_offer_actions` are present in the exported
package. **If either is missing, STOP and report — make no further calls.**

**B. Capture the before-state**, all by `GET`, saved to the scratchpad:
- `GET {APP}/tools?pageSize=50` → save the full `widgetTool` object of `claim_decision_card`
  to `card-before.json`. This is the regression baseline for Task 2.
- `GET {APP}/tools/1a02f494-691e-40c8-8472-adfe4307c930` → note its `etag`.
- `GET {APP}/agents/87551704-9bc1-4fe0-bde2-57d377cb8963` → save `instruction` verbatim to
  `claim_intake-before.txt` and note its `etag`. Expected length ~17,057 chars.
- Assert `uninsured_device` appears in the app's `variableDeclarations` (expected: 37 vars).

**Never print the source of `resolve_claim`.** It contains a live Resend API key. Filter tool
listings by `displayName` in a script; do not `cat` or echo whole tool payloads.

**C. Patch the tool.**
`PATCH {APP}/tools/1a02f494-691e-40c8-8472-adfe4307c930?updateMask=widgetTool`
Include the `etag` from step B at the top level of the body. On an etag/ABORTED failure, re-GET
and retry once. Body `widgetTool`:

- `name`: `cover_offer_actions` (unchanged)
- `widgetType`: `QUICK_ACTIONS` (unchanged)
- `description`: "Shows the cover offer as two tappable buttons. Call it in the SAME turn as
  your own one-sentence offer message — the widget renders only the buttons and has no slot
  for a question, so it cannot ask on your behalf. Only after the claim is resolved
  AUTO_APPROVE and uninsured_device is not empty. Never quote a price — there isn't one."
- `parameters`: `type: OBJECT`, `required: ["actions"]`, with exactly one property:
  - `actions`: `type: ARRAY`, description: "Exactly two buttons, in this order. The renderer
    calls .map() on this array unguarded, so it must always be present and non-empty or the
    widget throws and nothing renders at all." `items` is an `OBJECT` with
    `required: ["content","utterance"]` and exactly two properties:
    - `content` (`STRING`): "The button label the customer sees. Use exactly 'Add it' for the
      first and 'Not now' for the second."
    - `utterance` (`STRING`): "Sent into the conversation as the customer's own message when
      the button is tapped, and shown in the transcript as if they typed it — so it must read
      like a person. Use exactly 'Yes please, add it to my cover.' for 'Add it' and 'Not right
      now, thanks.' for 'Not now'. Never a bare token like 'accept'."
  - **Declare no `description` property on the items, and no `prompt`, `options`, `id` or
    `type` anywhere.** A `description` on any action flips the whole button set to a grid
    layout; `prompt`/`options` are read by nothing.
- `textResponseConfig`: `{"type": "LLM_GENERATED", "textResponseInstruction": "One short
  sentence offering to add the uninsured device to their cover, naming that specific device.
  The buttons carry only the answers, so this sentence is the only place the question is
  asked. Never quote a price, premium or monthly cost."}`
- Set **no** `dataMapping` — the actions are static strings the model emits verbatim, and
  adding a mapping would mean editing `resolve_claim`, the file holding the live secret.

**D. Do not touch `claim_decision_card` in this task at all.**
  </action>
  <verify>
    <automated>
POST `{APP}:retrieveToolSchema` for tool `1a02f494-…` and assert, in a script:
(1) the declared parameter keys are exactly `{"actions"}`;
(2) `prompt` and `options` appear nowhere in the schema;
(3) `actions.items` requires both `content` and `utterance`;
(4) `description` is not a declared item property;
(5) `textResponseConfig.type == "LLM_GENERATED"` and `textResponseInstruction` is non-empty.
Then re-GET `{APP}/tools` and assert `claim_decision_card`'s `widgetTool` object is
**byte-identical** to `card-before.json`, and that all 8 tools are still present.
    </automated>
  </verify>
  <done>`cover_offer_actions` declares only `actions[{content,utterance}]`, requires text, and `claim_decision_card` plus the other 7 tools are provably untouched.</done>
</task>

<task type="auto">
  <name>Task 2: Fix the Cross-sell instruction, cut a version, repoint the deployment</name>
  <files>server-side only: agents/87551704-9bc1-4fe0-bde2-57d377cb8963, a new version, deployment d7bfbb93</files>
  <action>
**A. Surgical instruction replacement.** Load `claim_intake-before.txt` from Task 1. In a
Python script, locate the single occurrence of `<step name="Cross-sell">` and the first
`</step>` after it (the existing block is 1,217 chars and is the last step before
`</subtask>`). Replace **that block only** with the two steps below. Change nothing else in
the 17k-char instruction — in particular leave rule 21, the device-mismatch offer at the
`<step>` around line 68, and the entire `Decide and close` subtask untouched.

Replacement (preserve the surrounding indentation of the original block):

```
<step name="Cross-sell">
    <trigger>The claim was AUTO_APPROVE, the email has been sent, and the call is winding down.</trigger>
    <action>
        Skip this step entirely on the HUMAN_REVIEW path - never sell to someone whose claim
        just went to a specialist. Skip it too if {uninsured_device} is empty.

        Say the offer OUT LOUD as your own one-sentence message, naming that specific device -
        for example "One more thing while I have you: your {uninsured_device} isn't on this
        policy. Would you like me to add it to your cover?" The buttons cannot ask the question
        for you - they only carry the answer - so if you do not say it, nothing asks it.

        In that same turn call {@TOOL: cover_offer_actions} with exactly two actions, in order:
        content "Add it" with utterance "Yes please, add it to my cover.", and content
        "Not now" with utterance "Not right now, thanks." Use those four strings verbatim and
        send no other fields.

        Never quote a price, a premium or a monthly cost - you do not have one, and you must
        not invent one. If they want numbers, say you will have someone send the options over.

        Do NOT end the session in this turn. The customer has not answered yet, and ending
        here kills the buttons before they can be tapped.

        If you ALREADY offered to add {uninsured_device} earlier in this call - because they
        mentioned it themselves - do not offer it a second time. Acknowledge what they decided
        then and go straight to the closing step.
    </action>
</step>

<step name="Close after the offer">
    <trigger>The customer has answered the cover offer, by tapping a button or by saying so.</trigger>
    <action>
        If they accept - "Yes please, add it to my cover." or anything equivalent - say you
        will get the options sent over, without quoting any price or premium. If they decline -
        "Not right now, thanks." or anything equivalent - accept it gracefully in one line and
        do not press.

        Either way close warmly in that same message, then call {@TOOL: end_session}. This is
        the last turn of the conversation.
    </action>
</step>
```

Before sending, assert in-script: the new instruction contains `Close after the offer`,
contains `Do NOT end the session in this turn`, no longer contains `the buttons ARE the
question`, no longer contains `(value "accept")`, still contains all of
`Never send the card without a sentence`, `WORD FOR WORD`, `Skip this step entirely on the
HUMAN_REVIEW path`, and `</taskflow>`, and that the length delta is between +500 and +1,500
chars. **If any assertion fails, STOP — do not PATCH.**

**B. Patch the agent.**
`PATCH {APP}/agents/87551704-9bc1-4fe0-bde2-57d377cb8963?updateMask=instruction` with the new
instruction and the etag from Task 1. Then re-GET and assert `validationErrors` is empty or
absent, `tools` still lists all 8 (including both widget tools), and `childAgents`/
`transferRules` are unchanged.

**C. Cut a version.** `POST {APP}/versions` with displayName/description
*"chat v8 - cover_offer_actions reshaped to SDK quick_actions contract; cross-sell step says
the offer aloud and defers end_session"*. Record the new version id.

**D. Repoint the deployment.**
`PATCH {APP}/deployments/d7bfbb93-8cee-43fe-9095-bc5775f353bd?updateMask=appVersion` with
`{"appVersion": "{APP}/versions/<new-version-id>"}`. Then **re-GET the deployment and read
back** `appVersion` — do not infer success from the response to the PATCH.
Assert `channelProfile.channelType == "WEB_UI"` and `webWidgetConfig.modality == "CHAT_ONLY"`
are unchanged.
  </action>
  <verify>
    <automated>
Fetch the new version snapshot (`GET {APP}/versions/<new-id>`) and assert: both
`claim_decision_card` and `cover_offer_actions` are present; `cover_offer_actions` declares
`actions` and neither `prompt` nor `options`; `claim_intake`'s instruction inside the snapshot
contains `Close after the offer`. Re-GET `d7bfbb93` and assert `appVersion` ends with the new
version id. Re-GET the app and assert `variableDeclarations` is still 37 entries.
    </automated>
  </verify>
  <done>A new chat version exists carrying both fixes, `d7bfbb93` provably serves it, both widget tools survived, and the decision card is untouched.</done>
</task>

<task type="auto">
  <name>Task 3: Prove it fires on one live conversation, then update the three docs</name>
  <files>.planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md, .planning/STATE.md, .planning/spec/DEMO-RUNBOOK.md</files>
  <action>
**A. ONE live conversation — and it must go one turn past the email.** That is the whole point
of this task; see the "WHY IT HAS NEVER FIRED" section.

Use `POST {APP}/sessions/<uuid-you-generate>:runSession` with body
`{"config": {"deployment": "{APP}/deployments/d7bfbb93-8cee-43fe-9095-bc5775f353bd"}, "inputs": [{"text": "..."}]}`.
Routing through the deployment (rather than the draft app) also proves the repoint.

Use a **keyboard fault, not a screen crack** — screen repairs hit the photo gate and the
executor cannot upload an image. This is the same path 260809-n1b used successfully.

| # | Text to send | Expect |
|---|---|---|
| 1 | `PDP100294, Jordan Rivera` | verified, handed to claim_intake |
| 2 | `the keyboard is broken, some keys don't work. no liquid, and it works normally otherwise` | priced, AUTO_APPROVE, `claim_decision_card` |
| 3 | `ok great` | the email turn, `send_claim_email` |
| 4 | `that's everything, thanks` | **the cross-sell turn — `cover_offer_actions` + a spoken offer** |

If turn 4 closes without the offer, send **one** more turn (`thanks, bye`) before concluding
it did not fire — the "winding down" trigger is a judgment call.

**Quota discipline.** ~120–150k input tokens per conversation against a 1,000/min limit. On
HTTP 429 `RESOURCE_EXHAUSTED`: pace, do not loop. **You are authorised for exactly one spaced
retry of the failed turn** in the *same* conversation after waiting ≥90s (this is the
precedent from 260809-n1b, where it worked). A second 429 ⇒ **STOP and report**, no third
attempt, no new conversation.

**B. Assert the emitted payload.** `GET {APP}/conversations/<session-id>` and locate
`turns[*].messages[*].chunks[*].toolCall` where `displayName == "cover_offer_actions"` (that
is the exact path — verified during planning). Assert:
1. the call exists at all — this alone is the headline result;
2. its argument keys are exactly `{"actions"}` — no `prompt`, no `options`;
3. `actions` is a list of length 2;
4. every entry has non-empty string `content` and `utterance`;
5. no entry carries a `description` key;
6. the `utterance` values are natural sentences, not bare tokens like `accept`/`decline`;
7. **in the same turn there is agent text naming the uninsured device** (`iPhone 16 Pro Max`
   for PDP100294) — this proves `LLM_GENERATED` gave the question a home;
8. `end_session` was **not** called in that same turn.

Also assert the run did not regress the card: a `claim_decision_card` `toolCall` is present
with a `productItem` carrying `title`, `subtitle`, a numeric `price.units`, `nanos` and
`currencyCode`, and an `imageUri`.

Record the exact emitted payload and the agent's offer sentence verbatim — the summary needs
both. Do not paste any tool *source* into the summary.

**C. Documentation.** Write only what the evidence supports; the on-screen render will still
be unverified.

`05-02-SUMMARY.md`:
- Replace the `## cover_offer_actions is very likely broken the same way — UNFIXED` section
  with a resolved section: the verified `quick_actions` contract (builder + renderer + the
  four field behaviours from this plan's contract table), the real reason it never fired
  (zero of 14 conversations reached a turn past the email; the one that got closest ended
  there), the three instruction defects, and the shipped payload.
- Update the "Still open on 05-02" list: mark the `cover_offer_actions` bullet resolved-with-a-
  caveat. **Keep `status: incomplete`** and state precisely why: the cross-sell's on-screen
  render is not yet confirmed by eye, and the silent-decision-turn defect and the unverified
  photo-contradiction path both remain open. Do not flip the status on a headless-only result.

`.planning/STATE.md`: rewrite the `cover_offer_actions` blocker bullet — it currently says
"very likely broken… UNFIXED… never fired". Replace with what happened, the new version id,
the "never fired = never reached, not miswired" finding, and what is still pending visually.
Add a Quick Tasks Completed row for `260810-ifr`. Update `last_updated`/`last_activity`.

`.planning/spec/DEMO-RUNBOOK.md`, chat section only: update the `Version live` row to the new
version. Add the cross-sell beat to Scenario C's turn table — a final row *"that's everything,
thanks"* → *offers to add the uninsured iPhone 16 Pro Max, with two buttons* — plus a short
presenter note that tapping a button types the sentence into the chat as if the customer had
written it, and that there is deliberately **no** cross-sell on the escalation path. Mark the
beat **"payload verified headlessly, on-screen render not yet confirmed by eye"**, in the same
style the runbook already uses for Scenario D. Leave the phone section alone.

Make no git commit — the orchestrator owns the docs commit.
  </action>
  <verify>
    <automated>
The Task 3B assertion script prints PASS/FAIL per numbered assertion and exits non-zero on any
failure. Then `grep -c` the three docs (with `grep -v '^#'` where counting) to confirm:
05-02-SUMMARY.md contains `quick_actions` and still has `status: incomplete`; STATE.md no
longer contains the string `very likely broken the same way`; DEMO-RUNBOOK.md contains the new
version id. `git status --short` shows changes confined to `.planning/`.
    </automated>
  </verify>
  <done>One live conversation contains a contract-shaped `cover_offer_actions` call with a spoken offer alongside it, and the three docs record the result honestly with the visual check named as still outstanding.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|---|---|
| widget button → agent | `utterance` is injected via `presenter.sendQuery()` as if the customer typed it — attacker-influenceable only by whoever authors the actions, i.e. the model |
| repo/summaries → git | `resolve_claim` holds a live Resend API key |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|---|---|---|---|---|
| T-ifr-01 | Information disclosure | live Resend key in `resolve_claim` | mitigate | `resolve_claim` is not read, edited, echoed or copied in this task. Tool listings are filtered by `displayName` in-script, never `cat`'d. No key in any summary or commit. |
| T-ifr-02 | Tampering | wrong app patched | mitigate | Every mutating URL must contain the literal `a2f621e4`; voice app `6e01e4a5` and fork `9ae7a0c3` receive no call of any kind. Pre-edit `exportApp` gate; post-push read-back of every mutation. |
| T-ifr-03 | Tampering | model-authored `utterance` injected into the conversation | accept | Mock demo, no real systems; the utterance strings are pinned verbatim in the schema descriptions and asserted on the live run. Escalates to `mitigate` (FIELD_MAPPING from a tool) only if the live run shows the model deviating. |
| T-ifr-SC | Tampering | package installs | n/a | No package-manager installs in this task. |
</threat_model>

<verification>
- Fresh `exportApp` taken before any edit, with both widget tools asserted present. STOP gate honoured.
- `retrieveToolSchema` shows `cover_offer_actions` declaring only `actions[{content,utterance}]`.
- `claim_decision_card`'s `widgetTool` object byte-identical before and after; 8 tools throughout; 37 session variables throughout.
- New version snapshot contains both widget tools and the new instruction; `d7bfbb93` read back serving it.
- One live conversation reaching turn 4+ with `cover_offer_actions` invoked, contract-shaped, alongside agent text naming the uninsured device, and no same-turn `end_session`.
- Voice app `6e01e4a5` and fork `9ae7a0c3`: zero API calls issued.
</verification>

<success_criteria>
1. `cover_offer_actions` emits `{actions:[{content,utterance},{content,utterance}]}` and nothing else, proven on a live conversation record.
2. The agent speaks a one-sentence offer naming the uninsured device in the same turn as the buttons.
3. The session is not ended in the button turn.
4. The decision card, the tariff pricing, the photo confirm path and single-send email are all unregressed.
5. `d7bfbb93` serves the new version; the voice app and the fork are untouched.
6. `05-02-SUMMARY.md`, `STATE.md` and `DEMO-RUNBOOK.md` state exactly what is proven and exactly what still needs the user's eyes.
</success_criteria>

<output>
Create `.planning/quick/260810-ifr-fix-cover-offer-actions-to-match-the-sdk/260810-ifr-SUMMARY.md` when done.

It must state: the new version id; whether `d7bfbb93` was repointed (read back, not inferred);
the exact emitted `actions` payload; the agent's offer sentence verbatim; the finding on why it
never fired; whether the visual check is still outstanding; and the stale-export warning for
future export→import tasks.

**Close with a short, specific list for the user to check at `http://localhost:3000`** (the rig
is already running — do not rebuild it). Run policy **PDP100294**, *Jordan Rivera*, **keyboard**
fault, "works normally otherwise", no photo needed; accept the decision, then say *"that's
everything, thanks"*. Expected: a sentence naming the **iPhone 16 Pro Max** and offering to add
it to cover, followed by **two buttons side by side in a single row**, labelled **Add it** and
**Not now**. Tapping one should type its sentence into the chat as the customer's own message
and the agent should close warmly. Failure signals worth reporting: **nothing at all appears
where the buttons should be** (the `.map()` throw — the whole message dies, it does not degrade
to JSON); buttons appear but **no question is asked above them**; the buttons are **stacked in a
grid with subtitles** rather than a plain row (a stray `description`); the transcript shows a
bare **`accept`**/**`decline`** instead of a sentence; or the conversation **ends before the
buttons can be tapped**.
</output>
