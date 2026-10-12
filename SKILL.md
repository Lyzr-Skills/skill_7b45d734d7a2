---
name: bucket-sla-support
description: Fact bucket for SERVICE COMMITMENTS - whether the standard Anaplan Availability and Support SLA applies or custom terms do, the uptime commitment and measurement period, service-credit tiers (trigger, % and basis), credit window, claim window and cap, sole-remedy wording, and support severity response / resolution times. Feeds the SLA variable-consideration check (high SLA liability).
metadata:
  bucket_id: sla_support
  schema_version: 2.0.0
  output_model: sla_support_models.py (SlaSupportOutput)
  brd_refs: Sec 4.1.1 step 3, 4.2.1, 5.1, 5.2.1
  tracker_rows: 50
---

# Bucket: SLA & Support

This skill is self-contained: Part A is how to extract (shared by every bucket), Part B is what
this bucket extracts, Part C is the exact output structure.

# Part A - How to extract

You extract facts for **one bucket** (SLA & Support) from **one document**. Part B says which facts,
Part C gives the exact output structure, and the output is validated by `sla_support_models.py`
(`SlaSupportOutput`). Your output feeds three kinds of downstream checks:

| Output section | Downstream consumer | Your job |
|---|---|---|
| `values` | Fact rules in code (exact match, tolerance, date difference) | Transcribe the single value exactly + light parse |
| `records` | Calculation rules in code (sums, joins, period comparisons) | Transcribe every row of a repeating structure, completely, with stable keys |
| `evidence` | Contextual agent (policy overrides, deviations, judgements) | Capture verbatim clauses and signals; give non-binding hints only |

**Golden rule: you transcribe; code and the contextual agent decide.**

## A1. Input
- An OCR reconstruction of the PDF in **markdown**. Pages are separated by page markers
  `<<<PAGE n>>>`, where `n` is the 1-based physical page of the PDF.
- Markdown formatting in the input (`#` headings, `**bold**`, `_italic_`, `|` table cells, `-` bullets,
  `>` quotes, backticks, `<br>`, `\` escapes) is layout produced by the OCR step, not document text.

Use nothing else: not Salesforce, not Anaplan, not other documents, not general knowledge about the
customer. If a fact is not in this document, it is `not_found`.

## A2. Applicability
Read Part B section 1 and the document.
- Bucket does not apply -> `applicable: false`, a one-sentence `applicability_reason`, and
  `values`, `records`, `evidence` set to `null`. `observations` and `notes` are `[]`.
- Bucket applies -> `applicable: true` and fill **every** field Part B defines.

"Applies" means the document is the kind that could contain these facts, not that it does.
Silence in an applicable document is evidence (no opt-out clause, no budget cap), so it is reported
as `not_found`, never skipped.

## A3. Citations
Every `found`, `inferred` or `unreadable` fact carries `sources`, a list of
`{"page_number": int, "quote": [string, ...]}`. The quotes are used to **highlight the fact in the
original PDF**, so they must match the OCR text exactly.

**Rules for each string in `quote`:**
1. It is a verbatim span from **one single line** of the page. Never join two lines in one string.
2. It contains **plain text only**: no markdown syntax (`#`, `**`, `__`, backticks, `>`, leading
   `-` / `*` / `+` bullets, `<br>`, HTML tags, `\` escapes) and **never a `|` character**. Drop the
   formatting characters and keep the words between them: `**Order Schedule Number:** ACM-1`
   is quoted as `Order Schedule Number: ACM-1`.
3. Copy the characters exactly as the OCR text has them, including OCR typos, casing, punctuation,
   currency symbols and number formatting. No ellipses, no paraphrase, no translation.
4. Make it long enough to identify the fact unambiguously on the page (label + value, or the
   sentence that states it), but no longer than the line.

**When a fact spans several lines or cells, use one string per line or cell, in reading order:**
- Address block on 5 lines -> 5 strings.
- Label on one line, value on the next -> 2 strings.
- Table row -> one string per relevant cell (`| Solution Architect | 200 |` ->
  `["Solution Architect", "200"]`).
- A printed value that itself contains `|` (e.g. DocuSign `10-Apr-26 | 5:35:23 PM EDT`) -> split at
  the pipe: `["10-Apr-26", "5:35:23 PM EDT"]`. The `raw` field still keeps the full printed value.
- A fact on two pages -> two source objects, one per page.

**By status:**
- `inferred`: quote the text you inferred from. Never invent text for something that is absent.
- `unreadable`: quote the partially legible text as the OCR gives it; if nothing is legible, use
  `["[illegible]"]` with the page number.
- `not_found` / `not_applicable`: `sources` is `[]`.

## A4. Status and confidence
| Status | Use when | raw / parsed |
|---|---|---|
| `found` | Stated in the document | raw = as printed (formatting removed); parsed = light parse |
| `inferred` | Not stated directly but follows from specific text | raw = the text relied on; parsed = your reading; confidence at most `medium` |
| `not_found` | Bucket applies, document is silent | both null, no sources, no candidates |
| `not_applicable` | This field cannot exist in this document type | both null, no sources, no candidates |
| `unreadable` | Text is there but illegible | raw = what is legible; parsed = null |

`confidence`: `high` = explicit and unambiguous; `medium` = explicit but wording or formatting
leaves some doubt, or inferred; `low` = conflicting or barely legible.

## A5. Light parsing
`raw` is the truth (as printed, markdown removed); `parsed` is a convenience that code re-checks.
Put `parsed: null` rather than guessing.

| Type in Part C | parsed format | Rules |
|---|---|---|
| Text | string | Trim whitespace and line breaks; do not rewrite. |
| Number | number | Remove thousands separators; respect decimal commas (`1.234,56` -> 1234.56). Percentages in percent units (`99.5%` -> 99.5). |
| Boolean | true / false | Only from explicit wording; otherwise null. |
| Date | `YYYY-MM-DD` | Only if unambiguous. `03/04/2026` with no locale evidence -> null + `normalization` note. |
| Time | `HH:MM:SS` (24 h) | Seconds `00` if not printed. |
| Timestamp | `YYYY-MM-DDTHH:MM:SS` | Local time as printed; **no conversion, no offset**. |
| UtcOffset | `+HH:MM` / `-HH:MM` | Only when printed as an offset or an unambiguous zone listed in Part B. |
| Money | `{"amount": number, "currency_iso": "USD"}` | ISO 4217. `$` alone is ambiguous -> `currency_iso: null` unless the document names the currency. |
| CountryCode | ISO 3166-1 alpha-2 (`US`, `GB`) | |
| SubdivisionCode | ISO 3166-2 (`US-IL`, `CA-ON`) | |
| LanguageCode | ISO 639-1 (`en`, `fr`) | |
| CurrencyCode | ISO 4217 (`USD`, `EUR`) | Only when the document names the currency or uses an unambiguous symbol (`€`, `£`). |
| enum (`"a" \| "b"`) | one listed value | If nothing fits, use `other` where listed and keep the wording in raw. Never force a fit. |

**Never compute.** No totals, sums, differences, term lengths, execution dates, discounts, FX or
time-zone conversions. If the document prints a total, transcribe it. Code does the arithmetic.

## A6. Conflicts -> candidates
If the document states the same fact differently in two places, put the reading you judge primary
in the value, every other reading in `candidates` (each with its own sources), set
`confidence: low`, and add a `conflict` note. Do not reconcile.

## A7. Records (repeating structures)
- One item per row / entry **as printed**. Do not merge, split, de-duplicate or re-order.
- `key`: follow the key rule in Part B. Keys must be unique within the collection.
- Leaves are `{"raw", "parsed"}`; `status` and `sources` sit on the item (quote the row's line or cells).
- A column that is blank for a row -> `{"raw": null, "parsed": null}`.
- `completeness`: `items_extracted` = number of items you output; `items_declared` = a count the
  document itself prints, else null; `continues_beyond_extraction` = true if the table visibly
  continues somewhere not in this document; `status` = `complete` / `partial` / `unknown`.
- `declared_totals`: every total / subtotal printed for the collection, transcribed.
- Empty collection: `items: []`, `items_extracted: 0`, status `complete`.

## A8. Evidence (for the contextual agent)
**Clauses**: capture each clause matching one of Part B's clause categories.
- `clause_id`: `C1`, `C2`, … in document order.
- `text_verbatim`: the clause text with markdown formatting removed and line breaks kept.
- Length: full text for categories marked ★ (full-verbatim). Otherwise up to 1,500 characters; if
  cut, set `text_truncated: true` and stop at a sentence end.
- `sources`: one string per line of the clause (and one source object per page).
- `overrides_hint`: fill it when the clause says it modifies, replaces, supplements or conflicts with
  another document or clause; otherwise null.

**Signals**: short factual pointers explaining why something may matter.
- `signal_id`: `S1`, `S2`, … Link `clause_id` when the signal comes from a captured clause.
- Record weak signals too. `description` states what is printed, not a conclusion.

**Hints**: one entry for every hint Part B defines, every time the bucket applies.
- `likely` must cite at least one `signal_id`; `unlikely` / `unclear` may have an empty list.
- Hints are leanings, not verdicts.

## A9. Observations and notes
- `observations`: useful facts this bucket does not define but that matter to contract review
  (cited). Do not repeat facts already in a field.
- `notes`: `conflict`, `normalization`, `missing` (expected but absent), `multi_document`,
  `truncation`, `other`, with `field_path` when about a specific field.

## A10. Fact paths
Downstream checks, agent findings and human overrides address facts by path:
`sla_support.values.<field>`, `sla_support.records.<collection>[<key>].<leaf>`,
`sla_support.evidence.clauses[<clause_id>]`, `sla_support.evidence.signals[<signal_id>]`.

## A11. What the validator checks
`sla_support_models.py` rejects the output if any of these fail; it is retried once with the errors,
then routed to a human.
1. Structure, field names, types and enum values match Part C exactly (no extra fields).
2. Every quote string is plain text (no `|`, no markdown syntax, no line break) and is found on a
   single line or table cell of its page.
3. Status rules: `found` / `inferred` / `unreadable` have sources; `not_found` / `not_applicable` have
   none and null raw / parsed; `inferred` is never `high` confidence.
4. `values` / `records` / `evidence` are null exactly when `applicable` is false.
5. `items_extracted` equals the number of items; record keys are unique; every item has a source.
6. Clause and signal IDs are unique and every reference to them exists.
7. Every hint appears exactly once; `likely` hints cite a signal; ★ clauses are not truncated.

# Part B - SLA & Support

**Core rule: transcribe the commitments; code compares them with the standard.** The standard
Anaplan Availability and Support SLA (BRD 5.1) is: 99.5% availability per calendar month; no credit
for the first failing month; 5% of monthly user and workspace fees for the second failing month and
10% for the third or later within a rolling 6-month period; claims within 14 days; support response
2 / 4 / 16 / 24 hours for Severity 1 / 2 / 3 / 4. The model never decides whether a term is "above
standard": code does.

## 1. Applicability
**Applies** to any document that sets, references or changes service levels or support
commitments: SLA exhibit, SSA / customer-paper master agreement, OS or amendment that mentions the
SLA, SOW with support commitments.

**Does not apply** to documents with no SLA or support content (PO, Deal Desk email, DPA).

A document that only incorporates the standard SLA by reference ("the Anaplan Availability and
Support SLA … is incorporated by reference") still applies: set `sla_reference_type` to
`anaplan_standard_by_reference` and leave the numbers `not_found`. Code then uses the standard.

## 2. Ownership boundaries
| Fact | Owner bucket | Note |
|---|---|---|
| The **right to terminate** for downtime / SLA failure | Term & Termination (trigger `sla_failure`) | here only a `termination_for_downtime` signal and the clause |
| Refunds or credits not tied to availability or support | Billing, Payment & Tax | |
| SLA wording inside a Special Terms section | Special Terms (verbatim text) | the numbers are still extracted here |

## 3. values (fact rules)
| Field | Type | Extraction notes |
|---|---|---|
| `sla_reference_type` | enum `anaplan_standard_by_reference`, `standard_with_modifications`, `custom_terms_in_document`, `no_sla` | `standard_with_modifications` when the standard SLA is referenced and then changed ("the SLA applies, except that…"). |
| `uptime_commitment_pct` | Number | 99.9% -> 99.9. Several figures (e.g. per service) -> primary here, others in `candidates`. |
| `measurement_period` | enum `monthly`, `quarterly`, `annual`, `other` | |
| `credit_window_months` | Number | "rolling six-month period" -> 6. |
| `claim_window_days` | Number | "within 14 days" -> 14; months are not converted (raw kept, parsed null). |
| `credit_cap_pct` | Number | Maximum total credit, if printed. |
| `credits_sole_remedy` | Boolean | true if credits are the sole and exclusive remedy; false if other remedies are expressly preserved. |

## 4. records (calculation rules)

### credit_tiers: one item per service-credit tier
| Leaf | Type | Notes |
|---|---|---|
| `trigger_raw` | Text | As printed: "second month in a rolling six-month period", "availability below 99.0%". |
| `failure_month_number` | Number | For month-count tiers: which failing month triggers it (first = 1, second = 2, "third or more" = 3). |
| `uptime_below_pct` | Number | For threshold tiers: the availability level below which the tier applies. |
| `credit_pct` | Number | Credit in percent units. |
| `credit_basis_raw` | Text | What the % applies to: "monthly user and workspace fees for the applicable month". |

Key `T1…` in printed order. Record a "no credit" tier (e.g. first failing month) too, with `credit_pct` 0.

### support_levels: one item per severity level
Key `sev<n>` when a level number is printed (`sev1`), else the label normalised (`critical`).
Leaves: `severity_label`, `severity_level`, `response_time_hours`, `resolution_time_hours`
(minutes / days are not converted: raw kept, parsed null), `coverage_raw` ("24x7", "business hours").

## 5. evidence (contextual agent)
**Clause categories** (★ = full-verbatim): `sla_reference`, `uptime_commitment` ★,
`service_credits` ★, `sla_termination` ★, `sole_remedy` ★, `support_terms`, `exclusions`, `other_sla`.

**Signal types**:
| Type | Record when |
|---|---|
| `custom_sla_terms` | SLA terms are written into this document rather than only referenced |
| `uptime_stated` | An uptime figure is printed (state it) |
| `credit_on_first_failure` | A credit applies from the first failing month / first breach |
| `credit_tier_stated` | A credit % is printed (state it) |
| `termination_for_downtime` | Customer may terminate for availability or support failures |
| `not_sole_remedy` | Credits are not the exclusive remedy, or damages are preserved |
| `penalty_language` | Liquidated damages, penalties or fee reductions beyond credits |
| `support_commitment_custom` | Support times or coverage differ from the printed standard table, or are customer-specific |
| `other` | Anything else about service levels a reviewer should see |

**Hint** (exactly once):
| Hint | Lean on… | Feeds |
|---|---|---|
| `high_sla_liability` | SLA terms expose Anaplan to more credits, refunds or termination than the standard | #50, Rule 5.2.1 |

## 6. Code-side derivations (never done by the model)
- Effective SLA: standard values where the document is silent and `sla_reference_type` is
  `anaplan_standard_by_reference` or `standard_with_modifications`.
- Deviations: uptime > 99.5; any credit at `failure_month_number` 1; credit % above 5 (2nd month) or
  10 (3rd+); credit window shorter than 6 months; claim window longer than 14 days; not sole remedy;
  any `sla_failure` termination right (Term & Termination).
- Support times shorter than the standard -> INFO (not a rev-rec trigger on its own).

## 7. Validations fed (rule registry)
| Rule | Type | Check | Fact paths |
|---|---|---|---|
| #50 SLA Credit / Termination (Rule 5.2.1) | calc | Any deviation in section 6 that increases credits or adds termination for downtime -> "Variable Consideration Risk - High SLA Liability" = TRUE; standard SLA -> FALSE | `values.*`, `records.credit_tiers`, Term & Termination `termination_rights[trigger=sla_failure]` |
| Rule 5.2.1 HITL | context | TRUE -> route to Revenue Leadership and Legal for possible revenue deferral | hint `high_sla_liability`, `evidence.clauses` |
| #28 SSA review (Rule 4.1.1 step 3) | context | SLA exhibit amendments listed in the SSA review | `values.sla_reference_type`, signals |
| Rule 1.3.1 Dual review | fact | High-liability SLA counts as high-risk -> senior reviewer | #50 result |

# Part C - Output structure

Return **one JSON object** exactly in this shape. Every key shown is required (use `null` /
`[]` as the rules allow); no other keys are allowed.

## C1. Shared shapes

```jsonc
Source        = { "page_number": int >= 1, "quote": [string, ...] }        // >= 1 plain single-line string
Value<T>      = { "raw": string | null, "parsed": T | null,
                  "status": "found" | "not_found" | "not_applicable" | "inferred" | "unreadable",
                  "confidence": "high" | "medium" | "low",
                  "sources": [Source], "candidates": [Candidate] }
Candidate     = { "raw": string, "parsed": string | number | boolean | Money | null, "sources": [Source] }
Leaf<T>       = { "raw": string | null, "parsed": T | null }
Money         = { "amount": number | null, "currency_iso": string | null }   // ISO 4217, 3 letters
Item          = { "key": string, "status": "found" | "inferred" | "unreadable", "sources": [Source] (>= 1), ...leaves }
Collection<I> = { "completeness": Completeness, "declared_totals": [DeclaredTotal], "items": [I] }
Completeness  = { "items_extracted": int, "items_declared": int | null,
                  "continues_beyond_extraction": boolean, "status": "complete" | "partial" | "unknown" }
DeclaredTotal = { "label_raw": string, "value": Leaf<Money>, "sources": [Source] }
Clause        = { "clause_id": string, "clause_number_raw": string | null, "heading_raw": string | null,
                  "text_verbatim": string, "text_truncated": boolean, "category_hints": [category],
                  "overrides_hint": OverridesHint | null, "sources": [Source] }
OverridesHint = { "target_document_raw": string | null, "target_clause_raw": string | null,
                  "relationship": "modifies" | "replaces" | "supplements" | "conflicts" | "unclear" }
Signal        = { "signal_id": string, "signal_type": signal_type, "description": string, "entity_raw": string | null,
                  "strength": "strong" | "moderate" | "weak", "clause_id": string | null, "sources": [Source] }
Hint          = { "hint": hint_name, "value": "likely" | "unlikely" | "unclear", "signal_ids": [string] }
Observation   = { "label": string, "value_raw": string, "sources": [Source] }
Note          = { "type": "conflict" | "normalization" | "missing" | "multi_document" | "truncation" | "other",
                  "field_path": string | null, "message": string }

Scalar types: Text = string, Number = number, Boolean = boolean, Date = "YYYY-MM-DD",
Time = "HH:MM:SS", Timestamp = "YYYY-MM-DDTHH:MM:SS", UtcOffset = "+HH:MM" | "-HH:MM",
CountryCode = "US", SubdivisionCode = "CA-ON", LanguageCode = "en", CurrencyCode = "USD", Money as above.
```

## C2. `sla_support` output

```jsonc
{
  "bucket": "sla_support",
  "schema_version": "2.0.0",
  "applicable": true,                // false -> values, records, evidence are null
  "applicability_reason": string,

  "values": {
    "sla_reference_type": Value<"anaplan_standard_by_reference" | "standard_with_modifications" | "custom_terms_in_document" | "no_sla">,
    "uptime_commitment_pct": Value<Number>,
    "measurement_period": Value<"monthly" | "quarterly" | "annual" | "other">,
    "credit_window_months": Value<Number>,
    "claim_window_days": Value<Number>,
    "credit_cap_pct": Value<Number>,
    "credits_sole_remedy": Value<Boolean>
  },

  "records": {
    "credit_tiers": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "trigger_raw": Leaf<Text>,
      "failure_month_number": Leaf<Number>,
      "uptime_below_pct": Leaf<Number>,
      "credit_pct": Leaf<Number>,
      "credit_basis_raw": Leaf<Text>
    }>,
    "support_levels": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "severity_label": Leaf<Text>,
      "severity_level": Leaf<Number>,
      "response_time_hours": Leaf<Number>,
      "resolution_time_hours": Leaf<Number>,
      "coverage_raw": Leaf<Text>
    }>
  },

  "evidence": {
    "clauses": [Clause],
    "signals": [Signal],
    "hints": [Hint]
  },

  "observations": [Observation],
  "notes": [Note]
}
```

Allowed lists for this bucket:

- `category` (clause `category_hints`): `sla_reference`, `uptime_commitment` ★, `service_credits` ★, `sla_termination` ★, `sole_remedy` ★, `support_terms`, `exclusions`, `other_sla`
  (★ = full-verbatim, never truncated)
- `signal_type`: `custom_sla_terms`, `uptime_stated`, `credit_on_first_failure`, `credit_tier_stated`, `termination_for_downtime`, `not_sole_remedy`, `penalty_language`, `support_commitment_custom`, `other`
- `hint_name` (each exactly once): `high_sla_liability`
