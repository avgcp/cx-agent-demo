---
phase: 05-demo-build-multimodal-backend-reveal-multilingual
plan: 02
status: incomplete
date: 2026-08-06
covers_success_criteria: []
---

# 05-02 — Decision card + quick actions (IN PROGRESS)

## Built

Two `widgetTool`s on the chat app `a2f621e4`, deployed as chat v3 `657ecb18`:

| Tool | Type | Wiring |
|---|---|---|
| `claim_decision_card` | `ORDER_SUMMARY` | `FIELD_MAPPING` from `resolve_claim`, `textResponseConfig: NONE` |
| `cover_offer_actions` | `QUICK_ACTIONS` | Model-supplied prompt + Add it / Not now |

## Discovery notes (no docs, no examples)

`uiConfig` is an undocumented bare object in the API schema and no widget tool existed
anywhere in the project to copy. `retrieveToolSchema` (v1 and v1beta, POST to
`{app}:retrieveToolSchema` with a tool resource name) was the way in: a widget returns
`{summary, widget_tool_status, payload}`, and the `payload` shape is whatever is declared in
the tool's own `parameters`. Reusable for any future widget.

Also learned the hard way: widget tools created directly via the API exist **only**
server-side. The local source had to be refreshed via `exportApp` before the next
`importApp`, which would otherwise have silently deleted them.

## Confirmed working

- `claim_decision_card` is invoked and receives the correct payload, verified on a live run
  (`CLM-24689`, keyboard replacement, $420, $25 excess, cutoff $1,500).
- The platform tags it `"type": "order_summary"`, so the widget is correctly formed.

## Not confirmed

- **Rendering.** The console Preview panel shows *"Agent response contained a custom
  payload"* rather than drawing a card. Consistent with rich response widgets being
  web-widget-only (`CLAUDE.md` §Tools). Needs a test through the real embed on deployment
  `d7bfbb93` — not Preview.
- **`cover_offer_actions` has never fired.** The tool exists and is wired, but no test run
  has reached the cross-sell step cleanly.

## Defect found during testing — NOT yet fixed

**The wrong-subject guard does not work.** A photo of a **laptop** was accepted and
auto-approved against **PDP100583** (Maria Santos, iPhone 16 Pro Max) — payload shows
`device: Apple iPhone 16 Pro Max`, `auto_approval_cutoff: 750`, `claim_amount: 420`, i.e.
28% of the iPhone's $1,500 value.

`assess_screen_crack` has a `WRONG_SUBJECT` branch for exactly this, but its
`image_shows_device` parameter asks the **model** to decide whether the photo matches the
policy. The model answered "yes" while looking at a laptop on a phone policy.

This is the same failure mode designed out everywhere else in this build during v1→v11: a
judgment call left with the model rather than enforced in code. It was inherited when the
tool was ported from `9ae7a0c3` and its interface was not re-examined.

**Proposed fix**: replace `image_shows_device` (a comparison) with `device_in_photo`
(an observation — `laptop` | `phone` | `other` | `unclear`) and compare it against
`device_category` from the policy record inside the tool. The model reports what it sees;
the tool decides whether it matches. It demonstrably describes images well — it produced
"several cracks branching across the display" unprompted — but is unreliable at holding two
facts in mind and comparing them.

## Verified separately

The vision read itself is good, on two real photos: *"I can see several cracks across the
screen"* and *"I can see several cracks branching across the display"*.
