---
quick_id: 260809-nt7
type: quick
status: complete
date: 2026-08-09
subsystem: planning-docs
tags: [documentation, verification-record, state-md, roadmap, runbook, chat-app]
files_modified:
  - .planning/STATE.md
  - .planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md
  - .planning/ROADMAP.md
  - .planning/spec/DEMO-RUNBOOK.md
---

# 260809-nt7 — Record live verification of decision-card rendering and photo upload through the real web widget

Documentation-only task. Recorded a single observed run — the user driving a full chat claim
through the real, console-generated chat-messenger SDK v1.16 widget embed — across the four
planning documents that still described three of its findings as open. No CES API call,
deployment change, or agent edit was made in this task; `git status` confirms all changes are
confined to `.planning/`.

## What was OBSERVED (the three closures)

On 2026-08-09 the user drove a full claim through the real widget embed at
`http://localhost:3000` (token broker, chat-messenger SDK v1.16), against deployment
`d7bfbb93-8cee-43fe-9095-bc5775f353bd` serving chat v7 `bb14cdcc-d723-4be1-85af-9f4451e22ed5`
of app `a2f621e4-9faf-505a-b804-22471f022366`, using policy `PDP100294` (Jordan Rivera, Apple
MacBook Pro 16").

1. **Widget rendering.** The decision card drew: title `Apple MacBook Pro 16"` with
   right-aligned price `$840.00`; subtitle `Screen replacement · CLM-24442 · $840 less $25
   excess = $815 to you · Approved on the spot (on-the-spot limit $1,500)`. The two cosmetic
   artifacts predicted by `260809-n1b` appeared (empty grey image tile at left, two trailing
   divider rules) and none of the five failure modes the SDK contract warned about occurred
   (no `$4,200.00`, `$NaN`, `$0.00` row, or broken-image glyph).
2. **Photo upload through the real web widget.** A real cracked-MacBook photo was uploaded
   through the browser, displayed inline, and read correctly by the model ("I can see the
   cracks on the screen there."). Previously this path had only been proven via the API and
   the simulator. A prior worry — that browser upload might need Cloud Storage bucket
   configuration — was disproven: `enable-file-upload` alone was sufficient.
3. **The live end-to-end chat claim through the real embed.** The full conversation (name +
   policy → device confirmation → damage description → photo → vision read → card →
   auto-approval) ran through the real deployed widget, not the simulator or the API.

## What remains OPEN (the four items — none softened or closed)

1. **The decision is not spoken.** The agent said only "I can see the cracks on the screen
   there." and produced the card with no sentence stating the decision — confirmed again on
   this run. Recorded as an open demo-quality issue with a suggested (not applied) direction:
   `textResponseConfig: {type: "LLM_GENERATED", ...}` plus resolving `claim_intake`'s
   self-contradictory instruction block.
2. **`cover_offer_actions` is still broken and has never fired.** Left entirely unchanged in
   STATE.md, as instructed — still declares `{prompt, options}` against an SDK that reads
   `{actions: [{content, description, utterance}]}`, and still throws on `.map()` rather than
   degrading gracefully.
3. **The photo CONTRADICTION path is still not verified live.** This run was the CONFIRM path
   only. The `260806-u21` STATE.md entry's "NOT YET VERIFIED LIVE" note was narrowed, not
   removed — it now applies specifically to the contradiction path, proven offline in
   `phototest2.py` only.
4. **The accepted limitation stands.** Photo assessment confirms damage, not device identity.
   This run used a matching device/policy pair, as required; the runbook's matching-pair
   callout was left untouched, per the plan's constraint.

## Where a document disagreed with this plan's premises

The plan's Task 3 anticipated the runbook might need a *new* chat section. On reading the
file, `DEMO-RUNBOOK.md` already had a full `# Chat channel — the photo demo` section
(Scenario C confirm, Scenario D contradiction, the matching device/policy-pair callout, the
seeded-policy table) — so this was an **update of an existing section**, not new authoring.
The plan itself flagged this correction in its Task 3 instructions, and the file matched that
expectation exactly; no other disagreement was found.

## Changes by file

**`.planning/STATE.md`**
- Widget-rendering blocker: leading status changed from "DIAGNOSED AND FIXED, awaiting the
  user's eyes" to "VERIFIED LIVE (2026-08-09, quick tasks 260809-n1b diagnosis + fix,
  260809-nt7 verification)"; `WHAT REMAINS` replaced with the verification record (card text,
  cosmetic artifacts, absent failure modes, CONFIRM-path note). ROOT CAUSE/CORRECTION history
  preserved verbatim.
- `260806-u21` accepted-limitation entry: final clause narrowed from a blanket "NOT YET
  VERIFIED LIVE against chat v6" to state the CONFIRM path is now verified live on chat v7,
  with the CONTRADICTION path remaining unverified. STANDING DEMO CONSTRAINT sentence
  untouched.
- New resolved-concern bullet added (Cloud Storage / `enable-file-upload` disproven concern).
- Silent-decision-turn entry: appended confirmation from this run, the demo-quality
  consequence, and the suggested (unapplied) direction. Stays open.
- `cover_offer_actions` entry: left completely unchanged, as instructed.
- Frontmatter `last_updated`, `last_activity`, body `Last activity:` and `Plan:` lines
  updated; new Quick Tasks Completed row added; progress counters untouched.

**`.planning/phases/.../05-02-SUMMARY.md`**
- New `## Rendering — CONFIRMED BY EYE 2026-08-09 (quick task 260809-nt7)` section added
  after the "What was built instead" heading, recording the verification evidence and stating
  the SDK contract is now verified-by-observation.
- "Still open on 05-02" list: first bullet (render unconfirmed) struck through and marked
  CLOSED with a pointer to the new section; contradiction bullet narrowed to name the confirm
  path as verified and the contradiction path as the sole remaining gap; a justification
  sentence added naming `cover_offer_actions` as the primary reason the plan stays
  `status: incomplete` and the silent decision turn as a lesser reason.
- Frontmatter `status: incomplete` — unchanged, now explicitly justified in the file body, per
  the hard constraint.

**`.planning/ROADMAP.md`** (Phase 5 only)
- 05-02 plan line: no longer claims rendering is unverified; states it is built, deployed and
  rendering-verified live with the `CLM-24442` figures; explains what keeps it `[~]`
  (`cover_offer_actions` + silent decision turn). Checkbox stays `[~]`.
- Success criterion 1: dated verified-live parenthetical appended; substance unchanged.
- Success criterion 4: file-upload-proven-with-no-storage-config note appended; phone
  deployment pin noted. Substance unchanged.
- Criteria 2 and 3 left untouched, as instructed (2 = contradiction path still unverified; 3 =
  05-03 scope, out of this run's evidence).
- "Built so far" line: version pointer corrected from stale `56a8b22a` (chat v6) to
  `bb14cdcc-d723-4be1-85af-9f4451e22ed5` (chat v7), with the rendering-verified note.

**`.planning/spec/DEMO-RUNBOOK.md`** (chat section only)
- Fact table: `Version live` corrected to chat v7 `bb14cdcc`; `Widget embed` filled in
  (console embed panel, token broker, `http://localhost:3000`, SDK v1.16,
  `enable-file-upload`, no Cloud Storage/`url-allowlist` needed).
- "Verified on a real photo" callout upgraded from simulator-credited to the real-widget run,
  with the exact observed dialogue and figures.
- Scenario C: added a card row to the turn table, the exact expected on-screen card text, an
  "Expected, not a bug" note for the grey tile and two dividers, a presenter warning that the
  agent likely says nothing (read the card aloud), and a note that terse input worked.
- Scenario D: added a prominent unverified-live caveat (proven offline in `phototest2.py`
  only, never run live) directly under the heading, with a rehearsal recommendation. Rest of
  the scenario, including the DL-5 note and non-accusatory quote, unchanged.
- Known cosmetic issues: two chat-channel entries added (grey tile + dividers, silent decision
  turn).
- Matching device/policy-pair callout: left exactly as it was, per the plan's constraint.

## Self-Check: PASSED

- `.planning/STATE.md` — modified, all required strings present (verified by grep in Task 1)
- `.planning/phases/05-demo-build-multimodal-backend-reveal-multilingual/05-02-SUMMARY.md` —
  modified, `status: incomplete` preserved with written justification (verified by grep)
- `.planning/ROADMAP.md` — modified, 05-02 line no longer claims unverified rendering, stays
  `[~]`, criteria 2/3 untouched (verified by grep)
- `.planning/spec/DEMO-RUNBOOK.md` — modified, chat section runnable with version, embed
  instructions, expected card text, and both warnings present (verified by grep)
- `git status --short` shows changes confined to `.planning/` plus two pre-existing untracked
  files outside this task's scope (`GE CX Demo - Notes.docx`, `meeting_context.md`)
- No overstatement found: none of the four files claim `cover_offer_actions` works, that the
  contradiction path is verified live, or that the agent speaks the decision
- No git commit made by this task, per instruction — the orchestrator owns the docs commit
