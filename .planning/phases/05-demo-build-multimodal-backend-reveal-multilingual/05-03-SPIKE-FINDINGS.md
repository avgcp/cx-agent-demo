# 05-03 Spike Findings

**Run:** 2026-08-10 · executor plan 05-03 · project `insurance-agent-demo-500614`, location `us`, API `https://ces.googleapis.com/v1beta`

Three answers downstream plans read by name. Headings below are literal and greppable —
**do not rename them.**

| Heading | Read by | Answer |
|---|---|---|
| `## LANGUAGE_SWITCH_VERDICT` | 05-08, 05-09 | ~~`LOCKS_AT_FIRST_UTTERANCE`~~ → **`FOLLOWS_WHEN_INSTRUCTED`** (corrected 2026-08-13, task 260813-nnm — the observation held, the attribution was wrong; read the correction block in that section before using this) |
| `## VOICE_BASELINE` | 05-05, 05-07, 05-10 | `draft_equals_v11: true` |
| `## PACKET_RECIPIENT` | 05-04, 05-05, 05-07 | `same-mailbox` — user decision 2026-08-11, supersedes `verify-domain`. Both emails → `akash.vinayak@nerdery.com`, assessor packet prefixed `[ASSESSOR]`. `from` stays `onboarding@resend.dev`. **No external blocker.** |
| `## RICH_CONTENT_SPANISH` | 05-08 | **UNRESOLVED** — not answerable within this plan's budget |

**Deployments did not move.** Chat `d7bfbb93` still serves v9 `160dc3b2`; voice `d28bbcb0`
still serves v11 `b17c9a26`. Every live conversation ran against the DRAFT app with no
`config.deployment`. Nothing customer-facing changed.

---

## LANGUAGE_SWITCH_VERDICT

~~LOCKS_AT_FIRST_UTTERANCE~~

**FOLLOWS_WHEN_INSTRUCTED**

> ### ⚠️ CORRECTED 2026-08-13 by quick task 260813-nnm — read this before acting on anything below
>
> **The observation recorded in this section is correct and was reproduced on voice. The
> attribution and the verdict token were wrong.**
>
> This section's ⚠ NOT VERIFIED note below offered two readings of the same evidence: (a) the first
> utterance sets the language, or (b) *"the app simply always answers in `defaultLanguageCode`
> regardless"*, and warned that under (b) **the two-conversation fallback does not work either.**
> 260813-nnm ran the cheap probe this section asked for — on **voice**, opening in Spanish — and
> **(b) is the correct reading.** Nothing locks at the first utterance. The response language was
> never a function of the utterance at all.
>
> | Probe (voice draft, TEXT/API, identical Spanish opener) | `spanish_markers` | `english_stopwords` |
> |---|---|---|
> | as-found (no `supportedLanguageCodes`, no `enableMultilingualSupport`) | 0 | 3 |
> | **+ this section's exact `languageSettings` applied** | **0** | 5 |
> | + a follow-the-caller block in `globalInstruction` | **7** | **0** |
>
> **The middle row is the correction.** Applying the very `languageSettings` this section proved
> settable changed nothing about the output language. `enableMultilingualSupport` governs **input**
> and declares the synthesis surface; it does not select the response language. **The response
> language is governed by the instruction, and by nothing else reachable through configuration.**
> Instructed to follow the caller, the agent follows from the first word — hence
> `FOLLOWS_WHEN_INSTRUCTED`.
>
> **Also disproven: the suspicion that this spike measured a prompt, not the platform.** 260813-nnm
> searched every instruction body, tool description and guardrail on **both** apps, live and in the
> deployed snapshots. **There is no English-only clause on either channel and there never was.**
> The *"I can only speak English"* sentence heard on a live Spanish phone call was emergent from
> the monolingual `languageSettings` on voice, not authored anywhere.
>
> **What downstream plans should now do:**
> - **Do NOT** treat the two-back-to-back-calls fallback as the only option — it would have failed for the same reason a mid-call switch did.
> - **Do NOT** rely on configuration alone for any Spanish beat on any channel.
> - The **mid-call EN→ES switch is once again an open question**, not a closed negative. It is now plausible (the shipped voice instruction explicitly tells the agent to switch mid-conversation) but **still untested**. One 2-turn probe settles it. Do not script it as a demo beat until someone runs that probe.
> - The instruction-level fix is **already live on voice** in v14 `5d02f14c` (app-level `globalInstruction`, so it governs all three agents). Chat `a2f621e4` has **not** received it.
>
> Full evidence: `.planning/quick/260813-nnm-remove-the-english-only-clause-and-enabl/260813-nnm-SUMMARY.md`

**Original 2026-08-10 record, retained unedited so the reversal is visible:**

| Field | Value |
|---|---|
| `spanish_markers` | **0** |
| `english_stopwords` | **4** |
| `Conversation.languageCode` | `en-US` |
| Session id | `ba782f0b-322a-4371-8cb1-c025002837c2` |
| App | chat `a2f621e4` — **DRAFT**, no `config.deployment` |
| Turns | 2 user turns (confirmed by `role == "user"` in the conversation record) |

Verdict rule was fixed in advance: `SWITCHES_MID_CALL` iff `spanish_markers >= 3 AND
english_stopwords <= 2`. Observed 0 and 4. Not close to the boundary — this is an
unambiguous result, not a marginal one.

**Turn 2 agent text, verbatim:**

> I'm sorry to hear that.
>
> Could you please attach a photo of the damage? I need to see the screen, in shot and reasonably lit.

### The distinction that matters most for 05-08 / 05-09

**Spanish INPUT was understood perfectly. Only the OUTPUT language stayed English.**

Turn 2's Spanish was parsed correctly and drove the deterministic path with no errors —
`run_diagnostic` fired with `{"issue_category": "physical_damage", "q1": "screen",
"q2": "no_liquid", "q3": "works_normally"}` and set `dx_outcome: REPAIRABLE`, which is
exactly what the equivalent English sentence produces. The agent then correctly demanded a
photo (the screen-crack gate). Comprehension is not the problem; **response-language
selection is.**

This corroborates 05-RESEARCH.md Pitfall 1 precisely, and matches `enableMultilingualSupport`'s
own documented description — *"agents in the app will use pre-built instructions to improve
handling of multilingual **input**."* The flag did what it says. It does not translate output.

### Setup this ran against (so nobody re-litigates it)

`languageSettings` was patched on the chat DRAFT app **before** the conversation and
**read back from a fresh GET**, not inferred from the request body:

```json
{"defaultLanguageCode": "en-US", "supportedLanguageCodes": ["es-US"], "enableMultilingualSupport": true}
```

**This is itself a finding.** The v1beta API accepts both `supportedLanguageCodes` and
`enableMultilingualSupport` on `apps.patch` with `updateMask=languageSettings`. Prior
research had the schema from the discovery doc but had never written the fields. They are
real, settable, and they persist. 05-06 can rely on this.

The rest of the app was asserted unchanged by the patch (`rootAgent`, `globalInstruction`,
`toolExecutionMode`, `modelSettings`, `guardrails`, `audioProcessingConfig`,
`variableDeclarations` all byte-identical before/after).

`languageSettings` was **deliberately not reverted** — 05-06 needs it under either verdict,
and so does the fallback.

### What this means for downstream plans

- **Do NOT script a mid-call EN→ES switch as a demo beat.** It does not work today.
- The fallback named in 05-RESEARCH.md — two back-to-back conversations, one per language —
  is now the only candidate mechanism. **But see the caveat immediately below: it is not
  itself verified.**

### ⚠ NOT VERIFIED — read this before building the fallback

**A conversation that starts in Spanish was NOT tested.** This spike's budget was two
conversations / six turns and both were spent on the two questions the plan named. So:

- **Proven:** a conversation that starts in English stays in English, even when the customer
  switches to Spanish.
- **Unproven:** whether a conversation whose *first* utterance is Spanish responds in Spanish.

The verdict token `LOCKS_AT_FIRST_UTTERANCE` describes the observed behaviour but **implies**
that the first utterance sets the language. The alternative reading — that the app simply
always answers in `defaultLanguageCode` regardless — is equally consistent with the single
observation made here, and would mean **the two-conversation fallback does not work either.**

`Conversation.languageCode` read `en-US`, which is also the app default, so that field
cannot discriminate between the two readings on this evidence.

**05-08/05-09 must run one cheap probe before committing to the fallback:** a single-turn
conversation on the chat draft opening in Spanish (e.g. `Hola, me llamo Jordan Rivera y mi
póliza es PDP100294`), then score the reply. One turn, ~5–10k tokens. If it answers in
English, the entire Spanish beat depends on the bilingual-template work in the tools that
05-RESEARCH.md recommends, and there is no configuration-only path at all.

---

## VOICE_BASELINE

```
draft_equals_v11: true
```

The editable voice draft is **byte-equal to the v11 snapshot the phone actually serves**.
Every later voice edit therefore builds on the deployed version, with no hidden drift to
discover mid-phase. Comparison was whole-object, not field-sampled: each agent and each tool
resource was serialised with the volatile identity fields (`name`, `createTime`,
`updateTime`, `etag`) stripped and SHA-256'd. Tool **source was hashed, never read or
printed.**

| Field | Value |
|---|---|
| App | `6e01e4a5-42a8-5213-b3da-c9053ff8ea52` |
| Version compared | v11 `b17c9a26-3485-4658-9259-dfa4839a7977` (*"v11 - no self-narration"*) |
| `draft_equals_v11` | **true** |
| `claim_intake` instruction length | **14140** chars (sha256[:16] `628548c1e582e34d`) |
| `claims_concierge` instruction length | 5126 chars (sha256[:16] `8f610408ce652a6c`) |
| Tool `displayName`s | `escalate_to_human`, `resolve_claim`, `run_diagnostic`, `send_claim_email`, `verify_identity` |
| Agent `displayName`s | `claim_intake`, `claims_concierge` (root) |
| `variableDeclarations` count | **33** (name set identical to snapshot, zero diff either way) |
| `predefinedVariableDeclarations` count | 2 |
| `languageSettings` | `{"defaultLanguageCode": "en-US"}` — untouched by this plan |
| `synthesizeSpeechConfigs` languages | `["en-US"]` only — **no `es`/`es-US` entry exists** |

**Regression anchor — later voice plans assert against these.** Whole-object sha256[:16],
volatile fields stripped:

| Resource | sha256[:16] |
|---|---|
| tool `escalate_to_human` | `0f30145634cca897` |
| tool `resolve_claim` | `898ebc7ab33d29af` |
| tool `run_diagnostic` | `8782c85ddfa1b433` |
| tool `send_claim_email` | `c0e80c7ef90d8d50` |
| tool `verify_identity` | `75349f96fb2b1ab7` |
| agent `claim_intake` | `c495723ddd92e699` |
| agent `claims_concierge` | `3318edbcdd40104b` |

Also identical draft↔snapshot: root `childAgents` (1) and `transferRules`, `claim_intake`'s
tool list (5) and `transferRules`, and app-level `globalInstruction`, `toolExecutionMode`,
`modelSettings`, `languageSettings`, `audioProcessingConfig`, `errorHandlingSettings`.

Machine-readable copy: `scratchpad/s03-voice-baseline.json`.

### It still resolves an escalated claim — verified live today

One conversation, three turns, Scenario B, against the voice **DRAFT** (`runSession`, no
`config.deployment`). Session `cae670d7-a6f1-491d-b782-921a53af6128`.

| Assertion | Result |
|---|---|
| `resolve_claim.decision` | **`HUMAN_REVIEW`** ✅ |
| `resolve_claim.claim_amount` | **`3000`** (JSON int) ✅ |
| `rules_fired` | **`["DL-3", "DL-2"]`** — both present ✅ |
| `send_claim_email` turns | **exactly one** (turn 2) ✅ |
| Cross-sell on the escalation path | **none** — no cross-sell tool, no offer prose ✅ |
| `escalate_to_human` fired | ✅ |
| Tools called | `verify_identity`, `run_diagnostic`, `resolve_claim`, `send_claim_email`, `escalate_to_human`, `end_session` |
| `d28bbcb0` after the run | still `b17c9a26` ✅ (re-read, not inferred) |

All values were parsed from the conversation record's `toolResponse`, not from agent prose.

### Pinned English `explanation` — 05-10 reads this as the ES pair's source

`resolve_claim`'s returned `explanation` on the HUMAN_REVIEW branch, verbatim, 240 chars
(pinned from the `toolResponse`, which is claim data — the tool's source was never read):

> So that comes to $3,000, and it looks like the device is a total loss. I'm going to pass this to one of our specialists. They'll already have everything we've talked about, so you won't need to go over it again. Your reference is CLM-24462.

`CLM-24462` is per-run; the reference number varies and the rest of the string does not.
Also on disk at `scratchpad/s03voice-explanation-EN.txt`.

### Note for 05-06 (voice TTS)

`audioProcessingConfig.synthesizeSpeechConfigs` on voice currently contains **`en-US` only**.
05-RESEARCH.md predicted this and it is confirmed: if the platform detects Spanish on a call
today, TTS behaviour is undefined by this project's own configuration. An `es-US` entry has to
be added explicitly — this plan deliberately did not add it (voice is not this plan's to
mutate).

---

## PACKET_RECIPIENT

> **⚠ SUPERSEDED 2026-08-11.** The `verify-domain` decision recorded on 2026-08-10 was
> reversed by the user the following day. The current, binding decision is `same-mailbox`
> below. The superseded reasoning is retained further down for the record.

**DECIDED BY THE USER, 2026-08-11: `same-mailbox`.**

Both the customer confirmation and the assessor briefing packet are delivered to
`akash.vinayak@nerdery.com` — the Resend account-owning mailbox, and therefore the one
address the shared `onboarding@resend.dev` sender is permitted to reach. They are told apart
by a subject-line prefix.

```
selected_option_id: same-mailbox
decision: SAME_MAILBOX
decided_by: user
decided_on: 2026-08-11
supersedes: verify-domain (2026-08-10)
distinct_recipient_available: NO — one mailbox, two subject prefixes
recipient_address:  akash.vinayak@nerdery.com
from_address:       onboarding@resend.dev   # UNCHANGED — do not touch
subject_prefix_assessor_packet: "[ASSESSOR]"
subject_prefix_customer_email:  ""          # unchanged from today's live behaviour
blocking_external_dependency: NONE
```

**How 05-04 / 05-05 / 05-07 must read this:**

1. **Build for one mailbox, two subject prefixes.** Do not build a second-recipient routing
   path and do not treat the addresses as PENDING — they are concrete above.
2. **Do NOT modify `send_claim_email`'s `from` address.** It stays `onboarding@resend.dev`.
   This is the single biggest win of the reversal: the customer-confirmation path remains
   exactly as verified (single-send, delivered), so nothing already proven has to be
   re-proven. Treat `send_claim_email` as byte-identical/untouched, as 05-04 and 05-05
   already assert.
3. **There is no external blocker.** The previous "blocking external dependency" on Resend
   domain verification and DNS propagation is GONE. Live delivery of the assessor packet can
   be verified in the same run that builds it.
4. **Say it honestly in the runbook.** Both emails land in one inbox. A presenter must not
   claim true separate routing. Criterion 3 asks for the packet to be *"delivered as a real
   message so the artifacts are demonstrable without a bespoke reveal screen"* — a real email
   in a real inbox satisfies that; the "different person" framing is narrative.
5. **Path to production:** verifying a domain at resend.com lifts the single-mailbox
   restriction and enables genuine separate routing. Record it in the spec as the production
   step, not as demo scope.

### ~~⚠ Blocking external dependency~~ — NO LONGER APPLIES

~~Nothing in 05-04 / 05-05 / 05-07 that actually delivers to a second mailbox can be verified
live until the domain is verified in the Resend dashboard and DNS has propagated.~~
Superseded by the `same-mailbox` decision above — there is no external dependency, and the
live delivery check is no longer gated.

### Superseded reasoning (2026-08-10, retained for the record)

The user initially declined the same-mailbox compromise, wanting the packet to reach a
genuinely different recipient, and accepted domain verification as the cost. That was
reversed on 2026-08-11 in favour of using the account-owning mailbox directly, on the
grounds that it works immediately, requires no DNS work, and avoids changing the `from`
address on an email path that is already verified.

### What changes once the domain is verified

1. **The `from` address moves off the shared `onboarding@resend.dev` sender** to an address on
   the verified domain. Per Resend's documentation (quoted below), this is precisely what
   lifts the single-mailbox restriction — the restriction is triggered by the `resend.dev`
   `from` domain, so changing `from` is not optional cosmetics, it is the whole mechanism.
2. **The existing customer-confirmation path `send_claim_email` also needs its `from`
   updated.** Both apps have their own copy of that tool. If only the new packet tool moves to
   the verified domain, the customer email stays on `resend.dev` and remains owner-only —
   which still works today but leaves two senders diverging.
3. **⚠ REGRESSION RISK — check both, do not assume.** Editing `send_claim_email`'s `from`
   means touching a tool whose source holds the live Resend API key. After any such change,
   re-assert on a live run that:
   - `send_claim_email` still fires in **exactly one** turn (the single-send invariant, held
     since v9 and re-verified for voice in `## VOICE_BASELINE` above), and
   - the customer email is still **actually delivered** (a newly-verified domain has its own
     deliverability warm-up and can silently land in spam).

   Both invariants are currently proven only for the `resend.dev` sender.

### Resend documentation finding — assumption A1 is now RESOLVED, and it is sender-level

**Source:** <https://resend.com/docs/knowledge-base/403-error-resend-dev-domain> (fetched
2026-08-10), corroborated by <https://resend.com/docs/api-reference/errors> (the
`validation_error` / 403 entry).

Documented verbatim:

> The `resend.dev` domain is only available for testing purposes and can only send emails to
> the email address associated with your Resend account. This restriction helps protect
> domain reputation and ensures proper email deliverability.

And the error text itself:

> You can only send testing emails to your own email address (your-email-address@domain.com).
> To send emails to other recipients, please verify a domain at resend.com/domains, and change
> the `from` address to an email using this domain.

**The restriction is documented as sender-level** — it is triggered by the `from` address
being on `resend.dev`, and it constrains the send to "the email address associated with your
Resend account" (singular). It is **not** documented as a per-recipient delivery outcome.
This matches, and now supersedes, the project's own 403 observation.

**Documented, HIGH confidence:**
- The restriction is caused by the `from` domain, not by the recipient count.
- Only the account-owning address can receive while `from` is `onboarding@resend.dev`.
- The only documented remedy is verifying a domain at resend.com/domains **and** changing the
  `from` address to that domain.

**NOT documented — explicitly stating this rather than inferring:**
- **Multi-recipient `to` lists are not mentioned anywhere.** The docs never say what happens
  when a `to` list mixes the owner address with a non-owner address.
- **Plus-aliases / subaddressing are not mentioned anywhere.** Whether
  `akash.vinayak+assessor@nerdery.com` counts as "the email address associated with your
  Resend account" is undocumented in either direction.

**Inference, flagged as inference (MEDIUM-HIGH, not proven):** the failure is classified as a
`validation_error` returned with HTTP 403 — a *request rejection*, not a partial-delivery
result. A request that is rejected does not deliver to anyone. So a `to` list containing a
second address most likely fails the **whole send**, losing the owner's copy too — which
would be worse on stage than not trying. A1's original framing ("assume it blocks a `to`
list") is therefore the safe assumption, and it is now backed by documentation rather than
guesswork. It is still not a live-tested fact.

**Bearing on `one-mailbox-plus-alias`:** the doc's phrasing is an equality against a single
stored address. A plus-alias is a different string, so a strict comparison rejects it and a
normalising comparison accepts it. Resend documents neither. This option therefore still
costs one live send to settle — exactly as the plan predicted — and that send would 403 the
whole request if the strict reading is right.

### The decision the user must make

**Question:** Where does the assessor briefing packet get delivered, given the shared Resend
sender delivers only to `akash.vinayak@nerdery.com`?

Success criterion 3 requires the packet be *delivered as a real message*. The customer
confirmation email already lands in `akash.vinayak@nerdery.com`. The packet is a different
artifact for a different reader (an internal assessor), and the demo beat is stronger if the
two are visibly distinct.

| Option id | What it means | Cost / risk now that the docs are read |
|---|---|---|
| **`one-mailbox`** | One mailbox, two clearly-differentiated subject lines | Zero new setup, nothing to break on stage, both artifacts visible side by side in one inbox. The presenter narrates *"in production these go to different people"* — the routing claim is told, not shown. **Lowest risk; ships today.** |
| **`verify-domain`** | Verify a domain at resend.com/domains, send the packet from it | The only option the documentation actually endorses, and the only one that yields genuinely different recipients. Requires DNS records on a domain the user controls **and** editing the `from` address inside the guarded tool source (which holds the live API key). Adds an external dependency and a DNS/deliverability failure mode on the day. |
| **`one-mailbox-plus-alias`** | Plus-alias / subaddress for the packet | Cheap, and may render as a distinct recipient in the inbox UI. **Undocumented either way** — one live send settles it, at the cost of a version cut and a conversation, and a strict-comparison rejection would 403 the whole send. |

**The user selected `verify-domain`.** The table above is retained as the costing that
justified the choice — `one-mailbox` and `one-mailbox-plus-alias` are **rejected options**
and must not be adopted as fallbacks by a downstream plan.

Note that `verify-domain` is also the only option Resend's own documentation endorses: it is
the single remedy the 403 error text names.

**Subject-line prefixes are still needed** and are `PENDING` alongside the addresses. Even
with two distinct mailboxes the two email kinds should stay visually unmistakable, since both
will often be demonstrated on one screen. Suggested shape for the user to accept or
overwrite at execution time — *not* a decision:

- customer confirmation → something like `[Meridian] Your claim`
- assessor packet → something like `[ASSESSOR REVIEW]`

They must be two distinct, non-empty strings.

**No tool source was read, printed or edited in gathering any of the above.** The finding
came entirely from Resend's public documentation.

---

## RICH_CONTENT_SPANISH

UNRESOLVED

**05-08 must still settle this in its Step F and its Spanish live run. Do not treat the
heading's presence as a verdict.** The token vocabulary 05-08 defines is
`RENDERS | DEGRADES | WITHHELD`; none of the three is asserted here.

The question splits cleanly in two, and this spike could only close one half.

### Half 1 — the SDK. SETTLED, and it points at `RENDERS`. [HIGH confidence]

Read offline from the deployed `chat-messenger` SDK **v1.16** bundle already on disk
(1,358,541 chars). No conversation was spent on this.

The `order_summary` `productItem` renderer is:

```js
function DF_MAz(a){
  if(!a.productItem) return DF_Mx(DF_Mmz);
  var b=a.productItem, c=b.title, d=b.subtitle, e=b.price, f;
  return DF_Mx(DF_Mnz, DF_MXp(b.imageUri, (f=a.v)==null?void 0:f.urlAllowlist), c, c, DF_MEz(e), d)
}
```

and its template is:

```html
<span class="product-title">${title}</span>
<span class="product-price">${price}</span>
<div class="product-subtitle">${subtitle}</div>
```

`DF_Mx` is lit-html's tagged-template factory (`_$litType$: 1`, `strings`, `values`), so
`title` and `subtitle` are **child-position interpolations rendered as DOM Text nodes**.

Evidence that nothing touches them:
- `title` and `subtitle` are passed through **unmodified** — no transform, no fold, no filter.
- The bundle contains **zero** `normalize(` calls, **zero** occurrences of `ASCII`, and
  **zero** `navigator.language` reads.
- All 6 `sanitiz` hits are the `goog.html.sanitizer` **tag/attribute** whitelist (it rejects
  non-`data-` attributes and non-whitelisted tags). It constrains markup, never text
  characters.
- All 5 `locale` hits are unrelated — a form country-picker's `localeCompare` sort and a
  `document.documentElement.getAttribute("lang")` read inside an unrelated proto. **There is
  no locale lookup on any render path.**

**Conclusion for half 1:** the SDK applies no locale gating and no character restriction to
our own free-text fields. Accented Spanish and `¿`/`¡` in `productItem.subtitle`, `title`,
and the `quick_actions` `content`/`utterance` labels will render as supplied. This
substantially narrows 05-RESEARCH.md assumption A2 — the doc's "rich content responses only
support English" is **not** implemented as an SDK-side restriction.

**One real, separate defect found while reading — record it, 05-08 needs it:**

```js
function DF_MEz(a){ ... new Intl.NumberFormat("en-US",{style:"currency",currency:a.currencyCode, ...}).format(b) }
```

The price is formatted with a **hardcoded `"en-US"` locale**. It is not derived from the
conversation language and there is no parameter to change it. On the Spanish card the price
will therefore still render US-style (`$840.00`, period decimal separator, no space before
the symbol) rather than Spanish convention. This is unavoidable client-side, it is cosmetic,
and it should go in the runbook as a known compromise rather than be treated as a bug.

### Half 2 — the platform. NOT SETTLED. [cannot be closed without a Spanish conversation]

The strongest reading of the web-widget doc's limitation is 05-08's `WITHHELD` case: that
**CES itself declines to emit the widget payload at all** on a non-English conversation. That
is a server-side behaviour. It is invisible in the SDK bundle and invisible in this spike's
two conversations, **both of which ran in English** — the Spanish turn in Task 1 came before
any widget and was answered in English anyway (see `## LANGUAGE_SWITCH_VERDICT`).

Settling it requires observing whether `payload` is present in the conversation record of a
run that reaches the decision card **in Spanish**. This plan's budget was two conversations /
six turns and both were spent on the questions the plan named, so it was not attempted.

**05-08 owns this.** Its Task 3 Spanish run already reaches the card, so the check is free
there: assert whether the decision turn carries a `payload` chunk at all. Absent ⇒
`WITHHELD`; present with intact accents ⇒ `RENDERS`; present but mangled ⇒ `DEGRADES`.
