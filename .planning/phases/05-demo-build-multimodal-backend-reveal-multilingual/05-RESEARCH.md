# Phase 5 (remainder): 05-03 Assessor Packet + 05-04 Spanish — Research

**Researched:** 2026-08-10
**Domain:** Google CX Agent Studio — multilingual configuration, agent-as-tool porting, transactional email constraints, cross-app synchronization
**Confidence:** MEDIUM overall (HIGH on platform mechanics verified live against the API; MEDIUM-LOW on whether the mid-call language-switch "wow moment" actually renders as hoped — that needs a live smoke test this research could not run)

## Summary

Both remaining Phase 5 items are buildable, but neither is a config flip. **05-04 (Spanish)** is a *content* problem wearing a *config* costume: the platform's multilingual machinery (`languageSettings`, automatic language detection/response-matching, 32 voice-language variants) genuinely exists and `es-US` is GA — but it does nothing for the strings this build has spent five days making deterministic. `resolve_claim`'s `explanation`, the assessor packet, the diagnostic questions, and the cross-sell offer are either hardcoded Python string templates or instruction-authored prose; the platform will not translate them. The only way to keep Spanish output deterministic (matching the project's own hard-won standard) is to make the tools themselves bilingual — a language-keyed dict of pre-written templates selected by a session variable, not LLM translation-on-the-fly. Free-form instruction-driven lines (diagnostic follow-ups, empathy) can safely rely on automatic detection + an explicit "respond in the customer's language" instruction, because they were never deterministic to begin with. The single biggest unresolved risk is whether a conversation's language can actually change *mid-call* — the API models language as one field on the `Conversation` resource, which reads more like "the conversation's language" than "the current turn's language," and this could not be settled without a live call.

**05-03 (assessor packet)** has real, working source material: `generate_case_summary` in the pre-hardening fork (`9ae7a0c3`) is a genuine agent-as-tool (`agentTool` type, `SYNCHRONOUS` execution, wired into `claim_intake` like any other tool) that composes the exact SUMMARY/ACTION/CLAIM/DIAGNOSTIC/RULES FIRED/FLAGS packet the roadmap wants — confirmed by reading its instruction directly. But it only *composes text*; nothing in the fork ever sends it anywhere ("Do not read it out to the customer — it is for the file. Carry on and close warmly."). Porting the agent alone does not satisfy success criterion 3's "delivered as a real message" requirement — a new delivery tool is needed, and it runs straight into the already-documented Resend constraint: the shared sender only delivers to one mailbox. There is no way to send a genuinely different "assessor" email to a genuinely different inbox without a verified domain. The workable near-term answer is one mailbox, two clearly-labelled subject lines.

**Cross-app convergence** has one real, hard-confirmed constraint: the CES API has **no shared-tool or shared-agent resource of any kind** — export/import (whole-package duplication) is the only mechanism, and it is exactly the mechanism that has already bitten this project twice via stale exports. The good news: voice's `claim_intake` instruction is clean — it never picked up the word-for-word/short-human-part contradiction that had to be fixed in chat (that bug was introduced by chat's widget-card work, which voice never got), so porting the decision-speech pattern to voice is not needed — it already works correctly and only needs bilingual content.

**Primary recommendation:** Treat 05-04 as "add a `preferred_language` session variable + bilingual string tables inside the deterministic tools," not "flip a platform switch." Treat 05-03 as "port `case_summary` + `generate_case_summary` unchanged, then build one new delivery tool, and get explicit user sign-off on the single-mailbox compromise before building anything mailbox-related." Do both waves — voice and chat — as separate, small, `apps.*.patch`-based edits per app, each gated by a fresh read-back, never a shared import.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Assessor packet composition (SUMMARY/ACTION/CLAIM/DIAGNOSTIC/RULES FIRED/FLAGS) | Agent/Instruction tier (agent-as-tool) | — | Pure text generation from already-computed session state; no new backend needed |
| Assessor packet delivery (email) | Custom Python tool (backend/API tier) | External service (Resend) | Same tier as the existing `send_claim_email` pattern — a deterministic HTTP call, not agent reasoning |
| Spanish decision explanation / email / packet text | Custom Python tool (backend/API tier) | — | Must stay deterministic; templated string selection belongs in code, not in the LLM |
| Spanish diagnostic questions / empathy lines / cross-sell offer | Agent/Instruction tier | Platform (automatic language detection) | These were never deterministic; safe to let the model compose them in the detected language |
| Voice TTS language switching | Platform (`audioProcessingConfig.synthesizeSpeechConfigs`) | Agent/Instruction tier | Native platform config surface, per documented API schema — not something a tool or instruction can substitute for |
| Widget card / quick-action rendering in Spanish | Client tier (deployed web-widget SDK) | Platform (rich-content English-only limitation) | The SDK's own template chrome is documented English-only; only the free-text fields we supply are ours to localize |
| Cross-app synchronization (voice ↔ chat) | Build/tooling process (export/import or direct patch scripts) | — | No platform primitive shares resources between apps; this is entirely an operational discipline, not an architecture layer |

## Standard Stack

No new external packages are required for either 05-03 or 05-04. Both are CX Agent Studio configuration + Python-tool-code changes within apps that already exist. The only "stack" additions are:

| Component | Status | Purpose | Why |
|-----------|--------|---------|-----|
| `App.languageSettings` (`defaultLanguageCode`, `supportedLanguageCodes`, `enableMultilingualSupport`) | GA — confirmed live in CES API discovery doc | Declares the app's language surface | [VERIFIED: CES API discovery doc, `https://ces.googleapis.com/$discovery/rest?version=v1beta`] — read directly, not inferred |
| `AudioProcessingConfig.synthesizeSpeechConfigs` (map of language code → `SynthesizeSpeechConfig`) | GA — confirmed live in CES API discovery doc | Per-language TTS voice/model selection for voice apps | [VERIFIED: same discovery doc] |
| `agentTool` tool type (`{name, agent}`) | GA — confirmed present and functioning in the fork app `9ae7a0c3` | The mechanism for porting `case_summary` as an agent-as-tool | [VERIFIED: read directly from the live `9ae7a0c3` app via `tools.get`] |
| Resend (existing, already in use by `resolve_claim`/`send_claim_email`) | Already integrated; do not re-verify or re-read its key | Email delivery for both the existing customer email and the new assessor packet | [VERIFIED: prior project work, STATE.md] — not re-verified here per the "never read `resolve_claim`'s source" constraint |

**Version verification:** Not applicable — no pip/npm packages. The relevant "versions" are CES API resource shapes, confirmed live against `https://ces.googleapis.com/v1beta` on 2026-08-10 (see Sources).

## Package Legitimacy Audit

Not applicable. This phase installs no external packages (no `npm install`, `pip install`, or `cargo add`). All work is CX Agent Studio app configuration and Python code embedded in existing platform tool resources.

## Architecture Patterns

### System Architecture Diagram — where 05-03/05-04 content plugs in

```
                         ┌─────────────────────────────┐
                         │   Customer input (voice/text) │
                         └───────────────┬───────────────┘
                                         │
                         ┌───────────────▼───────────────┐
                         │  claims_concierge (root agent)  │
                         │  verify_identity                │
                         └───────────────┬───────────────┘
                                         │ handoff
                         ┌───────────────▼───────────────┐
                         │        claim_intake (agent)     │
                         │  run_diagnostic → [assess_screen_crack: chat only]
                         │  resolve_claim  ◄── NEW: preferred_language
                         │        │  (returns bilingual, deterministic `explanation`)
                         │        │
                         │  ┌─────▼──────────────────────────────┐
                         │  │ Decision speech / card               │
                         │  │  voice: spoken verbatim (unchanged)  │
                         │  │  chat: dataMapping textResponse      │
                         │  │        ← explanation (unchanged      │
                         │  │        mechanism, now bilingual data)│
                         │  └───────────────────────────────────────┘
                         │        │
                         │  send_claim_email (existing, both apps)
                         │        │
                         │  ── NEW STEP ──
                         │  generate_case_summary  (agent-as-tool → case_summary agent)
                         │        │  produces packet text (deterministic template,
                         │        │  bilingual via same preferred_language pattern)
                         │        │
                         │  send_case_record_email (NEW tool, mirrors send_claim_email's
                         │        Resend pattern) — delivers the packet as a real message
                         │        │
                         │  cover_offer_actions [chat] / spoken offer [voice]
                         │        │
                         │  end_session
                         └─────────────────────────────────┘
```

### Recommended approach: bilingual deterministic tools, not LLM translation

**What:** Every tool that currently returns a fixed, hand-checked English string (`resolve_claim`'s `explanation`, the future assessor-packet composer, any other deterministic customer-facing line) should accept or read a `preferred_language` session variable and select from a small dict of pre-written templates (`{"en": "...", "es": "..."}`), rather than asking the model to translate.

**When to use:** Any string whose exact wording was hardened specifically to prevent the model from inventing, rounding, or paraphrasing figures (the decision explanation is the canonical example — this was the subject of the entire `260810-k5x` fix).

**When NOT to use:** Open-ended instruction-driven dialogue (diagnostic follow-ups, empathy responses, the cross-sell offer sentence) that was never deterministic — these can rely on the platform's documented automatic language detection + response matching, directed by an explicit instruction line ("respond in the language the customer is using").

**Example (illustrative, not sourced from the guarded `resolve_claim` file):**
```python
EXPLANATION_TEMPLATES = {
    "en": "Good news - that {issue} comes to ${amount}, which is under the ${cutoff} I can approve on the spot, so I can approve that for you right now. Your excess is ${excess}, and your reference is {claim_ref}.",
    "es": "Buenas noticias - {issue} tiene un costo de ${amount}, que está por debajo del límite de ${cutoff} que puedo aprobar de inmediato, así que puedo aprobar esto ahora mismo. Su deducible es de ${excess}, y su referencia es {claim_ref}.",
}
lang = context.state.get("preferred_language", "en")
explanation = EXPLANATION_TEMPLATES[lang].format(...)
```
This preserves the `dataMapping` `textResponse ← explanation` mechanism exactly as shipped in chat v9 — the field mapping is language-agnostic, it just copies whatever string the tool returns [VERIFIED: `260810-k5x-SUMMARY.md`, corroborated by direct inspection of the CES discovery doc's `dataMapping`/`FieldMapping` shape].

### Pattern: agent-as-tool, confirmed live

**What:** A tool resource of type `agentTool: {name, agent: <agent resource name>}`, `executionType: SYNCHRONOUS`, attached to a calling agent's `tools` array exactly like a Python tool. The target agent has its own instruction and its own (possibly empty, here just `end_session`) tool list, and returns its final text as the tool's synchronous response to the calling agent — it never talks to the customer directly.

**Confirmed structure, read directly from `9ae7a0c3` (source app, do not deploy):**
```json
{
  "displayName": "generate_case_summary",
  "executionType": "SYNCHRONOUS",
  "agentTool": {
    "name": "generate_case_summary",
    "agent": "projects/insurance-agent-demo-500614/locations/us/apps/9ae7a0c3-.../agents/eeb8edfd-97d3-42c7-ac84-dc5e89cd8000"
  }
}
```
The target agent (`case_summary`) reads eleven named session variables (`customer_name`, `policy_id`, `covered_device`, `issue_category`, `dx_answers`, `claim_amount`, `deductible`, `total_loss_flag`, `decision`, `rules_fired`, `photo_status`), branches on `decision` (`AUTO_APPROVE` → six-sentence customer email; `HUMAN_REVIEW` → the six-section briefing packet), and its `<constraints>` explicitly forbid inventing any figure, date, or reference — the same anti-hallucination posture as `resolve_claim`. [VERIFIED: read directly via `agents.get` on the live fork app, 2026-08-10]

**The gap:** `claim_intake`'s own instruction in the fork calls this tool and then says *"Do not read it out to the customer — it is for the file. Carry on and close warmly."* — there is no further step. The text goes nowhere. **Porting `case_summary` alone does not satisfy success criterion 3.** A new delivery tool and a new instruction step are both required.

### Anti-Patterns to Avoid

- **Letting the model compose the Spanish decision explanation live.** This is the exact class of bug five days of hardening (v7→v9) was spent eliminating for English. Do not reopen it for Spanish by relying on `enableMultilingualSupport` + instruction prose alone for deterministic figures.
- **Reusing a cached/exported zip for either app without a fresh `exportApp` first.** Confirmed twice already in this project (`260810-ifr`, `260810-k5x`) — `apps.tools.patch`/`apps.agents.patch` calls silently make every previously exported package stale, and a stale `importApp` will revert unrelated recent fixes.
- **Assuming a single Resend call can multi-recipient its way around the sender restriction.** Not tested in this research (would require touching `resolve_claim`'s guarded source), but the documented failure mode (403 to any non-owner address on the shared `onboarding@resend.dev` sender) is a sender-level restriction, not a per-call recipient-count restriction — assume it blocks a `to` list containing the assessor's address just as it blocked a single second address. Treat this as `[ASSUMED]` pending a narrow live test.
- **Porting the photo-gate logic (`PHOTO_REQUIRED`, `assess_screen_crack`) to voice.** GTP is audio-only; there is no upload surface. Voice's `resolve_claim` correctly has no photo gate today — keep it that way.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Detecting what language the customer is speaking (chat) | A custom language-sniffing tool/regex | Platform automatic language detection + response matching (documented GA capability) | Native, already covers "any language supported by Vertex AI Gemini models" for chat |
| Per-language TTS voice selection (voice) | Custom audio post-processing | `audioProcessingConfig.synthesizeSpeechConfigs` map, keyed by language code | Native API field, confirmed live in the discovery doc; the platform already does root-code fallback matching |
| Cross-app tool/agent sharing | A hand-rolled sync script that copies tool JSON between the two apps on every change | Nothing built-in exists — accept duplication as the model, and script the copy carefully (fresh export, explicit diff, per-app version cut) rather than trying to invent a "shared library" the API does not have | Confirmed absent from the CES API discovery doc (no `SharedTool`/library resource type of any kind) |

**Key insight:** The platform's multilingual primitives are real and solid at the *transport* layer (detect language, respond in kind, synthesize the right voice) but stop at the boundary of anything this project has made deterministic. Every deterministic string in this build is now a translation problem the platform will not solve — budget for it as content-authoring work, not configuration work.

## Common Pitfalls

### Pitfall 1: Assuming `enableMultilingualSupport: true` translates the app for you
**What goes wrong:** Flip the flag, ship the demo, and the decision explanation, the email, and the assessor packet all still come out in English because they are literal Python string templates or field-mapped tool outputs.
**Why it happens:** The flag's own description is genuinely about input handling — *"agents in the app will use pre-built instructions to improve handling of multilingual input"* [VERIFIED: CES API discovery doc, `LanguageSettings.enableMultilingualSupport`] — not output translation.
**How to avoid:** Treat the flag as necessary-but-far-from-sufficient. The bilingual-template work in the tools is the actual deliverable.
**Warning signs:** A live Spanish-language conversation still prints English figures in the decision line or the email.

### Pitfall 2: Assuming "es-US voice is GA" means "mid-call language switching is GA"
**What goes wrong:** Scripting a demo beat where the presenter switches from English to Spanish mid-call and expecting the agent to follow, based on the press-reported "audio-to-audio translation" capability.
**Why it happens:** `es-US` being GA (confirmed) is a fact about *voice synthesis availability*, not about *live translation or mid-conversation switching*. Those remain press-only claims (Constellation Research, TTEC Digital, CMSWire, all reporting the same Jan 2026 announcement) that could not be located anywhere in the current `docs.cloud.google.com` CX Agent Studio technical reference — including the Languages reference page itself, which was fetched directly for this research and contains zero mentions of "audio-to-audio" or live translation.
**How to avoid:** Script the fallback the project's own prior research already named correctly: the agent responds in whichever supported language it detects per turn (documented, GA), not a branded live-translation feature.
**Warning signs:** The plan or the runbook asserts "the agent translates my English into Spanish live" rather than "the agent switches to speaking Spanish once it detects Spanish input."

### Pitfall 3: The `Conversation.languageCode` field may mean "one language per call," not "current turn's language"
**What goes wrong:** Building a demo beat that depends on the agent switching languages *mid-call*, only to find in front of an audience that the language locks in at the first detected utterance and does not change again.
**Why it happens:** The CES API models `languageCode` as a single read-only field on the `Conversation` resource (*"Output only. The language code of the conversation."*) [VERIFIED: CES API discovery doc, `Conversation.languageCode`] — singular, resource-level, not obviously a per-turn value. This directly contradicts the more optimistic reading of the Agents doc's *"they will automatically switch languages to match the user input"* sentence, and this research had no way to reconcile the two without placing a live call.
**How to avoid:** This is the single highest-priority open item for a Wave 0 live smoke test before committing to any "watch it switch mid-call" demo beat. If it doesn't switch mid-call, the fallback is still solid: two separate conversations, one per language, run back-to-back (same pattern already used for Scenario A/B contrast).
**Warning signs:** Cannot be detected without a live test; this is a genuine open question, not a hypothesis this research could confirm or deny.

### Pitfall 4: Treating the assessor packet as done once `case_summary` is ported
**What goes wrong:** Porting `case_summary` + `generate_case_summary`, testing that it produces the right text, and considering success criterion 3 met.
**Why it happens:** The fork's own instruction explicitly tells `claim_intake` not to surface the packet anywhere ("it is for the file"). The roadmap's success criterion specifically requires it be **"delivered as a real message,"** which the fork never does.
**How to avoid:** Plan the delivery tool (new, Resend-based, modeled on `send_claim_email`) as a first-class task, not an afterthought of porting.
**Warning signs:** A plan whose tasks stop at "port `case_summary`" without a distinct "build and wire a delivery tool" task.

### Pitfall 5: Rich-content widgets are documented English-only, and this build's cards are custom text
**What goes wrong:** Assuming the decision card and cross-sell buttons will just work in Spanish because the text fields (`productItem.subtitle`, the cross-sell `content`/`utterance`) are freely supplied strings, not platform templates.
**Why it happens:** The official web-widget documentation states, verbatim, in its Limitations section: *"Presently, rich content responses only support English."* [CITED: `docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/deploy/web-widget`, fetched 2026-08-10]. It is genuinely unclear from the doc whether this refers to the SDK's own hardcoded chrome (confirmed elsewhere to be English string literals — `"Subtotal"`, `"Sales tax"`, etc., disassembled directly from `chat-messenger.js` v1.16 in `05-02-SUMMARY.md`) or something broader that would also affect our custom `productItem`/`quick_actions` string fields.
**How to avoid:** This build's card already avoids the SDK's own hardcoded labels (deliberately dropped `costBreakdown`), so the risk is narrower than it looks — but it is not zero, and it cannot be resolved by more reading. Budget one live verification pass (deploy a Spanish-content card, look at it in the browser) before considering 05-04 done on the chat side.
**Warning signs:** A Spanish-language card renders with English labels ("Sales tax," "Add it" if a stale English button somehow leaks through) mixed into Spanish sentences.

## Code Examples

### Confirmed `LanguageSettings` shape (App-level)
```json
// Source: CES API discovery doc, https://ces.googleapis.com/$discovery/rest?version=v1beta
{
  "defaultLanguageCode": "en-US",
  "supportedLanguageCodes": ["es-US"],
  "enableMultilingualSupport": true
}
```
Both apps today have only `{"defaultLanguageCode": "en-US"}` set — confirmed by direct `apps.get` on both `6e01e4a5` (voice) and `a2f621e4` (chat), 2026-08-10. No `supportedLanguageCodes`, no `enableMultilingualSupport`, no language-related session variable exists on either app. The "config exists" note in `05-01-SUMMARY.md` refers to nothing more than this single default field.

### Confirmed `SynthesizeSpeechConfig` map shape (voice-only)
```json
// Source: CES API discovery doc
"audioProcessingConfig": {
  "synthesizeSpeechConfigs": {
    "en-US": {},
    "es-US": {}
  }
}
```
Fallback behavior is documented explicitly: *"If the configuration for the specified language code is not found, the configuration for the root language code will be used. ... if the map contains 'en-us' and 'en', and the specified language code is 'en-gb', then 'en' configuration will be used."* [VERIFIED: CES API discovery doc, `AudioProcessingConfig.synthesizeSpeechConfigs` description]. There is currently no `es` or `es-US` root entry on the voice app at all — if the platform detects Spanish input today, TTS behavior is undefined by this project's own configuration and must be added explicitly.

## State of the Art

| Old Approach (this project's prior research, CLAUDE.md) | Current Finding (this research, live-verified) | When Changed | Impact |
|--------------|------------------|--------------|--------|
| "28 voice-language variants (GA + Preview)" | The live Languages reference page currently lists **32** variants | Unknown exact date; platform has been iterating monthly per its own release notes | Cosmetic — the GA subset (en-US, fr-CA, fr-FR, de-DE, pt-BR, es-ES, es-US — 7 languages) is unchanged. `es-US` GA status is reconfirmed, not newly discovered. |
| "Config exists in both chat apps" (05-01-SUMMARY) for Spanish | Only `defaultLanguageCode: en-US` is set on either app — no other multilingual config exists anywhere | N/A — this is a correction of an imprecise prior note | Materially changes 05-04 scope: there is no partial multilingual scaffolding to build on; it starts from zero |
| Audio-to-audio translation assumed to be a documented CX Agent Studio feature | Still absent from the Languages reference page and every other current CX Agent Studio doc checked in this research | Unchanged since CLAUDE.md's original finding | No change in recommendation: press-only, do not script as guaranteed |

**Deprecated/outdated:** Nothing found to be actively deprecated in this research; the platform is young (GA since 2026-02-04) and the gaps here are "not yet documented," not "removed."

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | A Resend `to` list containing a second, non-owner address would be rejected the same way a single second address was (403) | Anti-Patterns / Pitfall discussion of B3 | If wrong, a genuinely separate assessor mailbox might be reachable in one call without a verified domain, which would remove the need for the single-mailbox compromise — worth a cheap live test that does not require reading `resolve_claim`'s source (e.g., checking Resend's own account dashboard/API docs for the shared-sender policy) before committing to the single-mailbox plan |
| A2 | The "rich content responses only support English" limitation applies broadly enough to be a real risk to this build's custom `productItem`/`quick_actions` string fields, not just the SDK's own hardcoded chrome labels | Pitfall 5 | If the limitation only affects platform-native templates (not our free-text fields), 05-04's chat-side risk is much smaller than flagged here — resolved only by a live browser test with Spanish content |
| A3 | `enableMultilingualSupport`'s "pre-built instructions to improve handling of multilingual input" does not also improve *output* composition quality for free-text (non-deterministic) lines | Pattern: bilingual deterministic tools | If wrong, less custom instruction-authoring may be needed for the free-text Spanish lines than assumed — lower-risk to be wrong in this direction (would just mean less work, not a broken demo) |

## Open Questions

1. **Does an app's language actually switch mid-conversation, or only at conversation start?**
   - What we know: the Agents doc says agents "automatically switch languages to match the user input"; the API models `Conversation.languageCode` as a single field.
   - What's unclear: whether that field updates turn-by-turn or is fixed once per conversation.
   - Recommendation: Wave 0 live smoke test — start an English conversation, switch to Spanish mid-call (chat is cheaper/faster to test than voice), and directly observe whether the response language follows. Do this before writing any plan task that assumes live mid-call switching as the demo mechanism; have the two-separate-conversations fallback ready either way.

2. **Does the shared Resend sender reject a second recipient in a `to` list the same way it rejected a second single-recipient send?**
   - What we know: a single non-owner address 403'd.
   - What's unclear: multi-recipient behavior specifically, and whether Resend's own dashboard documents this restriction more precisely than what this project discovered by testing.
   - Recommendation: A five-minute check of the Resend dashboard/docs (no code, no source-reading) before committing to the single-mailbox compromise in the plan.

3. **Does the "rich content responses only support English" limitation block Spanish content in a custom `widgetTool`'s own declared string fields?**
   - What we know: the limitation is stated in the web-widget doc's Limitations section, unqualified.
   - What's unclear: scope — platform templates vs. our own payload strings.
   - Recommendation: One live deploy-and-look test with a Spanish `productItem.subtitle` before considering 05-04 complete on chat.

4. **Does adding an `es-US` entry to `synthesizeSpeechConfigs` plus `enableMultilingualSupport` produce an audibly different (Spanish) voice on a live call, or does GTP need additional per-number/per-deployment configuration?**
   - What we know: the schema supports the config; nothing in this research's read-only pass could exercise a live phone call.
   - What's unclear: whether GTP's deployment layer needs its own language setting beyond the app-level config.
   - Recommendation: First live-call smoke test after the config change, before scripting a presenter beat around it.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `gcloud auth print-access-token` / CES API access | All API-based build and verification work | ✓ (used throughout this research) | v1beta CES API, project `insurance-agent-demo-500614` | — |
| CES API discovery doc | Confirming exact request/response shapes before writing patch scripts | ✓ (fetched successfully) | `v1beta` | — |
| Resend account/dashboard access | Verifying the B3 multi-recipient question (Open Question 2) | Not checked in this research (would require going outside the CES API) | — | Treat as `[ASSUMED]` until checked |
| Local web-widget rig (`http://localhost:3000`, token broker, chat-messenger SDK v1.16) | Any chat-side live visual verification (Open Question 3) | Reported running/available per `260809-nt7`/`260810-k5x` summaries; not independently re-verified in this research | v1.16 | — |
| Live phone number for voice app | Any voice-side live verification (Open Questions 1, 4) | Not checked in this research (read-only API scope) | — | Simulator/`runSession` API calls as a lower-fidelity fallback, per prior project practice |

**Missing dependencies with no fallback:** None — all blocking verifications have a live-test path available to the executing team; this research simply could not exercise them under a read-only constraint.

## Cross-App Convergence — Confirmed Mechanics

### C1 — No shared-resource mechanism exists
Directly inspected the CES API discovery document's full resource and schema listing: **zero** resource types or schemas contain "shared," "library," "template," or "clone" in their names. [VERIFIED: `https://ces.googleapis.com/$discovery/rest?version=v1beta`, full resource/schema enumeration, 2026-08-10]. Combined with WebSearch findings describing export/import as "duplicate and customize," "export and share collections with your team" — the platform's own vocabulary confirms this is a copy-paste model, not a live link. **There is no way to make voice and chat reference the same tool/agent definition; every port is a duplication that must be kept in sync by hand.**

### C2 — The safest mechanism, given the already-documented stale-export trap
Two of the last three chat edits (`260810-ifr`, `260810-k5x`) used **direct `apps.tools.patch` / `apps.agents.patch`** rather than export→edit→import, specifically because that pattern:
1. Requires no export at all (no staleness risk),
2. Lets a script verify the exact pre-state via `GET`, apply a narrow `PATCH` with an `updateMask`, and read back byte-for-byte,
3. Avoids the `REPLACE`-mode import behavior that does not clean up removed fields cleanly (confirmed separately in `260806-u21`: `importApp` did not drop a removed variable declaration; a follow-up explicit `apps.patch` on `variableDeclarations` was required).

**Recommendation for porting `case_summary`/`generate_case_summary` and the Spanish bilingual templates to voice:** use the same direct-patch, read-verify-patch-verify discipline already proven twice on this project, rather than export/import. If bulk copying an entire new agent (`case_summary`) from the fork to voice, the cleanest path is: `agents.get` the source agent from `9ae7a0c3` (read-only, already done in this research), construct the equivalent `agents.create` request body against the voice app directly, `POST`, then read back. This sidesteps export/import entirely for the one case where a genuinely new agent resource is needed.

### C3 — What is and is not shareable across channels
| Capability | Voice (GTP) | Chat (Web widget) | Shareable? |
|---|---|---|---|
| Photo upload / `assess_screen_crack` | Not applicable — GTP is audio-only | Already built | No — and should not be ported to voice |
| Decision card / `cover_offer_actions` widgets | Not applicable — no visual surface | Already built | No — voice conveys the same information purely via speech, which it already does correctly |
| `resolve_claim`'s deterministic `explanation` (content) | Needs bilingual templates | Needs bilingual templates | **Yes** — same underlying string-selection logic, ported by hand (each app has its own tool copy) |
| `case_summary` agent + `generate_case_summary` agent-as-tool | Needs porting | Needs porting | **Yes** — identical agent-as-tool pattern works on both; GTP has no visual constraint against it since it's a background/internal artifact, not a customer-facing rendering |
| New assessor-packet delivery tool (Resend) | Needs building | Needs building | **Yes** — same HTTP-call pattern as the existing `send_claim_email`, once per app |
| `preferred_language` session variable + `languageSettings` config | Needs adding | Needs adding | **Yes** — same shape, `apps.patch` on each app separately |

## Security Domain

Minimal applicability — this is an internal demo-build phase with no new external-facing surface, but two items carry over from prior phase work and are worth restating for the planner:

| Known Item | Control |
|---|---|
| Resend API key embedded in tool source (`resolve_claim`/`send_claim_email`), and now potentially a second delivery tool for the assessor packet | Do not read, echo, or copy the key in any plan artifact or executor session (per this research's own operating constraint); the existing DEMO-RUNBOOK.md "revoke the key after the demo" note applies equally to any new delivery tool |
| Synthetic PII only (policy records, names, emails) | Already project-standard; the assessor packet template must not introduce any new field sourced from anything other than the existing seeded mock data |
| Bilingual template content is presenter-facing, not a trust boundary | No injection/validation concern beyond what already applies to the English templates — `preferred_language` should be constrained to a small enum (`"en"`/`"es"`) in code, not free text, to avoid a `KeyError`-class failure live on stage |

## Sources

### Primary (HIGH confidence — read directly via API or fetched from current official docs)
- CES API discovery document, `https://ces.googleapis.com/$discovery/rest?version=v1beta` — full schema/resource enumeration, `LanguageSettings`, `AudioProcessingConfig`, `SynthesizeSpeechConfig`, `Conversation.languageCode`, confirmed absence of any shared-tool/library resource type
- Live `apps.get` / `agents.get` / `tools.get` / `deployments.get` calls against project `insurance-agent-demo-500614`, both apps (`6e01e4a5` voice, `a2f621e4` chat) and the read-only reference fork (`9ae7a0c3`) — 2026-08-10
- [Languages reference](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/reference/language) — fetched directly, 32-variant table, GA/Preview status, chat-language statement, confirmed absence of "audio-to-audio" anywhere on the page
- [Web widget deployment](https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/deploy/web-widget) — fetched directly, exact Limitations-section quote on rich content English-only

### Secondary (MEDIUM confidence — WebSearch, cross-referenced)
- WebSearch on CX Agent Studio export/import mechanics — corroborates "duplicate/copy" model, no live-link sharing
- WebSearch on `LanguageSettings`/`enableMultilingualSupport` — corroborates the discovery-doc-derived schema

### Tertiary (LOW confidence — press-only, explicitly flagged, unchanged from prior project research)
- Constellation Research, TTEC Digital, CMSWire coverage of the Jan 2026 announcement — audio-to-audio translation, ~10 languages, mid-call switching. Not found in any current official CX Agent Studio doc checked in this research. Treat exactly as CLAUDE.md already does: do not script as a guaranteed live demo moment.

## Metadata

**Confidence breakdown:**
- Spanish platform mechanics (config schema, GA language list): HIGH — verified live against the API and current docs
- Spanish content/determinism strategy: MEDIUM — the bilingual-template recommendation is sound architecture but untested live
- Mid-call language switching: LOW — genuinely unresolved, flagged as the top open question
- Assessor packet porting mechanics: HIGH — read directly from the live source agent
- Assessor packet delivery / Resend constraint: MEDIUM — the core constraint is already confirmed by prior project testing; the multi-recipient question is untested
- Cross-app convergence mechanics: HIGH — confirmed by direct discovery-doc inspection, not inference

**Research date:** 2026-08-10
**Valid until:** ~14 days (platform is GA-young and iterating monthly; live-API findings should be re-confirmed if planning is delayed past two weeks)
