# 260813-nnm — Spanish on the voice channel

**Run:** 2026-08-13 · project `insurance-agent-demo-500614`, location `us`, API `https://ces.googleapis.com/v1beta`
**Target:** voice app `6e01e4a5-42a8-5213-b3da-c9053ff8ea52` · deployment `d28bbcb0-066e-4127-a894-fbf9ba39789f` (GTP)

| | |
|---|---|
| `ROLLBACK_VERSION` | **`5d9df25c-3771-45bb-bd20-b28978cc5955`** (voice v13) — read live before the PATCH |
| **New version** | **`5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc`** — *voice v14 - Spanish (es-US)* |
| **Deployment** | **REPOINTED** — `d28bbcb0` now serves v14, read back not inferred, `channelProfile` byte-identical, `updateTime 2026-08-13T22:23:57.500510Z` |
| Chat `a2f621e4` | **read-only throughout** — zero non-GET calls. Still v15 `129f8b31` |
| Fork `9ae7a0c3` | never called |
| App list | **5**, unchanged. No probe app created |
| Conversations spent | 5 sessions / 16 turns, all paced ≥55 s. No retries, no loops |

---

## THE PREMISE WAS WRONG — there is no English-only clause

The brief's hypothesis was that *"there is an English-only clause somewhere in voice's agent
instructions and the model is obeying it."* **There is not.** Every authored surface on the voice
app was searched, on both the live draft and the deployed `5d9df25c` snapshot:

| Surface | chars | language-restricting clause? |
|---|---|---|
| `globalInstruction` | 1,720 | **none** |
| agent `claims_concierge` (root) | 5,126 | **none** |
| agent `claim_intake` | 18,215 | **none** |
| agent `case_summary` | 6,447 | one, and it is deliberate — see below |
| all 9 tool descriptions/docstrings | — | **none** (zero regex hits) |
| `Default Prompt Guardrail` | 4,160 | **none** — and it lists multilingual as *safe* |
| `Default Safety Guardrail` | 752 | **none** |

The draft and the deployed snapshot are byte-identical on all four instruction bodies
(`globalInstruction` sha16 `aa2316f48ab368ac`, `claims_concierge` `8f610408ce652a6c`,
`claim_intake` `e94f64c5dd5cf7da`, `case_summary` `2f74fa346a4122dc`), so the phone was
demonstrably running the same prose that was searched.

**The only two `English` hits, both benign, quoted as short matched spans:**

1. `case_summary` @1876 — `"The packet is always written in English, whatever language the customer used or prefers"`, followed at @2083 by `"Never translate this packet."` This governs the **internal assessor packet only**, whose reader is an assessor, not the customer. It is by design and 05-09's own plan reaffirms it. **Left untouched.**
2. `claim_intake` @1079 — `"The caller speaks naturally; you translate."` This is about normalising colloquial device names (*"my lapi"*, *"my Mac"* → laptop). Nothing to do with human languages.

**The `Default Prompt Guardrail` is the platform default and is byte-shared with chat.** It lists
`"Multilingual Support: Asking for support or to respond in another language"` under **Definitions
of Safe Inputs**, not malicious ones. It was never the cause.

### Does chat carry the clause too?

**No.** Chat was checked read-only (GET only, zero mutations). Chat's `claims_concierge` (8,075),
`claim_intake` (28,215) and `globalInstruction` (1,720) have **zero** language-restricting hits.
Chat's `case_summary` carries the identical assessor-packet-is-English clause, same design intent.
**Neither channel has, or ever had, an instruction telling the agent to refuse Spanish.**

### So where did *"I am sorry, I can only speak English"* come from?

It was **emergent, not authored.** The voice app's `languageSettings` read
`{"defaultLanguageCode": "en-US"}` with **no** `supportedLanguageCodes` and **no**
`enableMultilingualSupport`, while chat had carried both since 05-03. The model, running under a
monolingual app configuration on the live-voice model `gemini-3.1-flash-live`, verbalised that
constraint in its own words and then continued with the concierge's normal identity ask. The
sentence appears nowhere in the repo or the API because nobody wrote it.

---

## THE CONTROLLED EXPERIMENT — config is not the mechanism, the instruction is

Three single-turn probes, identical Spanish first utterance
(`"Hola. Hola, quiero reportar un reclamo."`), voice DRAFT, TEXT/API channel, scored with the
05-03 marker scorer verbatim:

| # | Session | State of the app | Agent reply | `spanish_markers` | `english_stopwords` |
|---|---|---|---|---|---|
| 1 | `nnm-esbefore-a1` | **as found** — no `supportedLanguageCodes`, no `enableMultilingualSupport`, no language instruction | *"Let me just check that. Could I confirm your full name and policy ID?"* | **0** | 3 |
| 2 | `nnm-esafter-b1` | **+ `languageSettings` patched** (`es-US`, `enableMultilingualSupport: true`) and **+ `es-US` TTS entry** | *"I can help with that. Please tell me your full name and policy ID."* | **0** | 5 |
| 3 | `nnm-esafter-c1` | **+ a `Language:` block in `globalInstruction`** | *"Hola. Puedo ayudarle con eso. ¿Me podría decir su nombre completo y su número de póliza, por favor?"* | **7** | **0** |

**Probe 2 is the important one.** Applying exactly the `languageSettings` that chat already carries
live — the change the brief specified — **changed nothing.** The agent still answered a Spanish
opener in English. `enableMultilingualSupport` did precisely what its own documentation says: it
improves handling of multilingual **input**, and translates no output. 05-RESEARCH.md's Pitfall 1
is confirmed a second time, now on voice and now on a *first-utterance* Spanish opener.

**Probe 3 is what made Spanish work.** The output language is governed by the **instruction**, and
by nothing else that was reachable here.

### What was actually changed

Three surgical edits, all `apps.patch` with an `updateMask`. **No tool was created, patched or
read.** No `importApp`. No agent instruction was touched.

1. **`languageSettings`** → `{"defaultLanguageCode": "en-US", "supportedLanguageCodes": ["es-US"], "enableMultilingualSupport": true}` — mirroring chat byte-for-byte. Read back, not inferred. Kept even though probe 2 showed it is not sufficient: it is the declared language surface, and it is what makes `es-US` a legitimate language for the platform to synthesise.
2. **`audioProcessingConfig.synthesizeSpeechConfigs`** → `{"en-US": {}, "es-US": {}}`, constructed as *snapshot-plus-one-key*. `en-US` still maps to `{}` byte-identically; `bargeInConfig` byte-identical; no `dtmf`/`timeout`/`interrupt` key appeared. **Without this there is no Spanish voice for the phone to speak with**, which is why it is not optional even though the text channel cannot observe it.
3. **`globalInstruction`** 1,720 → **2,364 chars** (+644), by scripted `str.replace()` from a file, anchored on voice's own CRLF bytes (`"Absolute rules:\r\n"`, asserted to occur exactly once). **Zero existing bytes deleted** — the changed region was a single contiguous *insertion* (prefix 1,255 / suffix 465 / `old_region == 0`). All 29 original non-blank lines still present; 10 named must-survive literals present; read back byte-identical.

The block was placed **before** `Absolute rules:` deliberately, so the instruction still *ends* on
the absolute rules — honouring the o5l finding that the last paragraph of a section wins. Choosing
`globalInstruction` (app-level) over the two agent instructions was also deliberate: it governs all
three agents in one edit, and it kept the 18,215-char `claim_intake` instruction — the one whose
editing has repeatedly broken this project — completely untouched.

The inserted block, in full (it contains no secret):

```
Language:
- Answer in the same language the customer is speaking, from their very first words. If they speak Spanish, you speak Spanish. If they speak English, you speak English.
- If the customer changes language part-way through, change with them and carry on.
- Never announce the language you are using, never apologise for it, and never ask which language they would prefer. Never tell a customer you can only speak one language.
- Figures never change with the language. Amounts, excesses, limits and claim references are said exactly as a tool returned them - never translated, reworded, rounded or converted to another currency.
```

---

## WHERE SPANISH WORKS, AND EXACTLY WHERE IT REVERTS

Full Spanish auto-approve claim, session `nnm-esafter-c1`, 5 turns, voice DRAFT, TEXT/API channel.
The whole conversation was conducted in Spanish; the customer never typed an English word.

| Turn | Layer | Language | `es` / `en` | Evidence |
|---|---|---|---|---|
| 1 | greeting | **Spanish** ✅ | 7 / 0 | *"Hola. Puedo ayudarle con eso. ¿Me podría decir su nombre completo y su número de póliza…"* |
| 2 | identity + handoff to `claim_intake` | **Spanish** ✅ | 3 / 0, 4 / 0 | *"Gracias, Jordan. Un momento mientras le transfiero…"* / *"Hola, Jordan. Tengo aquí su Apple MacBook Pro de 16 pulgadas, ¿qué le ha pasado?"* |
| 3 | **the decision sentence** | **ENGLISH** ❌ | **0 / 13** | see below |
| 4 | model's own re-statement of the decision | Spanish, **model-composed** ⚠️ | 14 / 0 | see below |
| 5 | send-away line **+ cross-sell** | **Spanish** ✅ | 20 / 0 | *"…Responda a ese correo adjuntando fotos de los daños… Veo que también tiene un Apple iPhone 16 Pro Max que no está protegido. ¿Le gustaría agregarlo a su cobertura?"* |

**Spanish stops at exactly one place, and it is the place the brief predicted: the deterministic
strings.** `resolve_claim`'s `explanation` is a hardcoded English literal that the agent relays
byte-verbatim, so the single most important sentence in the demo came out in English in the middle
of a Spanish call:

```
Good news - that screen replacement comes to $840, which is under the $1,500 I can approve on the spot, so I can approve that for you right now. Your excess is $25, and your reference is CLM-24654.
```

The conversational layer is fully bilingual. The deterministic layer is monolingual. The seam is
visible to the caller.

### The undocumented consequence 05-09's plan does not anticipate

**The model then re-delivered the decision in Spanish, on the next turn, unprompted** — because the
new `Language:` block told it to answer in the caller's language and it had just failed to:

```
. La buena noticia es que el reemplazo de la pantalla cuesta 840 dólares, que está por debajo de
los 1,500 dólares que puedo aprobar en el acto, por lo que puedo aprobárselo en este momento.
Su exceso es de 25 dólares y su referencia es CLM-24654.
```

This is **two defects at once** and both belong to 05-09:

1. **The decision is delivered twice** — once in English, once in Spanish. On a phone call the
   caller hears the whole approval read out to them two times.
2. **The Spanish figures were composed by the model, not by the tool.** The values survived intact
   (840 / 1,500 / 25 / CLM-24654), but `$` was silently reformatted to `dólares` and the currency
   position moved. That is exactly the non-determinism 05-09's threat `T-05-12` exists to prevent.
   Today it is right; nothing guarantees it is right on stage. Note also the stray leading `". "`.

### Other English residue on the Spanish run

- **The customer email is English**, subject and body: `"Claim CLM-24654 - please reply with photos"`. `send_claim_email` is templated in code, so 05-09 Task 1 Step D must take its **`templated`** branch, not the cheap `agent_supplied` one. (Recorded as `email_body_source: templated` — evidenced from the tool's own `toolResponse`, which returns a fully-formed `subject` and `body` the agent never supplied.)
- **The assessor packet stays English**, correctly and by design.

**Deliberately NOT done in this task, per the brief:** no bilingual tool literals, no
`preferred_language` parameter, no `_ES_LABEL` dictionary. `resolve_claim` was never read, never
patched, and its source hash is unchanged (`898ebc7ab33d29af`, matching the value 05-03 recorded).
No Resend code was touched and no secret was read or echoed.

---

## REGRESSION CANARIES — ALL HELD IN ENGLISH

Two full English conversations on the same build that ships as v14, TEXT/API channel (per
05-06's warning that byte-comparison on the AUDIO channel fails spuriously).

### Auto-approve — `nnm-encanary-d1`, 4 turns — **21/21 PASS**

| Canary | Result |
|---|---|
| `DECISION_SPEECH_EN` byte-identical to `05-06-VOICE-BASELINE.md` (CLM normalised) | **PASS** — 197 chars, exact |
| `decision_turn_agent_chunks == 1` | **PASS** — no self-narration, no doubling |
| `spoken == explanation` byte-identical | **PASS** |
| deterministic tariff | **PASS** — 840 / 25 / 1,500 / 3,000, `DL-1`, `AUTO_APPROVE`, `relay_mismatch: false` |
| no unrendered `{placeholder}` | **PASS** |
| **the send-away line is still SPOKEN** (the v11 silent-turn shape) | **PASS** — *"An email is on its way to the address on your policy; please reply to it with photos…"* |
| exactly one customer email (**structural only**) | **PASS** — `send_claim_email` count 1 |
| **the cross-sell still fires** | **PASS** — *"By the way, I see you also have an Apple iPhone 16 Pro Max - would you like to add that to your cover?"* |
| no price/premium quoted on the cross-sell | **PASS** |
| English run stayed English (`spanish_markers == 0`) | **PASS** |

### Escalation — `nnm-enesc-e1`, 4 turns — **16/16 PASS**

| Canary | Result |
|---|---|
| `HUMAN_REVIEW`, `total_loss_flag: true`, `DL-3`+`DL-2`, $3,000 / $25 | **PASS** |
| **assessor packet files with all six sections** | **PASS** — `SUMMARY:` `ACTION:` `CLAIM:` `DIAGNOSTIC:` `RULES FIRED:` `FLAGS:` |
| **`[ASSESSOR] [VOICE]` subject** | **PASS** — `[ASSESSOR] [VOICE] CLM-24240 - Jordan Rivera`, `sent: true`, `status_code: 200` |
| packet carries the deterministic figures, no `{placeholder}`, English | **PASS** |
| `escalate_to_human` once, `escalated: true`, `context_passed: true` | **PASS** |
| the escalated send-away turn is **not silent** | **PASS** |
| one customer email, no `[ASSESSOR]` leakage to the customer | **PASS** |

### Canaries proved structurally rather than by conversation

- **Deterministic tariff / the `DIAGNOSTIC_INCOMPLETE` guard** — both live inside `resolve_claim`, whose whole-object hash is **byte-identical** to the deployed `5d9df25c` snapshot (`898ebc7ab33d29af`). All 7 pre-existing tools and all 3 agents hash identically to v13. **No `tools.patch` or `agents.patch` was issued at any point in this task.** The guard cannot have moved.
- **Barge-in and the rest of `GTP_SURFACE`** — `bargeInConfig` is `{"bargeInAwareness": true}`, byte-identical before and after; `audioProcessingConfig` minus the new `es-US` key is byte-identical to the deployed snapshot; no DTMF, timeout, interruption, SIP or `customize_response` key appeared. Asserted individually by name.
- **Audio-channel barge-in and TTS quality remain unheard** — an agent cannot dial the number.

---

## DEPLOYMENT

`POST /versions` → **`5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc`**, *"voice v14 - Spanish (es-US):
agents follow the caller's language, es-US TTS voice declared, multilingual input enabled"*.

**The repoint was gated in code** on a 41-assertion battery re-run against the **new version's own
snapshot** (not the draft), covering: the three `languageSettings` fields; the `es-US` synth entry
with `en-US` still `{}`; the four `globalInstruction` content markers and its exact 2,364 length;
`bargeInAwareness`; `audioProcessingConfig` minus `es-US` byte-equal to v13; all 7 tools and all 3
agents byte-equal to v13; 33 `variableDeclarations`; `SEQUENTIAL`; `gemini-3.1-flash-live`;
`rootAgent`/`guardrails`/`errorHandlingSettings`/`defaultChannelProfile`/`timeZoneSettings`
byte-equal to v13; and `d28bbcb0` still serving `5d9df25c` immediately before the PATCH. All passed.

Read back after the PATCH, **not inferred**:

```
appVersion  → 5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc
channelType → GOOGLE_TELEPHONY_PLATFORM
updateTime  → 2026-08-13T22:23:57.500510Z
channelProfile byte-identical: true
```

**Then proved on the deployment itself**, via `config.deployment` on `runSession`
(session `nnm-live-v14-f1`) — a Spanish opener to the *live phone build* returned:

> *"Hola. Puedo ayudarle con eso. ¿Me podría decir su nombre completo y su número de póliza, por favor?"*

**ROLLBACK: one call —** `deployments.patch` with `updateMask=appVersion` back to
`5d9df25c-3771-45bb-bd20-b28978cc5955`.

### Carried into v14 as a side effect, recorded not hidden

v13 snapshotted **7** tools; the draft carries **9**, because plan **06-02** added `record_claim`
and `lookup_claim` to the voice app and cut no version. Cutting v14 necessarily snapshots them.
**Both are referenced by no agent** — `claim_intake` wires 7 tools, `claims_concierge` wires 2,
neither includes a store tool, and all three agent objects hash identically to v13. They ship
**inert**. `record_claim` remains unwired on voice, exactly as 06-04 requires. Asserted explicitly
as gate D before the repoint.

Also noted: `predefinedVariableDeclarations` (`current_date`, `_session`) is platform-supplied and
is absent from **every** version snapshot, v13 included. Not a difference introduced here.

---

## THE 05-03 VERDICT IS CORRECTED, NOT OVERTURNED

`05-03-SPIKE-FINDINGS.md` recorded `LANGUAGE_SWITCH_VERDICT = LOCKS_AT_FIRST_UTTERANCE`. Its
**observation was correct and reproduced here**; its **attribution was wrong**, and its own name is
misleading.

- **Not confounded by a prompt clause.** The brief's worry — that an English-only instruction had contaminated the spike — is **disproven on both channels**. No such clause exists on voice or chat, and 05-03 ran against a chat app that had no language instruction either way.
- **The verdict was right that config alone does not switch the output language.** Probe 2 above reproduces exactly 05-03's result on voice, with the identical `languageSettings` applied.
- **But `LOCKS_AT_FIRST_UTTERANCE` is a wrong description.** 05-03 itself flagged (⚠ NOT VERIFIED) that a conversation *opening* in Spanish had never been tested, and that the alternative reading was "the app simply always answers in `defaultLanguageCode`." **That alternative reading is the correct one.** Probe 1 opened in Spanish, on voice, and got English — nothing "locked" at the first utterance, because the language was never a function of the utterance at all. The two-back-to-back-calls fallback 05-03 recommended **would not have worked either**, exactly as 05-03 feared.
- **The real mechanism:** output language is governed by the **instruction**. `enableMultilingualSupport` and `supportedLanguageCodes` govern input handling and declare the synthesis surface; they do not select the response language. Instructed to follow the caller, the agent follows the caller from the first word.

**Replacement token: `FOLLOWS_WHEN_INSTRUCTED`** (config-only ⇒ never switches; instruction ⇒
switches, proven on a Spanish-opening conversation on voice).

**Still genuinely untested: the mid-call EN→ES switch.** The new instruction explicitly tells the
agent to change language part-way through, but no conversation in this task exercised it — the
Spanish run was Spanish throughout and the English runs were English throughout. It is now
*plausible* where before it was *disproven*, and it should be cheap to settle with one 2-turn probe.
Do not script it as a demo beat until someone has.

---

## WHAT 05-09 STILL HAS TO DO

This task did 05-09's **Task 2 Steps A and B in full**, and the substance of **Steps C/D** at the
`globalInstruction` level instead of per-agent. What remains:

**Still required — and now the only substantive work left:**

1. **05-09 Task 1 in full — the bilingual deterministic literals.** `resolve_claim` needs the `preferred_language` parameter, the non-raising `startswith("es")` coercion, the `_ES_LABEL` dictionary (`screen replacement` → `sustitución de pantalla`, etc.) and Spanish branches for the auto-approve, human-review and decline sentences. **This is what the Spanish call visibly lacks.** Every guardrail in 05-09's plan for doing it without reading the Resend key still applies unchanged.
2. **`send_claim_email` — take the `templated` branch.** Settled by observation here: the tool returns a fully-formed `subject` and `body` the agent never supplied. Record `email_body_source: templated`. Its literals need the same treatment.
3. **NEW, not in 05-09's plan: suppress the double delivery of the decision.** Once the tool speaks Spanish, the model's compensating re-statement must stop. Add the sentinel 05-09 already drafted — *"THE TOOL SUPPLIES THE DECISION SENTENCE IN BOTH LANGUAGES - never translate it yourself…"* — to `claim_intake`, and assert `decision_turn_agent_chunks == 1` **on the Spanish run too**, not only the English one. Without this, 05-09 can ship correct Spanish figures and still have the caller hear the approval twice.
4. **05-09 Task 3's assertions 1–7**, re-run against the bilingual tool — in particular assertion 3, the byte-comparison of the Spanish `explanation` against the authored template.
5. **05-09 Task 4 — the human phone call.** Unchanged and still blocking. Only a human ear can judge whether the new `es-US` voice sounds Spanish or like an English voice reading Spanish, whether barge-in still cuts in mid-Spanish-sentence, and whether TTS mangles `$840` in Spanish the way it already does in English.

**No longer needed:** 05-09 Task 2 Step A (`languageSettings`) and Step B (`es-US` TTS) are done and
shipped. Step C/D reduce to the `preferred_language` tool-argument rule and the anti-double-delivery
sentinel — the follow-the-caller and never-announce-the-switch rules are already live app-wide.

**Re-plan note:** 05-09's `<hard_constraints>` item 1 pins `ROLLBACK_VERSION` as *"expected v11
`b17c9a26` unless 05-07 cut a later one."* It is now **v14 `5d02f14c`**, and `05-06-VOICE-BASELINE.md`'s
recorded `claim_intake` length of 14,140 is stale — it is **18,215** today. 05-09 must re-read both
live, as its own plan instructs, rather than trusting the numbers written into it.

---

## Self-Check: PASSED

- `.planning/quick/260813-nnm-.../260813-nnm-SUMMARY.md` — created
- `d28bbcb0` → `5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc` — read back from the API after the PATCH
- App list = 5, no probe app created, no `git commit` made (tree left dirty as instructed)
- Zero non-GET API calls to `a2f621e4` or `9ae7a0c3`
