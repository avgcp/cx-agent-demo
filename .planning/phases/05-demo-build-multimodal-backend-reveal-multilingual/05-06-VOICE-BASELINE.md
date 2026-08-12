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
| `## DECISION_SPEECH_EN` | `spoken_equals_explanation: **true**` — voice relays the tool's string byte-for-byte, in one chunk. **Holds on TEXT/API only** — see the 2026-08-11 audio-normalisation note in that section |
| `## AUTO_APPROVE_PATH` | tariff `840 / 25 / 1500` PASS, one `send_claim_email`, ~~cross-sell did NOT fire ❌~~ → **CORRECTED 2026-08-11: cross-sell FIRES ✅** (`103r6XO3ZXjS3S97D-AtLK1ag`); the negative was a false negative |
| `## ESCALATION_PATH` | `escalation_source: reused cae670d7-a6f1-491d-b782-921a53af6128` |
| `## PHONE_CHECK` | **DONE 2026-08-12, CORRECTED 2026-08-11** — decision line PASS, single-send PASS, barge-in **PASS**, **cross-sell FIRES ✅**, repeated diagnostic question **intermittent, not systematic** |

> **Two rows in this table were corrected after the evidence came in.** Task 1 wrote the table
> ahead of Task 2 with the plan's *expectations* in it. Both were wrong, in opposite directions:
> the decision speech turned out to be **deterministic** (better than expected), and the cross-sell
> turned out **not to fire at all** (worse than expected). The section bodies are the authority;
> this table is a convenience index.
>
> **Corrected again 2026-08-11 (task 260811-tgm).** The "cross-sell does not fire" finding was
> itself wrong — it rested on two calls that both ended *before* the beat. A third live call,
> `103r6XO3ZXjS3S97D-AtLK1ag` (`2026-08-12T02:06:37Z`, AUDIO/LIVE, v11 `b17c9a26`), shows the
> offer firing verbatim. The cross-sell is **working**; treat any remaining "gap" language in this
> file as superseded history.

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

> ### ⚠️ 05-09 MUST READ — `spoken_equals_explanation: true` holds on TEXT/API, **NOT** on AUDIO
>
> Added 2026-08-11 (task 260811-tgm). The byte-identity result above was measured on the
> **text/API channel** (`runSession`). On the **AUDIO** channel the relayed text is normalised for
> speech, and **the normalisation is not even stable between runs of the same version**:
>
> | Source | Channel | Decision amount as recorded |
> |---|---|---|
> | pinned `resolve_claim.explanation` / tool string | — | `$420` |
> | conversation `081cCNZtVwgSGqmfMpFSpbxMQ` (`2026-08-12T01:37:41Z`) | AUDIO/LIVE, v11 `b17c9a26` | `420 dollars` |
> | conversation `103r6XO3ZXjS3S97D-AtLK1ag` (`2026-08-12T02:06:37Z`) | AUDIO/LIVE, v11 `b17c9a26` | `420` |
>
> Same app, same version, same deployment, 29 minutes apart — one call spoke *"420 dollars"*, the
> other *"420"*. The `$` sigil and the `CLM-` hyphen are likewise dropped on audio
> (`CLM-24464` → `CLM 24464`).
>
> **Therefore: 05-09's Spanish byte-identity assertions must be run against the TEXT/API channel.**
> A byte-comparison performed on an AUDIO-channel transcript **will fail spuriously** and will look
> like a paraphrase/self-narration regression when nothing is wrong. If 05-09 needs an audio-channel
> check, assert on *semantic* content (the four substantive fields are present, in order), never on
> bytes.
>
> Note this does **not** weaken the underlying result: `relay_mismatch: false` on the tool response
> in `103r6XO3ZXjS3S97D-AtLK1ag`, and the audio wording tracks the tool string clause-for-clause.
> The variance is TTS-side text normalisation, not the model rewriting.

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

### Cross-sell — ~~DID NOT FIRE ❌~~ → **FIRES ✅ (corrected 2026-08-11)**

> **Read the ⛔ SUPERSEDED block at the end of this subsection before acting on anything in it.**
> The `cross_sell_fired: false` result below is a **false negative** from a conversation that ended
> before the beat. Live call `103r6XO3ZXjS3S97D-AtLK1ag` (`2026-08-12T02:06:37Z`, AUDIO/LIVE, v11
> `b17c9a26`) shows the offer firing. The observations below are kept as the record of how the
> wrong conclusion was reached.

```
cross_sell_fired: false   ← FALSE NEGATIVE, see SUPERSEDED block below
```

~~**This is the one substantive defect this baseline records on the auto-approve path.**~~
**It is not a defect.** The auto-approve path records no substantive cross-sell defect.

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

> **UPDATE 2026-08-12 — explanation (1) is now substantially weaker.** The real GTP call recorded
> in `## PHONE_CHECK` reproduced `cross_sell_fired: false` on a **live audio call**, where the
> agent *can* volunteer a turn and where the customer's reply after the email confirmation was a
> neutral `"Okay."` — the natural opening for the offer — with `uninsured_device` populated
> (`HP Pavilion 15`). No offer came. That is **two independent observations**, one of them
> immune to explanation (1)'s "a text `runSession` cannot volunteer a turn" argument.
> **Not yet proven**, because the caller did not run the exact scripted beat
> (`"That's everything, thanks"` + 10 s of silence). Current status: **probable genuine gap
> (explanation 2), two observations, one scripted retest still outstanding.**

> ### ⛔ SUPERSEDED 2026-08-11 — THE CROSS-SELL FIRES. THERE IS NO GAP.
>
> **Everything above this line about a cross-sell gap is wrong and must not be carried forward.**
> It is retained verbatim so the reversal is visible, not silently rewritten.
>
> **Evidence:** conversation **`103r6XO3ZXjS3S97D-AtLK1ag`**, voice app
> `6e01e4a5-42a8-5213-b3da-c9053ff8ea52`, `2026-08-12T02:06:37Z` → `02:08:35Z` (1m58s),
> `channelType: AUDIO`, `source: LIVE`, deployment `d28bbcb0`, `appVersion` **`b17c9a26`** (v11),
> policy `PDP100583`, 9 turns. Verified by read-only `GET` on 2026-08-11.
>
> **Turn 8 of 9, verbatim from the record:**
>
> > "Please reply to that email with photos of the damage. Once those are in, allow three to seven
> > business days for a representative to confirm everything. **Also, I see you have an HP Pavilion 15
> > that isn't covered yet – would you like to add that to your policy?**"
>
> User: `"Uh sure."` → Agent: *"Great, I'll have someone send over the options for that. Thanks for
> calling … have a good day."* → `end_session`. `uninsured_device` = `HP Pavilion 15`, same as the
> two prior calls.
>
> **Both prior negatives were false negatives.** Neither call ever reached the beat:
> - The `runSession` observation said `"No thanks"` early, which reads as refusal → `end_session`.
> - The `## PHONE_CHECK` call `081cCNZtVwgSGqmfMpFSpbxMQ` closed at its turn 9 after the caller said
>   `"¿Qué?"`, and the agent went straight to the sign-off — verified in that conversation's record.
>
> This is exactly the failure mode the runbook's own presenter note warns about
> (*"Don't say 'no thanks' early"*). **Explanation (1) — runbook/caller script, not capability — is
> the correct one after all**, and even it is only a *presenter-behaviour* caveat: with a neutral
> reply the offer arrives unprompted.
>
> **Two shape facts worth carrying into 05-07 / the runbook:**
> 1. The offer fired **appended to the send-away instructions in the same turn**, not as a turn of
>    its own. Do not script it as a separate beat or wait for one.
> 2. It fired **immediately after a barge-in** (see `## PHONE_CHECK`), i.e. interruption does not
>    suppress it.
>
> **Status: cross-sell CONFIRMED WORKING on live voice v11. No retest outstanding. Not a blocker.**

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
phone_check: DONE 2026-08-12 - PASS decision line | PASS single-send email |
             FAIL repeated diagnostic question | cross-sell did NOT fire (2nd observation) |
             Spanish did not switch (suggestive) | call length 2m12s
```

**The call happened.** A real inbound call was placed to the GTP number on `2026-08-12`. Unlike
every other capture in this file, this one is **not** a `runSession` against the draft — it is the
live phone leg, and the conversation record proves it:

| Field (from the conversation resource) | Value |
|---|---|
| conversation | `081cCNZtVwgSGqmfMpFSpbxMQ` |
| `channelType` | **`AUDIO`** |
| `source` | **`LIVE`** |
| `inputTypes` | `["INPUT_TYPE_VARIABLES", "INPUT_TYPE_AUDIO"]` |
| `deployment` | **`d28bbcb0-066e-4127-a894-fbf9ba39789f`** |
| `appVersion` | **`b17c9a26-3485-4658-9259-dfa4839a7977`** (v11) |
| `startTime` → `endTime` | `2026-08-12T01:37:41.859Z` → `2026-08-12T01:39:54.102Z` |
| `turnCount` | **9** |
| `languageCode` | `en-US` |
| policy used | **`PDP100583`** — Apple iPhone 16 Pro Max, `coverage_limit` 1500, `auto_approval_cutoff` **750**, `deductible` 25 |

**Note the policy is not the one the script called for.** The script says
`Jordan Rivera / PDP100294` (MacBook, cutoff $1,500). The caller used **PDP100583** (iPhone, cutoff
**$750**), so every figure below is re-priced against that policy and is **not** directly comparable
to `## DECISION_SPEECH_EN`'s `$840 / $1,500`. This turns out to be *more* informative, not less —
see item 1.

Tool sequence, by turn: `verify_identity` (T1) → `run_diagnostic` (T2, T3, T4) → **`resolve_claim`
(T5 — rejected)** → `run_diagnostic` (T6) → `resolve_claim` (T6) → `send_claim_email` (T7) →
`end_session` (T8).

> **A SECOND live call was placed 29 minutes later and changes three of the findings below.**
> Conversation **`103r6XO3ZXjS3S97D-AtLK1ag`**, `2026-08-12T02:06:37Z` → `02:08:35Z` (1m58s),
> `channelType: AUDIO`, `source: LIVE`, deployment `d28bbcb0`, `appVersion` **`b17c9a26`** (v11),
> policy **PDP100583**, **9 turns** — identical configuration to the call above. Recorded by task
> `260811-tgm` on 2026-08-11, read-only. It changes:
>
> | Item | Was | Now |
> |---|---|---|
> | 3 — repeated diagnostic question | systematic FAIL | **intermittent** — ran clean on unfixed v11 |
> | 4 — cross-sell | did not fire (2nd observation) | **FIRES ✅** — both negatives were false negatives |
> | open item 4 — barge-in | untested | **PASS**, with content-aware recovery (new item 8) |
>
> It also adds a new open item (**9 — possible double email**) and the audio-channel text
> normalisation warning now recorded in `## DECISION_SPEECH_EN`, which 05-09 must read.
> Items 1, 2, 5, 6 and 7 below are unaffected.

---

### 1. PASS — the decision line, correctly re-priced

Spoken at turn 6, one agent chunk:

> Good news - that screen replacement comes to 420 dollars, which is under the 750 dollars I can
> approve on the spot, so I can approve that for you right now. Your excess is 25 dollars, and your
> reference is CLM 24552.

`resolve_claim`'s own `explanation` on the same turn:

```
Good news - that screen replacement comes to $420, which is under the $750 I can approve on the spot, so I can approve that for you right now. Your excess is $25, and your reference is CLM-24552.
```

**Same template as `## DECISION_SPEECH_EN`, byte-for-byte, with only the four policy-dependent
figures changed** — `$420` for `$840`, `$750` for `$1,500`, `$25` unchanged, and the run's own
`CLM-`. `toolResponse` corroborates every one: `decision AUTO_APPROVE`, `claim_amount 420`,
`deductible 25`, `auto_approval_cutoff 750`, `coverage_limit 1500`, `rules_fired ["DL-1"]`,
`total_loss_flag false`, `claim_ref CLM-24552`.

**This is a stronger result than the baseline could give.** The baseline proved the sentence is
relayed verbatim for *one* policy; this call proves the template **re-prices correctly off a
different policy's tariff** and still relays verbatim. The model contributed none of the numbers.

**TTS normalisation is expected, not a defect.** `$420` was voiced as *"420 dollars"* and
`CLM-24552` as *"CLM 24552"*. The transcript records the *spoken* form; the tool string is
unchanged. Any future assertion must compare against `toolResponse.explanation`, **not** against
the transcript, or it will fail spuriously on the currency and hyphen forms.

### 2. PASS — single-send email, now confirmed live

`resolve_claim` (T6) returned `email_queued: true` and `email_delivery` was set to `live`.
`send_claim_email` (T7) then returned:

```
"Email already sent when the claim was decided. Customer should reply with photos attached."
```

with `sent: true`, `delivery: "live"`, `claim_ref CLM-24552`, and `send_claim_email` appearing in
**exactly one turn**. **This confirms on a real phone call what 05-04 established from historical
records and what `## AUTO_APPROVE_PATH` / `## ESCALATION_PATH` established over `runSession`:** the
customer email is sent by `resolve_claim`; `send_claim_email` only reports on it. The single-send
mechanic is not an artifact of API-driven turns.

> The mailbox itself (question 7 of the original script — how many mails actually landed at
> `akash.vinayak@nerdery.com`) was **not reported back**. The record proves one send *call* and one
> `delivery: "live"`; it cannot prove one *delivery*. That check remains open.

### 3. FAIL — the agent re-asked a question the customer had already answered

**This is the phone check's one failure.** At turn 4 the agent asked *"Aside from the crack, does it
still switch on and respond normally?"*. At turn 5 the customer answered **"It does."** — and the
agent then called **`resolve_claim`** instead of `run_diagnostic`, guessing `diagnostic_outcome:
"REPAIRABLE"` on its own.

`resolve_claim` refused it, correctly:

```
{"error": "DIAGNOSTIC_INCOMPLETE",
 "message": "The diagnostic has not reached a terminal outcome yet. Keep calling run_diagnostic until it returns one."}
```

**The guard did its job — it is working hardening and must not be weakened.** The defect is
entirely in what the agent did *next*: rather than calling `run_diagnostic` with the answer it had
just been given, it re-interrogated the customer —

> *"Sorry, I just need to be sure—apart from the crack, does it still turn on and work as it
> should?"*

— to which the (audibly resigned) reply was *"Ah it does work as it should"*. Only then did
`run_diagnostic` run with `q3=works_normally`, return `terminal REPAIRABLE`, and the claim resolve.

**Cost:** one wasted customer turn and ~11 s of a 132 s call — about 8% of the demo, spent making
the agent look like it was not listening, immediately before the moment it is meant to look
brilliant.

**Fixed in the DRAFT on 2026-08-12 (quick task `260811-suy`), not yet in any version.** The fix is
in the recovery path, not the guard: on `DIAGNOSTIC_INCOMPLETE` the agent is now instructed to call
`run_diagnostic` immediately, carrying every answer it already has, and never to re-ask a question
the caller has already answered in order to recover. `claim_intake` 14,140 → **14,647** chars
(+507), one contiguous region. **The live phone number still exhibits this defect** until a version
is cut and `d28bbcb0` is repointed — see `260811-suy-SUMMARY.md`.

> **NUANCE ADDED 2026-08-11 — the defect is INTERMITTENT, not systematic.** Conversation
> `103r6XO3ZXjS3S97D-AtLK1ag` (`2026-08-12T02:06:37Z`, AUDIO/LIVE) ran on live **v11 `b17c9a26`**,
> which does **not** carry the `260811-suy` draft fix — and the diagnostic completed cleanly:
> `q2` supplied at T4 → tool asks `q1` → `q1=screen` at T5 → tool asks `q3` → `q3=works_normally`
> at T6 → `terminal REPAIRABLE`, `questions_asked: 3`, and `repeat_of_previous_question: false` on
> every response. **No repeated question, no `DIAGNOSTIC_INCOMPLETE`, no wasted turn.**
>
> Two consequences, both load-bearing:
> 1. **Demo risk is lower than `260811-suy` implied.** The failure depends on whether the model
>    jumps to `resolve_claim` before the diagnostic reaches terminal — it happens on some runs and
>    not others on the *same* version. It is a coin-flip, not a certainty.
> 2. **The draft fix cannot be validated by absence of the symptom.** A clean post-fix call proves
>    nothing, because a clean call already occurs pre-fix. Any validation of `260811-suy` must
>    assert on the *instruction text / recovery behaviour under an induced
>    `DIAGNOSTIC_INCOMPLETE`*, not on "we ran it and it didn't repeat."

### 4. Cross-sell did NOT fire — second independent observation

```
cross_sell_fired: false   (2nd independent observation; 1st was ## AUTO_APPROVE_PATH)
```

- `uninsured_device` **was populated** — `HP Pavilion 15`, written by `verify_identity` at turn 1.
- The claim **auto-approved**, so DL-4's stated precondition was met.
- The customer's reply after the email confirmation was a neutral **"Okay."** — the natural
  opening, and *not* the refusal (`"No thanks"`) that explanation (1) blamed last time.
- Regex over every agent chunk for
  `iphone | 16 pro | uninsured | add it to your cover | also insure | another device | bundle | pavilion`
  returned exactly **one** hit, and it is the intake line *"I've got your Apple iPhone 16 Pro Max
  here, what's happened to it?"* — the **covered** device, not an offer.

**This materially weakens explanation (1)** in `## AUTO_APPROVE_PATH`. That explanation rested on
the claim that a text `runSession` cannot let the agent volunteer an unprompted turn. This was a
live audio call, where it can, with a neutral customer turn to hang the offer off — and no offer
came.

**It is not yet proven, and must not be written up as a proven defect.** The caller did not run the
exact scripted beat: they said *"Okay."*, not *"That's everything, thanks"*, and did not hold ~10 s
of silence afterwards. **Status: probable genuine gap (explanation 2) — two independent
observations, one scripted retest still outstanding.** The retest is unchanged and still worth 3
minutes: reach the email-confirmation turn, say *"That's everything, thanks"*, then **say nothing
for 10 seconds**.

> #### ⛔ SUPERSEDED 2026-08-11 — the retest happened. **THE CROSS-SELL FIRES.**
>
> Item 4 above is **wrong**. It is left intact so the reversal is auditable, but nothing in it may
> be carried into 05-07, 05-09 or the runbook. `cross_sell_fired: **true**`.
>
> **Evidence — conversation `103r6XO3ZXjS3S97D-AtLK1ag`**, same app
> `6e01e4a5-42a8-5213-b3da-c9053ff8ea52`, same deployment `d28bbcb0`, same version **v11
> `b17c9a26`**, same policy **PDP100583**, `channelType: AUDIO`, `source: LIVE`,
> `2026-08-12T02:06:37Z` → `02:08:35Z` (**1m58s**), 9 turns — i.e. a call placed **29 minutes after**
> `081cCNZtVwgSGqmfMpFSpbxMQ`, on identical configuration. Verified by read-only `GET` 2026-08-11.
>
> **Turn 8 of 9, agent, verbatim:**
>
> > "Please reply to that email with photos of the damage. Once those are in, allow three to seven
> > business days for a representative to confirm everything. **Also, I see you have an HP Pavilion 15
> > that isn't covered yet – would you like to add that to your policy?**"
>
> Turn 9 — user `"Uh sure."` → agent *"Great, I'll have someone send over the options for that.
> Thanks for calling … have a good day."* → `end_session`. `uninsured_device` = `HP Pavilion 15`,
> exactly as in the call above.
>
> **Both prior negatives were false negatives — neither conversation reached the beat.**
> Re-reading `081cCNZtVwgSGqmfMpFSpbxMQ`'s own record: its turn 8 carried the send-away line, the
> customer said **"¿Qué?"** at turn 9, and the agent went straight to the sign-off + `end_session`.
> The call ended *before* the offer's slot, it did not skip it. The `runSession` observation in
> `## AUTO_APPROVE_PATH` failed the same way via an early `"No thanks"`. This is precisely the
> failure the runbook's presenter note warns about: *"Don't say 'no thanks' early."*
>
> **Explanation (1) was right all along**, and even then only as presenter guidance — the offer
> arrives unprompted off a neutral reply, with no 10 s of silence required. **Explanation (2), the
> "v11 capability gap", is disproven on v11 itself.**
>
> **Two shape facts for the runbook and 05-07:**
> 1. The offer is **appended to the send-away instructions inside the same agent turn** — it is not
>    a turn of its own. Do not script a pause waiting for a separate beat; the runbook's "three
>    separate turns" phrasing is inaccurate on this point.
> 2. It fired **on the turn immediately following a barge-in** (item 8 below), so interruption does
>    not suppress it.
>
> **No retest outstanding. Not a blocker. Cross-sell is CONFIRMED WORKING on live voice v11.**

### 5. Spanish — did not switch (suggestive, not conclusive)

At turn 8 the customer said **"¿Qué?"**. The agent replied in English —
*"Thanks for calling, &lt;name&gt; - have a good day."* — and called `end_session`.
`Conversation.languageCode` stayed **`en-US`** for the whole call.

Consistent with `LOCKS_AT_FIRST_UTTERANCE`, **but weak evidence**, and it should not be cited as
confirmation:

- the input was two syllables, likely at the edge of what language ID can act on;
- it arrived **during the close sequence**, after the claim was resolved, where the agent is
  instructed to wrap up — a language switch may simply have been out-competed by the close;
- `end_session` fired in the same turn, so there was no subsequent turn in which a switch could
  have shown itself.

**05-09 must not treat this as a settled negative.** A real Spanish test needs a full Spanish
sentence, early, mid-claim.

### 6. Call length — 2m12s

`132.2 s` wall clock (`01:37:41.859Z` → `01:39:54.102Z`), 9 turns. **A useful demo-timing data
point:** the whole auto-approve story — greet, authenticate, hand off, three diagnostic questions,
price, approve, email, close — fits in **about two minutes**, *including* the ~11 s wasted by the
item-3 defect. Per-turn span durations (these include the caller's own speaking time, so they are an
upper bound on dead air, not a measure of it): T1 25.5 s (authentication), T2 9.6 s, T3 6.9 s,
T4 6.8 s, T5 7.8 s (the wasted re-ask), **T6 12.3 s (the decision turn — two tool calls)**,
T7 11.9 s, T8 5.6 s.

The decision turn is the longest agent-side beat and is the one a presenter should be ready to talk
over. Nothing approached the ">8 s of dead air is new" threshold once caller speech is discounted.

### 7. The email asks the customer to reply with photos — and the runbook does not say so

`send_claim_email`'s result carries subject **`Claim CLM-24552 - please reply with photos`** and a
body that instructs, in capitals, *"Please REPLY TO THIS EMAIL attaching clear photos of the
damage - ideally one showing the whole device and one close-up of the damaged area"*, then
*"allow 3-7 business days"*. The agent voiced the same thing at turn 7.

**This is the voice channel compensating for having no photo-upload path** — chat takes the photo
in-session and assesses it live; voice defers it to email. It is a genuinely good answer to the
obvious *"but how do you see the damage over the phone?"* question a customer will ask in the room.

**It is not in `DEMO-RUNBOOK.md`, and it should be.** Two lines are enough: (a) tell the presenter
this beat exists so they can land it deliberately rather than have it arrive as a surprise, and
(b) flag the contrast with chat's live photo assessment, which is the natural segue between the two
channels. Filed as a runbook gap, not an agent defect.

---

### Not answered by this call — still open

The original script asked eight things. The conversation record can only answer what it contains;
these four were **not reported back** and remain unverified:

| # | Open item | Why the record cannot settle it |
|---|---|---|
| 1 | **The phone number**, and the `☎ ____` blank in `DEMO-RUNBOOK.md` | not API-retrievable; `telephony-caller-id` in the record is the *caller*, and is redacted |
| ~~4~~ | ~~**Barge-in** — does talking over it interrupt?~~ | **CLOSED 2026-08-11 — PASS.** See item 8 below |
| 6 | **The close** — were the literal `"` quote marks voiced? | this call closed on *"Thanks for calling, &lt;name&gt; - have a good day."*, which carries **no** quote marks — a **different sentence** from the `"Understood. Have a good day."` recorded in `## DECISION_SPEECH_EN`. BLOCKER 2 is therefore **untested**, not cleared |
| 7 | **The mailbox** — how many mails actually landed | the record proves one send call, not one delivery. **Sharpened 2026-08-11 — see item 9** |

Also unexercised: the **escalation** path (this was auto-approve) and any **photo** beat (voice has
none).

---

### 8. PASS — barge-in CONFIRMED WORKING (added 2026-08-11, closes open item 4)

Open item 4 above is **closed**. Evidence: conversation **`103r6XO3ZXjS3S97D-AtLK1ag`**
(`2026-08-12T02:06:37Z` → `02:08:35Z`, AUDIO/LIVE, deployment `d28bbcb0`, v11 `b17c9a26`).

The caller talked over the agent mid-sentence at turn 7. Turn 8's user message carries the
platform's own interruption marker, verbatim from the record:

```
<context>agent speaking was interrupted. user only heard 'Since we've' in the last agent response.</context>
```

Turn 7's agent text was *"Since we've approved that, an email is already on its way to the address
we have on…"* — cut off exactly where the marker says.

Two things are proven, and the second is the interesting one:

1. **Barge-in works** — the interruption was registered, not swallowed. `bargeInConfig` in
   `## GTP_SURFACE` is live and effective.
2. **The agent knows *how much* the caller heard** (`'Since we've'`) and **recovered by
   re-delivering the missed content** — turn 8 restates the reply-with-photos instruction and the
   3–7 business-day expectation before appending the cross-sell. This is materially better than
   "the interruption was detected"; the agent repaired the conversation without the caller having
   to ask.

**Demo implication:** interrupting the agent is safe and is itself a credible wow beat. Nothing is
lost when a presenter cuts in.

### 9. OPEN — `send_claim_email` was called twice; inbox confirmation still needed

In `103r6XO3ZXjS3S97D-AtLK1ag`, `send_claim_email(policy_id=PDP100583)` was called on **two**
turns — 7 and 8 — where every prior record (v1 → v11, item 2 above) shows single-send as an
invariant.

**The record does not show a defect, and none may be asserted from it.** Both calls returned
**identical** payloads:

| Turn | `sent` | `delivery` | `message` |
|---|---|---|---|
| 7 | `true` | `live` | `Email already sent when the claim was decided. Customer should reply with photos attached.` |
| 8 | `true` | `live` | *(same string)* |

`resolve_claim` at turn 6 had already returned `email_queued: true` — so the email was sent at
decision time, and **both** `send_claim_email` calls are idempotent reports of that one send, not
sends. That is the invariant holding, not breaking.

**Why it stays open anyway:** `sent: true` is ambiguous on its face — it reads the same whether the
tool sent or merely reported a prior send — and the tool's source cannot be read to settle it
(it holds a live Resend key). **The only thing that closes this is an inbox check: confirm exactly
ONE email for `CLM-24464` landed at the policy address.** Until then, treat it as an open question,
not a defect. It also folds in open item 7 above, which asks the same question with less evidence.
