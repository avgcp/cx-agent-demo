# 05-06 Voice Baseline — *Meridian Claim - Voice (demo-ready)*

**Captured (UTC):** `2026-08-11T03:10:51Z` · plan 05-06 · project `insurance-agent-demo-500614`,
location `us`, API `https://ces.googleapis.com/v1beta`

**This is a read-only capture.** Nothing in the voice app, its versions or its deployment was
mutated to produce it. Every non-`GET` call issued by this plan was a `runSession` against the
voice **DRAFT** app; a scripted method/URL gate refused everything else (log:
`scratchpad/s06-gate-log.txt`).

**How later voice plans must use this file.** Read it *before* you edit anything on
`6e01e4a5-42a8-5213-b3da-c9053ff8ea52`, and re-assert it *after*. The section headings below are
literal and greppable — **do not rename them.** 05-07 (assessor packet on voice) reads
`## VOICE_INVENTORY` and `## ESCALATION_PATH`; 05-09 (Spanish on voice) reads
`## DECISION_SPEECH_EN` and `## GTP_SURFACE`.

| Section | Answer in one line |
|---|---|
| `## VOICE_INVENTORY` | `draft_equals_v11: true` — 2 agents, 5 python tools, 33 variables, deployment pinned to v11 |
| `## GTP_SURFACE` | `audioProcessingConfig` has exactly two keys: `bargeInConfig` and `synthesizeSpeechConfigs` (`en-US` only) |
| `## DECISION_SPEECH_EN` | `spoken_equals_explanation: **true**` — voice relays the tool's string byte-for-byte, in one chunk |
| `## AUTO_APPROVE_PATH` | tariff `840 / 25 / 1500` PASS, one `send_claim_email`, **cross-sell did NOT fire** ❌ |
| `## ESCALATION_PATH` | `escalation_source: reused cae670d7-a6f1-491d-b782-921a53af6128` |
| `## PHONE_CHECK` | `phone_check: PENDING` — human task, see the section |

> **Two rows in this table were corrected after the evidence came in.** Task 1 wrote the table
> ahead of Task 2 with the plan's *expectations* in it. Both were wrong, in opposite directions:
> the decision speech turned out to be **deterministic** (better than expected), and the cross-sell
> turned out **not to fire at all** (worse than expected). The section bodies are the authority;
> this table is a convenience index.

---

## VOICE_INVENTORY

Captured `2026-08-11T03:10:51Z` by `GET` only. Machine-readable copy:
`scratchpad/s06-inventory.json`.

### App

| Field | Value |
|---|---|
| App | `6e01e4a5-42a8-5213-b3da-c9053ff8ea52` |
| `displayName` | `Meridian Claim - Voice (demo-ready)` |
| `modelSettings.model` | `gemini-3.1-flash-live` |
| `toolExecutionMode` | **`SEQUENTIAL`** |
| `defaultChannelProfile` | `{}` — present but **empty**; the channel type lives on the deployment, not here |
| `languageSettings` | `{"defaultLanguageCode": "en-US"}` — **no `supportedLanguageCodes`, no `enableMultilingualSupport`** |
| `variableDeclarations` count | **33** |
| `predefinedVariableDeclarations` count | 2 |

App-resource top-level keys present: `audioProcessingConfig`, `createTime`, `defaultChannelProfile`,
`deploymentCount`, `description`, `displayName`, `errorHandlingSettings`, `etag`,
`globalInstruction`, `guardrails`, `languageSettings`, `loggingSettings`, `modelSettings`, `name`,
`predefinedVariableDeclarations`, `rootAgent`, `timeZoneSettings`, `toolExecutionMode`,
`updateTime`, `variableDeclarations`.

**`variableDeclarations` names (sorted, 33):**

```
auth_attempts, auth_status, auto_approval_cutoff, carrier_name, claim_amount, claim_ref,
claim_seq, coverage_limit, covered_device, customer_email, customer_name, decision,
decision_explanation, decision_rules, deductible, device_category, dx_answers, dx_ask_count,
dx_issue, dx_last_ask, dx_last_signature, dx_outcome, dx_stall_count, email_body,
email_delivery, email_status, email_subject, issue_category, photo_status, policy_id,
rules_fired, total_loss_flag, uninsured_device
```

> Note `photo_status` is declared on the **voice** app even though voice has no photo path — it is
> inherited from the shared lineage with chat. Do not read its presence as a photo feature.

### Agents

Instruction **bodies were never read, printed or persisted** — only hashed.

| Agent | Root? | `instruction` chars | `instruction` SHA-256 (first 32) | Tools | Children | `transferRules` |
|---|---|---|---|---|---|---|
| `claims_concierge` | **yes** | **5126** | `8f610408ce652a6c…` (full below) | 2 | 1 (`87551704-9bc1-4fe0-bde2-57d377cb8963` = `claim_intake`) | absent (key not present) |
| `claim_intake` | no | **14140** | `628548c1e582e34d…` (full below) | 5 | 0 | absent (key not present) |

Full SHA-256 of the `instruction` string, for byte-exact regression checks:

```
claims_concierge  5126 chars  8f610408ce652a6c99b98364a21eba0af71fac81d1435ed9541b732bb43b52a0
claim_intake     14140 chars  628548c1e582e34d4e26ca56fb03a76c17bb04bceb8853331fc35ab603383fbd
```

Neither agent carries a `transferRules` key at all — routing between `claims_concierge` and
`claim_intake` is instruction-driven, **not** rule-driven. A later plan must not assume a
`transferRules` object exists to be preserved.

These 16-hex prefixes are **identical to the ones 05-03 recorded** for the same two agents, so the
draft has not moved between the two captures.

### Tools

**`tool_count: 5`**

`tool_displayNames` (sorted):

```
escalate_to_human, resolve_claim, run_diagnostic, send_claim_email, verify_identity
```

All five are `executionType: SYNCHRONOUS`, kind `pythonFunction`.

**Parameter names are not retrievable without reading tool source.** A `pythonFunction` tool on
this platform exposes only `{name, description, pythonCode}` — there is no declared input schema, so
the parameter names are derived by the platform from the Python signature inside `pythonCode`.
Hard constraint 3 forbids reading that. Parameter names observed from live `toolCall.args` are
recorded in `## AUTO_APPROVE_PATH` and `## ESCALATION_PATH` instead, which is the sanctioned route.

**Tool attachment per agent:**

| Agent | Tools attached |
|---|---|
| `claims_concierge` (root) | `end_session`, `verify_identity` |
| `claim_intake` | `end_session`, `escalate_to_human`, `resolve_claim`, `run_diagnostic`, `send_claim_email` |

**System tools.**
- `end_session` — **attached to BOTH agents.** It is referenced by agents but does **not** appear in
  `GET apps/6e01e4a5…/tools`, which returns the five python tools only. A later plan counting tools
  must be explicit about which it means: 5 python tool *resources*, 6 distinct tool *references*.
- `customize_response` — **NOT attached to any agent.** There is no barge-in/DTMF/timeout override
  anywhere in this app; whatever the call does on those is the platform default plus
  `audioProcessingConfig.bargeInConfig` (see `## GTP_SURFACE`).

### draft_equals_v11

```
draft_equals_v11: true
```

Compared against version `b17c9a26-3485-4658-9259-dfa4839a7977` (*v11 — no self-narration*), the
version the phone deployment serves. Comparands, all equal, none differing:

- each agent's `instruction` SHA-256 **and** character length (2 of 2 equal)
- each agent's tool count (2 of 2 equal)
- sorted tool `displayName` list (equal)
- tool count (5 == 5)
- `variableDeclarations` count (33 == 33) and sorted names (equal)
- root agent's `childAgents` (equal) and `transferRules` (both absent)
- app `languageSettings` (`{"defaultLanguageCode": "en-US"}` on both)
- app `audioProcessingConfig` — byte-identical draft ↔ snapshot

**Independently corroborated.** 05-03 reached `draft_equals_v11: true` earlier on 2026-08-10 by a
*different* method — whole-object SHA-256 of every agent and tool resource with volatile fields
stripped. This capture re-derives it field-by-field and agrees. 05-03's per-resource hashes remain
the finer-grained anchor:

| Resource | sha256[:16] (05-03) |
|---|---|
| tool `escalate_to_human` | `0f30145634cca897` |
| tool `resolve_claim` | `898ebc7ab33d29af` |
| tool `run_diagnostic` | `8782c85ddfa1b433` |
| tool `send_claim_email` | `c0e80c7ef90d8d50` |
| tool `verify_identity` | `75349f96fb2b1ab7` |
| agent `claim_intake` | `c495723ddd92e699` |
| agent `claims_concierge` | `3318edbcdd40104b` |

### Deployment pin

| Field | Value |
|---|---|
| Deployment | `d28bbcb0-066e-4127-a894-fbf9ba39789f` — *voice - meridian demo* |
| `channelProfile.channelType` | **`GOOGLE_TELEPHONY_PLATFORM`** ✅ |
| `appVersion` tail | **`b17c9a26-3485-4658-9259-dfa4839a7977`** ✅ |
| Deployment resource keys | `appVersion`, `channelProfile`, `createTime`, `displayName`, `name`, `updateTime` |

> **Schema correction for later plans.** There is **no top-level `channelType` field** on a
> deployment resource. It is nested: `deployment.channelProfile.channelType`. A plan asserting
> `deployment["channelType"] == "GOOGLE_TELEPHONY_PLATFORM"` will read `None` and fail spuriously.

---

## GTP_SURFACE

Captured `2026-08-11T03:10:51Z`. **Enumerated, not assumed** — what follows is the key set that
actually exists on the resource today.

### `audioProcessingConfig`

Exactly two keys present:

```
audioProcessingConfig.bargeInConfig
audioProcessingConfig.synthesizeSpeechConfigs
```

**`synthesizeSpeechConfigs`** is a **map keyed by language code**. Key list:

```
["en-US"]
```

and its value is the **empty object** `{}` — i.e. `{"en-US": {}}`. There is **no `es` and no
`es-US` entry**, and no voice name, speaking rate, pitch or volume is set for `en-US` either; the
platform default voice is in use. This confirms 05-RESEARCH.md's prediction and 05-03's note.

**05-09 (Spanish on voice) must read this.** Adding Spanish TTS means adding an `es-US` key to this
map. Today the map's single entry is empty, so the minimal, lowest-risk shape for a later patch is
`{"en-US": {}, "es-US": {}}` — adding a key, not restructuring the object. Whatever is added, the
`en-US` key must still be present and still map to `{}` afterwards, or English TTS has been changed
as a side effect.

**`bargeInConfig`**:

```
audioProcessingConfig.bargeInConfig.bargeInAwareness → true
```

That single boolean is the entire barge-in configuration. There is no barge-in *sensitivity*,
*timeout* or *no-speech* setting on this resource.

### Every key matching `interrupt|bargeIn|barge_in|dtmf|timeout|speech|voice`

Case-insensitive search over the whole app resource (`globalInstruction` excluded from the walk, as
instruction text is not configuration). Three hits, all of them the two keys already listed:

| path | value |
|---|---|
| `audioProcessingConfig.synthesizeSpeechConfigs` | `{"en-US": {}}` |
| `audioProcessingConfig.bargeInConfig` | `{"bargeInAwareness": true}` |
| `audioProcessingConfig.bargeInConfig.bargeInAwareness` | `true` |

The same search over the **deployment** resource `d28bbcb0` returned **zero** hits.

> **Keys not listed above were not present in the resource.** In particular there is **no** DTMF
> config, **no** interruption/no-input timeout, **no** speech-endpointing setting, **no** SIP or
> UUI header config, and **no** `customize_response` tool anywhere in the app. Telephony behaviour
> on the live number is therefore entirely platform default apart from `bargeInAwareness: true`.
> A later plan must not assert against a field named here-by-analogy; if it needs one of those
> knobs it has to establish that the field exists first.

**`audioProcessingConfig` is byte-identical between the draft and the v11 snapshot**, so the
telephony surface above is exactly what the phone number serves.

---

## DECISION_SPEECH_EN

Captured `2026-08-11T21:04Z` from session `962bc958-8dd0-4232-ae6d-7ec1c23e00cf`, Scenario A
against the voice **DRAFT** app (`runSession`, no `config.deployment`). Machine-readable:
`scratchpad/s06convA-final-assertions.json`.

**This is the section 05-09 (Spanish on voice) exists to localise.** Read it byte-for-byte.

### `resolve_claim.explanation` — the tool's own string, byte-pinned

Taken from `chunks[].toolResponse.response.result.explanation` on the `AUTO_APPROVE` branch.
**197 characters.**

```
Good news - that screen replacement comes to $840, which is under the $1,500 I can approve on the spot, so I can approve that for you right now. Your excess is $25, and your reference is CLM-24868.
```

| Property | Value |
|---|---|
| character length | **197** |
| encoding | pure ASCII (no smart quotes, no non-breaking spaces) |
| separator used | plain ASCII hyphen-minus `-` with surrounding spaces, **not** an em dash |
| per-run variable part | `CLM-24868` only — the claim reference changes every run; every other byte is stable |

```
explanation_sha256: 9dc9b34796c194aab5bb11b1b2a55f4ac1a4948acf76c5a74ce8cd36f4fb0b94
```

That hash covers the run-specific `CLM-24868`, so it is an exact-replay anchor rather than a
template anchor. To compare across runs, substitute the reference with `CLM-#####` first.

### The agent's own spoken text in the decision turn

Agent text chunk, user turn 2 (`I dropped my laptop and the screen is cracked…`), **197 characters**:

```
Good news - that screen replacement comes to $840, which is under the $1,500 I can approve on the spot, so I can approve that for you right now. Your excess is $25, and your reference is CLM-24868.
```

```
decision_turn_agent_chunks: 1
spoken_equals_explanation: true
```

**One chunk, not two.** The doubled-sentence trap that `260810-k5x` caught on chat does **not**
occur on voice: the decision turn emits exactly one agent text chunk, and that chunk is
byte-identical to the tool's `explanation`. There is no narration around it and no restatement
after it.

### Why `spoken_equals_explanation: true` is the important result here

The plan anticipated a paraphrase, on the reasoning that voice conveys the explanation by
instruction rather than by the `dataMapping` field-mapping chat v9 uses. **The observed behaviour is
stronger than that.** v11 (*"v11 - no self-narration"*) relays the tool string verbatim — the model
is not rewriting it.

Consequences for 05-09:

1. **The English decision line is deterministic**, so a Spanish line can be made deterministic the
   same way: emit it from `resolve_claim`, not from the model. A bilingual template inside the tool
   that returns the `es` string on the `es` branch will be spoken verbatim exactly as the English one
   is today.
2. **The four substantive fields the Spanish string must carry, field-for-field:** the repair
   figure (`$840`), the on-the-spot approval limit (`$1,500`), the excess (`$25`), and the claim
   reference (`CLM-…`). Plus the framing beat — *good news → amount → under the limit → approved now
   → excess → reference* — in that order.
3. **The regression check for any later voice edit is exact, not fuzzy.** Re-run Scenario A and
   assert (a) `decision_turn_agent_chunks == 1`, (b) `spoken == explanation` byte-for-byte, and
   (c) the string matches the block above with only the `CLM-` digits differing. Any of the three
   failing means self-narration or paraphrase has come back.

### The follow-on turns, verbatim (for completeness — 05-07/05-09 must not disturb these)

Turn 3 (`Okay that works for me`), 116 chars, one chunk:

```
Perfect, I've sent an email to the address on your policy, so please reply to it with photos of the damage attached.
```

Turn 4 (`No thanks`), one chunk — **recorded exactly as emitted, including the literal quotation
marks**, which are a cosmetic artifact, not a transcription of mine:

```
"Understood. Have a good day."
```

> **Cosmetic defect, recorded not fixed.** The closing line ships with literal `"` characters
> wrapping it. On the phone leg TTS may or may not voice them; Task 3's human check should listen
> for a spoken *"quote… unquote"* or an odd clipped delivery on the close. Same family as the 👂
> artifacts the runbook already lists. Raised as a blocker in `05-06-SUMMARY.md`.

---

## AUTO_APPROVE_PATH

```
auto_approve_source: fresh 962bc958-8dd0-4232-ae6d-7ec1c23e00cf
```

Scenario A, driven live against the voice **DRAFT** app (`runSession`, no `config.deployment`),
**4 user turns**, the runbook's script verbatim. Turns 1–3 sent `2026-08-11T03:13–03:15Z`, turn 4
sent `2026-08-11T21:04Z`. Every figure below comes from a `toolCall` / `toolResponse` object, never
from agent prose. Machine-readable: `scratchpad/s06convA-final-assertions.json`.

### Tariff — `resolve_claim.toolResponse`

| Assertion | Expected (runbook) | Observed | |
|---|---|---|---|
| `decision` | auto-approve | **`AUTO_APPROVE`** | ✅ |
| `claim_amount` | `840` | **`840`** (JSON int) | ✅ |
| `deductible` | `25` | **`25`** | ✅ |
| `auto_approval_cutoff` (the on-the-spot limit) | `1500` | **`1500`** | ✅ |
| `claim_ref` | a `CLM-` reference | **`CLM-24868`** | ✅ |
| `rules_fired` | — | **`["DL-1"]`** | — |
| `rule_text` | — | `"Auto-approve: repair <= 50% of coverage, covered, not total loss, diagnostic conclusive."` | — |
| `coverage_limit` | — | `3000` | — |
| `total_loss_flag` | — | `false` | — |
| `decided_from` | — | `"diagnostic record"` | — |
| `relay_mismatch` | — | `false` | — |
| `issue_label` / `device` | — | `"screen replacement"` / `Apple MacBook Pro 16"` | — |
| `email_queued` | — | **`true`** | — |

**`840 / 25 / 1500` — PASS. `AUTO_APPROVE` — PASS.** Pricing is deterministic from the tariff: all
four figures come from the tool's structured result, and the spoken sentence relays that result
verbatim (see `## DECISION_SPEECH_EN`), so the model contributes none of the numbers.

`DL-1` here is the mirror of the escalation path's `DL-3` + `DL-2`. The two paths are driven by the
same rule engine and differ only in which rules match.

### Email — exactly one send

```
send_claim_email_turn_count: 1
```

`send_claim_email` appears in **exactly one** turn (user turn 3). Its `toolResponse`:

| Field | Value |
|---|---|
| `sent` | **`true`** |
| `delivery` | **`"live"`** |
| `claim_ref` | `CLM-24868` |
| `subject` | `Claim CLM-24868 - please reply with photos` |
| `expectation_days` | `3-7 business days` |
| `message` | `"Email already sent when the claim was decided. Customer should reply with photos attached."` |
| `email_on_file` | a synthetic `…@example.com` address (masked here; see the caveat below) |

> **Same single-send mechanic as the escalation path — 05-07 must account for it.** `resolve_claim`
> already returns `email_queued: true`, and `send_claim_email`'s own message says the email was
> *already sent when the claim was decided*. **The customer email is sent by `resolve_claim`, not by
> `send_claim_email`.** `send_claim_email` reports on a send that has already happened. This is
> identical to what `## ESCALATION_PATH` records, so it is the app's general behaviour, not a
> branch quirk. A plan adding an assessor email must not treat `send_claim_email` as the only send
> site or it will double-send the customer.

> **The reported recipient is not the real one.** `email_on_file` reports a seeded
> `@example.com` address, while `delivery: "live"` means Resend actually delivered to the hardcoded
> mailbox inside the tool (`akash.vinayak@nerdery.com`). The `toolResponse` recipient is display
> data; do not use it to assert where mail landed. This is why Task 3's mailbox check exists.

### Cross-sell — DID NOT FIRE ❌

```
cross_sell_fired: false
```

**This is the one substantive defect this baseline records on the auto-approve path.**

| Check | Result |
|---|---|
| cross-sell tool called | **none** — the only tools in the whole conversation were `verify_identity`, `run_diagnostic`, `resolve_claim`, `send_claim_email`, `end_session` |
| cross-sell prose in any agent chunk | **none** — regex over every agent text chunk for `iphone \| 16 pro \| uninsured \| add it to your cover \| also insure \| another device \| bundle` returned **0 hits** |
| turn where the offer was expected | after the turn-3 email confirmation, per the runbook |
| what actually happened at that point | turn 4's `No thanks` was met with `"Understood. Have a good day."` and `end_session` |

The runbook scripts turn 4 (`No thanks`) as the customer *declining the cross-sell*, and promises
*"Then it offers to add the uninsured **iPhone 16 Pro Max**"* as one of *"three separate turns"*.
**No offer was made.** The presenter's `No thanks` landed on nothing and the agent closed.

**Two candidate explanations, and this plan cannot separate them over the API.** Both are recorded
because a later plan must not assume the wrong one:

1. **The trigger utterance is wrong in the voice script.** A text `runSession` gives the agent
   exactly one turn per user turn — it cannot volunteer an unprompted third turn. The offer
   therefore needed a *neutral* closing utterance to hang off. The runbook's chat table uses
   `"That's everything, thanks"` for exactly this beat; the voice script instead jumps straight to
   `"No thanks"`, which reads as a refusal and gets `end_session`. The runbook's own presenter note
   — *"Don't say 'no thanks' early"* — points at this.
2. **v11 genuinely does not offer on the voice auto-approve path.** The runbook's "three separate
   turns" phrasing implies the agent volunteers the offer unprompted, which a live phone call can
   do (continuous audio) and a text `runSession` cannot. If so, the API test simply cannot see the
   beat, and only the phone leg can.

Under (1) this is a **runbook script defect**; under (2) it is a **v11 capability gap**. Either way
the presenter following the runbook verbatim today does **not** get the cross-sell on the phone.

The app is *provisioned* for the beat regardless: `uninsured_device` is one of the 33 declared
variables (see `## VOICE_INVENTORY`), and the runbook's "actually it's my phone" optional beat shows
the same offer prose firing on a different trigger.

**Recorded, not fixed** — per this plan's hard constraint 8. Raised as the primary blocker in
`05-06-SUMMARY.md`. **`## PHONE_CHECK` is the tie-breaker** and is scripted below to distinguish
(1) from (2): the caller is told to say `"That's everything, thanks"` first and *wait*, before
saying `"No thanks"`.

### Session shape

| Field | Value |
|---|---|
| user turns sent | **4** |
| agent turns carrying text | **4** |
| tools, in call order | `verify_identity` (T1) → `run_diagnostic` (T2) → `resolve_claim` (T2) → `send_claim_email` (T3) → `end_session` (T4) |
| `end_session` | **present**, turn 4 — the call closes cleanly by the built-in mechanism |
| `Conversation.languageCode` | `en-US` |
| agent text chunks per turn | T1 **2**, T2 **1**, T3 **1**, T4 **1** (user echo excluded) |

**Observed `toolCall.args` parameter names** (the sanctioned substitute for reading tool source):

| Tool | arg names |
|---|---|
| `verify_identity` | `first_name`, `last_name`, `policy_id` |
| `run_diagnostic` | `issue_category`, `q1`, `q2`, `q3`, `q4`, `q5`, `q6` |
| `resolve_claim` | `diagnostic_outcome`, `issue_type`, `policy_id` |
| `send_claim_email` | `policy_id` |
| `end_session` | (no args on this path) |

Identical to the escalation path's arg names for every shared tool — the two paths call the same
tools with the same signatures.

> **Conversation-record caveat for later plans.** `GET …/conversations/{session}` returned **3**
> turns, not 4: the turn-4 `end_session` exchange is absent from the persisted record even on a
> re-fetch ~17 hours later. The turn-4 evidence above therefore comes from the `runSession` response
> itself (`scratchpad/s06convA-turn-4.json`), which is authoritative. **A later plan that asserts
> only against the conversation record will not see the closing turn.** Assert against the
> per-turn `runSession` responses when the close matters.

### Voice, not chat

`verify_identity` accepted the spoken-digits policy ID `P D P one zero zero two nine four` and
resolved it without a re-prompt — the voice-shaped input path works over text `runSession` too.
Turn 1 greeting: `Thanks, Jordan - putting you through now.` then
`I've got your Apple MacBook Pro 16 inch here, what's happened to it?` — note `16 inch` is spoken
out rather than `16"`, i.e. the agent is already TTS-shaping its own text.

---

## ESCALATION_PATH

```
escalation_source: reused cae670d7-a6f1-491d-b782-921a53af6128
```

**Reused under this plan's own quota rule, not skipped.** 05-03 drove Scenario B live against the
voice **DRAFT** app (`runSession`, no `config.deployment`) earlier on the same UTC date
(2026-08-11 UTC / late 2026-08-10 local), three user turns, and saved the full conversation record.
Every value below was re-parsed **from that saved record** by this plan — from
`chunks[].toolCall` / `chunks[].toolResponse.response.result` objects, never from agent prose —
rather than copied from 05-03's prose. Re-running it would have cost 120–150k input tokens against
a 1,000/min limit for evidence that already exists at the same freshness.

| Assertion | Expected | Observed | |
|---|---|---|---|
| `resolve_claim.decision` | `HUMAN_REVIEW` | **`HUMAN_REVIEW`** | ✅ |
| `resolve_claim.claim_amount` | `3000` | **`3000`** (JSON int) | ✅ |
| `resolve_claim.rules_fired` | `DL-3` and `DL-2` both present | **`["DL-3", "DL-2"]`** | ✅ |
| `resolve_claim.auto_approval_cutoff` | `1500` | **`1500`** | ✅ |
| `resolve_claim.deductible` | `25` | **`25`** | ✅ |
| `resolve_claim.total_loss_flag` | — | `true` | — |
| `send_claim_email` turn count | `1` | **`1`** (turn index 2) | ✅ |
| `send_claim_email.sent` / `.delivery` | — | `true` / `"live"` | ✅ |
| cross-sell tool fired | none | **`cross_sell_fired: false`** | ✅ |
| cross-sell *prose* ("add it to your cover") | absent | absent | ✅ |
| `escalate_to_human` fired | yes | yes (turn 2) | ✅ |
| session ended | yes | **yes — `end_session` called**, turn 2, args `{params, reason, session_escalated}` | ✅ |
| `Conversation.languageCode` | — | `en-US` | — |
| user turns | 3 | 3 | ✅ |

```
cross_sell_fired: false
```

**Tools called, in order:** `verify_identity` (T0) → `run_diagnostic` (T1) → `resolve_claim` (T1)
→ `send_claim_email` (T2) → `escalate_to_human` (T2) → `end_session` (T2).

**Observed `toolCall.args` parameter names** (the sanctioned substitute for reading tool source):

| Tool | arg names |
|---|---|
| `verify_identity` | `first_name`, `last_name`, `policy_id` |
| `run_diagnostic` | `issue_category`, `q1`, `q2`, `q3`, `q4`, `q5`, `q6` |
| `resolve_claim` | `diagnostic_outcome`, `issue_type`, `policy_id` |
| `send_claim_email` | `policy_id` |
| `escalate_to_human` | `reason` |
| `end_session` | `params`, `reason`, `session_escalated` |

### The verbatim specialist-handoff sentence — 05-07 must leave this untouched

Agent text chunk, turn 1, **240 characters**, byte-identical to `resolve_claim`'s `explanation` on
the `HUMAN_REVIEW` branch:

```
So that comes to $3,000, and it looks like the device is a total loss. I'm going to pass this to one of our specialists. They'll already have everything we've talked about, so you won't need to go over it again. Your reference is CLM-24462.
```

`CLM-24462` is per-run — the reference number varies, the rest of the string does not.

The close, one turn later (146 chars), also verbatim:

```
The specialist will need a few photos of the damage, so I've sent an email to the address on your policy - please reply to it with those attached.
```

> **Note for 05-07 (assessor packet).** `send_claim_email`'s own result on this path says
> `"Email already sent when the claim was decided."` with `email_queued: true` already set inside
> `resolve_claim`'s result. The customer email is **queued by `resolve_claim`, not by
> `send_claim_email`** — `send_claim_email` reports on a send that has already happened. A plan
> that adds a second (assessor) email must not assume `send_claim_email` is the only send site, or
> it will double-send the customer.

---

## PHONE_CHECK

```
phone_check: PENDING
```

**Human task — an agent cannot place a phone call.** Tasks 1 and 2 proved over the API that the
draft matches what the phone serves, that the tariff is right, that the email fires once, and
exactly what the agent says at the decision moment. What the API *cannot* prove is the telephony
leg: that the number answers, that speech recognition hears a spoken policy ID, that barge-in and
pacing feel right, that the email actually lands, and **whether the cross-sell beat exists at all**
(see `## AUTO_APPROVE_PATH` — this call is the tie-breaker).

Nothing downstream is blocked on this. 05-07 and 05-09 can proceed against the API evidence above;
this section stays `PENDING` until the call happens, then gets `phone_check: pass` or
`phone_check: fail` written into the fence at the top.

---

### 0. Get the number (2 minutes, console only)

It is **not retrievable via the API**. CX Agent Studio console → project
`insurance-agent-demo-500614` (location `us`) → app **Meridian Claim - Voice (demo-ready)**
(`6e01e4a5-42a8-5213-b3da-c9053ff8ea52`) → deployment `d28bbcb0-066e-4127-a894-fbf9ba39789f`
(*voice - meridian demo*) → the integration / telephony panel.

While you are there, **write the number into `DEMO-RUNBOOK.md`** — the `☎ ____` line under
*Phone channel → Call this number* (around line 20). It has been blank since the runbook was
written.

Before dialling, confirm the deployment still says **v11 `b17c9a26…`**. It did as of
`2026-08-11T21:05Z`.

### 1. What to say — say it out loud, do not type

Four beats. **The third one is deliberately different from the runbook**, to give the cross-sell a
chance to appear:

| # | Say | Then |
|---|---|---|
| 1 | *"Hi, my name is Jordan Rivera and my policy is P D P one zero zero two nine four"* | let it verify and hand you over |
| 2 | *"I dropped my laptop and the screen is cracked. No liquid, and it still switches on and works normally otherwise."* | **let the pause happen** — this is the decision turn |
| 3 | *"Okay that works for me"* | it should confirm the email |
| 4 | **_"That's everything, thanks"_** ← **not** *"No thanks"* | **now WAIT at least 10 seconds in silence.** This is the whole point of the call. If an offer to add the iPhone 16 Pro Max comes, it comes here. |
| 5 | only *after* waiting: *"No thanks"* | it should close |

If the offer arrives *unprompted* between beats 3 and 4 without you saying anything, note that
too — it means the agent volunteers turns on a live call in a way the API test cannot reproduce.

### 2. The decision sentence you are listening for

At beat 2 the agent should say this, essentially word for word (only the `CLM-` number changes):

> *"Good news - that screen replacement comes to **$840**, which is under the **$1,500** I can
> approve on the spot, so I can approve that for you right now. Your excess is **$25**, and your
> reference is **CLM-24868**."*

The API proved this string is relayed **verbatim from the pricing tool, in one chunk** — so on the
phone it should be a single fluent sentence, not a paraphrase and not said twice. If you hear it
reworded, or hear the numbers twice, that is a regression worth reporting.

### 3. Report these eight things

Please answer each one explicitly — they are transcribed straight into this section.

1. **The number you dialled**, and did it answer?
2. **The decision sentence as you heard it**, as close to word-for-word as you can manage. Were the
   three figures exactly **$840 / $1,500 / $25**? Any figure that differed?
3. **Cross-sell** — did it offer to add the uninsured **iPhone 16 Pro Max**? At which beat? Did it
   come unprompted, or only after *"That's everything, thanks"*, or not at all?
4. **Barge-in** — talk over it deliberately once, mid-sentence, during a long turn. Did it stop and
   listen, or talk through you? (App has `bargeInAwareness: true` and no `customize_response`
   override, so platform default applies.)
5. **Longest silence**, roughly, in seconds. The decision turn is known to take ~4 s. **Past ~8 s of
   dead air is new** and worth reporting.
6. **The close** — after *"No thanks"*, did it end cleanly, or hang / loop / drop mid-sentence? The
   API shows it calls `end_session` with the line `"Understood. Have a good day."` — **listen for
   whether the literal quotation marks get voiced or clip the delivery** (recorded cosmetic defect,
   see `## DECISION_SPEECH_EN`).
7. **The mailbox** — open `akash.vinayak@nerdery.com`. How many claim confirmation emails arrived
   **from this call**? Expected **exactly one**, subject `Claim CLM-24xxx - please reply with
   photos`. Report the integer and the claim reference on it, and whether that reference matches
   what you heard at beat 2.
8. **Anything else that felt off** — 👂 emoji artifacts, mishearing of the spoken policy ID,
   the agent reading out an email address (it should never read one out on the phone), odd prosody.

### 4. If something fails

**Report it, do not fix it.** This is a baseline; a recorded defect is a successful outcome. Any
failure gets written here *and* raised as a blocker in `05-06-SUMMARY.md`.

### 5. Recording the result

Replace the fence at the top of this section with `phone_check: pass` or `phone_check: fail`, add
the call date, and transcribe the eight answers verbatim below it. If the cross-sell answer to
question 3 is *"not at all"*, update `## AUTO_APPROVE_PATH`'s cross-sell block to resolve
explanation **(2)** — a genuine v11 gap — and open it as a blocker for a later voice plan.
