# Demo Runbook — Meridian Claims Concierge

**For the presenter. Read this standing up.**

Two channels, two apps: **phone** (voice) and **chat** (photo). They share the seeded
policies, the claim logic and the email, but they are separate deployments — a change to one
does not affect the other.

Everything below was verified on the deployed build. Figures come from the agent's own
pricing tool, not from the Mock Data Appendix (§4) — the implementation diverged from that
appendix and this runbook matches what actually happens on the line.

---

# Phone channel

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

# Chat channel — the photo demo

**A separate app from the phone.** Same claims logic, same seeded policies, same email —
but the customer can show you the damage, which the phone run cannot do.

| | |
|---|---|
| App | `a2f621e4-9faf-505a-b804-22471f022366` — *Meridian Claim - Chat (hardened)* |
| Version live | `160dc3b2-571c-480f-b901-e4dbe8947f70` — *chat v9, the agent speaks the decision explanation verbatim* (supersedes v8 `3f85b1d8` cross-sell buttons, and v7 `bb14cdcc`, which is still the version the on-screen card was verified against — **the card definition is byte-identical in v8 and v9**) |
| Deployment | `d7bfbb93-8cee-43fe-9095-bc5775f353bd` — *chat - meridian demo*, `WEB_UI` / chat only |
| Widget embed | The console-generated embed snippet from **Deployments → *chat - meridian demo* → the embed/integration panel**, served from a local page with a token broker at `http://localhost:3000`, on chat-messenger SDK **v1.16**, with `enable-file-upload` set on the `chat-messenger-container` element. No Cloud Storage bucket or `url-allowlist` configuration is needed — file upload worked with none, and the card's placeholder image is a `gstatic.com` URL the SDK hard-trusts. |

> **Verified on a real photo, through the real widget.** On 2026-08-09 the whole path was
> driven through the real widget in a browser: a real cracked-MacBook photo was uploaded,
> displayed inline, and read correctly (*"I can see the cracks on the screen there."*), then
> priced from the tariff at $840 and auto-approved as `CLM-24442` with the decision card
> drawn on screen. Still worth one rehearsal with **your** photo, since accuracy depends on
> the image.

**Have ready:** a clear photo of a cracked laptop screen, and — for the second beat — a
photo of an **undamaged** laptop.

## Scenario C — photo confirms the damage  *(the main chat run)*

| You type | Agent should |
|---|---|
| *"Hi, my name is Jordan Rivera and my policy is PDP100294"* | Verify and open with your MacBook |
| *"I dropped my laptop and the screen is cracked. No liquid, and it still switches on and works normally otherwise."* | **Ask for a photo** — it will not price anything first |
| *(attach the cracked-screen photo)* | Describe what it can see, then approve |
| *(the decision card draws)* | See below |
| *"Okay that works"* | Confirm the email |
| *"That's everything, thanks"* | **Offer to add the uninsured iPhone 16 Pro Max to cover, with two buttons** — the cross-sell beat |
| *(tap **Add it** or **Not now**)* | Say it will send the options over — or accept the decline — then close warmly |

**The moment to point at:** it says back what it actually sees — *"I can see the crack
running from the lower left across the panel"* — before any decision. That line is the proof
it looked at the image rather than taking your word for it.

Then the usual: **$840**, under the **$1,500** limit, **$25** excess, `CLM-24xxx`, and a real
email.

**What the card looks like on screen — verified live 2026-08-09:**

- Title `Apple MacBook Pro 16"` with `$840.00` right-aligned on the same row.
- Subtitle `Screen replacement · CLM-24442 · $840 less $25 excess = $815 to you · Approved on
  the spot (on-the-spot limit $1,500)` (the claim number will differ each run).

**Expected, not a bug.** A presenter will see two cosmetic artifacts and should not be
alarmed by either: a small **empty grey image tile** at the left of the card — there is no
product image — and **two horizontal divider rules** below the row with nothing after them —
no cost breakdown, payment method or buttons are sent, deliberately, because the SDK would
otherwise print an unchangeable "Sales tax $0.00" line and live buttons that inject text into
the conversation when tapped.

**The agent now states the decision out loud — you no longer read the card for it.** Fixed in
chat v9 `160dc3b2` (2026-08-10). At the moment of approval it says the rule explanation in
full, naming the amount, the on-the-spot limit it is under, and why it qualifies:

> *"Good news - that screen replacement comes to $840, which is under the $1,500 I can approve
> on the spot, so I can approve that for you right now. Your excess is $25, and your reference
> is CLM-24xxx."*

**Point at where those figures come from.** That sentence is not written by the model — it is
copied from the pricing tool's own output by a platform field mapping, so the agent
*physically cannot* round it, reword it or invent a number. That is worth saying aloud if
anyone asks how you stop an LLM quoting the wrong price.

**The card repeats the same facts in its subtitle. That overlap is deliberate** — the spoken
line carries the beat, the card reinforces it and leaves it on screen. It is not a bug and not
a duplicate.

> Verified on a live run (keyboard fault, 2026-08-10): the explanation was byte-identical to
> the pricing tool's string and appeared exactly once in the conversation record. **The
> on-screen render of that line has not yet been watched in a browser** — if you see the same
> sentence printed twice, once above the card and once attached to it, report it; the fix is
> known and small.

Terse input works: the verified run used just *"PDP100294, Jordan Rivera"* and *"cracked. no
water but it still works"*.

## The cross-sell beat — *cost centre → profit centre*

> ⚠ **Payload verified headlessly on 2026-08-10 (chat v8); the on-screen render is not yet
> confirmed by eye** — the same status Scenario D carries. The tool now fires reliably and
> emits the exact shape the widget SDK reads, but nobody has yet watched the buttons draw in
> a browser.

**This beat is new and had never once fired before 2026-08-10.** It was not miswired — no
conversation had ever taken a turn *past* the email confirmation, which is structurally what
the offer requires. Give it that turn and it fires.

After the email is confirmed, say **"that's everything, thanks"**. The agent should say one
sentence naming the device that is *not* on the policy — *"One more thing while I have you:
your Apple iPhone 16 Pro Max isn't on this policy. Would you like me to add it to your
cover?"* — followed by **two buttons in a row: `Add it` and `Not now`**.

**Presenter notes:**

- **Tapping a button types a sentence into the chat as if the customer had written it** —
  `Add it` sends *"Yes please, add it to my cover."* and `Not now` sends *"Not right now,
  thanks."* That is the SDK's behaviour, not a bug, and it looks natural on screen. You can
  also just type the answer instead of tapping.
- **The agent will never quote a price or a premium** — it does not have one and is
  instructed not to invent one. It says it will have someone send the options over. If a
  customer asks "how much?", that is the honest answer, not a dodge.
- **There is deliberately no cross-sell on the escalation path** (Scenario D). Never sell to
  someone whose claim has just gone to a specialist. If you want to show the discipline, run
  Scenario D and point out that the offer *doesn't* come.
- The conversation ends **after** the customer answers, not before — so the buttons stay
  live long enough to tap.

## Scenario D — the photo disagrees  *(the strongest beat)*

> ⚠ **Unverified live.** This scenario is proven **offline only** (`phototest2.py`) and has
> **never been run live through the widget**. Rehearse it with the undamaged-device photo
> before showing it to a customer.

| You type | Agent should |
|---|---|
| *"Hi, my name is Jordan Rivera and my policy is PDP100294"* | Verify |
| *"I dropped my laptop and the screen is cracked. No liquid, and it still works otherwise."* | Ask for a photo |
| *(attach the photo of the **undamaged** laptop)* | Decline to approve, route to a specialist |

It will **not** auto-approve, and it will **not** accuse anyone:

> *"Thanks for sending that over. I'd like one of our specialists to take a closer look at
> the photo before we settle this, so I'm passing it on with everything we've discussed —
> you won't need to go over it again."*

No price is quoted, because nothing was verified. Behind it, `photo_contradiction` is set and
**DL-5** lands in the audit trail.

This is the answer to *"what stops someone just claiming anything?"* — and it lands better
than any slide, because the audience watched it happen.

**Other photo cases**, if someone asks:
- A blurry or badly framed photo gets **one** retry, with a specific reason, then a human.

> ### ⚠ Run the demo with matching device/policy pairs
>
> **The photo confirms whether the reported damage is visible. It cannot verify that the
> photographed device is the insured device.** Jordan Rivera / **PDP100294 is a MacBook — use
> a laptop photo.** Maria Santos / PDP100583 is an iPhone, so that policy needs a phone photo,
> and so on down the seeded table below.
>
> **What happens if someone in the audience hands you a mismatched photo:** the agent will
> describe what it sees, confirm the crack and approve the claim at the **tariff price for the
> policy's own device** — a phone photo on the MacBook policy still prices as a MacBook screen
> at $840. It will not flag the mismatch. Expect this rather than being surprised by it.
>
> **The honest answer if you are asked directly:** we tried twice to make the model verify
> device identity from the image and it failed both times — it already knows what the policy
> covers and conforms its description to that rather than to the pixels. Rather than ship a
> check that reads like protection and isn't, we removed it (2026-08-06). Confirming that the
> item photographed is the item insured is a claims-ops control — serial numbers, IMEI, policy
> records — not a vision-model job. What the model *is* reliably good at is the thing the demo
> actually shows: reading the damage, and refusing to approve when the damage isn't there.

## What's different from the phone run

- It writes in short structured messages rather than one-thing-per-turn speech.
- It will show the email address on file; on the phone it never reads one out.
- The escalation and cross-sell beats behave the same.
- **The photo gate only applies to screen repairs.** A liquid total loss escalates
  immediately, exactly as on the phone — no photo needed.

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

## If something goes wrong  *(phone)*

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
- **Chat:** the decision card shows an empty grey image tile and two trailing divider rules
  with nothing after them. Expected — see the "Expected, not a bug" note under Scenario C.
- ~~**Chat:** the silent decision turn — the agent draws the card without a sentence stating
  the decision.~~ **Fixed in chat v9 `160dc3b2` (2026-08-10).** The agent now speaks the rule
  explanation verbatim from the pricing tool. See Scenario C. One thing left to watch for on
  screen: the sentence printing *twice*, once above the card and once attached to it.

---

## After the demo

**Revoke the Resend API key.** It's in the `send_claim_email` / `resolve_claim` tool source
and baked into every version snapshot from v4 onward, readable by anyone with read access
to the project. Delete it in the Resend dashboard once you're done.
