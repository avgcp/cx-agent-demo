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
| `uninsured_device` | Recently purchased smartphone (purchased 2026-06, NOT currently on any policy) — seeded cross-sell/personalization hook (per D-08) |

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

### Negative / Over-Fire Records

Records that must **NOT** trigger a branch — seeded specifically to catch over-firing during rehearsal and live delivery (per Pitfall 4 / the "Looks Done But Isn't" checklist).

**Escalation over-fire check — near-threshold claim that must still auto-approve**

| Field | Value |
|---|---|
| `claim_id` | CLM-24003 |
| `loss_type` | Near-threshold cracked-screen claim |
| `claim_amount` | $950 |
| `total_loss_flag` | false |
| `data_loss_flag` | false |
| `coverage_valid` | true |
| `expected_routing` | **AUTO-APPROVE.** Must NOT escalate, despite being close to the $1,000 ceiling. This record exists solely to catch a decision-logic bug that escalates on proximity to the threshold rather than the literal `amount > $1,000` rule. |

**Cross-sell over-fire check — fully insured claimant, no cross-sell hook**

| Field | Value |
|---|---|
| `claimant_name` | Sam Okafor |
| `policy_id` | PDP-100871 |
| `coverage_valid` | true (fully insured — no coverage gaps) |
| `uninsured_device` | none |
| `expected_behavior` | Cross-sell/upsell offer must **NOT** fire for this claimant. This record exists solely to catch a decision-logic bug that offers a device bundle regardless of whether an actual coverage gap exists. |

### Pre-Validated Damage-Image Library

A small library of named seed images per damage scenario, each with a pre-validated expected damage-analysis output. Per Pitfall 5, **ad-hoc or live-captured photos are forbidden in the standard demo path** — only these named seed images may be used. The implementation team must attach the actual image files to these references and re-validate the expected output against the live model before the demo is delivered; these are placeholder references, not attached files, as of this spec.

| Image ref | Scenario | Expected damage-analysis output |
|---|---|---|
| IMG-CRACK-01 | Cracked/cosmetic laptop screen | Moderate cosmetic screen damage, repairable, LOW severity, within auto-approve range |
| IMG-CRACK-02 | Cracked/cosmetic laptop screen (variant) | Moderate cosmetic screen damage, repairable, LOW severity, within auto-approve range |
| IMG-CRACK-03 | Cracked/cosmetic laptop screen (variant) | Moderate cosmetic screen damage, repairable, LOW severity, within auto-approve range |
| IMG-LIQUID-01 | Liquid-damaged/destroyed high-end laptop | Severe internal liquid damage, likely total loss, HIGH severity, route to human |
| IMG-LIQUID-02 | Liquid-damaged/destroyed high-end laptop (variant) | Severe internal liquid damage, likely total loss, HIGH severity, route to human |

`CLM-24001` uses `IMG-CRACK-01`; `CLM-24002` uses `IMG-LIQUID-01`. The remaining images (`IMG-CRACK-02`, `IMG-CRACK-03`, `IMG-LIQUID-02`) are held in reserve as swap-in recovery images per the Demo Runbook / Fallback Plan (Pitfall 5 recovery strategy) if a live photo-analysis result looks off during rehearsal or delivery.

### Multilingual Test Phrases

English ↔ Spanish (US) trigger phrases, given verbatim (per D-10; exact phrasing required per Pitfall 10 to avoid ASR-ambiguous language-switch triggers).

| Phrase role | Verbatim text |
|---|---|
| English FNOL open | "Hi, I dropped my laptop and the screen is cracked." |
| Language-switch trigger (customer) | "¿Podemos continuar en español?" |
| Spanish FNOL phrase | "Se me cayó la laptop y la pantalla está rota." |

**Note:** English (US) and Spanish (US) are both **GA voice variants**. Any language outside the confirmed audio-to-audio core set uses the **captioned-text fallback** (per D-11) — do not assume live audio-to-audio parity for any language pair not explicitly confirmed in-console.

### Traceability Field Registry

Every on-stage dollar figure and coverage term maps to exactly one canonical named field and owning record below. **Rule: nothing is free-generated on stage.** If a figure is not in this registry, it must not be spoken by the agent or the presenter.

| Value | Field | Owning record |
|---|---|---|
| $100 | `deductible` | Policy (PDP-100294) |
| $3,500 | `coverage_limit` | Policy (PDP-100294) |
| $450 | `claim_amount` | CLM-24001 (small/auto-approve claim) |
| $2,400 | `claim_amount` | CLM-24002 (large/HITL claim) |
| $950 | `claim_amount` | CLM-24003 (escalation over-fire negative record) |
| true | `total_loss_flag` | CLM-24002 |
| false | `total_loss_flag` | CLM-24001, CLM-24003 |
| true | `data_loss_flag` | CLM-24002 |
| false | `data_loss_flag` | CLM-24001, CLM-24003 |
| $1,000 | Decision threshold (auto-approve ceiling) | Decision Logic (Section 3) — not a claim field, the literal configurable rule value |
| recently purchased smartphone (uninsured) | `uninsured_device` | Jordan Rivera / PDP-100294 (cross-sell hook) |
| none | `uninsured_device` | Sam Okafor / PDP-100871 (cross-sell over-fire negative record) |
