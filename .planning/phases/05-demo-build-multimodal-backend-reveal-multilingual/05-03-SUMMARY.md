---
phase: 05-demo-build-multimodal-backend-reveal-multilingual
plan: 03
status: complete
date: 2026-08-10
subsystem: cx-agent-spike
tags: [spike, multilingual, language-settings, voice-baseline, resend, rich-content, ces]
server_changes:
  chat_app: a2f621e4-9faf-505a-b804-22471f022366
  chat_change: "languageSettings ONLY — supportedLanguageCodes:[es-US] + enableMultilingualSupport:true added to the DRAFT (apps.patch, updateMask=languageSettings)"
  mechanism: direct apps.patch (NO importApp, NO version cut, NO deployment repoint)
  untouched: [6e01e4a5 (voice — GET only), 9ae7a0c3 (fork — no call of any kind)]
  deployments_unchanged: ["d7bfbb93 -> chat v9 160dc3b2", "d28bbcb0 -> voice v11 b17c9a26"]
files_modified:
  - .planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-03-SPIKE-FINDINGS.md
requirements: []
---

# Phase 5 Plan 03: Pre-flight Spike Summary

**All three gating unknowns are resolved, and one of the three answers is worse than the
phase hoped.** Mid-call language switching does not work — the verdict is
`LOCKS_AT_FIRST_UTTERANCE`, decided by a pre-committed mechanical rule, not by eye. The voice
draft is proven byte-equal to the v11 the phone serves and still resolves an escalated claim
correctly today. The packet-recipient question went to the user, who chose domain
verification.

Output: `05-03-SPIKE-FINDINGS.md` with four literal headings downstream plans grep by name.

## The three consumed outputs — verdict, evidence, confidence

### 1. `LANGUAGE_SWITCH_VERDICT` → `LOCKS_AT_FIRST_UTTERANCE` — read by 05-08, 05-09

**Confidence: HIGH for what was tested. But read the caveat — it is load-bearing.**

Evidence: one live 2-turn conversation on the chat DRAFT (session
`ba782f0b-322a-4371-8cb1-c025002837c2`), English turn 1, Spanish turn 2. The reply to the
Spanish turn scored **0 Spanish markers / 4 English stop-words** against a rule fixed *before*
the run (`SWITCHES_MID_CALL` iff `spanish_markers >= 3 AND english_stopwords <= 2`). Nowhere
near the boundary. `Conversation.languageCode` read `en-US`.

The most useful part is the *decomposition*: **Spanish input was understood perfectly; only
the output language stayed English.** `run_diagnostic` fired from the Spanish sentence with
`{issue_category: physical_damage, q1: screen, q2: no_liquid, q3: works_normally}` — identical
to what the English equivalent produces. This confirms 05-RESEARCH.md Pitfall 1 exactly and
matches `enableMultilingualSupport`'s own documented wording ("...multilingual **input**").

**⚠ What 05-08/05-09 would be WRONG to assume:** that the two-back-to-back-conversations
fallback works. **It is not verified.** A conversation *starting* in Spanish was never tested —
the budget was two conversations and both were spent on the questions the plan named. The
token `LOCKS_AT_FIRST_UTTERANCE` describes observed behaviour but *implies* the first
utterance sets the language; an equally consistent reading of the same single observation is
that the app always answers in `defaultLanguageCode`, in which case **the fallback is broken
too** and there is no configuration-only Spanish path at all. `Conversation.languageCode`
cannot discriminate — `en-US` is also the app default. 05-08 must run one Spanish-opening
turn before committing. This is written into the findings file, not just here.

### 2. `VOICE_BASELINE` → `draft_equals_v11: true` — read by 05-05, 05-07, 05-10

**Confidence: HIGH.** Whole-object comparison, not field-sampling: every agent and tool
serialised with volatile identity fields stripped and SHA-256'd, draft vs the v11 snapshot.
All equal. `claim_intake` instruction 14,140 chars, 5 tools, 2 agents, 33
`variableDeclarations`. Per-resource hashes are recorded as the regression anchor later plans
assert against.

Live re-verification (session `cae670d7-a6f1-491d-b782-921a53af6128`, 3 turns, voice DRAFT),
all asserted from the conversation record rather than prose: `decision == HUMAN_REVIEW`,
`claim_amount == 3000`, `rules_fired == ["DL-3","DL-2"]`, `send_claim_email` in **exactly
one** turn, **no** cross-sell on the escalation path, `escalate_to_human` fired. The English
`explanation` (240 chars) is pinned for 05-10's bilingual pair.

Side finding for 05-06: voice's `synthesizeSpeechConfigs` contains **`en-US` only** — no
`es`/`es-US` entry. Spanish TTS behaviour on the phone is currently undefined by this
project's own config.

### 3. `PACKET_RECIPIENT` → `verify-domain` (DOMAIN_VERIFICATION) — read by 05-04, 05-05, 05-07

**User decision, 2026-08-10.** The user declined the one-mailbox compromise; the packet is to
reach a genuinely different recipient.

The documentation evidence resolves assumption A1 and it is **sender-level**, not
per-recipient — source <https://resend.com/docs/knowledge-base/403-error-resend-dev-domain>:
*"The `resend.dev` domain is only available for testing purposes and can only send emails to
the email address associated with your Resend account."* Multi-recipient `to` lists and
plus-aliases are **not documented in either direction**, and the findings file says so
explicitly rather than inferring. The only inference drawn is flagged as one: the failure is a
`validation_error`/403 (a request rejection, not a per-recipient outcome), so a mixed `to`
list most likely fails the whole send.

**⚠ What 05-04/05-05/05-07 would be WRONG to assume:** that concrete addresses exist. They are
**PENDING** — a distinct recipient *is* available by decision, but `from`, `to` and both
subject prefixes are supplied at execution time after the user verifies the domain and DNS
propagates. Do not hardcode, and do not silently fall back to the rejected `one-mailbox`
values. **This is a blocking external dependency:** no live second-mailbox delivery can be
verified until the user completes it in the Resend dashboard.

Flagged as a regression risk: moving `from` off `resend.dev` also requires updating
`send_claim_email`'s `from` in **both** apps — a tool whose source holds the live API key —
after which the single-send invariant *and* actual deliverability must both be re-proven on
the new domain.

### Bonus: `RICH_CONTENT_SPANISH` → `UNRESOLVED` (half settled) — read by 05-08

Settled **offline and free** from the on-disk SDK v1.16 bundle, no conversation spent. The
`order_summary` renderer passes `title` and `subtitle` **unmodified** into lit-html
child-position interpolations (`<div class="product-subtitle">${subtitle}</div>`) — DOM text
nodes. The bundle has zero `normalize(`, zero `ASCII`, zero `navigator.language`; all 6
`sanitiz` hits are the goog HTML tag/attribute whitelist (markup, not characters) and all 5
`locale` hits are unrelated. **The SDK applies no locale gating to our own free-text fields**,
which substantially narrows 05-RESEARCH.md assumption A2.

Real defect found while reading: `Intl.NumberFormat("en-US", ...)` is **hardcoded** for the
price, with no parameter. The Spanish card will still render `$840.00` US-style. Cosmetic and
unavoidable client-side — belongs in the runbook.

**Deliberately NOT asserted:** the token stays `UNRESOLVED` because the platform half is
untestable here. 05-08's `WITHHELD` case (CES declining to emit the payload at all on a
non-English conversation) is server-side, invisible in the bundle, and invisible in this
spike's two English conversations. 05-08's Spanish run reaches the card, so the check is free
there. Writing `RENDERS` on SDK evidence alone would have been exactly the confident guess
this plan exists to prevent.

## Deviations from Plan

**[Rule 1 — bug, in this task's own tooling] The user-turn counter mis-scored, twice
corrected without touching server state.** Task 1's first scoring pass reported "user turns
matched verbatim: 1" against a required 2. Cause: the conversation record **redacts the
customer name** (`Hi, my name is [REDACTED] and my policy is PDP100294`), so a verbatim
string comparison against the sent text can never match turn 1. Re-counted by `role == "user"`:
exactly 2 user turns, criterion met. Nothing was changed to make it pass. Worth knowing
generally — **any future assertion that matches sent user text verbatim against the record
will fail on PII-bearing turns.**

**[Rule 1 — bug, same class] The plan-level verification's two initial FAILs were both check
defects, not file or state defects.**
1. Heading counts read 2–3 instead of 1 because in-prose backticked references to
   `## PACKET_RECIPIENT` etc. were being counted alongside the real heading. Tightened to
   line-start `^## HEADING$`; all four now count exactly 1. The in-prose mentions are
   legitimate and were kept — downstream plans reference these names in their own text too.
2. The secret-scan regex `re_[A-Za-z0-9_\-]{12,}` flagged three **pre-existing** files. All
   three are false positives on ordinary identifiers containing `...re_...` —
   `befo|re_model_callback`, `befo|re_tool_callbacks`, `p|re_verification`. Verified **without
   ever echoing a matched token**, by printing only each match's length and character-class
   shape plus preceding context. Tightened the regex to require ≥16 chars and at least one
   digit; zero hits. **No Resend key exists anywhere under `.planning/`.**

**[Rule 3 — blocking] The safety gate initially refused a required call.** `runSession` is a
POST, and Task 2 mandates running it against the voice DRAFT — but the blanket "no mutating
call without `a2f621e4`" gate refused it. Rather than weaken the gate or monkey-patch it per
call (the first attempt, discarded as fragile), the gate was made precise: definition-writes
against `6e01e4a5` and any call against `9ae7a0c3` are refused unconditionally; **the single
exception is `POST …:runSession`**, which writes no app definition. The gate was then
**unit-tested against five hostile URLs** before use — 4 refused, 2 allowed, as intended.

**Task 3 was not blocked on.** Per the orchestrator's standing instruction, the decision was
recorded with its evidence and everything independent of it was completed. The user's
`verify-domain` choice arrived mid-execution and is recorded.

**Executor crash and resume.** An earlier session of this plan died on an API transport error
(connection closed mid-response) after the `languageSettings` write had landed but before
anything reached the repo. State was re-audited read-only on resume; the landed write was
**verified, not reapplied**. Findings were then written to disk incrementally, section by
section, rather than composed at the end. Per-run output was held to lengths, booleans and
short excerpts throughout — no instruction body, no tool source, no large API response was
ever echoed.

**Quota:** exactly **2 conversations / 5 turns** against the plan's budget of 2 and 6. Turns
paced 60–62 s. **No 429 occurred**, so the authorised single spaced retry was never used.

## Verification Evidence

| Check | Result |
|---|---|
| Chat `languageSettings` read back from a fresh GET after PATCH | ✅ `es-US` + `enableMultilingualSupport: true` + `en-US` default |
| Rest of chat app unchanged by the patch | ✅ 7 fields byte-identical (`rootAgent`, `globalInstruction`, `toolExecutionMode`, `modelSettings`, `guardrails`, `audioProcessingConfig`, `variableDeclarations`) |
| Chat deployment `d7bfbb93` | ✅ still v9 `160dc3b2` — re-read after everything |
| Voice deployment `d28bbcb0` | ✅ still v11 `b17c9a26` — re-read after the conversation |
| Voice `languageSettings` | ✅ untouched, `{"defaultLanguageCode": "en-US"}` |
| Voice draft vs v11 snapshot | ✅ whole-object equal: 5 tools, 2 agents, 33 variables, both instructions |
| Voice escalation assertions | ✅ 4/4 from the record: `HUMAN_REVIEW`, `3000`, `DL-3`+`DL-2`, single `send_claim_email` |
| No cross-sell on the escalation path | ✅ no tool, no offer prose |
| Fork `9ae7a0c3` | **No API call of any kind was issued.** |
| Voice app writes | **None.** Gate refuses every non-`runSession` mutation; unit-tested. |
| Findings file headings | ✅ all four present exactly once at line start |
| Verdict token | ✅ first token after the heading is the literal `LOCKS_AT_FIRST_UTTERANCE` |
| Secret scan under `.planning/` | ✅ zero hits after false-positive triage |

**Secret handling:** `resolve_claim`'s and `send_claim_email`'s source was never read, echoed
or copied. Tool listings were filtered by `displayName`. Tool source was **hashed but never
printed** for the draft-vs-v11 comparison. The pinned `explanation` came from a conversation
`toolResponse` — claim data, not source.

## Known Stubs

None. This plan produced findings and one config field; no placeholder values were shipped.

The `PACKET_RECIPIENT` addresses are recorded as `PENDING` by explicit user decision, not as
a stub — the decision is made, the concrete values are supplied at execution time after an
external dependency the user owns.

## Threat Flags

None. No new network endpoint, auth path, file-access pattern or schema at a trust boundary
was introduced. The one server change is a language-config field on a draft app.

## Self-Check: PASSED

- `05-03-SPIKE-FINDINGS.md` — created (22,176 chars), all four headings verified present
- `05-03-SUMMARY.md` — created
- Every server assertion re-read from the API after the write, never inferred from a request body
- No git commit made; working tree left dirty for the orchestrator
- `STATE.md` and `ROADMAP.md` deliberately untouched
