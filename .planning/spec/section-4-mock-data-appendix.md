## Mock Data Appendix

**Dataset version:** v1.0

This appendix is the **single source of truth** for every dollar figure, coverage term, claim record, damage image, and multilingual phrase used anywhere else in the CX Agent V2 Demo specification. Sections 1 (Demo Narrative) and 3 (Decision Logic) reference the named record IDs and field names below — they do not invent, restate, or vary any value. If a number is not in this file, it must not be spoken on stage.

### Synthetic-Data Standard

All names, policy numbers, phone numbers, emails, and addresses in this dataset are **synthetic and drawn from a fictitious namespace**. Specifically:

- No valid-format Social Security Numbers appear anywhere in this dataset or in any demo artifact.
- Phone numbers use the reserved fictitious range **555-01xx** (e.g., `555-0142`) — never a real-format 10-digit number.
- Email addresses use `.test` or `example-*` domains (e.g., `jordan.rivera@example-mail.test`) — never a real or realistic-looking domain.
- The carrier name (`Meridian Device Protection`) is an invented, white-label placeholder, not a real insurer.
- **Presenters must NEVER improvise real personal details live.** Every name, number, and claim value spoken during a demo run comes from this canonical dataset — no typing a real prospect's name, a presenter's own phone number, or "just making something up" to personalize a run on the fly. This applies to rehearsals and live delivery alike.

### Carrier

| Field | Value |
|---|---|
| `carrier_name` | Meridian Device Protection |
| `carrier_type` | Generic personal-lines / gadget & device insurer |
| `branding_note` | White-label placeholder — swappable per account; not a real brand. Represents the pattern for any personal-lines/device/contents insurer generally. |

### Primary Claimant Profile

Drives the demo run, the cross-sell moment, and the personalization "wow" (seeded uninsured device makes the offer feel earned rather than generic).

| Field | Value |
|---|---|
| `claimant_name` | Jordan Rivera |
| `policy_id` | PDP-100294 |
| `email` | jordan.rivera@example-mail.test |
| `phone` | 555-0142 |
| `tenure` | Policyholder since 2023 |
| `uninsured_device` | Recently purchased smartphone (purchased 2026-06, NOT currently on any policy) — seeded cross-sell/personalization hook |

### Policy & Coverage

Insurance framing (per D-02) — deductible, coverage limit, and validity are the canonical coverage vocabulary for this policy.

| Field | Value |
|---|---|
| `coverage_type` | Personal device / gadget insurance |
| `deductible` | $100 |
| `coverage_limit` | $3,500 per device |
| `coverage_valid` | true |

### Anchor Laptop Claims (Run Back-to-Back)

Two claims under the same policy/claimant, engineered to land on opposite sides of the literal **$1,000** auto-approve/escalate threshold (per D-04). Run in sequence in the same demo so the contrast between the two outcomes is the decision-branch wow moment.

**Small claim — auto-approve side**

| Field | Value |
|---|---|
| `claim_id` | CLM-24001 |
| `loss_type` | Accidental cracked screen / cosmetic laptop damage |
| `claim_amount` | $450 |
| `total_loss_flag` | false |
| `data_loss_flag` | false |
| `coverage_valid` | true |
| `damage_image_ref` | IMG-CRACK-01 |
| `expected_routing` | Auto-approve (amount ≤ $1,000, no high-risk flag) |

**Large claim — human-in-the-loop (HITL) side**

| Field | Value |
|---|---|
| `claim_id` | CLM-24002 |
| `loss_type` | Liquid-damaged high-end laptop (destroyed) |
| `claim_amount` | $2,400 |
| `total_loss_flag` | true |
| `data_loss_flag` | true |
| `coverage_valid` | true |
| `damage_image_ref` | IMG-LIQUID-01 |
| `expected_routing` | Route to human assessor (amount > $1,000 AND total_loss_flag = true — always escalates per D-05, regardless of amount) |

$450 (CLM-24001) sits below the $1,000 ceiling; $2,400 (CLM-24002) sits above it — the two anchor claims straddle both sides of the threshold as required.

**These field names — `claimant_name`, `policy_id`, `email`, `phone`, `uninsured_device`, `coverage_type`, `deductible`, `coverage_limit`, `coverage_valid`, `claim_id`, `loss_type`, `claim_amount`, `total_loss_flag`, `data_loss_flag`, `damage_image_ref` — are the canonical tokens that Sections 1 (Demo Narrative) and 3 (Decision Logic) must reference by name. No section may restate or invent a different value for any of these fields.**
