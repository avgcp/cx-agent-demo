# 260813-olv — the language reset at the agent handoff

**Run:** 2026-08-13 · project `insurance-agent-demo-500614`, location `us`, API `https://ces.googleapis.com/v1beta`
**Target:** voice app `6e01e4a5-42a8-5213-b3da-c9053ff8ea52` · deployment `d28bbcb0-066e-4127-a894-fbf9ba39789f` (GTP)

| | |
|---|---|
| `ROLLBACK_VERSION` | **`5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc`** (voice v14) — read live before and after |
| **New version cut** | **`17b2e438-b132-49f1-8b32-190b132225ae`** — *voice v15 - language follows the caller across the agent handoff (NOT DEPLOYED - probe 4 mid-call switching still fails)* |
| **Deployment** | **HELD.** `d28bbcb0` still serves **v14 `5d02f14c`**, `updateTime` still `2026-08-13T22:23:57.500510Z` — byte-identical to its value at task start, re-read after the version cut. **No `deployments.patch` was issued at any point.** |
| **Why held** | **Probe 4 FAILS.** The gate was four probes plus the canary set; probes 1–3 and 42/42 canary checks pass, probe 4 fails in both directions. Per the gate: hold and report. |
| Chat `a2f621e4` | **zero calls of any method.** Still v15 `129f8b31` |
| Fork `9ae7a0c3` | never called |
| App list | **5**, unchanged. No probe app created |
| Conversations spent | 8 sessions / 34 turns, paced ≥60 s. One 529 mid-run (server-side, transient); no retry loop |

---

## THE DEFECT, REPRODUCED IN TWO TURNS ON THE TEXT CHANNEL

Session `olv-diag-a1`, voice **DRAFT** at live-v14 bytes, TEXT/API. No audio, no phone.

| Turn | User | Agent | Language |
|---|---|---|---|
| 1 | `Hola, ¿cómo estás?` | `claims_concierge` — *"Hola, ¿con quién tengo el gusto de hablar?…"* | **ES** ✓ |
| 2 | `Yeah, my name is Jordan Rivera and my policy is PDP100294.` | `claims_concierge` — *"Thanks, Jordan - putting you through now."* | **EN** ✓ switched correctly |
| 2 | *(same turn)* `agentTransfer` → `claim_intake` | `claim_intake` — *"Hola Jordan, tengo tu Apple MacBook Pro 16" aquí, ¿qué le ha pasado?"* | **ES** ⚠ **RESET** |
| 3 | `Yeah, I cracked my screen. It doesn't have any liquid damage…` | *"Buenas noticias, ese reemplazo de pantalla cuesta $840…"* | **ES** — stuck |

Two turns is the whole reproduction. It needs no phone call and no Spanish beyond the greeting.

**Turn 3 also translated the deterministic decision sentence.** `resolve_claim.explanation` is an English
literal that voice relays byte-verbatim (`## DECISION_SPEECH_EN`); once stuck in Spanish the model
translated it instead. That is a *consequence* of the reset, not an independent defect — it disappears
when the language is right.

---

## THE LEADING HYPOTHESIS WAS WRONG — IT IS NOT THE FIRST UTTERANCE

The brief's hypothesis was that the rule *"is being re-evaluated on sub-agent entry against
conversation history where Spanish appeared first."* **A control refutes the "first" part.**

Session `olv-diag-b1`, same build, the mirror image — **English first, Spanish latest**:

| Turn | User | `claim_intake` at the handoff |
|---|---|---|
| 1 | `Hi there, I need to report a claim.` (EN) | — |
| 2 | `Sí, me llamo Jordan Rivera y mi póliza es PDP100294.` (ES) | *"Tengo su Apple MacBook Pro 16" aquí, ¿qué le ha pasado?"* — **ES** |

| Control | first utterance | latest utterance | `claim_intake` spoke |
|---|---|---|---|
| `olv-diag-a1` | **ES** | EN | **ES** |
| `olv-diag-b1` | EN | **ES** | **ES** |

Under "first utterance wins", `olv-diag-b1` should have opened in English. It did not. **Neither
position predicts the outcome — Spanish does.** The mechanism is:

> On sub-agent entry `claim_intake` resolves its output language against the **whole transcript**
> rather than against any particular turn, and when the transcript contains any Spanish at all,
> Spanish wins — irrespective of where it appeared or what the caller's most recent words were.
> Having resolved, it does not re-decide: three consecutive English turns on the live call, and one
> in `olv-diag-a1`, failed to pull it back.

`claims_concierge` does **not** have this problem — it tracked the latest turn correctly in both
controls (EN in a1, ES in b1). The asymmetry is the tell: the `globalInstruction` follow-the-caller
block governs the root agent's turns, but inside `claim_intake` an 18,215-char agent instruction with
no language rule of its own dominates the context.

**And the wording made it worse.** 260813-nnm's block opened with *"Answer in the same language the
customer is speaking, **from their very first words**"* — intended as "starting immediately", readable
as "the language of their first words". It was a plausible contributor, but the `olv-diag-b1` control
shows it cannot be the whole story: removing it was necessary and not sufficient (see edit 1 below).

---

## WHAT WAS CHANGED — TWO EDITS, BOTH ON VOICE, BOTH SCRIPTED

Both by scripted `str.replace()` **from a file** (`edits.json`, `edits2.json`), anchors asserted to
occur exactly once in voice's own CRLF bytes, must-survive literal lists, every original non-blank
line re-checked present, bounded delta, byte-for-byte read-back. **`case_summary` and
`claims_concierge` were never patched** — their instruction hashes are byte-identical to v14
throughout (`2f74fa346a4122dc`, `8f610408ce652a6c`). No tool was created, patched or read;
`resolve_claim` and every other tool hashes identically to v14. No Resend code or secret was read.

### Edit 1 — `globalInstruction`, 2,364 → 2,859 (+495)

One contiguous region replaced (prefix 1,268 / suffix 846 / old 250 / new 745). The `Language:`
heading, the never-announce bullet and the figures bullet are untouched; only the first two bullets
were rewritten, and *"from their very first words"* is gone. The replacement (it contains no secret):

```
- Decide the language of every reply from THE CUSTOMER'S MOST RECENT MESSAGE ALONE - the words they have just said, on this turn. Not the language the call opened in, not a language used earlier in the call, not the language of anything you have already said. Their latest message in Spanish means you reply in Spanish; their latest message in English means you reply in English.
- Re-decide this from scratch on every turn, and especially on the first turn after you take over the call from another agent: match the customer's most recent message, never restart in the language the call began in.
- If the customer changes language part-way through, change with them on that same turn and carry on in the new language for as long as they use it.
```

### Edit 1 — `claim_intake`, 18,215 → 19,214 (+999), a **pure insertion** (`old_region == 0`)

Rules `22`, `22b`, `22c` appended inside `<constraints>`, in the section's own numbered-rule
convention, immediately before `</constraints>`. `<constraints>` was chosen deliberately: it is the
home of cross-cutting rules, so no `<step>` was reordered and the o5l *"last paragraph of a step
wins"* trap was avoided entirely.

**Edit 1 was necessary and NOT sufficient.** On the edit-1 build, `olv-p3` turn 2 still reset to
Spanish at the handoff. It did, however, fix the steady-state: `olv-p3` turn 4 followed a post-handoff
ES→EN switch, which live v14 could not do.

### Edit 2 — `claim_intake`, 19,214 → 20,711 (+1,497), two insertions

Edit 1 failed on exactly one turn: the one governed by `<step name="Confirm the device">`, whose
trigger is *"The caller has been transferred to you."* Per the project's own o5l finding —
**instruction position is load-bearing and a step's own action out-competes a distant rule on the
turn that step fires** — the rule was placed *inside that step*, not shouted louder.

1. **A recency tie-break appended to the opener paragraph of the transfer step.** This is the clause
   edit 1 lacked: it does not merely say "most recent", it says what to do when two languages are
   present.

```
Decide the LANGUAGE of that opening sentence before you say it. Scan back to the last
thing the CALLER themselves said - their own words, not yours and not the previous
agent's - and open in that language. When more than one language appears earlier in
the call, the LATER one wins: Spanish spoken earlier does not make this a Spanish
call if the caller's last words were English, and English spoken earlier does not
make it an English call if their last words were Spanish. Match their last words.
```

2. **The empty `<examples>` block filled with three few-shot demonstrations** (ES→EN before handoff,
   EN→ES before handoff, and a mid-claim switch). `<examples>` was `<examples>\r\n\r\n</examples>` —
   genuinely empty, so this displaced nothing.

**Deliberately NOT done, and why:**

- **`case_summary` was not touched.** The brief allowed it *"if it speaks"*. It composes the internal
  assessor packet, and it carries a deliberate *"The packet is always written in English… Never
  translate this packet"* clause. A follow-the-caller rule there would fight that clause and risk
  translating the assessor packet. The escalation canary confirms the packet still files in English.
- **`claims_concierge` was not touched.** It already gets the language right on every observation.
  The defect is entirely on the receiving side of the transfer.
- **The handoff line was not made to carry the language forward.** It was tested and it does not
  help: in `olv-diag-a1` the concierge said *"Thanks, Jordan - putting you through now."* in English
  and `claim_intake` opened in Spanish regardless. In `olv-p3b` turn 2 and `olv-p4` turn 2 the
  concierge emitted **no** send-away line at all, so it is not a reliable carrier.
- **No third edit was attempted.** The remaining failure (probe 4) sits in `<step name="Ask and
  record">`, whose action **ends on the `260811-suy` repeated-question / `DIAGNOSTIC_INCOMPLETE`
  recovery rule**. o5l's recorded warning is explicit: *"Do not add a competing paragraph to the end
  of a step that already ends on a rule you depend on."* That rule has been broken three times by
  three different causes. Trading a working diagnostic for an unproven language fix is a bad trade.

---

## THE FOUR PROBES — PER-TURN LANGUAGE, MARKER COUNTS

All against the voice **DRAFT**, TEXT/API channel, scored with the 05-03 marker scorer
(`es` = Spanish markers, `en` = English stopwords, `acc` = accented/inverted-punctuation characters).

### Probe 1 — open in Spanish, stays Spanish through the handoff ✅ **PASS**

`olv-p1`, 6 turns, customer never typed an English word until turn 6.

| Turn | Agent | Verdict | es / en / acc |
|---|---|---|---|
| 1 | `claims_concierge` | **ES** | 12 / 0 / 5 |
| 2 | `claims_concierge` | **ES** | 4 / 1 / 0 |
| 2 | **`claim_intake` (handoff)** | **ES** ✓ | 4 / 0 / 3 |
| 3 | `claim_intake` (decision) | **ES** | 22 / 0 / 1 |
| 4 | `claim_intake` (send-away) | **ES** | 12 / 2 / 4 |
| 5 | `claim_intake` (cross-sell) | **ES** | 8 / 1 / 5 |

Zero English turns after the handoff.

### Probe 2 — open in English, stays English through the handoff ✅ **PASS**

`olv-p2`, 5 turns. **Total Spanish markers across every agent chunk in the whole session: 0.**

| Turn | Agent | Verdict | es / en / acc |
|---|---|---|---|
| 1 | `claims_concierge` | **EN** | 0 / 6 / 0 |
| 2 | `claims_concierge` | **EN** | 0 / 2 / 0 |
| 2 | **`claim_intake` (handoff)** | **EN** ✓ | 0 / 3 / 0 |
| 3 | `claim_intake` (decision) | **EN** | 0 / 18 / 0 |
| 4 | `claim_intake` (send-away) | **EN** | 0 / 21 / 0 |
| 5 | `claim_intake` (cross-sell) | **EN** | 0 / 8 / 0 |

No over-correction toward Spanish. This was the regression risk and it did not materialise.

### Probe 3 — open Spanish, switch to English before the handoff ✅ **PASS** — *the exact live failure*

`olv-p3b`, 4 turns.

| Turn | User | Agent | Verdict | es / en / acc |
|---|---|---|---|---|
| 1 | `Hola, como estas?` (ES) | `claims_concierge` | **ES** | 10 / 0 / 7 |
| 2 | `Yeah, my name is Jordan Rivera…` (EN) | **`claim_intake` (handoff)** | **EN** ✓ | 0 / 3 / 0 |
| 3 | (ES) | `claim_intake` (decision) | **EN** | 0 / 18 / 0 |
| 4 | (EN) | `claim_intake` (send-away) | **EN** | 0 / 19 / 0 |

**Two controls, same script, earlier builds — the fix is what moved it:**

| Build | `claim_intake` at the handoff |
|---|---|
| **live v14 `5d02f14c`** (`olv-diag-a1`) | **ES** ❌ es=3 / en=0 / acc=3 — the defect |
| **edit 1 only** (`olv-p3`) | **ES** ❌ es=3 / en=0 / acc=4 — still broken |
| **edit 1 + edit 2** (`olv-p3b`) | **EN** ✅ es=0 / en=3 / acc=0 |

### Probe 4 — switch language AFTER the handoff, inside `claim_intake` ❌ **FAIL, both directions**

This is the beat a demo would actually use, and it does not work.

| Direction | Session / turn | Caller said | Agent replied | Verdict | es / en / acc |
|---|---|---|---|---|---|
| **EN → ES** | `olv-p5` t3 | `Se me cayo y se rompio.` (ES) | *"Got it, I'm sorry to hear that. Was there any liquid involved?"* | **EN** ❌ | 0 / 4 / 0 |
| **EN → ES** | `olv-p3b` t3 | `Si, se me rompio la pantalla…` (ES) | (decision line, English) | **EN** ❌ | 0 / 18 / 0 |
| **ES → EN** | `olv-p6` t4 | `The screen is cracked, and there was no liquid at all.` (EN) | *"Y, aparte de la rotura, ¿el ordenador se enciende…"* | **ES** ❌ | 4 / 0 / 1 |
| **ES → EN** | `olv-p1` t6 | `Actually no thanks. Can you tell me again…` (EN) | *"Por supuesto. Una vez que recibamos sus fotos…"* | **ES** ❌ | 10 / 1 / 3 |
| ES → EN, **edit-1 build** | `olv-p3` t4 | `Okay, that's fine by me.` (EN) | *"I've sent an email to the address on your policy…"* | **EN** ✅ | 0 / 11 / 0 |

Both `olv-p5` t3 and `olv-p6` t4 are **clean mid-diagnostic turns** — deliberately not the decision
turn, where an English tool literal is relayed verbatim and would confound the reading. `olv-p1` t6
is a close-sequence turn and is the weak observation of the four (05-06 item 5 already records that
a switch can be out-competed by the close); the other three are not.

**Honest reading of the last row:** on the edit-1 build a post-handoff ES→EN switch was followed once.
After edit 2 it is not. Either edit 2 traded mid-claim switching for the handoff turn, or the single
edit-1 observation was noise — one observation cannot separate those, and it was on a close turn,
the weakest kind. It is recorded rather than resolved.

**Probe 4 has never worked on any build of this project.** 05-03's original chat finding was exactly
this — an English conversation, a Spanish turn 2, an English reply. It was attributed to
`LOCKS_AT_FIRST_UTTERANCE`, then re-attributed by 260813-nnm to `FOLLOWS_WHEN_INSTRUCTED`. Neither
verdict is wrong about what it measured, but **mid-call switching inside a sub-agent has been
observed to fail on every build ever tested, and it still fails.** It is not a regression introduced
here.

---

## REGRESSION CANARIES — 42/42 PASS, ALL IN ENGLISH UNLESS MARKED

Anchored on `05-06-VOICE-BASELINE.md` and nnm's 21/21 + 16/16. All on the TEXT/API channel, per
05-06's warning that byte-comparison on AUDIO fails spuriously.

### English auto-approve — `olv-p2`, 5 turns

| Canary | Result |
|---|---|
| `DECISION_SPEECH_EN` byte-identical to `05-06-VOICE-BASELINE.md` (CLM normalised) | **PASS** — 197 chars, exact |
| `spoken == resolve_claim.explanation` byte-identical | **PASS** |
| `decision_turn_agent_chunks == 1` | **PASS** — chunks=1, no self-narration, no doubling |
| deterministic tariff | **PASS** — 840 / 25 / 1,500 / 3,000, `DL-1`, `AUTO_APPROVE`, `relay_mismatch: false` |
| no unrendered `{placeholder}` | **PASS** |
| **the send-away line is still SPOKEN** (the v11 silent-turn shape) | **PASS** |
| exactly one `send_claim_email` (structural) | **PASS** |
| **the cross-sell still fires** | **PASS** — *"…I see you also have an Apple iPhone 16 Pro Max - would you like to add that to your cover?"* |
| no price/premium quoted on the cross-sell | **PASS** |
| English run stayed English | **PASS** — `spanish_markers == 0` across the entire session |

### Spanish run — `olv-p1`, 6 turns

| Canary | Result |
|---|---|
| **`decision_turn_agent_chunks == 1` ON THE SPANISH RUN** (nnm's double-delivery canary) | **PASS** — chunks=1 |
| **no unprompted re-statement of the decision on the following turn** | **PASS** — turn 4 contains no `840` and no `CLM-` |
| Spanish tariff identical to English | **PASS** — 840 / 25 / 1,500 from `resolve_claim` |
| cross-sell fires in Spanish | **PASS** |
| exactly one `send_claim_email` | **PASS** |
| send-away line SPOKEN in Spanish | **PASS** |

> **nnm's double-delivery defect did NOT recur, but it did not recur for a different reason than
> "fixed".** nnm saw the tool's English string relayed, then re-stated in model-composed Spanish on
> the next turn. Here the model **translated the decision on the decision turn itself** and never
> re-stated it — one chunk, one delivery. So `spoken != explanation` on the Spanish run: the figures
> (840 / 1,500 / 25 / CLM-24495) were **model-composed, not tool-supplied.** They survived intact and
> `$` was preserved this time, but nothing guarantees that on stage. **This is threat `T-05-12` and it
> is still open. 05-09 owns it** — the fix is the bilingual literal inside `resolve_claim`, which this
> task was explicitly forbidden to touch. Recorded, not fixed.

### English escalation — `olv-p4`, 5 turns

| Canary | Result |
|---|---|
| `HUMAN_REVIEW`, `total_loss_flag: true`, $3,000 / $25 | **PASS** |
| `DL-3` + `DL-2` both fired | **PASS** — `["DL-5", "DL-3", "DL-2"]` (DL-5 additionally, diagnostic inconclusive — the caller's script gave a total loss the tool could not confirm) |
| **assessor packet files with all six sections** | **PASS** — `SUMMARY:` `ACTION:` `CLAIM:` `DIAGNOSTIC:` `RULES FIRED:` `FLAGS:` |
| **`[ASSESSOR] [VOICE]` subject at 200** | **PASS** — `[ASSESSOR] [VOICE] CLM-24176 - Jordan Rivera`, `sent: true`, `status_code: 200` |
| packet carries no unrendered `{placeholder}`, and is in English | **PASS** — `case_summary` was never patched |
| `escalate_to_human` fired exactly once | **PASS** |
| **the escalated send-away turn is NOT silent** | **PASS** — spoken, and emitted *before* all five tool calls |
| one customer email, no `[ASSESSOR]` leakage to the customer | **PASS** |
| escalation ran entirely in English | **PASS** |

### Canaries proved structurally

Against the **new version's own snapshot** `17b2e438`, not the draft — 18/18:

- **Deterministic tariff and the `DIAGNOSTIC_INCOMPLETE` guard** live inside `resolve_claim` and
  `run_diagnostic`. **All nine tools are byte-identical to v14** by whole-object hash with volatile
  fields stripped — `resolve_claim` included, which was never read, patched or echoed. No
  `tools.patch` was issued at any point.
- **`GTP_SURFACE` byte-identical:** `audioProcessingConfig` is
  `{"synthesizeSpeechConfigs": {"en-US": {}, "es-US": {}}, "bargeInConfig": {"bargeInAwareness": true}}`,
  byte-equal to v14. No DTMF, timeout, interruption, SIP or `customize_response` key appeared.
- `languageSettings` byte-equal to v14; 33 `variableDeclarations`; `SEQUENTIAL`;
  `gemini-3.1-flash-live`; `case_summary` and `claims_concierge` instructions byte-equal to v14.
- **`record_claim` is still wired to NO agent** — `claim_intake` wires 7 tools, unchanged and
  byte-identical to v14. It ships inert, exactly as 06-04 requires.
- **Audio-channel behaviour remains unheard.** An agent cannot dial the number.

---

## WHY THE DEPLOYMENT WAS HELD — AND THE ARGUMENT FOR OVERRIDING THAT

**Held, per the gate as written:** *"The repoint is gated on the four probes plus the canary set. Any
failure → leave `d28bbcb0` on `5d02f14c` and report."* Probe 4 fails. `d28bbcb0` was not patched.

**The counter-argument, so the decision is yours and not mine.** On every dimension actually tested,
`17b2e438` is **strictly better than the build on the phone right now**:

| | live v14 `5d02f14c` | v15 `17b2e438` |
|---|---|---|
| Probe 1 — ES through the handoff | ✅ | ✅ |
| Probe 2 — EN through the handoff | ✅ | ✅ |
| Probe 3 — switch before the handoff | ❌ **catastrophic** — the rest of the call is stuck in the wrong language and unrecoverable | ✅ |
| Probe 4 — switch after the handoff | ❌ | ❌ |
| English canaries | ✅ | ✅ 42/42 |

Probe 4 fails on **both** builds and has never passed on any build. Probe 3 is a **live-observed,
call-ending defect** — conversation `119vrJXUcbjQCO4DWJ_0w7xxw` ended with the caller hanging up — and
it is fixed. Holding leaves the worse build serving the phone.

**If you want it shipped, it is one call** (re-read `d28bbcb0` first, as always):

```
PATCH .../apps/6e01e4a5-.../deployments/d28bbcb0-...?updateMask=appVersion
{"appVersion": ".../versions/17b2e438-b132-49f1-8b32-190b132225ae"}
```

**ROLLBACK is the same call** with `5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc`.

> ### ⚠ `draft_equals_v14` IS NOW **FALSE** — 06-04 MUST READ THIS
>
> The voice **draft** carries the two edits and is byte-equal to **v15 `17b2e438`**, not to the
> deployed v14. `05-06-VOICE-BASELINE.md`'s `draft_equals_v11` invariant, which later voice plans
> assert, no longer holds against the *deployed* version.
>
> **Consequence for 06-04 (wire the claim store into voice), which is the next dispatch:** any version
> it cuts from this draft **will include these language edits**. That is not harmful — they fix probe 3
> and regress nothing in 42/42 canaries — but it must be a decision, not a surprise, and 06-04 must
> say so in its own summary. Current live `claim_intake` is **18,215** chars; the draft is **20,711**.
> Re-read both live rather than trusting any number written into a plan.

---

## ALSO RECORDED, NOT FIXED — SPEECH-TO-TEXT MANGLES THE POLICY ID

From the same live call (`119vrJXUcbjQCO4DWJ_0w7xxw`, `2026-08-13T22:36:23Z`, AUDIO/LIVE, v14
`5d02f14c`), independent of the language defect:

**Capturing one policy ID took five turns.** The transcribed attempts ran `PGP` → `PDP` → `1000294`
→ `TDP100294` before the correct `PDP100294` landed. `verify_identity` fired **three times**, and a
barge-in occurred mid-sequence.

Speech-to-text mangles alphanumeric identifiers. `PDP` is heard as `PGP` and `TDP`; the digit run
loses or gains a leading character. On the auto-approve path the whole claim takes about two minutes
(05-06 item 6), so roughly a third of the demo can be spent failing to authenticate — immediately
before the moment the agent is meant to look brilliant, and in front of the room.

**Not fixed here, deliberately.** It is the same problem Phase 6 criterion 4 was written around, and
06-01 already shipped the relevant mitigation on the store side: keys are normalised
(`PDP-100294` → `PDP100294`) *"so a spoken policy id survives speech-to-text, which is what makes
criterion 4 reachable without anyone reading a claim reference aloud."* The equivalent normalisation
on the **authentication** path — `verify_identity` — does not exist, and `verify_identity` was out of
scope for this task.

**Two candidate mitigations for whoever picks this up**, neither attempted:

1. **Normalise inside `verify_identity`** — strip non-alphanumerics, upper-case, and accept a small
   confusion set on the letter prefix (`PGP`/`TDP`/`BDP` → `PDP`) plus a digit-length tolerance. This
   is the 06-01 pattern applied one layer earlier and it is invisible to the caller.
2. **Presenter mitigation, zero build cost** — the runbook should tell the presenter to say the policy
   ID **digit by digit, slowly**, and to expect one retry. This belongs in `DEMO-RUNBOOK.md` whether
   or not (1) is ever built.

---

## WHAT THIS CHANGES FOR OTHER PLANS

- **05-09 (Spanish deterministic strings) is unchanged in scope and is now the only thing standing
  between this and a clean Spanish call.** This task confirms `resolve_claim.explanation` is
  model-translated on a Spanish run (`spoken != explanation`, figures model-composed) — the same
  threat `T-05-12` nnm recorded, arriving by a different route. nnm's *double-delivery* symptom did
  not reproduce; do not assume it is gone, it was simply out-competed by an outright translation.
- **`05-03`'s `LANGUAGE_SWITCH_VERDICT` needs no further correction.** `FOLLOWS_WHEN_INSTRUCTED` is
  still the right token for the *root* agent. What this task adds is that it does **not** generalise
  to sub-agents: a `globalInstruction` is not re-applied per turn inside `claim_intake`, and a
  sub-agent needs the rule in its own instruction. Even then, mid-call switching inside the sub-agent
  does not work.
- **Do not script a mid-call language switch as a demo beat.** 05-03 said this, nnm re-opened it as
  plausible, and this task closes it again with four direct observations: **it works at the handoff
  and only at the handoff.** The demonstrable beat is *"the caller speaks Spanish and the whole call
  is in Spanish"* (probe 1) or *"the caller starts in one language and settles into another before
  authentication completes"* (probe 3). Neither requires switching mid-claim.
- **A new, cheap reproduction exists.** Two text turns against the draft reproduce a defect that
  previously needed a phone call: Spanish greeting, then English identity, then read `claim_intake`'s
  opener. Any future language work should start there.

---

## Self-Check: PASSED

- `.planning/quick/260813-olv-.../260813-olv-SUMMARY.md` — created
- `d28bbcb0` re-read after all work → **`5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc`**, `updateTime`
  `2026-08-13T22:23:57.500510Z`, unchanged. **No repoint.**
- New version `17b2e438-b132-49f1-8b32-190b132225ae` verified 18/18 against v14 for isolation
- App list = 5, no probe app created, no `git commit` made (tree left dirty as instructed)
- Zero API calls of any method to `a2f621e4` or `9ae7a0c3`
- No tool source read, no Resend key read or echoed, `record_claim` left unwired
