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

## Defect found during testing — TWO FIXES ATTEMPTED, BOTH FAILED

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

**Fix applied** *(chat v4 — verified offline only; it failed live and no longer exists in the
build, see Resolution below)*: replaced `image_shows_device` (a comparison) with `device_in_photo`
(an observation — `laptop` | `phone` | `other` | `unclear`) compared against `device_category`
from the policy record inside the tool. Verified: laptop-on-phone and phone-on-laptop both
rejected; matching device still approves at $840 / $420; wrong object, unclear, contradiction
and bad-enum cases all still behave. Five stale assertions in phototest.py were migrated —
they passed the old yes/no values and would have started silently returning INVALID_VALUE. The model reports what it sees;
the tool decides whether it matches. It demonstrably describes images well — it produced
"several cracks branching across the display" unprompted — but is unreliable at holding two
facts in mind and comparing them.

## Verified separately

The vision read itself is good, on two real photos: *"I can see several cracks across the
screen"* and *"I can see several cracks branching across the display"*.


---

# Wrong-subject guard: removed by decision (resolved 2026-08-06, chat v6 `56a8b22a`)

*Sections below record the investigation as it stood at chat v5 `5ccd34a4`. The
**Resolution** section at the end records the decision taken and what the build does now.*

## The finding

**The photo assessment cannot verify *which device* is in the image, and this is not
fixable by prompting.** Two structural attempts both failed against live testing.

| Version | Approach | Outcome |
|---|---|---|
| v4 `7710088b` | Model reports the device category (`device_in_photo`), tool compares against `device_category` in code | Model reported `"phone"` while looking at a laptop |
| v5 `5ccd34a4` | Model reports **physical features** (`keyboard_visible`, `hinged_lid`) plus a named description; tool infers the device and compares | Model reported `keyboard_visible: "no"`, `hinged_lid: "no"`, and *"a black phone with several cracks on the screen"* |

## The evidence

`evidence-laptop-photo.png` in this directory is the exact image uploaded in
`simulator-da97c67a` against **PDP100746 (Apple iPhone 16)**. It is an unmistakable MacBook:
a full keyboard with individual keys fills the lower half of the frame, with a trackpad and
a hinged aluminium lid. It was approved at $280 as an iPhone screen repair.

Retrieved by decoding the base64 PNG stored inline in the conversation record
(`turns[n].messages[].chunks[].image.data`) — worth knowing, as it makes any past image
claim auditable after the fact.

## Root cause

The model already knows what the policy covers by the time it sees the photo, and its
report conforms to that rather than to the image. This is not a labelling problem: it
answered three independent, concrete, physical questions falsely against overwhelming
visual evidence.

Note also that the vision capability itself is **good** — the same model produced
"several cracks branching across the display" accurately on two occasions. It reports the
damage faithfully and the identity unfaithfully, because only the identity conflicts with
the surrounding context.

## Why the offline tests did not catch either attempt

Both suites supplied the observation values directly, which proved the comparison logic and
never the observation. A test that feeds the tool `device_in_photo="laptop"` cannot detect
that the model would have said `"phone"`. The suites now include the misleading inputs the
model actually produced, but that is a regression guard, not a fix.

## Options

1. **Isolate the vision call** in an agent-as-tool with no policy knowledge — it receives
   only the image and answers what it shows. Removing the context is the only structural
   fix. Unknown: whether CES gives such a sub-agent a clean context or passes conversation
   history through. If it inherits history, this fails identically. The pattern exists
   (`generate_case_summary` in `9ae7a0c3` is an agent-as-tool).
2. **Accept the limitation.** Remove the wrong-subject guard, document that photo
   verification confirms *damage* but not *device identity*, and keep demos on matching
   device/policy pairs. **A guard that does not work is worse than none** — it reads as
   protection in both the code and the runbook.

The contradiction path (crack reported, none visible) is **unaffected** and still works: it
only asks about damage, which the model reports honestly.

## Resolution — option 2, accept the limitation (user decision, 2026-08-06)

Executed as quick task `260806-u21`.

**The decision, in the user's framing:** *a guard that does not work is worse than none,
because it reads as protection in both the code and the runbook.* Option 1
(agent-as-tool vision isolation) was explicitly **not** attempted — it rests on an unverified
assumption about whether CES gives a sub-agent a clean context, and a third failed structural
attempt would buy nothing the documentation cannot state honestly today.

**What was removed** from `assess_screen_crack` (chat v6 `56a8b22a-2baf-4b18-9540-cdc6185acbee`):

| Removed | Note |
|---|---|
| `keyboard_visible`, `hinged_lid` parameters | Signature is now exactly `(crack_visible, what_you_see)` — confirmed server-side via `retrieveToolSchema` |
| The device-inference block | Laptop/phone signal counting and keyword matching, and the comparison against `device_category` |
| The `WRONG_SUBJECT` return and `photo_check="rejected"` | `"rejected"` is now unreachable; it is retained defensively in `resolve_claim`'s `photo_blocked` set with a comment saying so |
| The `photo_device_seen` session variable | Declaration removed from the app; nothing else referenced it |
| The instruction paragraph telling the agent to answer physical questions and ignore the policy | Replaced with one sentence: describe the object and the damage plainly in `what_you_see` |

A code comment at the top of the function records **why** the check is absent, so a future
reader does not "fix" the omission by reintroducing the same failure.

**What still holds** — all proven offline against the edited source in `phototest2.py`:

- The contradiction path: a reported crack absent from the photo sets `photo_contradiction`,
  returns `NO_DAMAGE_VISIBLE`, fires **DL-5**, forces `HUMAN_REVIEW`, and says nothing
  accusatory.
- The unusable-photo path: exactly one retry, then a human.
- Deterministic tariff pricing — $840 for a screen, $3,000 total loss — never from the vision read.
- The total-loss path, the bad-enum rejection, and all v1→v11 hardening.

**A junk photo now lands in `unclear`, not in a rejection.** A picture with no device or no
screen in it is caught by `crack_visible="unclear"` → one retry → human review, which is the
same end state the removed guard produced. That enum's description was deliberately
strengthened to name "no screen and no device visible in the picture at all" as one of its
cases, because it is now the only thing standing between a junk photo and an approval.

**The limitation, stated for reuse:**

> **Photo assessment confirms whether the reported damage is visible. It cannot verify that
> the photographed device is the insured device. Demos must use matching device/policy pairs.**

`phototest2.py` now asserts this as an explicit passing test — it feeds the exact false report
the model produced live (`"a black phone with several cracks on the screen"` against
PDP100294, a MacBook) and asserts the claim proceeds and is priced from the tariff at $840.
The limitation is a visible test result, not a silent absence.

**Version and deployment:** chat v6 `56a8b22a-2baf-4b18-9540-cdc6185acbee` was cut, and the
customer-facing chat deployment `d7bfbb93` was repointed to it (confirmed serving). Both
widget tools survived the export→edit→import round trip, verified in the fresh export before
editing and again server-side after the push. Voice app `6e01e4a5` untouched, still pinned to
v11 `b17c9a26`.

**Still open on 05-02** (why this plan stays `status: incomplete`): widget rendering is still
unverified through the real embed, and `cover_offer_actions` has still never fired. Separately,
the contradiction path has not been re-run live against chat v6 — it needs a photo of an
undamaged device, which the executing session did not have; it is proven offline only.
