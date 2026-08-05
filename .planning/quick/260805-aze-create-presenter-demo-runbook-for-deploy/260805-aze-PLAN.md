---
quick_id: 260805-aze
type: quick
autonomous: true
files_modified: [.planning/spec/DEMO-RUNBOOK.md]
---

<objective>
Author `.planning/spec/DEMO-RUNBOOK.md` — a presenter-facing runbook for the **deployed**
"Meridian Claim - Voice (demo-ready)" agent in CX Agent Studio.

This is operational, not specification. Sections 1–6 of the spec describe what the demo
*should* be; this describes how to *run* the thing that now exists on a phone line. Every
figure in it comes from the deployed build's own tool code and from verified call
transcripts, not from the Phase 1 mock-data appendix — the implementation team's PoC
diverged from that appendix (tariff-based pricing, 50%-of-coverage threshold, no photo
upload), and the runbook must match reality on the line.
</objective>

<context>
Deployed target at time of writing:
- Project: insurance-agent-demo-500614 (location `us`)
- App: 6e01e4a5-42a8-5213-b3da-c9053ff8ea52 "Meridian Claim - Voice (demo-ready)"
- Version: v7 `718b6fb3-eb4d-4b56-a7d6-39eb3f81c875`
- GTP deployment: "voice - meridian demo" `d28bbcb0-066e-4127-a894-fbf9ba39789f`

The app is a copy; the original PoC (`f5ee2711-f865-43f2-9f10-67a3c15008ab`) is untouched.
Deployments pin an app *version*, so any future edit needs a new version AND a deployment
repoint — the runbook must say so, because it is the single most likely way a future change
silently fails to reach the phone number.
</context>

<tasks>

<task type="auto">
  <name>Task 1: Author the demo runbook</name>
  <files>.planning/spec/DEMO-RUNBOOK.md</files>
  <action>
    Write `.planning/spec/DEMO-RUNBOOK.md` with these sections, in this order:

    1. **Call this number** — a fill-in blank for the GTP number (not retrievable via the
       CES or Dialogflow APIs; it lives in the console on the deployment), plus the app,
       version and deployment IDs.
    2. **Scenario A — auto-approve** and **Scenario B — escalation**: verbatim caller lines
       proven to work on real calls, each with what the agent should say back.
    3. **Seeded policies** — all six, with device, coverage, on-the-spot limit and excess;
       note every email routes to one inbox regardless of policy.
    4. **Presenter notes** — how the agent actually behaves: compound answers are fine, a
       mis-heard policy ID self-corrects from the digits, out-of-scope questions deflect,
       the close arrives as three separate turns.
    5. **If something goes wrong** — rollback via a single deployment PATCH, with the full
       version ladder and an explicit warning that v6 has an audible double-speaking bug.
    6. **Known cosmetic issues** — ear emoji in transcripts, occasional ~4s pause on the
       decision turn.
    7. **After the demo** — revoke the Resend API key.

    Keep it scannable: tables over prose, short lines, no spec-style narrative. A presenter
    reads this standing up, possibly on a phone screen.
  </action>
  <acceptance_criteria>
    - File exists with a fill-in blank for the phone number and all three resource IDs
    - Both scenarios present with verbatim caller lines and expected agent responses
    - Six-row policy table with the one-inbox note
    - Rollback section lists v1–v6 with the v6 warning
    - Post-demo key revocation noted
  </acceptance_criteria>
  <verify>
    test -f .planning/spec/DEMO-RUNBOOK.md &&
    grep -q "718b6fb3" .planning/spec/DEMO-RUNBOOK.md &&
    grep -q "PDP100294" .planning/spec/DEMO-RUNBOOK.md &&
    grep -q "ecab48ad" .planning/spec/DEMO-RUNBOOK.md &&
    grep -qi "revoke" .planning/spec/DEMO-RUNBOOK.md
  </verify>
  <done>A presenter with no context can dial the number, run either scenario, know what to expect, and recover if it misbehaves.</done>
</task>

</tasks>

<verification>
- Every dollar figure traces to the deployed `resolve_claim` tariff, not to §4 of the spec
- Version ladder matches the app's actual version list
- No claim in the runbook that was not observed on a real call or in the deployed tool source
</verification>

<success_criteria>
`.planning/spec/DEMO-RUNBOOK.md` is accurate to deployed v7 and usable cold by someone who
was not part of building it.
</success_criteria>

<output>
`.planning/quick/260805-aze-create-presenter-demo-runbook-for-deploy/260805-aze-SUMMARY.md`
</output>
