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
| Version live | `a6f6b620-af15-4e43-b0d8-bfbbb2d64a46` — **voice v18, *the phone can now READ A CLAIM BACK* (2026-08-14, plan `06-04`). A caller who says who they are and asks about a claim they already filed hears its real status, read from the shared claim store — including claims filed in the CHAT widget. Supersedes v17 `dcc20863` (the call actually opens with the greeting), v16 `09a1f14d`, v15 `17b2e438`, v14 `5d02f14c` (Spanish), v13 `5d9df25c` (assessor packet on the phone), v11 `b17c9a26` (no self-narration).** |
| Roll back to | `dcc20863-3746-4e43-a2c9-ed30e0611479` — voice v17. **Loses only the claim-status beat** — the phone goes back to being able to take a new claim but not to read an existing one back. Every other beat is byte-identical (same `claim_intake`, same nine tools, same config, same opening line). See *If something goes wrong (phone)* below. |
| Deployment | `d28bbcb0-066e-4127-a894-fbf9ba39789f` — *voice - meridian demo* |
| Claim email goes to | `akash.vinayak@nerdery.com` |
| Assessor packet goes to | `akash.vinayak@nerdery.com` — **same mailbox**, told apart by the `[ASSESSOR] [VOICE]` subject |

> **Confirm the version before you present.** A deployment pins a *version*, so the phone
> number serves whatever it was last pointed at:
> ```bash
> curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
>   "https://ces.googleapis.com/v1/projects/insurance-agent-demo-500614/locations/us/apps/6e01e4a5-42a8-5213-b3da-c9053ff8ea52/deployments" \
>   | grep -o 'versions/[a-f0-9-]*'
> ```
> Expect `a6f6b620…`. Anything else and the email may go elsewhere — see the version table
> at the bottom.
>
> **Faster and better: just listen to the first sentence.** The opening line is now a
> deterministic literal, so it is a one-second version check by ear. If the call opens
> *"…To pull up your policy, could I take your full name and your policy ID?"* you are on
> **v16 or older**, whatever the deployment says.
>
> ⚠ **NOT YET CONFIRMED BY EAR — and the last call that WAS made found a real defect.** Everything
> since v13 was proven end to end over the API — the new opening, the Spanish call, the language fix
> at the handoff, the packet, Resend HTTP 200 — but the only real call since then, on
> 2026-08-14, opened with the **old** identity demand and is what produced `260813-ui0`. That is now
> fixed and re-proven on the deployed build, **but nobody has dialled v17.** Three things need an ear:
>
> 1. **The first sentence** must be *"Hello, you're connected to Meridian Device Protection. I'm
>    Alex. How can I help you today?"* — then **answer with what you want, not your name**; identity
>    should be asked for once, after that, never before and never twice.
> 2. **Hang up mid-call.** The call should close cleanly — that branch is separate code and must
>    still work.
> 3. **Say nothing for ~10–15 seconds** after the greeting. You should hear *"Are you still there?"*
>
> How the greeting *sounds*, and whether the Spanish voice sounds Spanish, still cannot be
> established from a conversation record.
>
> ⚠ **DO NOT SCRIPT A BEAT AROUND AN EMAIL ARRIVING.** The user's inbox check on 2026-08-13
> found that **only about two of seven test emails arrived** — roughly a **30% delivery
> rate** — against seven consecutive Resend HTTP 200s. Acceptance is not delivery. The cause
> is almost certainly the unverified shared sender `onboarding@resend.dev`. **Say the email
> is on its way; do not open a mailbox on stage to prove it.** The fix is a verified sending
> domain at resend.com/domains — an account action, not a code change. Check spam.
>
> **Only one mailbox can receive.** The sender is Resend's shared `onboarding@resend.dev`,
> which on a free account delivers **only to the address that owns the Resend account**.
> That account is `akash.vinayak@nerdery.com`, so mail to anyone else is rejected and the
> agent quietly falls back to a drafted message. To send to a second person, verify a domain
> at resend.com/domains and change the `from` address to it.

The agent answers as **Alex**, for the carrier **Meridian Device Protection**.
Chat/photo upload is **not** part of this demo — it's a phone call.

**Not English only any more.** Since voice v14 the phone agent follows the caller's language, and
since v15 it keeps following it across the handoff to claim handling. Opening the call in Spanish
works end to end. **Three caveats, all firm.** (1) The deterministic strings (the decision
sentence and the customer email) are still English. (2) **Switching language mid-claim does not
work and must not be scripted as a beat** — it works at the handoff and only at the handoff.
(3) **The very first sentence of every call is English, always** — the greeting is a fixed
literal that runs before the model, so nothing can localise it (`260813-ui0`). The agent then
switches from the caller's first real utterance and stays switched, handoff included, verified
end to end:

> **Agent:** Hello, you're connected to Meridian Device Protection. I'm Alex. How can I help you today?
> **Caller:** Hola, buenos días. Necesito presentar una reclamación, se me cayó el portátil.
> **Agent:** Siento mucho escuchar eso, pero puedo ayudarle. ¿Me podría decir su nombre completo y su número de póliza?

**Script this as "watch it switch," not "it answers in Spanish from hello."** An earlier note in
this runbook promised a Spanish *greeting* (*"Hola, bienvenido a Meridian Device Protection…"*);
that was measured over the API without a real session-start event and **is not what a caller
hears**. Switching on the caller's first sentence is the better beat anyway — the audience sees
the change happen.

---

> ### NEW — the agent no longer opens by demanding your identity
> **Voice v17 `dcc20863` / chat v18 `d0e4bfef` (2026-08-13, quick tasks `260813-tgq` + `260813-ui0`).**
> Both channels now open with a greeting and an open question, and ask who you are only *after*
> you have said what you want. **The opening sentence is a fixed literal — it is word-for-word
> identical on every call and in every chat session:**
>
> > **Agent:** Hello, you're connected to Meridian Device Protection. I'm Alex. How can I help you today?
> > **You:** I need to file a claim.
> > **Agent:** Of course — could I take your full name and your policy ID?
>
> > **⚠ If you presented between tgq and ui0, you did not get this.** v16/v17 changed the
> > *instruction*, but a callback was still hard-returning the old *"To pull up your policy, could
> > I take your full name and your policy ID?"* on the real channels — and on **chat** it then
> > asked a second time. Only **voice v17 `dcc20863` / chat v18 `d0e4bfef`** actually deliver the
> > new opener. The line above is the version check: if you hear the old one, you are on an old build.
>
> **Let it ask.** Say *"Hi there"* or *"Hello"* first and let the greeting land — it is a better
> first impression than being interrogated, and it is the beat the change exists to produce.
> If you volunteer everything in one breath it will still skip straight to verifying you and
> **will not** make you repeat yourself, but you lose the beat. **On the escalation run,
> volunteering everything at once is actively harmful — see Scenario B.**

## Scenario A — auto-approve  *(the main run)*

| You say | Agent should |
|---|---|
| *"Hi there"* | **Greet you and ask how it can help** — no request for details yet |
| *"I need to file a claim"* | Ask for your full name and policy ID, in one question |
| *"Jordan Rivera, policy P D P one zero zero two nine four"* | Verify, then put you through to claim handling |
| *"I dropped my laptop and the screen is cracked. No liquid, and it still switches on and works normally otherwise."* | Price it and approve on the spot |
| *"Okay that works for me"* | Confirm the email is on its way |
| *"That's everything, thanks"* | Offer the cross-sell, then close warmly |

> Say *"That's everything, thanks"* rather than *"No thanks"* on the last turn. *"No thanks"*
> reads as a refusal and can close the session **before** the cross-sell arrives.

**What lands:** screen replacement **$840**, under the **$1,500** it can approve on the
spot, excess **$25**, reference **CLM-24xxx**. A real email arrives at
`akash.vinayak@nerdery.com`. Then it offers to add the uninsured **iPhone 16 Pro Max**.

The decision, the email and the cross-sell come as **three separate turns**. Let it finish
each one — don't talk over the pause.

---

## Scenario B — escalation  *(the contrast)*

| You say | Agent should |
|---|---|
| *"Hi there"* | Greet you and ask how it can help |
| *"I need to file a claim"* | Ask for your full name and policy ID |
| *"Jordan Rivera, policy PDP100294"* | Verify and put you through |
| *"I dropped my laptop down a flight of stairs"* | Ask which part is damaged |
| *"The screen is completely smashed and it will not turn on at all now"* | Ask about liquid |
| *"No liquid at all, just the drop"* | Ask whether it still works |
| *"No, it is not working at all"* | Call it a total loss ($3,000) and route to a human |
| *"Okay"* | **Speak the send-away line**, then file the case and close |

> ### ⛔ ON THE PHONE, DO NOT OPEN THE ESCALATION WITH A SPILL
> *"I spilled coffee into my laptop"* **stalls the diagnostic on the voice channel.** The model
> passes the diagnostic tool its answers wrapped in quote marks, the tool correctly refuses them,
> and the agent re-asks the same question indefinitely — five turns and no decision. Verified
> **pre-existing** against the previous build through the live deployment (plan `06-04`), not a
> regression, and not fixed. **Use the drop script above on the phone.** The spill is still fine
> on chat.
>
> ### ⚠ On the escalation run, do NOT open by volunteering everything at once
> *"Hi, I'm Jordan Rivera, policy PDP100294, I spilled water on my MacBook"* in one breath works
> — you are verified immediately and not asked to repeat yourself — **but it collapses the rest
> of the call onto a single turn, and that turn goes silent.** The case gets filed, the
> specialist gets the packet, the session closes, and **the agent says nothing at all** while it
> happens. Verified as a **pre-existing** behaviour (it does the same on the previous build), not
> a regression, but it is ugly in front of a room. **Use the four separate turns above.**

**What lands:** **$3,000** total loss, rules **DL-3** (total loss always escalates) and
**DL-2** (over the limit), routed to a specialist with the case already packaged. Email
sends on this path too.

**The call ends after escalation** — that's deliberate. There is no cross-sell on this
path; the agent will not try to sell to someone whose claim just went to a specialist.

Run A then B back-to-back. The contrast between the two is the point.

### The briefing packet on the phone — *the backend claims-processing reveal*

**New in voice v13 `5d9df25c` (2026-08-12).** Same beat the chat channel has had since v10,
now on the call.

On the escalation path *and nowhere else*, the moment the agent tells the caller the email
is on its way, it silently composes a six-section **assessor briefing packet** from the
session and sends it as a **second, separate email**. The caller hears **nothing** about it —
no packet, no case record, no assessor, no second email.

| Email | Subject | Who it's for |
|---|---|---|
| Customer confirmation | `Claim CLM-24xxx - please reply with photos` | Jordan Rivera |
| **Briefing packet** | **`[ASSESSOR] [VOICE] CLM-24xxx - Jordan Rivera`** | the specialist picking the case up |

Both land in `akash.vinayak@nerdery.com` — see the one-mailbox note at the top. The
**`[VOICE]` token is how you tell a phone packet from a chat one** (`[ASSESSOR] [CHAT] …`)
when you have run both channels into the same inbox.

The packet reads:

```
SUMMARY: ... Apple MacBook Pro 16" which is liquid damaged and does not turn on.
ACTION: Assess the claim details ... to determine the outcome.
CLAIM: Customer: Jordan Rivera; Policy: PDP100294; Device: Apple MacBook Pro 16";
       Issue: liquid_damage; Amount: $3,000; Excess: $25; Total-loss flag: true.
DIAGNOSTIC: Customer reported: q1=no_power.
RULES FIRED: DL-3; DL-2.
FLAGS: Total loss indicated.
```

**Open it on screen while the call is still fresh.** This is the "what happens behind the
scenes" answer, and it is stronger on the phone than in chat: the caller said four sentences
and a structured case file already exists.

> **Two things to watch on the call, and roll back if either is wrong.**
> 1. **Listen for a pause after the send-away line.** The packet costs about **6 extra
>    seconds** of tool work on that turn (measured: 3.3 s on v11, 9.5 s on v13, same script,
>    same channel). The agent is *instructed to say the whole send-away sentence before it
>    calls any tool*, and the API record confirms the text comes first — so the work should
>    run while you are still being spoken to. A second of quiet before the call closes is
>    fine; several seconds of dead air is not.
> 2. **Listen for a leak.** It must not say "packet", "case record", "assessor", or read any
>    heading aloud. Nothing leaked on the API runs, but audio is its own channel.
>
> Either one goes wrong → roll back to v11 `b17c9a26`, one call, below.

---

## Scenario B2 — "where's my claim got to?" on the PHONE  *(the cross-channel beat)*

**New in voice v18 `a6f6b620` (2026-08-14).** The phone can now read an **existing** claim back —
including one filed in the **chat widget**. This is the beat that shows the two channels are one
system rather than two demos.

| You say | Agent should |
|---|---|
| *"Hi, this is Jordan Rivera, policy PDP100294, and I'm just checking on a claim I put in."* | Verify you, then read **one** claim back and ask whether that is the one |
| *"Yes, that's the one."* | Read the full status: reference, device, amount, excess, what comes to you, when it was filed, **and which channel it came from** |
| *"Could you also check CLM-60203 for me?"* | *"I can't find a claim with that reference on this policy"* — and **quote no figure at all** |

**The line to point at:** *"…Filed on 14 August 2026 **via chat**."* You are on the phone, hearing a
claim that was filed in the chat window. Every figure in that sentence is composed in Python from the
stored record and relayed word for word — **the model contributes none of the numbers.**

> ### 👂 Two things to know before you run this live
>
> 1. **`PDP100294` currently holds 25 test claims**, so the agent will say *"I can see 25 claims on
>    this policy."* That is correct and it is a bad line in front of a customer. Either prune the
>    store first, or run this beat on **Maria Santos / PDP100583**, which has exactly one claim (but
>    was filed on the phone, so you lose the cross-channel punch).
> 2. **You are never asked to read a claim number out.** That is deliberate and it is worth saying
>    out loud in the room: speech-to-text mangles `CLM-24690`, so the agent identifies you by name
>    and policy and finds the claim itself. If it ever *does* ask you to read a reference aloud,
>    that is a defect — stop and report it.

**The pause before the answer is about three seconds.** It is the shortest tool-backed turn in the
whole demo. Talk over it if you want to — barge-in works.

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
| Version live | **`8a95ab02-e653-4d1e-baee-eb500c52710c` — chat v20, *confirming a claim read-back works, and the `[RECORD]` email actually sends*** (2026-08-14, quick task `260814-8rv`). Two fixes: answering *"yes"* / *"that's the one"* / *"correct"* to the status read-back used to return **"there is nothing for me to read back"**; and on the auto-approve path the cross-sell widget was ending the turn before `send_case_record_email` ran, so **no `[RECORD]` email was ever sent**. Both fixed; both tools now fire in the same turn, mailer first. Supersedes v19 `619e13a1` (non-colliding 8-digit claim references + cross-policy overwrite refusal, `260814-80f`), v18 `d0e4bfef` (**the widget's first message actually greets you** — `260813-ui0`). v17 rewrote the instruction but a callback still hard-returned the identity demand as the widget's opening message **and the agent then asked a second time** — so v17's opener was never seen by a user. Supersedes v17 `64be15eb` (the instruction change), v16 `be4c83bb` (photo on every claim + `[RECORD] [CHAT]` record email), v15 `129f8b31` (claim store + status lookup), v14 `cdca14e3` (photo attached to the packet), v13 `1eb3fd5c`, v12 `26c3aebd`, v10 `658472a0`, v9 `160dc3b2`, v8 `3f85b1d8`, and v7 `bb14cdcc`, which is still the version the on-screen card was verified against — **the card definition is byte-identical in v7 through v18** |
| Roll back to | **`619e13a1-2a31-4627-aca3-2ff3a32e336b` — chat v19.** **Loses two things and you must know which**: (a) confirming a claim read-back breaks again — answering *"yes"* returns *"there is nothing for me to read back"*, so **do not run the claim-status beat on v19**; and (b) the `[RECORD]` email stops being sent on the auto-approve path. Everything else is byte-identical (same 13 tools, same config, same opening line, same card, same cross-sell). The older v17 `64be15eb` rollback is retained for history only — do not roll back that far. See *If something goes wrong (chat)* below. |
| Deployment | `d7bfbb93-8cee-43fe-9095-bc5775f353bd` — *chat - meridian demo*, `WEB_UI` / chat only |
| Widget embed | The console-generated embed snippet from **Deployments → *chat - meridian demo* → the embed/integration panel**, served from a local page with a token broker at `http://localhost:3000`, on chat-messenger SDK **v1.16**, with `enable-file-upload` set on the `chat-messenger-container` element. No Cloud Storage bucket or `url-allowlist` configuration is needed — file upload worked with none, and the card's placeholder image is a `gstatic.com` URL the SDK hard-trusts. |

> **Verified on a real photo, through the real widget.** On 2026-08-09 the whole path was
> driven through the real widget in a browser: a real cracked-MacBook photo was uploaded,
> displayed inline, and read correctly (*"I can see the cracks on the screen there."*), then
> priced from the tariff at $840 and auto-approved as `CLM-24442` with the decision card
> drawn on screen. Still worth one rehearsal with **your** photo, since accuracy depends on
> the image.

> ### Ten-second pre-flight — read the widget's FIRST message before you type
> Open the widget and look at the opening message. It must be, word for word:
>
> > Hello, you're connected to Meridian Device Protection. I'm Alex. How can I help you today?
>
> If it says *"To pull up your policy, could I take your full name and your policy ID?"* you are
> on **chat v17 or older** — and that build also asks a **second** time after you answer, which
> reads badly in a room. Repoint to **v18 `d0e4bfef`**. This is the fastest version check there
> is; it needs no terminal.

**Have ready:** a clear photo of a cracked laptop screen, and — for the second beat — a
photo of an **undamaged** laptop.

## Scenario C — photo confirms the damage  *(the main chat run)*

> **New in chat v16 `be4c83bb` (2026-08-13): the photo beat is no longer exclusive to this
> scenario.** The agent now asks for a photo on **every** chat claim — a total loss and a liquid
> spill get the ask too, in the same turn it acknowledges the loss. On those claims the photo is
> **filed, not assessed**: a separate tool puts it on the claim record and the agent says only
> *"thanks, I've got that on the file."* It never describes a photo it has not assessed, and the
> vision assessor is refused in code on anything that is not a cracked screen — a dead or submerged
> device has no crack to see, and a model asked to find one will invent it. **Only this scenario
> produces the "I can see the crack running across the panel" moment.**
>
> **And every resolved chat claim now leaves a photo-carrying artifact behind** — see *The record
> email* below. Before v16 an approved claim's photo drove the decision and was then discarded.
>
> **A customer with no photo is never blocked.** One ask, and if they decline or just answer the
> next question, the claim proceeds and resolves identically — same tariff, same card, same email.
> If a presenter wants the short path, decline the photo and nothing is lost.


| You type | Agent should |
|---|---|
| *"Hi there"* | **Greet you and ask how it can help** — no request for details yet. Actually, the widget greets you *before* you type anything; this row is your cue to answer it with what you want, not with your name (new in chat v18 `d0e4bfef`) |
| *"I need to file a claim"* | Ask for your full name and policy ID, in one question |
| *"Jordan Rivera, policy PDP100294"* | Verify and open with your MacBook |
| *"I dropped my laptop and the screen is cracked. No liquid, and it still switches on and works normally otherwise."* | **Ask for a photo** — it will not price anything first |
| *(attach the cracked-screen photo, **on its own turn, after it asks**)* | Describe what it can see, then approve |
| *(the decision card draws)* | See below |
| *"Okay that works"* | Confirm the email |
| *"That's everything, thanks"* | **Offer to add the uninsured iPhone 16 Pro Max to cover, with two buttons** — the cross-sell beat |
| *(tap **Add it** or **Not now**)* | Say it will send the options over — or accept the decline — then close warmly |

> ### ⚠ Attach the photo ONLY when it asks, on its own turn
> **Do not attach the cracked-screen photo in the same message that describes the damage.** If you
> do, the agent tries to assess the image before it has classified the claim as a screen, its own
> safety gate refuses the assessment, and the claim then **deadlocks** — it will keep asking for
> "another photo" and never price anything. Verified as a **pre-existing** defect (it does the same
> on chat v16, the previous build) and it is the top follow-up item. Two safe routes: attach the
> photo **after** it asks for one, or run a **non-screen** fault (a dead keyboard), where the photo
> is filed rather than assessed and the claim resolves normally at **$420**.

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

## Scenario C2 — "where's my claim got to?" in the CHAT WIDGET

**New in chat v20 `8a95ab02` (2026-08-14).** The claim-status beat now works in chat as well as on
the phone, and — this is the part that changed — **confirming the read-back works.** On v15 through
v19 the widget read one claim back, asked *"Is that the one you mean?"*, and then answered *"yes"*
with **"I can't match that to a claim on this policy, so there is nothing for me to read back."**
That is fixed. Do not run this beat on any build older than v20.

| You type | Agent should |
|---|---|
| *"Hi, this is Jordan Rivera, policy PDP100294. Any update on my claim?"* | Verify you, then read **one** claim back and ask whether that is the one |
| *"yes"* — or *"that's the one"*, or *"correct"* | Read the full status: reference, device, amount, excess, what comes to you, when it was filed, and which channel it came from |
| *"no, the older one"* (the alternative branch) | Move to a **different** claim and read that one back instead — it never reads you a list |
| *"Could you also check CLM-99999999?"* | *"I can't find a claim with that reference on this policy"* — and **quote no figure at all** |

**Any plain agreement works** — the agent resolves the confirmation using the reference *it* just
gave you, so you are never asked to type a claim number back. Same design as the phone; the two
channels behave identically here.

> **Same caveat as the phone beat:** `PDP100294` holds 25 test claims, so the widget opens with
> *"I can see 25 claims on this policy."* Prune the store first, or run it on
> **Alex Chen / PDP100017**, which holds exactly one (`CLM-24599`, Dell XPS 14, $560) and therefore
> resolves in a single turn with no confirmation step at all — so rehearse whichever one you intend
> to show.

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

### The briefing packet — *the backend claims-processing reveal*

**New in chat v10 `658472a0` (2026-08-11), made reliable in v12 `26c3aebd` (2026-08-12), and
in v14 `cdca14e3` (2026-08-13) the packet now carries the customer's own photo as a real
file attachment.** Every escalated chat claim now writes a
six-section **assessor briefing packet** and sends it as a **second, separate email** — the
specialist's handover, composed from the session's own recorded facts. The customer never sees
it, is never told about it, and hears nothing different.

**Since chat v16 `be4c83bb` (2026-08-13) this happens on EVERY resolved chat claim, not only the
escalated ones** — so two emails land for every claim, and the subject line says which kind it is
without opening it:

| | Subject | Who it is for |
|---|---|---|
| Customer confirmation | `Claim CLM-24xxx - please reply with photos`, or — when the customer already uploaded one — `Claim CLM-24xxx - photo received, nothing needed` | the customer |
| **Record, auto-approved claim** | **`[RECORD] [CHAT] CLM-3xxxxxxx - Jordan Rivera - APPROVED`** | the file. **New in v16** — before this, an approved claim left no record and no photo. **⚠ It was silently NOT being sent between v16 and v19** — the cross-sell widget was ending the turn before the mailer could run. **Fixed in chat v20 `8a95ab02` (2026-08-14, quick task `260814-8rv`)**; do not trust a `[RECORD]` email from a build older than v20 |
| **Briefing packet, escalated claim** | **`[ASSESSOR] [CHAT] CLM-24xxx - Jordan Rivera`** | the specialist picking the case up |

`[ASSESSOR]` still means *a person must act on this*, which is why its subject is unchanged from
every packet already in the mailbox. **The token that selects the whole corpus is the channel one,
`[CHAT]` / `[VOICE]`**, which is on both forms — that is the stable filter for a downstream agent
reviewing claim photos against a policy document. Both kinds carry the customer's photo as a real
attachment when one was given, and both carry a `PHOTO: attached` / `PHOTO: not provided` line
computed by the mailer itself, so it cannot claim an attachment that is not there.

**Voice is unchanged** — it is audio-only and still asks for photos by email reply.

**The photo the customer uploaded is attached to the packet as a real file** (new in v14
`cdca14e3`, 2026-08-13), named `CLM-24xxx-damage.png`. Open it on screen: the specialist gets
the evidence, not a description of it. If no photo was ever assessed, the packet still sends,
unchanged, with no attachment — the reveal never depends on it.

**And the agent stops asking for something it already has.** When a photo has been assessed, the
send-away line becomes *"the photo you sent is already attached to the claim, so there is nothing
further you need to send us"*, and the customer email says the same. This is worth pointing at:
the old behaviour asked a customer to email in a photo they had uploaded thirty seconds earlier.

Both arrive at **`akash.vinayak@nerdery.com`**. That is a demo constraint, not the design —
Resend's shared `onboarding@resend.dev` sender only delivers to the account-owning mailbox.
**Say so honestly if asked:** in production these route to two different people; verifying a
domain at resend.com/domains is the one step that unlocks it. The `[ASSESSOR]` prefix is what
makes them unmistakable side by side on one screen, and the **`[CHAT]` token separates this
packet from the phone channel's `[ASSESSOR] [VOICE] …`** when both have been run into the
same inbox.

**What is in the packet** — six headings, every value pulled from the conversation, nothing
invented:

```
SUMMARY:     Review liquid damage claim for Jordan Rivera's Apple MacBook Pro 16"
             which has no power following water contact.
ACTION:      Review claim details, rules fired, and photo evidence when received
             to make a final determination.
CLAIM:       Customer Jordan Rivera, Policy PDP100294, Device Apple MacBook Pro 16",
             Issue liquid damage, Claim amount $3,000, Excess $25, Total loss true.
DIAGNOSTIC:  The customer reports the device has no power.
RULES FIRED: DL-3, DL-2.
FLAGS:       none
```

> **✅ Both packet-presentation defects are fixed as of chat v13 `1eb3fd5c` (2026-08-12).**
> Recorded because the old wording appears in earlier summaries.
>
> 1. **Money used to render as a raw float** — `Amount: 3000.0; Excess: 25.0`, no `$`, no
>    thousands separator. It now reads **`$3,000`** and **`$25`**. Safe to put on a projector.
> 2. **The subject used to carry no channel token** — `[ASSESSOR] CLM-24xxx - …` was identical
>    in shape whether the claim came from chat or from the phone. Both channels now stamp
>    themselves: **`[ASSESSOR] [CHAT] …`** and **`[ASSESSOR] [VOICE] …`**. The `[ASSESSOR]`
>    prefix is unchanged, so any mailbox filter on it still works.

**✅ Nothing extra to do — corrected in chat v12 `26c3aebd` (2026-08-12).** The packet is filed
on the **same turn as the email confirmation**, silently, together with the handoff to the
specialist and the close. Say *"okay, thanks"*, read the email line, and you are done: the case
record is already sent. **You do not need to send another message.**

> **Superseded, and worth knowing why.** Chat v10 filed the packet on the turn *after* the email
> confirmation, so the runbook used to tell presenters to type one more thing. That was fragile
> and it failed in the wild: a real conversation on v10 escalated correctly and filed **no packet
> at all**, because the customer said *"ok"*, got the send-away line, and stopped — the natural
> end of the conversation. The turn the packet needed never came, and nothing on screen said
> anything was wrong. v12 moves the filing onto the turn that always happens.

**Presenter notes:**

- **It fires on every escalated chat claim**, not only the photo-disagreement one. The fastest
  way to show it is the liquid total loss — *"I spilled a full glass of water on my MacBook and
  now it won't turn on at all"* — which escalates in two turns and needs no photo.
- **Point at `RULES FIRED`.** `DL-3` and `DL-2` are the same rule IDs the decision engine
  computed; the packet did not re-derive them. That is the audit trail the "how do you govern
  this?" question is really asking about.
- **Nothing in the packet is written by the model from scratch** — it is composed from eleven
  recorded session variables. If a value is missing it says so; it does not guess.
- Verified server-side on 2026-08-11 (session `esc-25e925f8`, `CLM-24413`): the packet was
  composed once, handed to the mailer byte-for-byte unaltered, and Resend returned **HTTP 200**.
  **The inbox arrival has not yet been confirmed by eye** — check the mailbox once before you
  present.

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
- **The briefing packet fires on both channels now** (voice v13 `5d9df25c` / chat v14
  `cdca14e3`), on the escalation path only, with identical six sections. Two differences:
  the subject token — `[ASSESSOR] [VOICE]` vs `[ASSESSOR] [CHAT]` — and **only the chat packet
  can carry the customer's photo as an attachment**, because only chat can receive one.
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

**NEW BEAT — "where has my claim got to?"** (chat, v15 `129f8b31`, 2026-08-13)
Every claim resolved on chat is now written to a shared claim store the moment it is priced.
So you can **file a claim, close the chat, open a fresh one, and ask about it** — the agent
verifies you, then reads the claim back with its reference, device, amount, excess and filing
date. Two things to know before you run it:

- **Identify by name and policy ID, exactly as you would for a new claim. Never read a claim
  reference out.** The agent is deliberately built not to ask for one, because reference
  numbers do not survive speech-to-text. If you *volunteer* one it will use it, but it is a
  bonus path, not the way in.
- **`PDP100294` carries several claims, so the agent will ask which one you mean.** That is
  the designed behaviour, not a stumble — answer naturally (*"the older one"*, *"the laptop
  one"*, *"the one from June"*) and it resolves. If you want a single clean answer with no
  disambiguation, use **`PDP100583` / Maria Santos**, which has exactly one claim on file.
- Every figure in that sentence is composed in code, not by the model, and an unknown or
  wrong-policy reference gets a flat *"I can't find a claim with that reference on this
  policy"* with **no figure of any kind** — the agent will not invent a claim. That refusal is
  worth showing deliberately if anyone asks about hallucination.

**Send the photo when it asks for it, not after the decision.** (chat, v14 `cdca14e3`)
The photo is attachable only on the turn it arrives, and the customer's confirmation email is
written the moment the claim is priced. Upload it when prompted — or, if you are volunteering
one on the total-loss run, do it **before** the price appears. Volunteer it after the decision
and the packet still gets the attachment, but the email will read *"please reply with photos"*
while the agent says the photo is attached.

**On the cross-sell, tap "Add it".** Declining works but can close the chat without a farewell
line. Accepting gives you *"I will get the options sent over"* and a warm close — the better
ending anyway.

**Mentioning liquid is safe now.** If you say something loose like *"there was water on the
ground where it fell"*, it asks whether any of it actually got on the device rather than
assuming the worst. Only liquid that reached the device makes it a total loss. Real liquid
damage (*"I spilled a glass of water on it"*) still escalates immediately.

**Talk normally.** It handles a whole claim in one breath — *"dropped it, screen's cracked,
no liquid, still works otherwise"* gets you straight to the decision without a Q&A.

**⚠ SAY THE POLICY ID DIGIT BY DIGIT, SLOWLY — AND EXPECT A RETRY.** *(corrected 2026-08-13 after
a live call; the previous "it no longer matters" wording was too confident.)* On the **phone**,
speech-to-text mangles alphanumeric identifiers badly. A real call on 2026-08-13
(`119vrJXUcbjQCO4DWJ_0w7xxw`) took **five turns and three `verify_identity` calls** to capture one
policy ID — the transcript went `PGP` → `PDP` → `1000294` → `TDP100294` before `PDP100294` landed.
That is roughly a third of a two-minute demo spent failing to authenticate, immediately before the
moment the agent is meant to look brilliant.

Mitigations, in order of usefulness:
- **Say it slowly, letter by letter and digit by digit**, and pause between the prefix and the
  digits. Do not rattle it off.
- **Do not talk over it while it is confirming** — a barge-in mid-sequence made this call worse.
- If it mishears, **just say the number again**; it does recover, it simply costs turns.
- On **chat** this is a non-issue — type it.

*(The store-side lookups in Phase 6 already normalise keys so a spoken **claim reference** survives
transcription. The **authentication** path has no equivalent normalisation, which is why this still
bites. Recorded as a demo risk, not yet fixed — see `260813-olv-SUMMARY.md`.)*

**⚠ LANGUAGE: pick one language before you authenticate, and stay in it.** *(phone, live v17
`dcc20863`, 2026-08-13.)* Spanish works end to end — identity, handoff, decision, send-away and
cross-sell — **provided the caller does not change language mid-call**, and provided you expect
the **opening sentence to be English** (it is a fixed literal; see the top of this runbook):

- **Switching language before authentication completes used to be a call-killer.** The agent
  reset to the language the call opened in at the moment it handed you to claims, and did not
  come back. **Fixed in voice v15 `17b2e438` and carried forward into the live v17** — no longer
  a hazard on the build that is serving.
- **Switching language AFTER the handoff does not work on any build.** Do not script it. It is the
  one language beat that has never worked.
- The demonstrable beats are **"the whole call in Spanish"** and, on v15, **"the caller starts in
  one language and settles into another before authenticating."** Both land without a mid-claim
  switch.

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

**The named rollback is now voice v17 `dcc20863-3746-4e43-a2c9-ed30e0611479`** — the build the
phone served immediately before the 2026-08-14 03:50 UTC repoint. Rolling back costs **only the
claim-status beat**: the phone can still take a new claim, it just can no longer read an existing
one back. The opening line, the decision line, the tariff, the customer email, the cross-sell, the
assessor packet, Spanish, the handoff language fix and barge-in are all **byte-identical** between
v17 and v18 — v18 adds one subtask to `claims_concierge` and attaches one tool, and touches nothing
else. Substitute it for `<VERSION_ID>` above.

*(The older v11 `b17c9a26` rollback below is retained for history only. Do not roll back that far
unless the packet itself is the problem — you would also lose Spanish and the handoff language fix.)*

| Version | ID | Notes |
|---|---|---|
| **v18** | `a6f6b620-af15-4e43-b0d8-bfbbb2d64a46` | **Current.** The phone can read an existing claim back. `lookup_claim` attached to `claims_concierge` behind `verify_identity`; the caller is never asked to say a claim reference out loud; several claims are disambiguated by reading the most recent one back; the status sentence is composed in Python and relayed word for word. 47/47 canaries green |
| **v17** | `dcc20863-3746-4e43-a2c9-ed30e0611479` | **Roll back to this.** The call actually opens with the greeting. Everything except the claim-status beat is byte-identical to v18 |
| v16 | `09a1f14d-be24-40ba-abbd-a06e495f5d0d` | Superseded. The instruction that greets and asks *"how can I help you today?"* — but on a real call it still opened by demanding name and policy ID. Everything except the opening line is byte-identical to v17 |
| v15 | `17b2e438-b132-49f1-8b32-190b132225ae` | Superseded. Language follows the caller across the `claims_concierge` to `claim_intake` handoff — the defect that ended a live call with a hang-up |
| v14 | `5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc` | The phone agent speaks Spanish, following the caller from the first word. Conversational layer fully bilingual; the deterministic strings are still English |
| v13 | `5d9df25c-3771-45bb-bd20-b28978cc5955` | Superseded. Assessor briefing packet on the escalation path, filed on the email-confirmation turn; `[ASSESSOR] [VOICE]` subject; money as whole dollars; also carries the `DIAGNOSTIC_INCOMPLETE` recovery fix |
| v12 | `9227210b-e46b-41bd-af7d-59db48abb3a6` | **Never deployed. Do not roll onto it.** Cut mid-plan; its packet mailer throws on every send (unsupported `timeout` argument) so no assessor email is ever delivered |
| **v11** | `b17c9a26-3485-4658-9259-dfa4839a7977` | **Roll back to this.** No self-narration; no packet |
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

**Stay on v13.** If it misbehaves, drop to **v11**, then **v10**, then **v9** — same conversation
quality, emails you, only difference is the email is composed at a slightly later step.

The two unnumbered versions above were cut during a parallel edit and point the claim email
at a different mailbox. Don't roll onto them by accident.

**The email cannot break the call.** If the network or the mail provider fails, the agent
falls back to a drafted message and the conversation continues untouched. You lose the
inbox moment, nothing else.

**Editing anything?** A deployment pins a *version*. Changing the app does **not** change
what the phone number serves — you must cut a new version and repoint the deployment.

---

## If something goes wrong  *(chat)*

**Roll back the chat deployment to v19 `619e13a1`** — the version live before v20. You lose
**exactly two things, and both matter**: (a) **the claim-status confirm path breaks** — answering
*"yes"* to *"Is that the one you mean?"* returns *"there is nothing for me to read back"*, so
**Scenario C2 must not be run on v19**; and (b) **the `[RECORD] [CHAT]` email stops being sent** on
the auto-approve path, because the cross-sell widget ends the turn before the mailer runs. The
opening greeting, the photo ask, the claim store, the status lookup itself, the decision card, the
tariff, the customer email, the assessor packet and the cross-sell are all **byte-identical** between
v19 and v20 — v20 changes two instructions and nothing else, and touches no tool. One call:

```bash
curl -X PATCH \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"appVersion":"projects/insurance-agent-demo-500614/locations/us/apps/a2f621e4-9faf-505a-b804-22471f022366/versions/619e13a1-2a31-4627-aca3-2ff3a32e336b"}' \
  "https://ces.googleapis.com/v1/projects/insurance-agent-demo-500614/locations/us/apps/a2f621e4-9faf-505a-b804-22471f022366/deployments/d7bfbb93-8cee-43fe-9095-bc5775f353bd?updateMask=appVersion"
```

Then **reload `http://localhost:3000`** so the widget picks the version up.

> **v14 is live and the draft matches it byte for byte** (quick task `260812-o5l`,
> 2026-08-13). Both canaries that held the gate on 2026-08-12 were **real defects and both are
> fixed**: the cross-sell was being killed by an `end_session` on the send-away turn (caused by
> the new closing wording, proven against a live-v13 control), and the photo could not be
> captured mid-diagnostic because a `beforeToolCallback` refused `assess_screen_crack` before
> its body ran. The full canary set — decision card, cross-sell, spoken send-away, one customer
> email, deterministic tariff, six-section packet on the email turn, attachment present and
> correctly absent — passed on the **deployment itself**, not only the draft.

| Version | ID | Notes |
|---|---|---|
| **v18** | `a6f6b620-af15-4e43-b0d8-bfbbb2d64a46` | **Current.** The phone can read an existing claim back. `lookup_claim` attached to `claims_concierge` behind `verify_identity`; the caller is never asked to say a claim reference out loud; several claims are disambiguated by reading the most recent one back; the status sentence is composed in Python and relayed word for word. 47/47 canaries green |
| **v17** | `dcc20863-3746-4e43-a2c9-ed30e0611479` | **Roll back to this.** The call actually opens with the greeting. Everything except the claim-status beat is byte-identical to v18 |
| v16 | `09a1f14d-be24-40ba-abbd-a06e495f5d0d` | Superseded. The instruction that greets and asks *"how can I help you today?"* — but on a real call it still opened by demanding name and policy ID. Everything except the opening line is byte-identical to v17 |
| v15 | `17b2e438-b132-49f1-8b32-190b132225ae` | Superseded. Language follows the caller across the `claims_concierge` to `claim_intake` handoff — the defect that ended a live call with a hang-up |
| v14 | `5d02f14c-8cba-4bf4-aa3a-b9caf57ffddc` | The phone agent speaks Spanish, following the caller from the first word. Conversational layer fully bilingual; the deterministic strings are still English |
| v13 | `5d9df25c-3771-45bb-bd20-b28978cc5955` | Superseded. Assessor briefing packet on the escalation path, filed on the email-confirmation turn; `[ASSESSOR] [VOICE]` subject; money as whole dollars; also carries the `DIAGNOSTIC_INCOMPLETE` recovery fix |
| v12 | `9227210b-e46b-41bd-af7d-59db48abb3a6` | **Never deployed. Do not roll onto it.** Cut mid-plan; its packet mailer throws on every send (unsupported `timeout` argument) so no assessor email is ever delivered |
| **v11** | `b17c9a26-3485-4658-9259-dfa4839a7977` | **Roll back to this.** No self-narration; no packet |
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

**Stay on v13.** If it misbehaves, drop to **v11**, then **v10**, then **v9** — same conversation
quality, emails you, only difference is the email is composed at a slightly later step.

The two unnumbered versions above were cut during a parallel edit and point the claim email
at a different mailbox. Don't roll onto them by accident.

**The email cannot break the call.** If the network or the mail provider fails, the agent
falls back to a drafted message and the conversation continues untouched. You lose the
inbox moment, nothing else.

**Editing anything?** A deployment pins a *version*. Changing the app does **not** change
what the phone number serves — you must cut a new version and repoint the deployment.

---

## If something goes wrong  *(chat)*

**Roll back the chat deployment to v12 `26c3aebd`** — identical behaviour to what is live,
except the packet's money prints as `3000.0` / `25.0` and the subject carries no `[CHAT]`
token. One call:

```bash
curl -X PATCH \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"appVersion":"projects/insurance-agent-demo-500614/locations/us/apps/a2f621e4-9faf-505a-b804-22471f022366/versions/26c3aebd-d72b-4ec5-861d-8a9fabb140cf"}' \
  "https://ces.googleapis.com/v1/projects/insurance-agent-demo-500614/locations/us/apps/a2f621e4-9faf-505a-b804-22471f022366/deployments/d7bfbb93-8cee-43fe-9095-bc5775f353bd?updateMask=appVersion"
```

Then **reload `http://localhost:3000`** so the widget picks the version up.

> **✅ Resolved.** That draft became v14 `cdca14e3` (2026-08-13), and v15 `129f8b31` now
> supersedes it. The chat draft and the deployment are in step: `d7bfbb93` serves v15.

| Version | ID | Notes |
|---|---|---|
| **v18** | `d0e4bfef-6d5f-43b3-b490-3d9036d030e2` | **Current.** The widget's first message ACTUALLY greets you. v17 changed the instruction but a callback still hard-returned the identity demand as the opening message, and the agent then asked a SECOND time. 93/93 canaries green |
| **v17** | `64be15eb-947f-4f36-8af2-2aefd225742b` | **Roll back to this.** The instruction that greets and asks *"how can I help you today?"* — but the widget still opened by demanding identity, twice. Everything except the opening message is byte-identical to v18 |
| v16 | `be4c83bb-825a-4feb-af88-379113543aa7` | Superseded. A photo is asked for on every chat claim, and every resolved claim files a photo-carrying record email — `[RECORD] [CHAT] … - APPROVED` on auto-approve, `[ASSESSOR] [CHAT] …` on escalation |
| v15 | `129f8b31-f06e-48cc-90fb-15dcf8611db1` | Superseded. Every resolved claim is written to the shared claim store on the decision turn, on both branches, and a verified customer can ask for the status of a claim filed in an earlier session and have it read back. Nothing the customer sees on any existing beat changed — the full v14 canary set was re-run green before this shipped |
| v14 | `cdca14e3-5b0e-4675-9861-f5f22736362f` | The customer's uploaded photo is attached to the assessor packet as a real file; the spoken send-away line and the customer email both stop asking for a photo already sent; `assess_screen_crack`'s callback gate re-keyed so a photo volunteered mid-diagnostic is still capturable; the AUTO_APPROVE send-away turn can no longer close the call and kill the cross-sell. Losing v15 costs only the claim-status beat |
| v13 | `1eb3fd5c-5aff-46c8-b572-e3fe18bf966f` | Packet money as whole dollars (`$3,000` / `$25`) and an `[ASSESSOR] [CHAT]` subject token. `claim_intake` is byte-identical to v12 — only the packet composer and the mailer's subject changed |
| v12 | `26c3aebd-d72b-4ec5-861d-8a9fabb140cf` | Packet filed on the email-confirmation turn; send-away line still spoken; escalation and close actually fire |
| v11 | `838b6d2b-1e9f-44f8-b319-941c3e4ea10b` | **Never deployed. Do not roll onto it.** Cut mid-task; files the packet on the right turn but the agent says *nothing* to the customer on that turn |
| v10 | `658472a0-05be-4a28-9d9f-9774ebe0dd05` | Assessor briefing packet, but on the turn *after* the email — needs one extra customer message or nothing is filed |
| v9 | `160dc3b2-571c-480f-b901-e4dbe8947f70` | Spoken decision explanation, no packet |
| v8 | `3f85b1d8-4810-44eb-85e6-39adc42593c9` | Cross-sell buttons; decision turn may say nothing |
| v7 | `bb14cdcc-d723-4be1-85af-9f4451e22ed5` | Decision card fixed; no working cross-sell |

Rolling back to v9 costs you the packet beat and nothing else — the card, the spoken decision,
the cross-sell, the tariff and the customer email are byte-identical in v9 and v10.

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

**Revoke the Resend API key.** It's in the `send_claim_email` / `resolve_claim` /
**`send_case_record_email`** tool source and baked into every version snapshot from v4 onward
(for `send_case_record_email`, from chat v10 `658472a0` and from **voice v12 `9227210b` /
v13 `5d9df25c`** onward), readable by anyone with read access to the project. Delete it in the
Resend dashboard once you're done — one key revocation covers **both apps and all three tools**,
since voice and chat each hold their own copy of the same key.
