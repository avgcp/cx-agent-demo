# Demo Runbook — Meridian Claims Concierge (voice)

**For the presenter. Read this standing up.**

Everything below was verified on the deployed build. Figures come from the agent's own
pricing tool, not from the Mock Data Appendix (§4) — the implementation diverged from that
appendix and this runbook matches what actually happens on the line.

---

## Call this number

```
☎  ___________________________________
```

> Not retrievable via API — it's in the CX Agent Studio console on the deployment below.
> Fill it in before you present.

| | |
|---|---|
| Project | `insurance-agent-demo-500614` (location `us`) |
| App | `6e01e4a5-42a8-5213-b3da-c9053ff8ea52` — *Meridian Claim - Voice (demo-ready)* |
| Version live | **v11** `b17c9a26-3485-4658-9259-dfa4839a7977` |
| Deployment | `d28bbcb0-066e-4127-a894-fbf9ba39789f` — *voice - meridian demo* |
| Claim email goes to | `akash.vinayak@nerdery.com` |

> **Confirm the version before you present.** A deployment pins a *version*, so the phone
> number serves whatever it was last pointed at:
> ```bash
> curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
>   "https://ces.googleapis.com/v1/projects/insurance-agent-demo-500614/locations/us/apps/6e01e4a5-42a8-5213-b3da-c9053ff8ea52/deployments" \
>   | grep -o 'versions/[a-f0-9-]*'
> ```
> Expect `b17c9a26…`. Anything else and the email may go elsewhere — see the version table
> at the bottom.
>
> **Only one mailbox can receive.** The sender is Resend's shared `onboarding@resend.dev`,
> which on a free account delivers **only to the address that owns the Resend account**.
> That account is `akash.vinayak@nerdery.com`, so mail to anyone else is rejected and the
> agent quietly falls back to a drafted message. To send to a second person, verify a domain
> at resend.com/domains and change the `from` address to it.

The agent answers as **Alex**, for the carrier **Meridian Device Protection**.
English only. Chat/photo upload is **not** part of this demo — it's a phone call.

---

## Scenario A — auto-approve  *(the main run)*

| You say | Agent should |
|---|---|
| *"Hi, my name is Jordan Rivera and my policy is P D P one zero zero two nine four"* | Verify, then put you through to claim handling |
| *"I dropped my laptop and the screen is cracked. No liquid, and it still switches on and works normally otherwise."* | Price it and approve on the spot |
| *"Okay that works for me"* | Confirm the email is on its way |
| *"No thanks"* | Close warmly |

**What lands:** screen replacement **$840**, under the **$1,500** it can approve on the
spot, excess **$25**, reference **CLM-24xxx**. A real email arrives at
`akash.vinayak@nerdery.com`. Then it offers to add the uninsured **iPhone 16 Pro Max**.

The decision, the email and the cross-sell come as **three separate turns**. Let it finish
each one — don't talk over the pause.

---

## Scenario B — escalation  *(the contrast)*

| You say | Agent should |
|---|---|
| *"Hello, this is Jordan Rivera, policy PDP100294"* | Verify and put you through |
| *"I spilled a full glass of water on my MacBook and now it won't turn on at all"* | Call it a total loss and route to a human |

**What lands:** **$3,000** total loss, rules **DL-3** (total loss always escalates) and
**DL-2** (over the limit), routed to a specialist with the case already packaged. Email
sends on this path too.

**The call ends after escalation** — that's deliberate. There is no cross-sell on this
path; the agent will not try to sell to someone whose claim just went to a specialist.

Run A then B back-to-back. The contrast between the two is the point.

---

## Optional beat — "actually it's my phone"

If someone asks what happens when the caller names a device that isn't covered, try it:

> *"I cracked the screen on my iPhone 16 Pro Max"*

It declines the claim and turns it into an offer, without being asked to:

> *"Your iPhone 16 Pro Max isn't covered by this policy, so I can't open a claim for it.
> We can add it to your cover, though, so you're protected in future. Would you like me to
> arrange that?"* → then steers back to the MacBook.

Name a device on **no** record (*"my Dell monitor got smashed"*) and it declines cleanly
without inventing cover or offering to add it. Good answer to "does it just say yes to
everything?".

---

## Seeded policies

| Policy ID | Name | Covered device | Coverage | Approves up to | Excess | Uninsured (cross-sell) |
|---|---|---|---|---|---|---|
| **PDP100294** | Jordan Rivera | MacBook Pro 16" | $3,000 | $1,500 | $25 | iPhone 16 Pro Max |
| PDP100017 | Alex Chen | Dell XPS 14 | $2,000 | $1,000 | $25 | iPhone 16 |
| PDP100583 | Maria Santos | iPhone 16 Pro Max | $1,500 | $750 | $25 | HP Pavilion 15 |
| PDP100746 | David Okafor | iPhone 16 | $1,000 | $500 | $25 | Dell XPS 14 |
| PDP100862 | Sarah Lindqvist | HP Pavilion 15 | $1,000 | $500 | $25 | Samsung Galaxy A55 |
| PDP100935 | Tom Brennan | Samsung Galaxy A55 | $500 | $250 | $25 | MacBook Pro 16" |

Use **PDP100294** unless you have a reason not to. The others exist if someone asks to see
a different device or a smaller policy.

> Every email goes to `akash.vinayak@nerdery.com` regardless of which policy you use. The
> addresses on the policy records are `example.com` and undeliverable by design. The agent
> never reads an address aloud.

---

## Presenter notes

**Mentioning liquid is safe now.** If you say something loose like *"there was water on the
ground where it fell"*, it asks whether any of it actually got on the device rather than
assuming the worst. Only liquid that reached the device makes it a total loss. Real liquid
damage (*"I spilled a glass of water on it"*) still escalates immediately.

**Talk normally.** It handles a whole claim in one breath — *"dropped it, screen's cracked,
no liquid, still works otherwise"* gets you straight to the decision without a Q&A.

**If it mishears the policy ID, just say the number again.** It matches on the digits, so
"PDP" coming through as "TDP" or "BDP" no longer matters. Saying "P D P" and the digits in
separate turns also works.

**Off-script questions are safe.** Ask it about the weather. It declines and steers back to
the claim without breaking character — a good moment to point out it isn't a script.

**Let the pauses happen.** Three beats, three turns. If you answer before it finishes, you
can knock it off sequence.

**Don't say "no thanks" early.** It reads as declining the claim and the close gets muddled.
Save it for the cross-sell.

---

## If something goes wrong

**Roll back the deployment.** One call, takes seconds:

```bash
curl -X PATCH \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"appVersion":"projects/insurance-agent-demo-500614/locations/us/apps/6e01e4a5-42a8-5213-b3da-c9053ff8ea52/versions/<VERSION_ID>"}' \
  "https://ces.googleapis.com/v1/projects/insurance-agent-demo-500614/locations/us/apps/6e01e4a5-42a8-5213-b3da-c9053ff8ea52/deployments/d28bbcb0-066e-4127-a894-fbf9ba39789f?updateMask=appVersion"
```

| Version | ID | Notes |
|---|---|---|
| **v11** | `b17c9a26-3485-4658-9259-dfa4839a7977` | **Current.** No self-narration |
| v10 | `6ec881a1-081f-4859-8edb-4329d801c3d8` | Liquid-ingress disambiguation |
| v9 | `ff095eeb-95a9-42f1-987d-8f2ed61d9304` | Device framing + mismatch cross-sell |
| v7 | `718b6fb3-eb4d-4b56-a7d6-39eb3f81c875` | Previous good build, assumes the covered device |
| — | `5d24c721-4ecc-4166-a595-9d2e151a2e16` | ⚠️ Mails a different address — do not use |
| — | `a435521a-87ad-4e65-912f-ec86cc747a67` | ⚠️ Mails a different address, no working key — do not use |
| v6 | `ecab48ad-5dc2-438b-987d-e47f561bf79c` | ⚠️ **Avoid** — repeats lines aloud |
| v5 | `0dd52030-d0a8-44d7-8750-cc0ef6c31962` | Safe fallback, emails you |
| v4 | `81f80c75-e8de-4b3d-8f05-30455d8c01d5` | First with live email |
| v3 | `9eba0634-93aa-448c-9b61-91ffb2818930` | No live email |
| v2 | `c3ede5f3-dd3e-475c-b658-3fd660e2c384` | |
| v1 | `a49ca4f7-6e6a-4bfd-90df-f9e90596056c` | Earliest |

**Stay on v11.** If it misbehaves, drop to **v10**, then **v9** — same conversation quality, emails you,
only difference is the email is composed at a slightly later step.

The two unnumbered versions above were cut during a parallel edit and point the claim email
at a different mailbox. Don't roll onto them by accident.

**The email cannot break the call.** If the network or the mail provider fails, the agent
falls back to a drafted message and the conversation continues untouched. You lose the
inbox moment, nothing else.

**Editing anything?** A deployment pins a *version*. Changing the app does **not** change
what the phone number serves — you must cut a new version and repoint the deployment.

---

## Known cosmetic issues

- An 👂 emoji occasionally appears in transcripts when the caller pauses mid-sentence.
  Doesn't affect the call.
- The decision turn can take ~4 seconds when the agent reaches for pricing slightly early.
  It self-corrects; it just sits quiet a beat longer.

---

## After the demo

**Revoke the Resend API key.** It's in the `send_claim_email` / `resolve_claim` tool source
and baked into every version snapshot from v4 onward, readable by anyone with read access
to the project. Delete it in the Resend dashboard once you're done.
