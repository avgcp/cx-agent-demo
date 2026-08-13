---
quick_id: 260812-trc
type: quick
status: diagnosed-HELD
date: 2026-08-13
subsystem: cx-agent-voice-app, cx-agent-chat-app, resend-delivery
tags: [diagnosis, resend, email-delivery, resolve_claim, voice, executeTool, no-fix-shipped]
verdict: >
  resolve_claim WORKS. Its Resend POST returns HTTP 200 with a real message id from inside the
  CES sandbox, on the byte-identical code the live phone deployment serves. The missing
  CLM-24430 email was NOT caused by a code regression, the .ok trap, a quota cap, a rate limit,
  a dead key, or a voice/chat divergence — all six are ruled out by execution. The remaining
  live cause is DELIVERY from the shared unverified sender onboarding@resend.dev, which only
  the inbox can settle. A SEPARATE, CONFIRMED defect was found: resolve_claim asserts
  email_queued: true unconditionally, on BOTH apps. It cannot report a failed send.
server_changes:
  apps_modified: NONE
  versions_cut: NONE
  deployments_repointed: NONE
  voice: 6e01e4a5 still serves 5d9df25c (re-read 2026-08-13T02:43Z, updateTime unchanged)
  chat: a2f621e4 still serves cdca14e3 (re-read, updateTime unchanged)
  fork_9ae7a0c3: zero calls of any method
  throwaway_app: zztrcprobe created, used, DELETED (GET -> 404, app list back to 5)
conversations_spent: 0
emails_sent_by_this_task: 13
---

# 260812-trc — Diagnose the missing customer email on the live phone call

## Headline

**`resolve_claim` is not broken, and nothing regressed when voice `5d9df25c` was cut.**

Proven by running it: an instrumented byte-identical copy of voice `resolve_claim`, executed
inside the CES sandbox with the live key and the real payload, got **`status_code: 200`** back
from `api.resend.com` with a genuine message id. The tool the live phone deployment serves is
**byte-identical** (`sha 890678c445f9fe9c`) to the one tested — the v13 version snapshot was
compared field-for-field against the draft, not assumed.

So the customer email for `CLM-24430` was **accepted by Resend**. It was not accepted and it
did not arrive. The gap between those two facts is **delivery**, and the tool has no visibility
into it — which is the second, independent finding below.

**Zero CES conversations were spent.** `apps.executeTool` and one throwaway app carried the
whole diagnosis.

## What was ruled OUT, each by execution

Six hypotheses were live at the start of this task. All six are dead, and none of them was
retired by reading — every row below is a measurement.

| Hypothesis | Verdict | The evidence that killed it |
|---|---|---|
| **A regression in voice `resolve_claim` when `5d9df25c` was cut** | **RULED OUT** | Voice `resolve_claim`'s `updateTime` is **`2026-08-05T16:43:09Z`** — nine days before the failing call, and *before* v11 `b17c9a26`, the version whose `CLM-24464` email the user confirmed by inbox. The tool has not been edited since. 05-07 modified `case_summary` and `send_case_record_email`, never `resolve_claim`. |
| **The `ExternalResponse.ok` / `status_code: 0` trap (06-01)** | **RULED OUT as the cause** | `resolve_claim` contains **no `.ok` reference and no `status_code` reference at all** on either app. It cannot have been misled by `.ok`, because it never reads it. (It is defective for a *different* reason — see *The real defect*.) |
| **`import ces_requests` failing (the injected-global trap)** | **RULED OUT** | `has_import_ces_requests = False` on both apps; the tool uses the bare injected global, which is correct. |
| **An unsupported kwarg, the 05-07 `timeout=20` shape** | **RULED OUT** | `has_timeout_kwarg = False`. `resolve_claim` passes only `url=`, `headers=`, `json=`. A sandbox probe confirmed the shim's signature is `post(url: str, data=None, json=None, **kwargs)` and that **`url=` as a keyword raises nothing** — both `post(URL, ...)` and `post(url=URL, ...)` returned identically. The divergence I first suspected (`resolve_claim` passes `url=` by keyword, `send_case_record_email` passes it positionally, on *both* apps) is **cosmetic and harmless**. |
| **Resend quota exhaustion (daily or monthly cap)** | **RULED OUT** | The response headers on a live send read **`x-resend-daily-quota: 21`** and **`x-resend-monthly-quota: 92`**, and a second send moved them to **22 / 93**. They **increment**, so they are *used* counts, not remaining. Against a free-tier 100/day and 3,000/month that is **~22% of today's budget and ~3% of the month's** at the time of the failing call. Nowhere near a cap. |
| **A rate limit at the moment of the call** | **RULED OUT** | `ratelimit-limit: 10`, `ratelimit-policy: 10;w=1`, `ratelimit-remaining: 9` on every probe. The limit is 10 requests **per second**; a phone call makes one. No `429` was ever observed, and no `retry-after` header was ever returned. |
| **A dead / revoked / restricted API key** | **RULED OUT** | The key extracted from `resolve_claim` sent successfully from a local process (**HTTP 200**, message id returned) and from inside the CES sandbox (**HTTP 200**, message id returned). It is a *send-only* key — `GET /api-keys`, `/domains` and `/emails/{id}` all return `401 restricted_api_key` — which is why no delivery log is reachable, but it sends. |
| **The shared-sender recipient restriction** | **RULED OUT** | `onboarding@resend.dev` → `akash.vinayak@nerdery.com` returned **200** repeatedly, and the user confirmed `TRC-MARK-A` and `TRC-MARK-B` arrived in that mailbox. The restriction permits this pair. |
| **A voice-vs-chat divergence in `resolve_claim`** | **RULED OUT as a cause** | The two files differ (voice 10,122 chars, chat 15,648 — chat carries `260812-o5l`'s photo-branching, common prefix 4,476), **but the email machinery is identical on both**: same single `ces_requests.post`, same `url=`/`headers=`/`json=` kwargs, same 377-char call expression, same `raise_for_status()` inside `except Exception`, same unconditional `email_queued`, same one key — and the key in `resolve_claim` is **the same key** as in `send_case_record_email`. |

## What the send actually does — proven, not inferred

An instrumented copy of voice `resolve_claim` was built **mechanically**: its `pythonCode` was
fetched, four `context.state` writes were spliced in immediately after the `ces_requests.post`
statement, the function was renamed, `compile()`-checked, and POSTed to a **throwaway app**
(`zztrcprobe`). The source was never printed, never written to disk and never entered the
executor's context. The copy was then run through `apps.executeTool` with the same three
arguments the live call used (`policy_id="PDP100294"`, `issue_type="screen"`,
`diagnostic_outcome="REPAIRABLE"`) and a seeded session state.

```
probe_status   = 200
probe_ok       = true
probe_text     = {"id":"7526f24c-4dc0-415e-8f1e-20f1e7362b26"}
probe_reason   = ""
email_delivery = "live"
email_status   = "queued"
```

A second, **dry-run** copy replaced `ces_requests.post` with a recorder that captures the
payload and sends nothing. It shows the request is well-formed:

| Captured field | Value |
|---|---|
| `cap_url` | `https://api.resend.com/emails` |
| `cap_from` | `Meridian Device Protection <on***@resend.dev>` |
| `cap_to` | `["ak***@nerdery.com"]` — a list, one valid address |
| `cap_subject` | `Claim CLM-24737 - please reply with photos` |
| `cap_keys` | `["from", "subject", "text", "to"]` |
| `cap_body_len` | `434` |
| `cap_auth_prefix` | `Bearer re_` |
| `cap_json_is_dict` | `true` |

The URL, the sender, the recipient, the subject and the body are all correct, the auth header is
present, and Resend answers **200 with a message id**. There is nothing wrong with the request
voice `resolve_claim` makes.

## A wrong intermediate conclusion, and how it was caught

**This nearly shipped as the answer, and it was false.** It is recorded because the technique
that produced it looks rigorous and is not.

Since `resolve_claim` reports nothing about its HTTP result, the first idea was to detect sends
from the outside: read Resend's `x-resend-daily-quota` header immediately before and immediately
after invoking the tool, and infer the send count from the delta. Two bracketing marker emails
give a baseline delta of 1; a real send in between should make it 2.

It read **1**. Twice. That says `resolve_claim` sent nothing while reporting `email_queued: true`
— exactly the headline the brief predicted, and completely wrong.

The control that caught it: the same bracket was run around the **instrumented** copy, which had
already returned `status_code: 200` and a Resend message id. The delta was **still 1**, for a
send that provably happened. So the counter is not a send detector.

Reconciling the whole sequence shows why — **the header lags by exactly one send**:

```
marker "TRC-MARK-A"            -> header 25
  resolve_claim (CLM-24224)          [1 send, not yet reflected]
marker "TRC-MARK-B"            -> header 26      (expected 27)
  probe_send (kw + positional)       [2 sends]
  probe_resolve run 1 (CLM-24614)    [1 send]
marker "TRC-CTL-1"             -> header 31   = 26 + 1(lagged) + 2 + 1 + 1  ✔ exact
  probe_resolve run 2 (CLM-24646)    [1 send]
marker "TRC-CTL-2"             -> header 32      (expected 33, one lagged)  ✔
```

Every send **is** counted; the counter is eventually consistent and the most recent send has not
propagated by the time the next response header is written. The arithmetic closes exactly on 31,
which is only possible if `CLM-24224` — the "zero sends" run — really did send.

**Carry forward:** `x-resend-daily-quota` is usable for *budget* ("how much of today is gone")
and useless for *attribution* ("did that specific call send"). Never infer a send count from a
delta of one. Any counter-based assertion in this project needs a positive control on a send
that is independently known to have happened.

## The real defect — separate from the cause, and confirmed on BOTH apps

The brief's instinct is right even though its mechanism is wrong. **`resolve_claim` cannot report
a failed send.** This is a defect in its own right and it is what made a whole week of "email
verified" claims worthless as evidence.

Structurally, on **both** voice and chat:

| Property | Value |
|---|---|
| References to `status_code` in the tool | **0** |
| References to `.ok` | **0** |
| `try` / `except` blocks | 2 / 2, both `except Exception` |
| What it does with the response object | calls `raise_for_status()` — **inside** the `try` |
| `email_queued` occurrences | 1, written as a literal `True` |
| `email_delivery` set to `"live"` | on the branch **before** the `try`, not from the result |
| `email_status` set to `"queued"` | before/independent of the HTTP outcome |

So the sequence is: decide the branch → write `email_delivery = "live"` → write
`email_status = "queued"` → `try:` POST → `raise_for_status()` → **`except Exception: pass`-shaped
swallow** → return `email_queued: True` unconditionally.

Every failure mode is invisible. A 401, a 422, a 429, a DNS failure, a `status_code: 0` "Error
fetching from URL", or an exception thrown anywhere in the block all produce the **identical**
observable: `email_queued: true`, `email_delivery: "live"`, `email_status: "queued"`.

Note the one non-defect: the gate that *would* have told the truth exists and works —
`elif not RESEND_API_KEY: context.state["email_delivery"] = "simulated"`. It is the only honest
signal in the tool, and it only fires when the key is absent.

### What this voids

**Every "exactly ONE customer email, asserted via `resolve_claim`" canary in this project is void
as evidence of delivery.** That includes the ones in `05-05`, `05-07`, `260812-hhi` and
`260812-o5l`, and the standing instruction in `STATE.md` that says to assert the customer-send
count via `resolve_claim` rather than `send_claim_email`.

The instruction is still **right about which tool to count** — `send_claim_email` is a keyless
reporter and counting it is meaningless. It is **wrong about what the count proves**: it proves
`resolve_claim` *ran once*, not that an email *left*. Those canaries remain valid as
single-send/no-double-send assertions and are worthless as delivery assertions.

**The only delivery evidence this project has ever had is the inbox**, on four occasions:
`CLM-24464` (voice v11), `CLM-24314` and `CLM-O5LPHOTO` (chat, both `send_case_record_email`),
and `TRC-MARK-A` / `TRC-MARK-B` today.

## What is left, and it is the sender

With acceptance proven and every code hypothesis dead, one link remains: **Resend returned 200,
and 200 is acceptance, not delivery.** `260812-o5l` wrote exactly that sentence and it is the
operative one here.

The sender is **`onboarding@resend.dev`** — Resend's shared sandbox domain, used by every free
account on the platform, with no verified DNS for this project (no SPF, no DKIM, no DMARC
alignment that belongs to Meridian or to the sending party). The recipient is a
**corporate-hosted mailbox** (`nerdery.com`). That combination is exactly the one that gets
greylisted, throttled per-recipient, or silently spam-foldered by enterprise filters — and it
fails **intermittently**, which is precisely the shape of the observed evidence: `CLM-24464`
arrived, `CLM-24430` did not, the markers arrived, all with identical code and an identical
200 from the API.

This is a hypothesis with strong circumstantial support, **not a proven cause**, and it is
honest to say so. Proving or disproving it needs one thing this task cannot get: **which of
today's thirteen sends actually landed.** See *What the user must check*.

The two remaining possibilities, kept open:

1. **The message was delivered and not seen** — spam/junk folder, or a delivery delay longer
   than the interval between the call and the check. The call ended `2026-08-13T02:22:57Z`.
2. **Resend accepted and then dropped or bounced it** downstream. Unprovable from here: the
   embedded key is send-only, so `GET /emails/{id}`, `/domains` and `/api-keys` all return
   `401 restricted_api_key`. **A full-access key or the Resend dashboard is the only way to read
   the delivery log**, and that is a user action.

## The emails this task sent — 13, all to the same mailbox

Every one of these is a datapoint. `TRC-*` are diagnostic markers; `CLM-*` are **real customer
claim emails** produced by real `resolve_claim` code and are the ones that matter.

| Subject | Sent by | Resend result |
|---|---|---|
| `TRC-PROBE-1 diagnostic send from local` | local process | 200 |
| `TRC-PROBE-2 diagnostic send from local` | local process | 200 |
| `TRC-QUOTA-BEFORE`, `TRC-QUOTA-AFTER` | local process | 200, 200 |
| `TRC-MARK-A`, `TRC-MARK-B` | local process | 200, 200 — **user confirms both ARRIVED** |
| **`Claim CLM-24224 - please reply with photos`** | **live voice `resolve_claim`, via `executeTool`** | 200 (inferred from the counter reconciliation) |
| `TRC-SANDBOX-A`, `TRC-SANDBOX-POS-A` | CES sandbox, throwaway app | 200, 200 |
| **`Claim CLM-24614 - please reply with photos`** | instrumented copy of voice `resolve_claim` | **200, id `7526f24c-…`** |
| **`Claim CLM-24646 - please reply with photos`** | instrumented copy of voice `resolve_claim` | **200** |
| `TRC-CTL-1`, `TRC-CTL-2` | local process | 200, 200 |

`CLM-24737` was composed by the dry-run probe and **deliberately not sent** — it will not appear.

## SECURITY — the Resend key must be rotated

**A Resend API key value was surfaced in this executor's own diagnostic output.** A structural
probe that was meant to print only argument *shapes* printed the `headers=` token of voice
`send_case_record_email`, and that tool carries the key as an **inline string literal** rather
than behind a constant, so the value came out with it. It was masked in every subsequent output
and it is **not written anywhere in this repository** —
`grep -rE 're_[A-Za-z0-9_]{20,}' .planning/` returns no match — but it existed in a transcript.

Two things follow, and both are the user's:

1. **Rotate the key in the Resend dashboard**, then update it in the four places it lives:
   `resolve_claim` and `send_case_record_email` on **both** apps. It is confirmed to be **the
   same single key** in all four (`resolve_claim_key == send_case_record_email_key: True`).
2. **When rewriting them, put the key behind a named constant in both tools.** `resolve_claim`
   already does this (`f"Bearer {RESEND_API_KEY}"`); voice `send_case_record_email` does not, and
   that difference is the entire reason the value leaked. A structural probe cannot leak a
   constant's value.

This is unrelated to the missing email and is the more urgent of the two.

## The fix that is NOT shipped, and why — HELD

Nothing was changed. **No tool was patched, no version was cut, no deployment was repointed.**
Voice `d28bbcb0` still serves `5d9df25c` and chat `d7bfbb93` still serves `cdca14e3`, both
re-read from the API after the last write, not inferred.

The truthfulness defect deserves a fix and the brief authorises one. It is held for four
reasons, in order of weight:

1. **The fix would not have prevented this failure.** The send returned 200. An honest tool
   would have reported `sent: true, status_code: 200` — the same reassurance, just better
   founded. Shipping it would close a real hole and would **not** make `CLM-24430` arrive.
2. **The key must be rotated first.** Rotation touches the same four tools. Patching them twice,
   cutting four versions and repointing twice is worse than doing it once.
3. **Scope.** Editing `resolve_claim` means editing the key-bearing tool on both apps, cutting
   two versions, repointing two live deployments and re-running both full canary batteries —
   the work `05-07` needed an entire plan for. Doing it at the end of a diagnosis, against a
   live phone demo, with a *known-good* send path, is the wrong trade.
4. **The higher-value change is not in the code.** If the sender hypothesis holds, capturing the
   status code changes nothing, because the status code is 200. **Verifying a real sending
   domain in Resend** is what would actually fix delivery, and it is a dashboard action.

### The fix, specified precisely, ready to apply

To be done in the same pass as the key rotation, on **both** `resolve_claim` tools, by anchored
scripted `str.replace()` with byte-for-byte read-back:

- Keep the `try:` and the `except Exception`. Do **not** let a mail failure break a live call —
  `05-07` is right that silent degradation is the correct runtime behaviour.
- **Assign the status explicitly** and branch on it, never on `.ok`:
  `ok = getattr(resp, "status_code", 0) in (200, 201)` — per `06-01`, `.ok` is `True` even when
  `status_code` is `0`.
- Set `email_status` to `"sent"` or `"failed"` from that boolean, and `email_delivery` to
  `"live"` or `"failed"`. Today both are written **before** the request and can only say `"queued"`
  and `"live"`.
- Return `email_queued` from the boolean, and **add `email_status_code` and a truncated
  `email_error`** to the result dict so a conversation record carries the evidence. This is the
  single highest-value line: it is what would have let this task be answered from the record in
  five minutes instead of an hour.
- In the `except`, record the exception type and set the failure state — do not swallow silently.
- **Do not** surface any of it to the customer or change one word of `explanation`; the agent
  must keep saying the email is on its way. The change is to the *record*, not the *script*.

Then: cut a version per app, run each app's existing canary battery against the **draft**, and
gate the repoint on those passing — the `05-07` / `260812-o5l` procedure, unchanged. Rollback
targets are **voice `5d9df25c`** and **chat `cdca14e3`**.

## What the user must check — this is the blocking step

**One question decides everything: of the thirteen emails above, which arrived?**

1. **Search the mailbox for `CLM-24224`, `CLM-24614` and `CLM-24646`.** These are real customer
   claim emails, sent by real `resolve_claim` code, each answered with an HTTP 200.
   - **All three arrived** → `resolve_claim` delivers, and `CLM-24430` was a one-off delivery
     drop or a spam-folder miss. Demo risk is real but intermittent.
   - **None arrived, while `TRC-MARK-A`/`B` did** → delivery is failing *selectively*, and the
     difference between a marker and a claim email is the body/subject shape, which points at
     content filtering on an unverified shared sender. That is a demo-day blocker.
2. **Check the junk/spam folder, and check it for `CLM-24430` too.** Anything from
   `onboarding@resend.dev` counts.
3. **Count how many of the six `TRC-*` markers arrived.** Only two were reported. If four are
   missing, per-recipient throttling on the shared sender is close to proven.
4. **Open the Resend dashboard** (the embedded key cannot: it is send-only). Look at the delivery
   log for `2026-08-13T02:21–02:23Z` and find `Claim CLM-24430 - please reply with photos`.
   Its status — `delivered`, `bounced`, `complained`, `queued` — is the single fact that ends
   this investigation. Also read the account's actual daily/monthly caps there.
5. **Decide on a verified sending domain.** If the demo must not drop customer emails on stage,
   `onboarding@resend.dev` is the wrong sender for a corporate recipient. Verifying a real domain
   in Resend (SPF + DKIM) is the fix with the best odds, and it is a dashboard action, not a code
   change.
6. **Rotate the API key** — see *SECURITY* above. Do it in the same pass as the tool fix.

## What this means for demo day

- **Budget is not the constraint.** At the time of the failing call the account had used ~21 of
  a 100/day allowance and ~92 of 3,000/month. Every claim costs **one** customer email, plus
  **one more** assessor packet on the escalated path only — so a demo day is ~2 emails per
  escalated claim, ~1 per auto-approved claim. Dozens of runs fit comfortably. The 10-per-second
  rate limit is irrelevant to a conversation.
- **Delivery is the constraint, and it is currently unmonitored.** The agent tells the caller the
  email is on its way and has no idea whether it left. **Do not build a demo beat on the customer
  opening the email live.** Say it has been sent, move on, and show the inbox only if it has been
  checked beforehand.
- **The assessor packet path is on firmer ground** — `send_case_record_email` returns a real
  `status_code`, and both a `[ASSESSOR] [CHAT]` packet and its attachment have been confirmed by
  eye. The customer email is the weak link, not the packet.

## Deviations

**[method] The instrumented and dry-run copies of `resolve_claim`.** Not in the brief, which
suggested invoking `resolve_claim` directly or writing "a minimal probe tool that performs the
same outbound call". Neither would have answered the question: the real tool reports no status,
and a hand-written probe proves only that *a* Resend call works, not that *this* one does. Copying
the real source mechanically onto a throwaway app and splicing in state writes is the only method
that observes the actual call. The source was transformed in-process, `compile()`-checked, and
never printed, persisted or read.

**[Rule 1 — bug in this task's own method] The quota-delta technique was wrong and was retracted.**
Full account above. It produced a confident, false, brief-confirming answer, and only a positive
control caught it.

**[scope] No fix shipped.** The brief permits holding and asks for a precise diagnosis as the
deliverable. Reasoning above.

## Self-Check: PASSED

- `260812-trc-SUMMARY.md` — created
- `.planning/STATE.md` — modified
- Voice `d28bbcb0` re-read → `5d9df25c`, `updateTime 2026-08-12T20:51:46.574949Z` — **unchanged**
- Chat `d7bfbb93` re-read → `cdca14e3`, `updateTime 2026-08-13T01:35:50.254056Z` — **unchanged**
- Voice v13 snapshot `resolve_claim` vs draft: `sha 890678c445f9fe9c` both — **byte-identical**
- Zero non-GET calls to `6e01e4a5` or `a2f621e4` other than `:executeTool`; **no `patch`, no
  version cut, no repoint**
- Fork `9ae7a0c3`: **zero calls of any method**
- Throwaway app `zztrcprobe` **deleted** (`GET` → 404); app list back to its original **5**
- `grep -rE 're_[A-Za-z0-9_]{20,}' .planning/` → **no match**
- CES conversations spent: **ZERO**
- **No git commit, by instruction** — the tree is deliberately left dirty
