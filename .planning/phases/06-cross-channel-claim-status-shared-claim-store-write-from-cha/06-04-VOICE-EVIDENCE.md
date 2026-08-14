---
phase: 06
plan: 04
artifact: VOICE-EVIDENCE
date: 2026-08-14
status: live
purpose: >
  The voice-side evidence 06-05 re-checks, plus the literal phone script the human check runs.
  Section headings are literal and greppable. Do not rename them.
---

# 06-04 — Voice evidence

## VOICE_BASELINE_CAPTURE

Captured by `GET` only at plan start, `2026-08-14`, **before any write**. Every figure here was read
live from `https://ces.googleapis.com/v1beta`. **No figure was copied from a planning document** —
the plan's own table, `05-06-VOICE-BASELINE.md`, `06-02-STORE-CONTRACT.md` and `06-03-SUMMARY.md`
were all stale by between one and six version cuts by the time this plan ran.

### The rollback target, read live

| Thing | Value read at plan start |
|---|---|
| Deployment | `d28bbcb0-066e-4127-a894-fbf9ba39789f` — *voice - meridian demo* |
| **`appVersion` it serves — THE ROLLBACK TARGET** | **v17 `dcc20863-3746-4e43-a2c9-ed30e0611479`** |
| `updateTime` | `2026-08-14T03:15:07.395898Z` |
| `channelProfile.channelType` | `GOOGLE_TELEPHONY_PLATFORM` |
| `channelProfile` sha256 | `e7197eb367fa9cf0…` (this executor's serialiser, `sort_keys` + `(",",":")`) |
| Deployment resource keys | `appVersion`, `channelProfile`, `createTime`, `displayName`, `name`, `updateTime` |

> The plan's environment table named voice v13 `5d9df25c` as the live version. It has not been live
> since `260813-nnm`. Six cuts intervened (v14 `5d02f14c`, v15 `17b2e438` never deployed, v16
> `09a1f14d`, v17 `dcc20863`). **`dcc20863` is the rollback target for this plan** and it was
> re-read live again immediately before the repoint — see `## VOICE_SHIP`.

### Voice app `6e01e4a5-42a8-5213-b3da-c9053ff8ea52`

| Field | Value |
|---|---|
| `displayName` | `Meridian Claim - Voice (demo-ready)` |
| `modelSettings.model` | `gemini-3.1-flash-live` |
| `toolExecutionMode` | `SEQUENTIAL` |
| `variableDeclarations` count | **33** |
| `audioProcessingConfig` | `{"synthesizeSpeechConfigs":{"en-US":{},"es-US":{}},"bargeInConfig":{"bargeInAwareness":true}}` |
| `languageSettings` | `{"defaultLanguageCode":"en-US","supportedLanguageCodes":["es-US"],"enableMultilingualSupport":true}` |
| `globalInstruction` | 2,859 chars |

`audioProcessingConfig` now carries an `es-US` key (added by `260813-nnm`), so the `{"en-US": {}}`
figure quoted in `05-06-VOICE-BASELINE.md ## GTP_SURFACE` is **stale**. The byte-equality canary in
this plan is anchored on the value read above, not on the Phase 5 text.

### Agents — instruction lengths and SHA-256, read live

| Agent | Root? | chars | `instruction` SHA-256 | line endings | tools | children |
|---|---|---|---|---|---|---|
| `claims_concierge` | **yes** | **6,642** | `be2fdeb31854ac06…` | 69 CRLF, **0 bare LF** | 2 (`end_session`, `verify_identity`) | 1 (`claim_intake`) |
| `claim_intake` | no | **20,711** | `f52491844b53204c…` | 256 CRLF, 7 bare LF | 7 | 0 |
| `case_summary` | no | **6,447** | `2f74fa346a4122dc…` | 0 CRLF, 65 LF | 0 | 0 |

Neither agent carries a `transferRules` key — routing is instruction-driven, exactly as 05-06
recorded. `claims_concierge` carries `beforeModelCallbacks[0].pythonCode`, **869 chars**, the
`260813-ui0` build. **`claim_intake` is not touched by this plan** and is asserted byte-identical to
the row above at every gate.

### Tools — 9 python tool resources, whole-object SHA-256 with volatile fields stripped

`escalate_to_human`, `generate_case_summary`, **`lookup_claim`**, `record_claim`, `resolve_claim`,
`run_diagnostic`, `send_case_record_email`, `send_claim_email`, `verify_identity`.

All nine `SYNCHRONOUS`, all `pythonFunction`. **`resolve_claim`'s source was never read, echoed,
printed or persisted by this plan** — it is asserted byte-unchanged by whole-object hash only.
`lookup_claim` existed on voice but was **attached to no agent** at plan start; `record_claim`
remains attached to no agent (the voice **write** is out of scope here — see below).

### Read-only control on the other channel

| Thing | Value at plan start |
|---|---|
| Chat deployment `d7bfbb93-8cee-43fe-9095-bc5775f353bd` | v18 **`d0e4bfef-6d5f-43b3-b490-3d9036d030e2`**, `updateTime 2026-08-14T03:15:20.124453Z`, `WEB_UI` |
| Chat agents | `claims_concierge` 9,591 · `claim_intake` 32,498 · `case_summary` 6,977 |
| Chat tools | 13 |
| App list | **five** |
| Fork `9ae7a0c3` | **zero calls of any method** |

### THE VOICE WRITE IS OUT OF SCOPE FOR THIS PLAN

`record_claim` is **not** wired into voice `claim_intake` by 06-04, deliberately and per the plan.
A claim filed on the phone is therefore still **not** written to the shared store. The read path
lives entirely on `claims_concierge`; the write would live in `claim_intake`, the instruction whose
editing has repeatedly broken this project. **06-05 owns the voice write.** Until it lands, the
cross-channel loop is closed in one direction only: chat writes, voice reads.

## VOICE_IDENTIFICATION

**Criterion 4 is the design problem of this plan, and it is a speech problem, not a lookup problem.**
Speech-to-text mangles alphanumerics. On the live call recorded in `260813-olv` a single policy ID
took **five turns and three `verify_identity` attempts** (`PGP` → `PDP` → `1000294` → `TDP100294`).
A claim reference is worse: `CLM-24841` has no phonetic redundancy at all. A demo that fails while
the customer is still identifying themselves has failed before it starts.

### How the caller is identified

**By name and policy ID, through the `verify_identity` step voice already has.** Nothing new was
built for identification and no new authentication surface exists.

- `verify_identity(first_name, last_name, policy_id)` — the arg names observed live in 05-06 — must
  have returned `authenticated = true` **before** any lookup. The instruction states this as a
  precondition, and the live battery asserts the ordering from the conversation record: a
  `lookup_claim` preceding `verify_identity` is a **failure**, not a warning.
- The control is structural as well as instructional. `lookup_claim` reads
  `context.state['policy_id']` and, whenever it is present, **ignores the model-supplied `policy_id`
  argument entirely**, reporting `policy_source: "session"` and `policy_id_arg_ignored: true`.
  Re-proven on the **wired** voice tool by invocation: seeded with session `PDP-100583` and the
  argument `policy_id="PDP-100294"`, it returned `PDP-100583`'s claim. The property survived wiring.
- 05-06 recorded that `verify_identity` **already accepted the spoken-digit form**
  `P D P one zero zero two nine four` without a re-prompt. That single observation is why criterion
  4's identification path is policy-ID-based rather than reference-based.

### The caller is NEVER asked to read a claim reference aloud

The instruction carries an explicit prohibition — *"NEVER make the caller spell a claim reference
back to you, and never invite one"* — with the reason stated (spoken letters and digits are the
least reliable thing on a phone line) so the model does not treat it as an arbitrary rule.

- A reference is **not needed**: `lookup_claim` takes the policy from the verified session, so the
  agent calls it with no policy of its own.
- A **volunteered** reference is passed as `claim_ref`. It is a bonus path, never a requirement, and
  the instruction states that its absence never stops the lookup.
- A description — a device, a date, *"the older one"* — is passed as `selector`.

Proven by grep over the new region before the write: zero hits for `read.*reference`,
`claim number.*aloud`, `what is your claim ref`, `could i take your claim`, and
`ask…for…claim (reference|number)`; and **every one of the two literal `claim reference` mentions
sits inside a sentence carrying an explicit negation** (a stricter check than the plan's, because a
prohibition and a request are both string matches and only the negation check tells them apart).

### How several claims disambiguate — read the most recent back, never enumerate

When `match_count > 1` the tool returns `disambiguation_line`, which names the most recent claim and
asks whether that is the one. The agent relays that line **verbatim, once**, and waits.

**It does not speak `alternatives`.** The instruction says so in terms: *"NEVER speak the
alternatives list - do not say the other claims, their devices or their dates out loud."* On a phone
call a spoken list of devices and dates is dead air and forces the caller to hold three items in
their head; a read-back-and-confirm is one turn, and the most recent claim is overwhelmingly the one
being asked about. `alternatives` exists so the **second** call can resolve, not to be said.

Two answers are possible and **both** have a deterministic path back to a `status_line`:

| Caller says | Agent does |
|---|---|
| *"Yes, that's the one"* | calls `lookup_claim` again with **`claim_ref` set to the reference the disambiguation line itself just gave it** — never asking the caller to repeat it |
| *"No, the older one"* / names a device or a date | calls `lookup_claim` again with **`selector`** set to what they said |

> **[Rule 2 — missing critical functionality, found before any conversation was spent.]** The
> confirm path did not exist in 06-03's chat region, which says only *"call `lookup_claim` again with
> `selector` set to what they said"*. `selector` resolution has no branch for *"yes"* — it would fall
> through to `NO_MATCH` and the agent would tell a caller there is nothing on file **immediately
> after reading their claim back to them**, which is the worst failure this branch can produce. The
> fix routes the confirmation through the reference **the tool supplied**, so criterion 4 still holds
> — the caller never speaks it. This is a latent defect in the shipped chat build too and 06-05
> should carry it across.

### A policy with nothing on file, and mismatches

When `found` is false the agent says the tool's line and stops — no guess, no figure, no speculation
about why, no restating of any claim detail. Then, and only then, it offers to take a new claim,
which routes to `claim_intake` through the existing path, unchanged.

A reference belonging to **another** policy is treated **identically** to one that does not exist.
That is structural, not textual: `lookup_claim` never reads `claims/by-ref/` at all — it performs one
`GET` of the authenticated policy's object and resolves every reference inside that array, so another
customer's claim is not hidden from the response, it is **never fetched**. Re-proven by invocation on
the wired tool: `CLM-60203` (a real claim on `PDP-100583`) queried against session `PDP-100294`
returned a response **byte-identical** to the wholly invented `CLM-99999` — same five keys, same
literal, `canon(a) == canon(b)` **true**.

### How intake and status are told apart

Stated in the instruction with **literal trigger families**, and the ambiguous case is named:

| Opening | Route |
|---|---|
| *"I dropped it"*, *"I spilled something on it"*, *"the screen is cracked"*, *"I need to file a claim"*, *"I want to make a claim"* | **`claim_intake`, exactly as today.** The instruction says the existing routing is untouched, and no byte of the existing routing sentence was changed. |
| *"I'm checking on my claim"*, *"chasing up my claim"*, *"any update"*, *"what's happening with my claim"*, *"did it go through"*, *"the status of my claim"* | the status branch |
| *"I'm calling about my claim"*, *"it's about a claim"* | **ONE clarifying question** — is this something new that has just happened, or a claim already put in — then route on the answer. **Never on a guess.** Guessing wrong sends a caller with a fresh loss into a look-up that finds nothing. |

### Speak before you resolve — the dead-air rule

The agent emits a short acknowledgment **before** it calls `lookup_claim` in that turn — one clause,
no figures, no reference, no promise about the outcome, and nothing about systems. The instruction
states the intra-turn order explicitly (*"your line first, the tool second, the tool's line third"*)
rather than leaving it implicit, because the v11 `838b6d2b` regression came from implicit ordering.
The example given is a person's line (*"Of course, let me see where that has got to."*), not a
holding phrase about systems.

Silence posture is phrased as *"say nothing **ADDITIONAL** about how you found it"* — deliberately
**not** as *"the only thing they hear is X"*, which is the phrasing that became a gag order and
silenced an entire turn on chat v11.

### The edit

| | |
|---|---|
| Agent | voice `claims_concierge` (root) — `claim_intake` untouched |
| Instruction | 6,642 → **11,658** (**+5,016**) |
| Contiguous regions changed | **1** — common prefix 6,072, common suffix 570, old span **0**, new span 5,016 |
| Anchor | `</subtask>\r\n    <subtask name="Anything else">`, sliced from voice's **own CRLF bytes**, asserted **unique** (1 occurrence) |
| Method | scripted `str.replace()` with the region read **from a file** in LF and converted to CRLF in-process; instruction handled through JSON only, never through a text-mode round trip |
| Read-back | **byte-identical to the string the script intended to write**, sha `b0553997ae8b343f…` |
| Tools | `lookup_claim` attached; tool count 2 → **3**; `updateMask=instruction,tools` |
| `beforeModelCallbacks` | **SURVIVED byte-identical** (869 chars, sha unchanged) — asserted, because this is exactly what a careless `updateMask` wipes |
| Agent key set | unchanged, 11 keys; `afterModelCallbacks` and `childAgents` byte-identical; `transferRules` still absent |

**Pre-write battery 24/24, post-write 8/8, isolation 23/23.**

> **Deviation, recorded rather than hidden: the delta bound was re-declared once, before running.**
> The plan predicted *"roughly +900 to +2,600"*. That band predates 06-03, whose equivalent chat
> region alone is **2,949** characters — the plan's upper bound was already below the size of the
> thing it was asking to be mirrored. The first pass declared 3,600–4,800 and measured 4,740; the
> confirm-path fix above then required +276, so the band was re-declared **3,600–5,200 before the
> second run** and measured 5,016. Both passes were re-derived from the **plan-start bytes**, so
> "exactly one contiguous region differs from the baseline" is true of the shipped instruction, not
> merely of each increment. The bound's purpose (T-06-13 — catch a runaway edit) is served by the
> band plus the contiguity measurement plus the byte-identical survival of the `<role>`,
> `<constraints>` and `Verify the caller` slabs, all of which passed.

## VOICE_STATUS_LOOKUP

Session `v04-status-c`, voice **DRAFT** (`runSession`, no `config.deployment`), TEXT/API channel, on
the exact bytes that became the shipped version. Every figure below is read from a
`toolCall`/`toolResponse` object. **Byte assertions run on TEXT/API only** — on AUDIO the platform
normalises `$420.00` and drops the `CLM-` hyphen, unstably between runs of the same version, so an
audio byte comparison fails spuriously (`05-06-VOICE-BASELINE.md ## DECISION_SPEECH_EN`).

### The beat, four turns

| Turn | Caller | What fired | What was said |
|---|---|---|---|
| 0 | *"Hi, this is Jordan Rivera, policy P D P one zero zero two nine four, and I'm just checking on a claim I put in."* | `verify_identity` → `lookup_claim{policy_id:"PDP100294"}` → `match_count: 25` | *"Thanks, Jordan. **Of course, let me see where that has got to.** I can see 25 claims on this policy. The most recent is CLM-24690, your Apple MacBook Pro 16" - keyboard replacement, filed on 14 August 2026. Is that the one you mean?"* |
| 1 | *"Yes, that's the one."* | `lookup_claim{claim_ref:"CLM-24690"}` → `match_count: 1` | the `status_line`, **byte-identical, alone in its chunk** |
| 2 | *"Could you also check C L M six zero two zero three for me?"* | `lookup_claim{claim_ref:"CLM-60203"}` → `found: false` | the not-found line, verbatim |
| 3 | *"Alright, what about C L M nine nine nine nine nine?"* | `lookup_claim{claim_ref:"CLM-99999"}` → `found: false` | the not-found line, verbatim |

### CRITERION 2 — proven CROSS-CHANNEL with a chat-written claim, not a fixture

The sentence spoken on the phone at turn 1, byte-identical to `lookup_claim.status_line`:

```
Claim CLM-24690 on policy PDP100294: your Apple MacBook Pro 16", keyboard replacement. It was
approved for $420.00, less your $25.00 excess, so $395.00 comes to you. Filed on 14 August 2026
via chat.
```

**`CLM-24690` was written by CHAT.** `channel: "CHAT"` in the `toolResponse`, and the tool's own
Python composition renders that as *"via chat"* in the sentence the caller hears. It was filed by
chat's `record_claim` on 14 August 2026 during the `260813-ui0` chat runs, into
`gs://meridian-claim-store-500614/claims/by-policy/PDP100294.json`, and read back on the **voice**
channel by an agent that has never had `record_claim` wired. **This is a real cross-channel proof,
not the 06-02 fixture path** — the fixtures were available and were not needed for criterion 2.

| Assertion | Result |
|---|---|
| S9 `status_line` relayed **byte-identically**, in exactly one chunk, exactly once | **PASS** |
| S11 exactly **one** agent chunk on that turn carries a currency figure or a `CLM-` string, and it **is** the relay | **PASS** |
| S12 nothing outside the relayed sentence carries a figure — the remainder of the chunk is the empty string | **PASS** |
| S13 `channel: CHAT`, *"via chat"* present in the spoken sentence | **PASS** |
| S6 `disambiguation_line` relayed **byte-identically**, exactly once | **PASS** |

**The model composed no figure at any point.** The amount, the excess, the net, the device, the
issue, the date and the reference all arrive inside one Python-composed string.

### CRITERION 4 — the caller was never asked for a claim reference

| Assertion | Result |
|---|---|
| S4 zero request-shaped mentions of a claim reference or claim number across **every** agent chunk in three status conversations | **PASS** |
| S1 `verify_identity` fired **strictly before** `lookup_claim` on the status turn | **PASS** |
| S2 `verify_identity` returned `authenticated: true` before any lookup | **PASS** |
| S3 every `lookup_claim` reported `policy_source: "session"` and `policy_id_arg_ignored: true` | **PASS** |
| S3b the model **did** supply `policy_id: "PDP100294"` and the tool **ignored it** | **PASS** — T-06-03's structural control survived wiring, proven live rather than assumed |
| S5 several-claims branch reached, `match_count: 25` | **PASS** |
| S7 **zero** of the 24 `alternatives` were spoken — no other claim reference, device or date appears in any agent chunk on that turn | **PASS** |
| S8 the confirm path passed **the reference the tool itself supplied** as `claim_ref`, with no `selector`, so the caller never spoke it | **PASS** |
| S10 `match_count` collapsed to **1** on the confirm call | **PASS** |

### The ambiguous opening gets exactly one question — session `v04-status-a`

| Turn | Caller | Agent |
|---|---|---|
| 0 | *"Hi, I'm calling about my claim."* | *"I can help with that. **Is this something new that has just happened, or a claim you have already put in?**"* — one question, no tool call, no guess |
| 1 | *"It's one I already put in - I'm just checking where it's got to."* | *"Of course, let me see where that has got to. Could I take your full name and your policy ID?"* |

### A caller who cannot be authenticated reaches nothing — session `v04-fail-b`

An unintended but valuable control. The script gave *Jordan Rivera* against `PDP100583`, which
belongs to **Maria Santos**. `verify_identity` returned
`{"authenticated": false, "reason": "NAME_MISMATCH", "attempts": 1}` and **`lookup_claim` was never
called on any of the four turns**, including two on which the caller volunteered claim references.
The agent re-asked for the name and disclosed nothing — no device, no policy detail, no claim.
**T-06-03's instruction gate holds under a caller actively pushing references at it.**

## VOICE_GRACEFUL_FAILURE

Three negative paths, all proven, all producing the tool's own literal and **zero figures of any
kind**.

| Path | How proven | `found` | `lookup_status_code` | Line spoken |
|---|---|---|---|---|
| **Reference belonging to ANOTHER policy** — `CLM-60203`, a real claim on `PDP-100583`, asked for on `PDP100294` | live, `v04-status-c` T2 | `false` | `200` | *"I can't find a claim with that reference on this policy, so there is nothing for me to read back."* |
| **Unknown reference** — `CLM-99999` | live, `v04-status-c` T3 | `false` | `200` | the **same** line |
| **Policy with nothing on file** — `PDP-100746` | `apps.executeTool` on the shipped tool (no conversation) | `false` | `404` | *"There are no claims on file for this policy at the moment."* |

**The empty-policy case is stated plainly as proven by invocation, not by conversation.** It could
not be driven live because `verify_identity` has no synthetic identity for `PDP-100746`, so a caller
can never authenticate onto it — which is itself the correct behaviour and is recorded rather than
worked around.

### Assertions

| Assertion | Result |
|---|---|
| S14 cross-policy response carries exactly the five negative-path keys — `found`, `lookup_status_code`, `match_count`, `policy_source`, `status_line` — and **zero claim fields** | **PASS** |
| S15 unknown-reference response carries the **same five keys** | **PASS** |
| **S16 T-06-03: the two responses are BYTE-IDENTICAL** — `canon(cross) == canon(unknown)` | **PASS** |
| S17 the not-found line relayed verbatim, exactly once, on both paths | **PASS** |
| S18 **zero currency figures and zero `CLM-` strings in ANY agent chunk** on either turn, asserted by regex over every chunk rather than by inspection | **PASS** |
| S19 empty policy returns the `EMPTY` literal, the same five keys, `404` | **PASS** |

**Nothing was fabricated on any negative path.** There is nothing to fabricate *from*: a not-found
response contains no claim field at all, so the model has no figure in context to hallucinate with.
A caller probing for the difference between "not yours" and "does not exist" gets byte-identical
responses and cannot tell them apart.

### T-06-03R — the residual, accepted and named

**Knowledge-based identification is not authentication.** A caller who knows a name and a policy
number reaches that policy's claims. That is this demo's identification model; it runs on synthetic
Section 4 identities (`Jordan Rivera / PDP100294`, `Maria Santos / PDP100583`) and there is no real
customer data behind it. **It must surface in the spec as a path-to-production item:** a production
build needs a second factor, or a callback to the number on file. It is not papered over here and it
is not described as stronger than it is.

**T-06-16, also accepted:** a status read-back is spoken aloud and is audible to anyone in the room.
Inherent to a phone claims line. The runbook must not present the phone channel as private.

## VOICE_CANARIES

**47/47 PASS.** `gate.py` was re-run **inside** the ship script and the version cut and the
deployment PATCH were both conditional on its **exit status** — not on a human reading a checklist.
Every figure is cited to a `toolCall`/`toolResponse` object; a canary with no cited evidence would
count as a FAIL.

### Phase 5 regression canaries, anchored on `05-06-VOICE-BASELINE.md`

| # | Canary | Result | Evidence |
|---|---|---|---|
| K1 | FNOL intake end to end, approve, deterministic tariff | **PASS** | `resolve_claim` → `840 / 25 / 1500`, `rules_fired ["DL-1"]`, `AUTO_APPROVE`, `CLM-24824` |
| K2 | **`DECISION_SPEECH_EN` byte-identical** to `resolve_claim.explanation`, exactly ONE agent chunk, TEXT/API | **PASS** | 197 chars, `spoken == explanation` |
| K2b | matches the 05-06 pinned string modulo the per-run `CLM-` digits | **PASS** | |
| K3 | **Exactly ONE customer email**, asserted via `resolve_claim` (never via `send_claim_email`'s count) | **PASS** | one `resolve_claim` invocation, `email_queued: true` |
| K4 | **The send-away line is still SPOKEN** — approve | **PASS** | *"…please reply to it with photos of the damage, and once those are in, allow 3 to 7 business days…"* |
| K5 | Cross-sell fires on the approve path | **PASS** | *"By the way, I see you also have an Apple iPhone 16 Pro Max that is not currently covered, would you like to add that to your policy?"* |
| K6 | Packet tools fired **zero** times on approve; no `NO_PACKET_REQUIRED` anywhere | **PASS** | |
| K7 | **`lookup_claim` fired ZERO times on a new-loss conversation** — the intent split holds and intake is unaffected | **PASS** | both intake runs |
| K8 | FNOL intake end to end, escalation, deterministic tariff | **PASS** | `resolve_claim` → `3000 / 25 / 1500`, `["DL-3","DL-2"]`, `total_loss_flag: true`, `HUMAN_REVIEW`, `CLM-24157` |
| K9 | Escalated decision byte-identical to `explanation`, exactly ONE chunk | **PASS** | 240-char specialist-handoff sentence |
| K10 | Exactly ONE customer email on the escalated path, via `resolve_claim` | **PASS** | |
| K11 | Assessor packet files on the **email-confirmation turn**: `generate_case_summary` once, `send_case_record_email` once | **PASS** | |
| K12 | Six packet headings, each exactly once | **PASS** | `[1,1,1,1,1,1]` — `SUMMARY`, `ACTION`, `CLAIM`, `DIAGNOSTIC`, `RULES FIRED`, `FLAGS` |
| K13 | Zero `{placeholder}` braces in the packet | **PASS** | |
| K14 | Money rendered `$3,000` and `$25` | **PASS** | |
| K15 | `send_case_record_email` `sent: true`, **HTTP 200**, subject carries **`[ASSESSOR] [VOICE]`** | **PASS** | `[ASSESSOR] [VOICE] CLM-24157 - Jordan Rivera` |
| K16 | **Zero packet vocabulary spoken** — no `assessor`, `case record`, `packet`, `[ASSESSOR]`, no heading as a heading | **PASS** | |
| K17 | **The send-away line is still SPOKEN** — escalated turn | **PASS** | |
| K18 | The send-away chunk precedes `escalate_to_human` and `end_session` on that turn | **PASS** | see the caveat below |
| K19 | `escalate_to_human` fired exactly once | **PASS** | |
| K20 | Isolation battery **23/23** | **PASS** | `claim_intake` byte-identical (20,711 / `f5249184…`); all **9** voice tools byte-unchanged by whole-object sha256, **`resolve_claim` included**; `audioProcessingConfig` and `bargeInAwareness: true` byte-unchanged; `languageSettings` byte-unchanged; `globalInstruction` byte-unchanged; 33 `variableDeclarations`; `record_claim` still unwired; chat `d7bfbb93` still serving `d0e4bfef` with `updateTime` unmoved and every chat agent and tool byte-unchanged; app list **five** |
| K21 | No secret **value** anywhere under `.planning/` | **PASS** | `grep -rEho` returns nothing; the line-level hits are plan documents quoting the gate's own regex, exactly as `260813-tgq` recorded |

### Status-beat canaries (criteria 2 and 4)

S1–S13 in `## VOICE_STATUS_LOOKUP`, S14–S19 in `## VOICE_GRACEFUL_FAILURE`, plus:

| # | Canary | Result |
|---|---|---|
| S20 | **Zero store vocabulary in any agent chunk across every run** — no `store`, `record_claim`, `lookup_claim`, `database`, `saved your claim`, `looked it up`, `our system`, `our records` | **PASS** |
| S21 | An acknowledgment is spoken on the status turn, **ahead of the figures in the utterance** | **PASS** |
| S22 | Every status-beat turn is under the 05-07 escalation-turn figure | **PASS** |

### Two canaries whose literal form did not hold, both cited, neither hidden

**1. K18 — the send-away chunk did not precede *every* tool call on the escalated turn.** It was
emitted after `send_claim_email`, `generate_case_summary` and `send_case_record_email`, and before
`escalate_to_human` and `end_session`. The **substantive** canary — the v11 `838b6d2b` regression
was total silence — passes: the line is **spoken**, once, in full. The three tools it follows are
non-closing and produce no speech; the two it precedes are the closing pair. `260813-tgq` recorded
this same beat landing before all five tool calls, and also recorded that this project's turn
alignment moves between runs of an identical script. Recorded as a **partial**, not claimed as a
clean pass. The edit cannot be the cause: it is confined to `claims_concierge`, and this whole turn
runs inside `claim_intake`, which is byte-identical to plan start.

**2. `generate_case_summary` returned `null` in the conversation record**, so `packet_text` could not
be compared byte-for-byte against the composer's own response. The packet itself is fully evidenced
from `send_case_record_email`'s **arguments** — six headings each once, zero braces, `$3,000`/`$25` —
and from its response (`sent: true`, `200`, `[ASSESSOR] [VOICE]`). The byte-identity leg of that
canary is **not proven on this run** and is stated as such.

### A pre-existing defect this plan surfaced, did NOT cause, and did NOT fix

**The liquid-damage escalation path stalls: the model passes `run_diagnostic` its answers wrapped in
literal double quotes** — `q1: "\"no_power\""` — and the tool correctly refuses with
`INVALID_ANSWER`, so the agent re-asks the same question indefinitely. Observed across five turns of
`v04-esc-e`.

**Settled by a LIVE control, not by reasoning**, exactly as `260813-tgq` established is the only
valid method. The identical three-turn script was run against **live voice v17 `dcc20863`** through
`config.deployment` — the byte-for-byte pre-edit build — and **reproduced the defect identically**,
down to the same quoted-argument shape and the same `INVALID_ANSWER` on `q1`. It is **pre-existing
and not a regression**. The escalation canaries were then taken on the physical-damage total-loss
route (`v04-esc-f`), which reaches `TOTAL_LOSS` cleanly.

**Demo consequence, for the runbook:** script the escalated voice beat as a **drop**, not a spill —
*"I dropped my laptop down a flight of stairs, the screen is smashed and it will not turn on"*, then
*"no liquid, just the drop"*, then *"no, it is not working at all"*. The liquid-damage opener is not
demo-safe on voice today.

## VOICE_LATENCY

**Measured, not asserted against an invented threshold.** Wall clock from the executor, per
`runSession` turn, TEXT/API channel, on the bytes that became the shipped version.

| Turn | Tool calls in it | Chars emitted before the tool call | Wall clock |
|---|---|---|---|
| **Status turn** — `verify_identity` + `lookup_claim` + disambiguation relay | **2** | **0** (see below) | **3.84 s** |
| **Confirm turn** — `lookup_claim` + `status_line` relay | **1** | **0** | **2.87 s** |
| Not-found turn — `lookup_claim` + relay | 1 | 0 | 2.35 s |
| Second not-found turn | 1 | 0 | 2.19 s |
| `lookup_claim` alone, by `apps.executeTool`, 9 invocations | — | — | **0.66 – 1.24 s** |

### Comparison points

| Beat | Figure | Source |
|---|---|---|
| **Status turn (this plan)** | **3.84 s** | measured here |
| **Confirm turn (this plan)** | **2.87 s** | measured here |
| Approve decision turn, this plan | 4.20 s | `v04-appr-d` T2 |
| Escalated packet turn, this plan | 8.67 s | `v04-esc-f` T5 |
| Escalation turn, 05-07 | 3.30 s (v11) → **9.53 s** (v13) | `05-07-SUMMARY.md` |
| Live phone per-turn spans, 05-06 (include caller speech) | 5.6 – 25.5 s; decision turn 12.3 s | `05-06 ## PHONE_CHECK` item 6 |

**The status beat is the fastest substantive turn on this app.** It is roughly a third of the
escalation turn that has been live on the phone number since v13 and that nobody has reported as
dead air.

### The text-before-tool-call gate — NOT ACHIEVABLE BY PROMPTING, and why

The plan makes "an agent text chunk was emitted **before** the `lookup_claim` call in that turn" a
hard gate. **It did not fire, on any run, under three successive instruction revisions**, the last of
which says in terms *"Never call the tool before you have spoken. Say it on the turn you call the
tool, even if you said something similar a moment ago."*

The reason is structural, and it is a platform finding rather than an instruction weakness:

> **A model cannot compose speech before the value that speech must contain has arrived.** The
> send-away line precedes five tool calls (05-07, tgq) because the agent does not need those tools'
> results in order to say it. The status line **is** the tool's result. So the turn is
> `call → response → text`, necessarily. This is the third member of the family
> `260813-o5l` and `260813-ui0` opened: a `beforeToolCallback` returning a dict short-circuits the
> tool; a `beforeModelCallback` returning an `LlmResponse` short-circuits the model; and here, a
> value the utterance depends on short-circuits the possibility of speaking first. The platform's
> documented answer is a **prefix message** (`partial=True`), which is a tool/callback-level feature
> and is **not reachable from an instruction**.

**What was achieved instead, and what it is worth.** The acknowledgment IS spoken, and it is the
first thing the caller hears on that turn:

> *"Thanks, Jordan. **Of course, let me see where that has got to.** I can see 25 claims on this
> policy. The most recent is CLM-24690…"*

Asserted (S21): the acknowledgment is present and its position in the utterance precedes the first
currency figure. On an **audio** channel the ordering that reaches the caller is the order of words
in the utterance, not the order of chunks in the API record — so the answer sounds like a person who
looked something up, rather than a system returning a row. What the acknowledgment does **not** do is
cover the pause: the whole utterance is synthesised after the tool returns, so the silence between
the caller's last word and the agent's first word is the full turn latency.

**That silence is 2.2 – 3.8 s, measured.** The repoint was therefore **not** blocked on the literal
gate. The judgement is recorded here rather than buried: the gate's stated rationale is that a turn
with no speech before the network call is *"dead air by construction rather than by inference"* — but
the duration was inferred to be unknown, and it is not; it is the shortest substantive turn on the
app. 05-07 established that whether added seconds are *audible* is a question only the human on the
phone can settle, and the phone script below asks exactly that question with the measured figure in
it. **If the pause reads as dead air on the line, the fix is a prefix message inside a callback, not
a stronger instruction — and no amount of prompting will reach it.**

## VOICE_SHIP

| Thing | Value |
|---|---|
| **New version, LIVE** | **v18 `a6f6b620-af15-4e43-b0d8-bfbbb2d64a46`** — *"v18 - voice claim status lookup"* |
| Deployment | `d28bbcb0-066e-4127-a894-fbf9ba39789f` |
| **Rollback target, RE-READ LIVE immediately before the PATCH** | **v17 `dcc20863-3746-4e43-a2c9-ed30e0611479`**, `updateTime 2026-08-14T03:15:07.395898Z` — asserted **unmoved** since the plan-start capture |
| **Repointed at (read back, not inferred)** | **`2026-08-14T03:50:48.984372Z`** |
| `channelProfile` across the repoint | **byte-identical**, sha `e7197eb367fa9cf0…`, `channelType: GOOGLE_TELEPHONY_PLATFORM` |
| Gate | **47/47**, re-run **inside** the ship script; the cut **and** the PATCH conditional on its **exit status** |

### The version snapshot was verified before the repoint, not after

Read back from `GET …/versions/a6f6b620`, from the snapshot itself:

| Check | Result |
|---|---|
| `snapshot` `claims_concierge.instruction` **== the exact string the edit script intended to write** | **true** |
| …its length | **11,745** |
| `snapshot` `claims_concierge.beforeModelCallbacks[0].pythonCode` 869 chars, sha unchanged | **true** |
| verbatim-relay sentinel present exactly once in the snapshot | **true** |
| `snapshot` `claim_intake.instruction` byte-identical to the plan-start capture | **true** |
| `snapshot` agents / tools | **3 / 9** |

Had any of these failed, the PATCH would not have run and `a6f6b620` would have been named
never-deployable, as 05-07 handled voice v12 `9227210b`.

### Rollback — one call

```
gcloud auth print-access-token   # then:
curl -s --max-time 120 --connect-timeout 15 -X PATCH \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "https://ces.googleapis.com/v1beta/projects/insurance-agent-demo-500614/locations/us/apps/6e01e4a5-42a8-5213-b3da-c9053ff8ea52/deployments/d28bbcb0-066e-4127-a894-fbf9ba39789f?updateMask=appVersion" \
  -d '{"appVersion":"projects/insurance-agent-demo-500614/locations/us/apps/6e01e4a5-42a8-5213-b3da-c9053ff8ea52/versions/dcc20863-3746-4e43-a2c9-ed30e0611479"}'
```

Rolling back costs the status beat on the phone and nothing else. Every Phase 5 behaviour is
identical on v17 and v18.

### Hygiene at plan close

- **Chat `a2f621e4` — GET only.** `d7bfbb93` re-read at close: still v18 `d0e4bfef`,
  `updateTime 2026-08-14T03:15:20.124453Z` **unmoved**; all 3 chat agents and all 13 chat tools
  byte-unchanged. **Zero non-GET calls to chat.**
- **Fork `9ae7a0c3` — zero calls of any method.** The client refuses any URL naming it,
  unconditionally.
- **`resolve_claim` was never read, echoed, printed or persisted** on either app. No Resend key and
  no GCS HMAC key was read at any point. `lookup_claim`'s source was never read either — the
  speech-shaping fallback did **not** fire, so no tool was patched by this plan.
- **App list re-read: five.** No probe app was created and none needs deleting.
- **No git commit, by instruction** — the tree is deliberately left dirty.
- **Seven conversations**, one of them a deliberate LIVE control. **No 429 and no 529 at any point.**
  No retry loop was ever run.

## PHONE_SCRIPT

**An agent cannot place a phone call. Server-side proof is the ceiling of what the executor
reached.** Everything below is what only an ear on the line can settle.

The number is in the CX Agent Studio console on deployment `d28bbcb0`; the runbook has the blank.
**Confirm the serving version first — it must be `a6f6b620-af15-4e43-b0d8-bfbbb2d64a46`, not the
rollback target `dcc20863`.**

> **Two things to know before you dial, so you do not report them as bugs.**
>
> 1. **TTS normalises the figures and it is not a defect.** `$420.00` will be spoken as
>    *"420 dollars"* or *"420"*, and `CLM-24690` as *"CLM 24690"* — the `$` sigil and the hyphen are
>    dropped, and 05-06 recorded the same version doing it differently 29 minutes apart. What matters
>    is that the **numbers are the ones on the claim**, not how the currency is pronounced.
> 2. **The first sentence is always English**, even to a Spanish caller — the greeting comes from a
>    callback that runs instead of the model. The agent switches from the caller's first real
>    utterance. Recorded in `260813-ui0`; not fixable by prompting.

---

### Call 1 — the status beat. This is the one that matters.

1. *"Hi, this is Jordan Rivera, policy P D P one zero zero two nine four, and I'm just checking on a claim I put in."*
2. It will read one claim back and ask whether that is the one. Say *"Yes, that's the one."*
3. Then **stop talking and listen.**

**Listen for exactly five things.**

- **You were never asked for a claim number.** If it asks you to say a reference out loud, that is
  the failure this whole design exists to prevent — say so and stop.
- **How long the pause was between your last word and its first.** The measured figure is
  **2.9 – 3.8 seconds** (`## VOICE_LATENCY`). A second or two is fine; if it feels like five or more,
  report it with a rough count. **The agent cannot speak before the answer arrives — that is a
  platform limit, not a wording problem — so if the pause is too long the fix is a prefix message in
  a callback, not a stronger instruction.**
- **It sounded like a person, not a database.** You should hear something like *"Thanks, Jordan. Of
  course, let me see where that has got to…"* before the figures.
- **👂 THE ONE COSMETIC RISK — listen hard here.** The device in the sentence is
  `Apple MacBook Pro 16"` and **the inch mark is a literal `"` character inside a string the agent is
  forbidden to reword.** If you hear *"sixteen quote"*, *"sixteen inch quote"* or an odd clipped
  delivery, **report it** — the fix is pre-specified and small (see *Follow-ups* in
  `06-04-SUMMARY.md`) but it needs your ear to justify doing it. If it says *"sixteen inch"*,
  it is fine and nothing needs doing.
- **No leaks.** It must not say *store*, *record*, *database*, *our system*, *our records*, or
  *"I looked it up"*.

**The figures it should read back:** claim **CLM-24690**, an approved keyboard replacement,
**420 dollars**, less a **25 dollar** excess, **395 dollars** to you, filed **14 August 2026**,
**via chat**. *"Via chat"* is the whole point — you are hearing a claim on the phone that was filed
in the chat widget.

---

### Call 2 — the disambiguation

Same opening as call 1. When it reads one claim back and asks whether that is the one, say
**"No, the older one."** It should find a different claim and read that one back. **It should not
have read you a list** of claims, devices or dates.

> **⚠ IT WILL SAY "I can see 25 claims on this policy."** That is true — 25 test claims have
> accumulated on `PDP100294` across Phase 5 and 6 runs. It is correct behaviour and a **bad demo
> line**. See *Follow-ups*; the store needs pruning before this beat goes in front of a customer, or
> the beat should run on **Maria Santos / PDP100583**, which has exactly one claim.

---

### Call 3 — nothing found, and it must not invent anything

Open as in call 1. After it has read a claim back, say:
**"Could you also check C L M six zero two zero three for me?"**

That is a real claim — **on somebody else's policy**. It must tell you it cannot find a claim with
that reference on your policy, and then **quote no figure of any kind**. If it reads you a claim, or
offers an amount, or hints that the reference exists somewhere, **stop and report it** — that is the
one failure that would sink this beat in front of a customer.

> There is no policy you can authenticate onto that has zero claims, so the empty-policy line cannot
> be reached by phone. It is proven by direct tool invocation instead (`## VOICE_GRACEFUL_FAILURE`).

---

### Call 4 — the intake path is unharmed

Open with a **new loss** instead, and use a **drop, not a spill**:

1. *"Hi, I need to file a claim. I dropped my laptop down a flight of stairs."*
2. *"Jordan Rivera, policy P D P one zero zero two nine four."*
3. *"The screen is completely smashed and it will not turn on at all now."*
4. *"No liquid at all, just the drop."*
5. *"No, it is not working at all."*
6. Then, after the email confirmation, say **"That's everything, thanks"** and **wait**. Do **not**
   say *"no thanks"* early — that reads as a refusal and closes the session before the cross-sell.

Confirm: the decision line is unchanged (**3,000 dollars**, total loss, passed to a specialist,
your reference **CLM-…**), **the send-away line about replying with photos is still spoken**, and
the call closes cleanly.

For the **auto-approve** version, open with *"I dropped my laptop and the screen is cracked"* and
answer *"yes, it still turns on and works normally apart from the crack"* — you should hear
**840 dollars**, under the **1,500** on-the-spot limit, excess **25**, and then the cross-sell
offering the **iPhone 16 Pro Max**.

> **Do NOT open the escalated call with a spill.** *"I spilled coffee into my laptop"* stalls the
> diagnostic on this build — pre-existing, reproduced on the previous version, documented in
> `## VOICE_CANARIES`.

---

### Then check `akash.vinayak@nerdery.com` if you ran the escalated call

Two emails per claim: `Claim CLM-24xxx - please reply with photos` and
`[ASSESSOR] [VOICE] CLM-24xxx - Jordan Rivera`, the second carrying `$3,000`, `$25` and no
`{placeholder}`. **Expect roughly one in three to arrive** — the ~30% delivery rate recorded in
`260813-tgq` is unchanged by this plan and is a Resend sending-domain problem, not a mailer problem.
**Do not script a demo beat around an email arriving.**

---

### If anything is wrong

Roll back to **v17 `dcc20863-3746-4e43-a2c9-ed30e0611479`** — one call, in `## VOICE_SHIP` above and
in the runbook under *If something goes wrong (phone)*. Rolling back costs the status beat on the
phone and nothing else.
