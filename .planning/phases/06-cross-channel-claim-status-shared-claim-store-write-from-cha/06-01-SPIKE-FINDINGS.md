---
phase: 06
plan: 01
artifact: SPIKE-FINDINGS
date: 2026-08-13
verdict: FEASIBLE — GCS + HMAC over the S3-compatible XML API, proven by round-trip from inside a CES Python tool
conversations_spent: 0
probe_app: 618417eb-7c66-472b-b4fc-3f5797038d31 (ZZ-SPIKE-06-01, deleted at plan close)
---

# 06-01 — SPIKE: what can a CES Python tool authenticate to, and what store backs the claim record?

## Headline

- **A Google-credentialed token is NOT obtainable from a CES Python tool.** The metadata server is
  unreachable (`status_code: 0`, `reason: "Error fetching from URL"`), `google.auth` /
  `google.cloud.*` are `ModuleNotFoundError`, and none of the 12 environment variables is a
  credential.
- **Outbound egress is OPEN, not allowlisted to `api.resend.com`.** `https://example.com/` → 200,
  `https://api.github.com/zen` → 200, `http://example.com/` → 200.
- **Google API hosts are reachable, only the credential is missing** — `storage.googleapis.com` and
  `firestore.googleapis.com` both returned **401**, not a connection error.
- Therefore the store is **GCS accessed with a shared secret** — an HMAC key against the
  S3-compatible XML API. Same shape as the proven Resend call (outbound HTTPS + a secret in tool
  source), applied to a Google store inside the customer's own project.
- **Zero CES conversations were spent.** All probing ran through `apps.executeTool` on a disposable
  app.

## SANDBOX_CAPABILITIES

Every probe below was executed inside a CES `pythonFunction` tool on the disposable app
`618417eb-7c66-472b-b4fc-3f5797038d31`, invoked via `POST {app}:executeTool`. No conversation.

| Probe | Result | Evidence (raw `response.result`) |
|---|---|---|
| **A. Imports — available** | `json`, `os`, `base64`, `hmac`, `hashlib`, `datetime`, `urllib.request`, `uuid`, `ssl`, `socket`, `http.client`, `pydantic` all import | `A_imports` all `true` |
| **A. Imports — absent** | `google.auth`, `google.auth.transport.requests`, `google.cloud.storage`, `google.cloud.firestore`, `requests`, **and `ces_requests`** all raise `ModuleNotFoundError` | `A_imports` = `"ModuleNotFoundError"` |
| **A′. `ces_requests` is a GLOBAL, not an import** | `import ces_requests` **fails**; `globals()["ces_requests"]` **succeeds**. Same for `requests`, `context`, `tools`, `async_tools`. | `G0_ces_requests_global: true`; globals = `Blob, CallbackContext, Content, ExternalResponse, FunctionCall, FunctionResponse, GenerateContentConfig, HttpMethod, LlmRequest, LlmResponse, Part, StatusError, Tool, async_tools, ces_internal, ces_requests, context, get_variable, importlib, remove_variable, requests, set_variable, tools` |
| **B. Environment** | **12** variables, **none a credential**: `BORG_CONTAINER_RUNTIME, GPG_KEY, LANG, MPLCONFIGDIR, PATH, PYTHONPATH, PYTHON_GET_PIP_SHA256, PYTHON_GET_PIP_URL, PYTHON_PIP_VERSION, PYTHON_VERSION, SANDBOX_INPUT_OUTPUT_DIRECTORY, XBOX_PARALLEL_SOCKET`. `GOOGLE_CLOUD_PROJECT`, `GOOGLE_APPLICATION_CREDENTIALS`, `GCP_PROJECT`, `K_SERVICE`, `GAE_ENV`, `GCE_METADATA_HOST` all **false**. | `B_env.count: 12`, `B_env.cred_markers` all `false`. Identical to the 260812-o5l finding — independently reproduced. |
| **C. ToolContext — session state IS readable AND writable from tool code** | `context` is a global of type `ToolContext` from module `ces_internal`. `context.state` is a **plain `dict`**; a write during the probe came back in the `executeTool` response's `variables` block. Methods present: `get_variable`, `set_variable`, `remove_variable`, `parts()`, `get_last_user_input()`, `get_last_agent_output()`. Fields: `actions, agent_name, events, function_call_id, invocation_id, session_id, state, streaming_stage, user_content, variables`. | `C_context.state_type: "dict"`, `state_writable: true`, `session_id_present: true`, response `variables: {"probe_write":"ok"}` |
| **D. Egress — arbitrary public host** | **OPEN.** `https://example.com/` → **200** (559 bytes), `https://api.github.com/zen` → **200** (26 bytes), `http://example.com/` → **200**. Egress is *not* allowlisted to `api.resend.com`. | `D_egress_public`, `F2_metadata_detail.http_example` |
| **E. Google APIs, unauthenticated** | **REACHABLE — PASS.** `storage.googleapis.com/storage/v1/b?project=…` → **401** *"Anonymous caller does not have storage.buckets.list access"*. `firestore.googleapis.com/v1/projects/…/databases` → **401** *"Request is missing required authentication credential."* Both are credential failures, not connection failures. | `E_google_apis` |
| **F. Metadata server** | **NO TOKEN.** `http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token` → `status_code: 0`, `reason: "Error fetching from URL"`, 0 bytes. Same for `http://169.254.169.254/…`. **Not a scheme block** — plain `http://example.com/` returned 200 in the same invocation. No token value was ever produced, so none was recorded. | `F_metadata`, `F2_metadata_detail` |
| **G. `ces_requests` kwarg surface** | **Accepted:** `headers`, `params`, `data` (and `json`, per signature). **Rejected with `TypeError: Requests.request() got an unexpected keyword argument`:** `timeout`, `allow_redirects`, `verify`. Signatures: `get(url, params=None, **kwargs)`, `post/put(url, data=None, json=None, **kwargs)`, `delete(url, **kwargs)`. | `G_kwargs` — generalises the 05-07 `timeout=20` defect |
| **G′. Verbs** | `get, post, put, patch, delete, head, options, request` **plus** `async_get, async_post, async_put, async_patch, async_delete, async_head, async_options, async_request`. Return type is `ces_internal.ExternalResponse` with `status_code, ok, reason, text, json, raise_for_status`. | `G0_attrs` |
| **H. Async execution** | `ExecutionType` accepts exactly **`SYNCHRONOUS`** and **`ASYNCHRONOUS`** — a `PATCH` with `"ASYNC"` returned `400 Invalid value at 'tool.execution_type'`, with `"ASYNCHRONOUS"` returned **200**. All four Warranty-app tools are `SYNCHRONOUS`. So `record_claim` **can** be made asynchronous, and `ces_requests.async_*` exists as a second lever. | live `PATCH …/tools/probe_sandbox?updateMask=executionType` |

### Traps worth carrying into 06-02

1. **`response.ok` is `True` even when `status_code` is `0`.** The metadata probe returned
   `ok: true, status_code: 0`. **Never branch on `.ok` — branch on `.status_code`.** This would
   have read as a silent success in any store tool that trusted `.ok`.
2. **`import ces_requests` fails.** Use the injected global. Any tool that wraps its HTTP call in a
   bare `try/except` around `import ces_requests` will silently do nothing (the 05-07 shape).
3. **`timeout=` is still rejected**, now confirmed alongside `allow_redirects` and `verify`.
4. **Tool code changes are not immediately live.** Every probe here waited ~30 s after `PATCH`
   before `executeTool` — the 260812-o5l trap, re-honoured.
5. **`apps.executeTool` body shape**: `{"tool": "<FULL resource name>", "args": {...}}`. A bare tool
   id returns `400 "Either tool name or toolset name is required."` The response is
   `{tool, response:{result}, variables}` — `variables` echoes anything the tool wrote to
   `context.state`.

## AUTH_LADDER

Walked in order. Stopped at the first rung producing a working write-then-read round-trip executed
from inside a CES Python tool.

| Rung | What it was | Outcome | Ruled out by / proven by |
|---|---|---|---|
| **1. Google-credentialed direct** (Firestore/GCS with a bearer token minted in the tool) | Requires an OAuth token from inside the sandbox | **RULED OUT** | Probe F: metadata server `status_code: 0`, `reason: "Error fetching from URL"` on both `metadata.google.internal` and `169.254.169.254`, while plain `http://example.com/` returned 200 in the same invocation. Probe A: `google.auth` = `ModuleNotFoundError`. Probe B: zero credential env vars. Three independent negatives. |
| **2. GCS with a shared secret, no Google credential** — HMAC key against the **S3-compatible XML API** (AWS SigV4) | Outbound HTTPS + a secret in tool source, exactly the Resend shape, aimed at a Google store in the customer's own project | **CHOSEN — proven** | Probe D (egress open) + probe E (`storage.googleapis.com` reachable, 401 only) + probe A (`hmac`, `hashlib`, `base64`, `datetime` all importable) made it viable; the round-trip below proved it. |
| **3. A small Cloud Run service fronting the store** | One HTTPS POST with a bearer shared secret; the ROADMAP's named fallback | **NOT NEEDED — not built** | Costed only. Requires enabling `run.googleapis.com`, `artifactregistry.googleapis.com` and `cloudbuild.googleapis.com` (all three confirmed **disabled** on `insurance-agent-demo-500614` at execution time), a container build, and a component somebody has to operate for the life of the demo. The executor's account *does* hold `serviceusage.services.enable` and `run.services.create`, so it is available if rung 2 is rejected — but it buys nothing rung 2 does not already deliver, and costs an operated component. |
| **4. A third-party HTTPS JSON store with a bearer token** | The literal Resend analogue; needs no GCP change at all | **NOT NEEDED — not adopted** | Costed only. Candidate would be a hosted key-value/JSON service. Rejected on the merits *before* the checkpoint even had to weigh it: it puts synthetic claim records on a host outside the customer's project and adds a second vendor to the spec's path-to-production story, in exchange for nothing — rung 2 needs no new GCP service either. |

### Why rung 2 is the strong outcome, not a consolation

Rung 2 needs **no service enablement** (`storage.googleapis.com` was already enabled), **no new
component to operate**, and **no data leaving the project**. It is the only rung that is
simultaneously (a) inside the customer's project, (b) reachable with the one credential shape a CES
tool can actually carry, and (c) already-enabled infrastructure. Its one real cost is a long-lived
HMAC secret embedded in tool source — identical in kind to the Resend key already living in
`resolve_claim`, so it introduces no new class of risk to this demo.

### The round-trip, executed from inside a CES Python tool

Tool `probe_store` on the disposable app, invoked twice via `apps.executeTool` (zero conversations).

**Run 1 — single-object round trip** (`spike/roundtrip1.json`, 99-byte payload):

| Step | Status | Assertion |
|---|---|---|
| `PUT` object | **200** | `wrote: true` |
| `GET` object, signed | **200** | `read_back: true`, **`payload_byte_identical: true`**, `read_len: 99` |
| `GET` object, **no auth header** | **403** | `anon_read_denied: true` — satisfies T-06-04, the store is not publicly readable |
| `DELETE` object | **204** | `deleted: true` |
| `GET` after delete | **404** | object gone |

**Run 2 — the real access pattern** (`spike/rt2/…`, two full claim records):

| Step | Status | Assertion |
|---|---|---|
| `GET by-policy` before any write | **404** | cold start handled — absence is a 404, not an error |
| `PUT by-policy` (1 claim, 594 B) + `PUT by-ref` | **200 / 200** | |
| `GET by-policy`, insert newest-first, `PUT` again (2 claims, 1121 B) | **200 / 200** | read-modify-write append works |
| `GET by-policy` — **one read** | **200** | `claim_count: 2`, `most_recent_ref: "CLM-24902"`, `most_recent_is_newest: true`, `most_recent_status: "HUMAN_REVIEW"` |
| `GET by-ref` — **one read** | **200** | `lookup_by_ref_matches: true` |
| `GET by-ref` for an unknown reference | **404** | `unknown_ref_is_none: true` — criterion 5's "never fabricate" has a hard 404 to branch on |
| `GET ?list-type=2&prefix=…` with a correctly signed canonical query string | **200** | `list_sees_policy_key: true`. (An earlier 403 on this call was a **signing defect** — an unsigned canonical query string — **not** a permission problem. Recorded so nobody re-litigates it. The design does not need LIST.) |
| `DELETE` ×3, then `GET` | **204 ×3, 404** | `deleted: true`; the bucket is empty, verified out-of-band with `gcloud storage ls` |

Total wall time for run 2 — **ten** signed HTTP calls in one tool invocation — was **3.56 s**
end-to-end including CES overhead. A real `record_claim` is 1 GET + 2 PUTs.

## STORE_MECHANISM

**Google Cloud Storage, reached over the S3-compatible XML API, signed with AWS Signature V4 using
a GCS HMAC key held in the tool source.** Everything below is literal — 06-02 writes tool source
from this section without re-deriving anything.

### The concrete resources

| Thing | Value | Status |
|---|---|---|
| Bucket | `gs://meridian-claim-store-500614` | **CREATED** by this plan, `US` multi-region, **uniform bucket-level access**, **public access prevention ENFORCED** |
| Service account | `claim-store-writer@insurance-agent-demo-500614.iam.gserviceaccount.com` | **CREATED** by this plan |
| IAM | `roles/storage.objectAdmin` on the bucket only — no project-level grant | **BOUND** |
| Credential | A GCS **HMAC key** for that service account: a 61-char access id beginning `GOOG1…` and a **40-char** secret | Minted, used, then **deactivated and deleted** at plan close. 06-02 mints a fresh one. |
| GCP service required | `storage.googleapis.com` — **already enabled**, nothing new to turn on | ✔ |

### The HTTP contract

| | |
|---|---|
| **Host** | `storage.googleapis.com` |
| **URL template** | `https://storage.googleapis.com/{BUCKET}/{OBJECT_KEY}` |
| **Verbs** | `PUT` to write, `GET` to read, `DELETE` to remove. All via the injected `ces_requests` global — `ces_requests.put(url, data=body, headers=h)` / `.get(url, headers=h)` / `.delete(url, headers=h)`. |
| **Payload encoding** | UTF-8 JSON string, `json.dumps(obj, sort_keys=True, separators=(",", ":"))`. **`sort_keys=True` is load-bearing** — it is what made `payload_byte_identical` assertable. Do **not** pass `json=` to `ces_requests.put`; pass the pre-serialised string as `data=`, because the SigV4 payload hash must be computed over the exact bytes sent. |
| **Auth header** | `Authorization: AWS4-HMAC-SHA256 Credential={ACCESS_ID}/{yyyymmdd}/us/s3/aws4_request, SignedHeaders=host;x-amz-content-sha256;x-amz-date, Signature={hex}` |
| **Companion headers** | `x-amz-date: {yyyymmddTHHMMSSZ}` (UTC) and `x-amz-content-sha256: {sha256hex of the body, empty-string hash for GET/DELETE}` |
| **SigV4 region / service** | region `us` (the bucket's `US` location, lowercased), service `s3` |
| **Not-found signal** | `GET` on a missing key returns **404** with no body. This is the branch criterion 5 uses. |
| **Timeouts** | There are none to set — `timeout=` is rejected by the shim (probe G). Budget for it structurally instead: the whole 10-call probe took 3.56 s. |

### The signing routine, exactly as proven

```
canonical_uri     = "/" + BUCKET + "/" + OBJECT_KEY
canonical_query   = ""                       # or the literal encoded query string, if one is sent
payload_hash      = sha256(body_utf8).hexdigest()
canonical_headers = "host:storage.googleapis.com\n"
                    "x-amz-content-sha256:" + payload_hash + "\n"
                    "x-amz-date:" + amzdate + "\n"
signed_headers    = "host;x-amz-content-sha256;x-amz-date"
canonical_request = METHOD \n canonical_uri \n canonical_query \n canonical_headers \n
                    signed_headers \n payload_hash
scope             = datestamp + "/us/s3/aws4_request"
string_to_sign    = "AWS4-HMAC-SHA256" \n amzdate \n scope \n sha256hex(canonical_request)
key = HMAC(HMAC(HMAC(HMAC("AWS4"+SECRET, datestamp), "us"), "s3"), "aws4_request")
signature = HMAC_hex(key, string_to_sign)
```

Standard library only — `hmac`, `hashlib`, `datetime`, `json`. **No third-party dependency, no
package install** (operational rule 7 satisfied). ~40 lines, identical in both apps; 06-02 should
copy it verbatim rather than reimplement it per tool.

### Access posture (T-06-04)

- Public access prevention is **enforced** and uniform bucket-level access is **on**.
- An **unauthenticated** `GET` of a written object returned **403** from inside the sandbox
  (`anon_read_denied: true`) — the store is not publicly readable, proven, not assumed.
- The service account can only touch this one bucket (`roles/storage.objectAdmin` bound at the
  bucket, not the project).
- The HMAC secret is a long-lived bearer credential living in tool source, **the same risk class as
  the Resend key already in `resolve_claim`** — no new category of exposure for this demo. Rotation
  is `gcloud storage hmac create/delete`; revocation is instant and does not touch the agents' logic,
  only the constant.

### Where the secret lives, and where it must never live

It is embedded in the `pythonCode` of the store tools on each app, and nowhere else. During this
plan the secret existed only inside a single executor process — minted, substituted into the tool
payload, and POSTed in one uninterrupted `python3` invocation. **It was never written to disk, never
echoed, and appears in no file in this repo.** 06-02 must mint its own key the same way.

## CLAIM_SCHEMA

### Key structure — two objects per claim, both written by `record_claim`

```
claims/by-policy/{POLICY_KEY}.json   ->  { policy_id, updated_at, claims: [ record, ... ] }   # newest FIRST
claims/by-ref/{CLAIM_KEY}.json       ->  record                                               # one claim
```

**Why both, in two sentences.** Criterion 4 makes lookup by policy id load-bearing — a caller must
never be asked to read `CLM-24253` aloud — and a per-policy object holding that policy's claims
newest-first answers *"the most recent claim for this policy"* in **one `GET`**, with no query
engine, no index to keep consistent and no LIST call. Criterion 1 keys the record by claim
reference, so the same record is also written to a claim-keyed object, which answers *"tell me about
CLM-24253"* in one `GET` too; the duplication is deliberate, both writes happen inside the same tool
invocation, and a stale copy is impossible because a claim record is only ever written once per
resolution.

### Key normalisation — required, because voice will mangle these

| Input seen in the wild | Key written |
|---|---|
| `PDP-100294`, `PDP100294`, `pdp 100294` | `PDP100294` — uppercase, strip every non-alphanumeric |
| `CLM-24253`, `CLM 24253`, `clm24253` | `CLM24253` — same rule |

The **display** forms (`PDP-100294`, `CLM-24253`) are carried inside the record as
`policy_id_display` and `claim_ref`, so nothing the customer hears changes. Normalising the key is
what makes a spoken policy id survive speech-to-text.

### The record, field by field

| Field | Type | Source | Why it is here |
|---|---|---|---|
| `schema_version` | int, `1` | constant | lets 06-02..06-04 evolve the record without a migration guess |
| `claim_ref` | string, `"CLM-24253"` | `resolve_claim.claim_ref` | **criterion 1 key**, display form |
| `policy_id` | string, `"PDP100294"` | normalised | **criterion 4 key** |
| `policy_id_display` | string, `"PDP-100294"` | as given | spoken/printed form |
| `customer_name` | string, `"Jordan Rivera"` | session state | read-back on identification |
| `channel` | `"CHAT"` \| `"VOICE"` | which app wrote it | criterion 3, both directions |
| `created_at` | ISO-8601 UTC, `"2026-08-13T02:00:00Z"` | tool clock | ordering; newest-first insert |
| **`status`** | `"APPROVED"` \| `"HUMAN_REVIEW"` | `resolve_claim.decision` | **criterion 1** |
| **`device`** | string, `"Aether Pro 14 laptop"` | `resolve_claim.device` | **criterion 1** |
| **`claim_amount`** | number, `420.50` | `resolve_claim.claim_amount` | **criterion 1** |
| **`excess`** | number, `75.00` | `resolve_claim.deductible` | **criterion 1** — stored under the customer-facing name `excess`, since that is the word both agents speak |
| **`photo_assessed`** | boolean | `photo_status` present and not `"none"` | **criterion 1** |
| **`rules_fired`** | list of strings | `resolve_claim.rules_fired` | **criterion 1** |
| `issue_label` | string, `"Cracked screen"` | `resolve_claim.issue_label` | the sentence the lookup composes |
| `total_loss_flag` | boolean | `resolve_claim.total_loss_flag` | explains a `HUMAN_REVIEW` without re-deriving it |
| `auto_approval_cutoff` | number, `1000.00` | `resolve_claim.auto_approval_cutoff` | criterion 2: the figures spoken back must be the *stored* ones |
| `coverage_limit` | number, `2500.00` | `resolve_claim.coverage_limit` | same |
| `photo_status` | string, `"confirmed"` \| `"contradicted"` \| `"unclear"` \| `"retry"` \| `"none"` | session state | distinguishes *a photo was assessed* from *what it showed* |

### Rules on the shape — not negotiable

1. **No pre-composed prose in the record.** There is deliberately no `status_sentence`,
   `explanation` or `summary` field. The spoken/printed status sentence is composed **at read time**
   by `lookup_claim` in Python, for exactly the reason `resolve_claim.explanation` is composed in
   Python: it is then a tool-computed string the agent relays verbatim, the wording can be corrected
   later without rewriting a single stored record, and the model is never the author of a figure.
2. **Every number is stored as a number**, formatted at read time. Never store `"£420.50"`.
3. **Concurrency: last-write-wins, and that is acceptable for a demo.** The per-policy object is
   read-modify-written with no ETag precondition; two claims filed for the same policy in the same
   second could lose one. The demo files one claim at a time. *(If it ever matters, the XML API
   supports `x-goog-if-generation-match` — noted, not implemented.)*
4. **A missing key is a 404 and must be surfaced as "not found", never as an empty claim.**
   Criterion 5's "never fabricate" is enforced in `lookup_claim`: a non-200 returns a
   `found: false` result and the agent has no figures to read.
5. **Synthetic data only.** Every record originates in the Section 4 namespace — `Jordan Rivera`,
   `PDP-100294` / `PDP100294`, `PDP-100871`, `PDP100583`. No real PII, ever.

### Worked example (the exact object round-tripped in this spike)

```json
{"schema_version":1,"claim_ref":"CLM-24902","policy_id":"PDP100294",
 "policy_id_display":"PDP-100294","customer_name":"Jordan Rivera","channel":"CHAT",
 "created_at":"2026-08-13T02:00:00Z","status":"HUMAN_REVIEW","device":"Aether Pro 14 laptop",
 "issue_label":"Cracked screen","claim_amount":1180.0,"excess":75.0,
 "auto_approval_cutoff":1000.0,"coverage_limit":2500.0,"total_loss_flag":false,
 "photo_assessed":true,"photo_status":"confirmed",
 "rules_fired":["TARIFF_SCREEN_REPLACEMENT","EXCESS_APPLIED","UNDER_AUTO_APPROVAL_CUTOFF"]}
```

594 bytes for a one-claim policy object, 1121 bytes for two. Nowhere near any limit — and note the
record is **never** carried in a session variable, so the 262,144-byte session-variable ceiling
found in 260812-o5l does not apply to it.

## STORE_DECISION

**Status: LOCKED by the user, 2026-08-12.** Rung 2 is the mechanism for the whole of Phase 6.
Rungs 3 (Cloud Run) and 4 (third-party host) are **formally rejected** — Cloud Run was costed but
never built, and nothing in this phase may re-open them. The user's literal words are recorded
verbatim under **User's lock** below.

### The decision, as it must be presented

1. **Can a CES tool get a Google credential?** **No.** The metadata server returns `status_code: 0`,
   `reason: "Error fetching from URL"` on both `metadata.google.internal` and `169.254.169.254`,
   while plain `http://example.com/` returned **200** in the same tool invocation. `google.auth` is
   `ModuleNotFoundError` and there are no credential environment variables.
2. **Is egress open or allowlisted?** **Open.** `https://example.com/` → **200**,
   `https://api.github.com/zen` → **200**. It is not restricted to `api.resend.com`.
3. **Rung chosen:** **Rung 2 — GCS via the S3-compatible XML API, signed with a GCS HMAC key**,
   because it is the only mechanism that is simultaneously inside the customer's own project,
   reachable with the one credential shape a CES tool can carry, and built on a GCP service that was
   already enabled.
4. **The cost, named literally:**
   - **Services that must be enabled: NONE.** `storage.googleapis.com` was already on. Cloud Run,
     Artifact Registry, Cloud Build, Firestore and Secret Manager all stay **disabled**.
   - **A new component to operate: NONE.** No service, no container, no deployment, no uptime.
   - **New resources created by this plan** (all synthetic-demo-scoped, all deletable in one
     command): bucket `gs://meridian-claim-store-500614` (US, uniform access, public access
     prevention **enforced**); service account
     `claim-store-writer@insurance-agent-demo-500614.iam.gserviceaccount.com`;
     `roles/storage.objectAdmin` bound **on that bucket only**, not the project.
   - **A new shared secret: YES.** A GCS HMAC key (61-char `GOOG1…` access id, 40-char secret),
     embedded in the store tools' Python source on each app — **the same risk class as the Resend
     key already living in `resolve_claim`**. The spike's key was **deactivated and deleted** at plan
     close; 06-02 mints a fresh one inside a single process and never writes it to disk.
   - **Does any data leave the project? NO.** Records stay in a private bucket in
     `insurance-agent-demo-500614`. Nothing goes to a third-party host.
5. **The record schema:** `schema_version, claim_ref, policy_id, policy_id_display, customer_name,
   channel, created_at, status, device, claim_amount, excess, photo_assessed, photo_status,
   rules_fired, issue_label, total_loss_flag, auto_approval_cutoff, coverage_limit`. No pre-composed
   prose field — the status sentence is composed at read time in Python.
6. **"The most recent claim for this policy" is answered in ONE `GET`** of
   `claims/by-policy/{POLICY_KEY}.json`, whose `claims` array is newest-first. No LIST, no index, no
   query engine. Keys are normalised (`PDP-100294` → `PDP100294`) so a spoken policy id survives
   speech-to-text — which is what makes criterion 4 reachable without anyone reading `CLM-24253`
   aloud.
7. **Can `record_claim` run asynchronously? YES.** `ExecutionType` accepts `ASYNCHRONOUS`
   (`"ASYNC"` was rejected with a 400, `"ASYNCHRONOUS"` accepted with a 200), and
   `ces_requests.async_post/put/get` exist as a second lever. **So 06-04 may not have a voice
   latency problem at all** — and even synchronously, the whole ten-call probe finished in 3.56 s,
   against a real write of 1 GET + 2 PUTs.

### Rejected rungs — closed, do not re-litigate

**Rung 3 (a Cloud Run service fronting the store) — REJECTED.** It was costed, not built. It remains
technically available (the executor's account holds `serviceusage.services.enable` and
`run.services.create`, verified by `testIamPermissions`), but it is **not proven by execution**,
it would require enabling `run.googleapis.com`, `artifactregistry.googleapis.com` and
`cloudbuild.googleapis.com` — all three currently disabled — and it adds a component somebody has to
operate for the life of the demo, in exchange for nothing rung 2 does not already deliver.

**Rung 4 (a third-party HTTPS JSON store) — REJECTED.** It would put synthetic claim records on a
host outside the customer's project and add a second vendor to the spec's path-to-production story,
again for no gain over rung 2.

### User's lock

**Locked 2026-08-12. The user's literal reply:**

> `as-proven`

The lock accepts, explicitly, all three costs:

1. **A GCS HMAC secret embedded in tool source on both apps** — accepted as the same risk class as
   the Resend key already living in `resolve_claim`.
2. **The two resources created by this plan are ACCEPTED and must NOT be deleted** — bucket
   `gs://meridian-claim-store-500614` (US, uniform bucket-level access, public access prevention
   enforced) and service account
   `claim-store-writer@insurance-agent-demo-500614.iam.gserviceaccount.com` with
   `roles/storage.objectAdmin` **on that bucket only**.
3. **The schema exactly as written above** — two objects per claim
   (`claims/by-policy/{POLICY_KEY}.json` newest-first, `claims/by-ref/{CLAIM_KEY}.json`), the field
   list as specified, normalised keys, and **no pre-composed prose field**; the status sentence is
   composed at read time in Python.

**Rung 2 is the locked mechanism for the whole of Phase 6.** 06-02, 06-03 and 06-04 build on it
without re-deriving anything from this document.

### MANDATORY for every store tool written in 06-02, 06-03 and 06-04

Three platform facts, each proven here by execution. The first two would each let a store tool
**silently report success on a write that never happened** — read them before writing a line of
`record_claim` or `lookup_claim`.

1. **`import ces_requests` FAILS with `ModuleNotFoundError`.** It is an **injected global**, not an
   importable module. Get it with `globals()["ces_requests"]`. The same is true of `requests`,
   `context`, `tools`, `async_tools`, `Part`, `Blob` and `StatusError`. A tool that wraps its HTTP
   call in `try: import ces_requests / except: return {...}` will do **nothing** and report whatever
   its fallback says — the 05-07 swallowed-exception shape, with a store write at stake this time.
2. **`ExternalResponse.ok` is `True` even when `status_code` is `0`.** Observed directly: the
   metadata probe returned `ok: true, status_code: 0, reason: "Error fetching from URL"` on a request
   that never reached a server. **Every store tool MUST assert on `status_code` explicitly** — a
   write is `status_code in (200, 201)`, a read is `status_code == 200`, a miss is
   `status_code == 404` — and **must never branch on `.ok`**.
3. **`ExecutionType` accepts `ASYNCHRONOUS`** (a `PATCH` with `"ASYNC"` returned **400**, with
   `"ASYNCHRONOUS"` returned **200**), and `ces_requests.async_get/post/put/patch/delete` exist as a
   second lever. **This may remove 06-04's voice-latency concern entirely** — evaluate it there
   before designing around a latency problem that may not exist. Note that the synchronous path is
   already fast: ten signed HTTP calls in one invocation took 3.56 s, against a real `record_claim`
   of 1 `GET` + 2 `PUT`s.

### Plan-close assertions

| Assertion | Result |
|---|---|
| Disposable SPIKE app deleted | `DELETE` → **200** (LRO), then `GET .../apps/618417eb-…` → **404** |
| App list restored | **5 apps**, exactly the five that existed at plan start |
| Chat deployment `d7bfbb93` unchanged | serves version **`cdca14e3`**, `updateTime 2026-08-13T01:35:50.254056Z` — **byte-identical to the reading taken at plan start**, re-read not assumed |
| Voice deployment `d28bbcb0` unchanged | serves version **`5d9df25c`**, `updateTime 2026-08-12T20:51:46.574949Z` — **identical to plan start** |
| Fork `9ae7a0c3` | **never called by any method**, GET or otherwise |
| Non-GET calls to `6e01e4a5`, `a2f621e4` or `9ae7a0c3` | **ZERO.** Every non-GET in this plan targeted `618417eb` (the disposable app) or GCS. The client-side URL gate refused nothing, because nothing was ever attempted. |
| `spike/` objects in the store | **all deleted** — the tool deleted them (204 ×3, then 404), and `gcloud storage ls -r gs://meridian-claim-store-500614/**` reports **no objects** |
| Spike HMAC key | **deactivated then deleted**; `gcloud storage hmac list` returns **empty** |
| Secret-leak gate | `grep -rE 're_[A-Za-z0-9_]{20,}\|ya29\.\|AKIA\|GOOG1E'` over this findings file returns **0 matches**; the only repo-wide hits are the plan files quoting the gate's own regex |
| CES conversations spent | **ZERO.** `apps.executeTool` carried every probe. |
