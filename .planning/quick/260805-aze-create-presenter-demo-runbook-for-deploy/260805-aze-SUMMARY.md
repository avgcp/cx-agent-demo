---
quick_id: 260805-aze
status: complete
date: 2026-08-05
files_modified: [.planning/spec/DEMO-RUNBOOK.md]
---

# Quick Task 260805-aze — Summary

## What was built

`.planning/spec/DEMO-RUNBOOK.md` — a presenter runbook for the deployed voice agent.
Operational, not specification: dial-in details, two verbatim call scripts, the seeded
policy table, presenter notes, rollback ladder, known cosmetic issues, post-demo cleanup.

## Deviation from the standard quick workflow

Executed inline rather than via `gsd-planner` + `gsd-executor` subagents. The operator
instruction in force for this session prohibits spawning subagents that were not explicitly
requested, and the task was a single document whose content was already established. All
quick-mode artifacts (PLAN, SUMMARY, STATE row, atomic commits) were produced as normal.

## Source of truth

Figures were taken from the **deployed tool source**, not from the Mock Data Appendix (§4).
The PoC diverged from that appendix — tariff-based pricing, a 50%-of-coverage threshold, no
photo upload — so §4's $1,000 threshold, $100 deductible and $3,500 limit are all stale
relative to what a caller actually hears. Cross-checked against the live app:

- 6/6 seeded policies match on coverage, cutoff and excess
- MacBook screen tariff = 28% of $3,000 = **$840** ✓
- Total loss = full device value = **$3,000** ✓

## Discovered mid-task: the deployment had moved

The runbook was drafted against v7 `718b6fb3`. Verification against the live deployment
found someone had cut **v8 `a435521a` "switched demo recipient"** at 12:44 and repointed the
deployment. The runbook was corrected to v8 before completion.

Diffed v8 against v7: **identical except the recipient constant.** Bounce-back guard, digit
rescue and send-once guard all present; the double-speaking narration stripper absent from
both. So v8 carries every v7 fix.

## Open risk recorded in the runbook

The v8 recipient is `aniket.kumar@nerdery.com`, but the sender is Resend's shared
`onboarding@resend.dev`, which on a free account only delivers to the **account owner's**
address. If the Resend account belongs to `akash.vinayak@`, mail to `aniket.kumar@` is
rejected and the agent falls back to a drafted message — the call still completes, but no
email arrives. Live delivery was only ever proven against `akash.vinayak@`.

Not verified here: doing so would mean sending an unsolicited message to a third party. The
runbook carries a five-second curl check and the rollback-to-v7 remedy.

## Verification

- Acceptance greps pass (phone blank, all three resource IDs, both scenarios, six policies,
  v6 warning, key-revocation note)
- Every figure cross-checked against deployed tool source — no value taken on trust
