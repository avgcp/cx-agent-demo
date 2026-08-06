---
phase: 05-demo-build-multimodal-backend-reveal-multilingual
plan: 01
status: complete
date: 2026-08-05
covers_success_criteria: [1, 2, 4]
---

# 05-01 — Chat channel with photo damage verification

## Process note

Executed without a PLAN.md. The work was done conversationally while the phase was being
defined, so this summary is written retrospectively against the phase's success criteria
rather than against a plan. Criteria 1, 2 and 4 are covered here; 3 (assessor briefing
artifact) and 5 (Spanish) are untouched.

## What was built

A **second app** — `Meridian Claim - Chat (hardened)`, `a2f621e4-9faf-505a-b804-22471f022366` —
forked from the hardened voice source (v11), not from the pre-hardening Chat fork.

| | |
|---|---|
| Deployment | `d7bfbb93-8cee-43fe-9095-bc5775f353bd` — `WEB_UI`, `CHAT_ONLY`, light theme |
| Version | `97f44790-7d15-49d3-8d58-0c216f268345` |
| Voice app | `6e01e4a5…` untouched, still v11 `b17c9a26` on the phone deployment |
| Aniket's Chat app | `9ae7a0c3…` untouched, kept as reference |

### Photo verification

`assess_screen_crack` ported from `9ae7a0c3` and adapted: it reads `covered_device` /
`device_category` from session state instead of carrying its own policy table (the same
de-duplication applied to `send_claim_email` earlier), and gained a `what_you_see`
parameter so the agent must say back what it observed — the line that proves it looked.

### Determinism preserved (success criterion 1)

The photo **cannot** produce the claim amount. Pricing stays in the tariff. Three
enforcement points, all in code rather than instructions, following the pattern established
during v1→v11 where every callback-level guard held and every prose-level one was ~50/50:

1. `before_tool_callback` — `PHOTO_REQUIRED` blocks `resolve_claim` on any screen repair
   until a photo has been assessed.
2. `resolve_claim` — photo status **outranks** the diagnostic. `contradicted` / `unclear` /
   `rejected` force `HUMAN_REVIEW` regardless of what the tree concluded.
3. `before_tool_callback` — `assess_screen_crack` is refused before the diagnostic completes.

### Anti-fraud path (success criterion 2)

Reported crack absent from the photo → `photo_contradiction = true`, `DL-5`, human review,
and a non-accusatory customer line. Two corrections made after testing:

- It was quoting *"that comes to $840"* for damage it could not verify. Now no figure is
  named on the contradiction path — naming one reads as though the claim had been valued.
- The replacement wording said "specialist" twice in one sentence.

### Chat-native instructions

The voice instruction set actively harms chat — *"Never use lists, bullets or headings —
everything you say is spoken aloud"*, *"Ask ONE question per turn"*, *"Keep sentences short
enough to say in one breath"*, *"Never read an email address aloud"*. All four are wrong in
a text channel. The global instruction was rewritten for chat and six voice-isms were
replaced in `claim_intake`. Separate apps rather than modality conditionals, per the
"voice is separate" decision.

## Verified

Offline (`phototest.py`), all passing:

- Photo confirms → $840, auto-approve, one email
- Photo contradicts → `HUMAN_REVIEW`, DL-5, no price quoted, no accusation
- Unclear photo → one retry, then human
- Wrong subject → rejected, names the covered device
- Total-loss path unaffected by photo logic

On the deployed chat channel: the diagnostic completes, the agent **asks for a photo**, and
`decision` / `claim_amount` remain unset — the gate holds.

## NOT verified — carried as a blocker

**The model's vision accuracy on a real photo.** Every branch of `assess_screen_crack` is
tested, but only by supplying the observation values directly. Whether the model correctly
reports `crack_visible` from an actual image has never been exercised, because the API can
be driven with text but not with a convincing photo of a cracked MacBook screen. If it
misreads a demo photo, Scenario C silently becomes Scenario D in front of an audience.

Requires one upload through the real widget to settle.

## Not done in this plan

- **Success criterion 3** — `case_summary` / assessor briefing packet. Deferred: the chosen
  UI scope was photo + decision card + quick actions, excluding the inline backend reveal.
- **Success criterion 5** — Spanish. Config exists in both chat apps but every
  customer-facing string is still English.
- **Decision card and quick actions** — `widgetTool` confirmed available (`ORDER_SUMMARY`,
  `QUICK_ACTIONS`, `dataMapping.sourceToolName`) but not built. Held deliberately until the
  photo path is confirmed working in the real widget, since everything else depends on it.
