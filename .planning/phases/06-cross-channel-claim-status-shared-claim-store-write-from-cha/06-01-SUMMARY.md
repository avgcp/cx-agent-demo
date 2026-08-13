---
phase: 06
plan: 01
subsystem: shared-claim-store
tags: [spike, ces, gcs, hmac, sigv4, claim-store, auth]
requires: []
provides:
  - "the store mechanism 06-02/06-03/06-04 hardcode: GCS + HMAC + S3-compatible XML API"
  - "the claim record schema and key layout"
  - "the sandbox capability map (egress open, no Google credential, ToolContext writable, async available)"
affects: ["06-02", "06-03", "06-04"]
tech-stack:
  added: ["Google Cloud Storage (XML API, S3-compatible)", "AWS SigV4 signing in stdlib hmac/hashlib"]
  patterns: ["outbound HTTPS + shared secret in tool source (the resolve_claim/Resend shape)"]
key-files:
  created:
    - ".planning/phases/06-cross-channel-claim-status-shared-claim-store-write-from-cha/06-01-SPIKE-FINDINGS.md"
  modified: []
decisions:
  - "LOCKED by the user 2026-08-12 (literal reply: `as-proven`). Rungs 3 (Cloud Run, costed not built) and 4 (third-party host) formally rejected."
  - "Rung 2 chosen: GCS via the S3-compatible XML API with a GCS HMAC key. No new GCP service, no operated component, no data leaves the project."
  - "Bucket gs://meridian-claim-store-500614 and SA claim-store-writer@ are ACCEPTED as permanent - do NOT delete."
  - "Store tools must assert on status_code, never on ExternalResponse.ok, which is True even at status_code 0."
  - "ces_requests is an injected global; `import ces_requests` fails."
  - "Two objects per claim — claims/by-policy/{POLICY_KEY}.json (newest-first array) and claims/by-ref/{CLAIM_KEY}.json — so both lookups are one GET."
  - "Keys normalised (PDP-100294 -> PDP100294) so a spoken policy id survives speech-to-text."
  - "No pre-composed prose in the stored record; the status sentence is composed at read time in Python."
metrics:
  ces_conversations_spent: 0
  completed: 2026-08-13
---

# Phase 6 Plan 01: Claim-Store Spike Summary

**Can a CES Python tool obtain a Google credential? NO** — the metadata server is unreachable
(`status_code: 0`, `reason: "Error fetching from URL"`) while plain `http://example.com/` returned
200 in the same invocation; `google.auth` is `ModuleNotFoundError`; no credential env vars exist.
**Is egress open? YES** — arbitrary public HTTPS hosts returned 200; egress is not allowlisted to
`api.resend.com`. **Rung chosen: Rung 2 — Google Cloud Storage over the S3-compatible XML API,
signed with a GCS HMAC key**, proven by a write→read→delete round-trip executed from inside a CES
Python tool (PUT 200, GET 200 byte-identical, anonymous GET 403, DELETE 204, GET-after-delete 404).
**Can `record_claim` run asynchronously? YES** — `ExecutionType` accepts `ASYNCHRONOUS`, and
`ces_requests.async_*` exists as a second lever, so 06-04 may have no voice-latency problem at all.
**Zero CES conversations were spent**; `apps.executeTool` on a disposable app carried every probe.

## What was done

Three things, in order.

**1. The sandbox was measured, not read about.** A disposable app `ZZ-SPIKE-06-01`
(`618417eb-7c66-472b-b4fc-3f5797038d31`) was created with one `pythonFunction` tool, and every
probe A–H ran through `POST {app}:executeTool` — no conversation, no quota. The findings file's
`## SANDBOX_CAPABILITIES` records each probe with the literal status code it produced.

Two discoveries change how tools get written from here on:

- **`import ces_requests` fails.** It is `ModuleNotFoundError`. `ces_requests` is an **injected
  global**, as are `requests`, `context`, `tools`, `async_tools`, `Part`, `Blob` and
  `StatusError`. A store tool that guards its HTTP call with a `try: import ces_requests` will
  silently do nothing — the exact 05-07 failure shape.
- **`ExternalResponse.ok` is `True` even when `status_code` is `0`.** The metadata probe returned
  `ok: true, status_code: 0, reason: "Error fetching from URL"`. Any tool branching on `.ok` would
  have reported a successful store write that never happened. **Branch on `.status_code`.**

Also settled: `ces_requests` accepts `headers`, `params`, `data`, `json` and rejects `timeout`,
`allow_redirects`, `verify` — the 05-07 `timeout=20` defect, generalised; and `context.state` is a
plain writable `dict`, so a lookup tool can read the session-authenticated policy id itself instead
of trusting a model-supplied parameter (the control for threat T-06-03).

**2. The auth ladder was walked and stopped at rung 2.** Rung 1 (Google-credentialed direct) died on
three independent negatives. Rung 2 succeeded: a GCS bucket, a service account scoped to that bucket
alone, and an HMAC key signed with stdlib `hmac`/`hashlib` against `storage.googleapis.com` — the
Resend shape (outbound HTTPS + a secret in tool source) pointed at a Google store inside the
customer's own project. Rungs 3 (Cloud Run) and 4 (third-party host) were costed and left unbuilt.

A second probe ran the **real access pattern**, not just a toy object: cold-start 404, write, append
a second claim by read-modify-write, one-GET lookup by policy returning the newest claim first,
one-GET lookup by reference, a 404 for an unknown reference, a correctly-signed LIST, then cleanup.
Ten signed HTTP calls in one tool invocation, **3.56 s** end to end.

**3. The schema was written down field-by-field** with every criterion-1 field given a home, and the
lookup-key question resolved in favour of two objects per claim so that neither lookup needs a query
engine.

## Deviations from Plan

### Auto-fixed issues

**1. [Rule 3 - Blocking] `apps.executeTool` was used instead of driving a conversation.**
- **Found during:** Task 1. The plan budgeted one conversation for the probe.
- **Issue:** A conversation costs quota this project has been rationing for weeks, and the prompt's
  standing guidance is to prefer `executeTool`.
- **Fix:** Every probe ran through `POST {app}:executeTool`. **Zero conversations spent** against
  the plan's budget of two.

**2. [Rule 3 - Blocking] The probe tool could not take `context` as a parameter.**
- **Found during:** Task 1. `POST .../tools` returned
  `400 "Python function probe_sandbox has parameters without types: [context]"`.
- **Fix:** `context` is not a parameter at all — it is an injected global. The probe was rewritten to
  read `globals()["context"]`, which is also how the live `assess_screen_crack` tool does it.

**3. [Rule 3 - Blocking] `curl` mangled the `:executeTool` URL when passed as a shell variable**,
  reporting `Could not resolve host: xecuteTool`. Fixed by passing the URL via `--url` with the
  literal string. Worth knowing — it looks like an API failure and is not one.

**4. [Rule 1 - Bug] The first LIST call returned 403 because the canonical query string was not
  signed**, not because of a permission gap. Fixed by including the encoded query string in the
  canonical request; LIST then returned 200. Recorded in the findings so no later plan re-litigates
  it as an IAM problem. (The design does not need LIST.)

**5. [Rule 2 - Missing critical functionality] An unauthenticated-read check was added to the store
  probe.** The plan required the store's access posture to be recorded; asserting it by execution
  (anonymous `GET` → **403**) rather than by configuration inspection is what actually closes threat
  T-06-04.

## Resources created (a cost the checkpoint must accept or reject)

| Resource | Detail |
|---|---|
| `gs://meridian-claim-store-500614` | US multi-region, uniform bucket-level access, **public access prevention enforced**, currently **empty** |
| `claim-store-writer@insurance-agent-demo-500614.iam.gserviceaccount.com` | new service account |
| IAM binding | `roles/storage.objectAdmin` **on that bucket only** |
| HMAC key | minted for the round-trip, then **deactivated and deleted**; `gcloud storage hmac list` is empty |

**No GCP service was enabled.** `run`, `artifactregistry`, `cloudbuild`, `firestore` and
`secretmanager` all remain disabled.

## Blocking external dependency

**NONE. Cleared.** Nothing needs deploying and no GCP service needs enabling before 06-02 can run.

The one blocking item — the user's lock (Task 3) — was **answered `as-proven` on 2026-08-12**. Rung
2 is LOCKED for the whole phase; rungs 3 (Cloud Run, costed but never built) and 4 (third-party
host) are formally rejected. The lock explicitly accepts the HMAC secret in tool source, the schema
as written, and **the bucket and service account as permanent — they must NOT be deleted.**
**06-02 is unblocked.**

## MANDATORY reading for 06-02 / 06-03 / 06-04

Two of these would each let a store tool **silently report success on a write that never happened**.

1. **`import ces_requests` FAILS** (`ModuleNotFoundError`) — it is an **injected global**. Use
   `globals()["ces_requests"]`. Same for `requests`, `context`, `tools`, `async_tools`, `Part`,
   `Blob`, `StatusError`.
2. **`ExternalResponse.ok` is `True` even when `status_code` is `0`.** Every store tool **MUST**
   assert on `status_code` (`200`/`201` write, `200` read, `404` miss) and **never** on `.ok`.
3. **`ExecutionType` accepts `ASYNCHRONOUS`** (`"ASYNC"` → 400, `"ASYNCHRONOUS"` → 200), and
   `ces_requests.async_*` exists — **this may remove 06-04's voice-latency concern entirely.**
   Check before designing around a problem that may not exist.

## Isolation and safety

- **Zero non-GET calls** to `6e01e4a5`, `a2f621e4` or `9ae7a0c3`. The fork `9ae7a0c3` was never
  called by any method.
- Chat deployment `d7bfbb93` still serves **v14 `cdca14e3`** (`updateTime 2026-08-13T01:35:50Z`) and
  voice deployment `d28bbcb0` still serves **`5d9df25c`** (`updateTime 2026-08-12T20:51:46Z`) — both
  re-read at plan close and byte-identical to the readings taken at plan start.
- The disposable app was **deleted**; `GET` on it returns **404**, and the app list is back to its
  original five.
- `resolve_claim`'s source was never read, echoed or persisted. The one tool-list read taken from the
  chat app printed **only `def` lines** for the email/claim tools and its response file was deleted
  in the same call.
- The HMAC secret existed only inside a single executor process. Secret-leak gate
  (`re_…|ya29\.|AKIA|GOOG1E`) returns **0 matches** in the findings file and **0** in the scratchpad;
  the only repo hits are the plan files quoting the gate's own regex.

## Known Stubs

None. Nothing in this plan renders to a user; the artifact is a written verdict backed by executed
round-trips.

## Self-Check: PASSED

| Claim | Verification |
|---|---|
| `06-01-SPIKE-FINDINGS.md` exists | FOUND, 342 lines (plan required ≥120) |
| Five literal headings, exactly one each | `SANDBOX_CAPABILITIES=1 AUTH_LADDER=1 STORE_MECHANISM=1 CLAIM_SCHEMA=1 STORE_DECISION=1` |
| Every criterion-1 and lookup field present in `## CLAIM_SCHEMA` | missing fields: **NONE** (16 fields checked programmatically against the section body) |
| Round-trip assertions | run 1 `wrote/read_back/payload_byte_identical/deleted = True/True/True/True`, statuses `200/200/204`; run 2 `deleted=True, most_recent_is_newest=True, lookup_by_ref_matches=True, unknown_ref_status=404` |
| Probes A–H all recorded, none "not run" | result keys `A_imports, B_env, C_context, D_egress_public, E_google_apis, F_metadata, F2_metadata_detail, G_kwargs, G0_*, H_verbs` + the live `executionType` enum test |
| No secret in any artifact or raw probe output | pattern scan over the findings file, the summary and every scratchpad file: **NONE** |
| SPIKE app gone | `GET` → **404**; app list back to 5 |
| Both deployments unchanged | `d7bfbb93` → `cdca14e3` @ `2026-08-13T01:35:50.254056Z`; `d28bbcb0` → `5d9df25c` @ `2026-08-12T20:51:46.574949Z` |
| No git commit made | tree deliberately left dirty per instruction |
