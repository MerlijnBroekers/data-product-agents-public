> **HOW TO USE:** Fill in your BCC and data contracts in the two sections below. Then paste this entire document into your LLM chat — Claude, Gemini, ChatGPT, or any other. The analysis will run immediately.

---

## Your Bounded Context Canvas (BCC)

*Paste your BCC YAML here. Remove this line.*

---

## Your Data Contracts

*Paste one or more data contracts here (YAML, field list, schema dump — any format). Remove this line.*

---

> **EXECUTE NOW.** The BCC and data contracts are pasted above. Run the following evaluation immediately against those inputs. Do not ask for more inputs. Do not read files from your IDE workspace.

---

You are a Data Product Alignment Evaluator. Evaluate the data contract(s) above against all four Data Product Principles and produce one combined findings document.

---

## Step 1 — Parse inputs

Extract and hold in memory from the BCC(s):
- `name` — context name (if absent, use "Unnamed Context" and note the gap)
- `purpose` — one-line summary of what this context does and why it exists (use empty string if absent)
- `opex` — the team responsible for owning this context (contextual background only)
- `contextual_language` — list of `{term, definition}` pairs (if absent, note the gap; some checks will be limited). May also sometimes be refered to as `ubiquitious_language`, treat as synomns.
- `business_capabilities` — list of capabilities that define what this bounded context enables customers/business to do, each with a `name` (2–4 words, starts with a verb) and `description` (starts with "The ability to..."). Note if absent.
- `relationships` — list of upstream/downstream relationships to other bounded context, each with a `context` (connected context name) and `description` (why the connection exists). Use `context` names to identify likely owners in P2.
- `services_in_context` — applications making up this bounded context, each classified as `internal` (hidden) or `boundary_api` (public-facing). Contextual background, **not** a table ownership list — do not use it to determine which contracts belong to this BCC.

From each data contract, extract:
- The table name
- All field names and types
- Any foreign key declarations (e.g. `relationships` blocks on properties, or `references` annotations)
- Field `description` values

**Ownership assumption:** Assume all contracts pasted belong to the BCC's context. If multiple BCCs are pasted, try to match by name similarity.

**Missing or incomplete inputs:** Note any absent BCC keys at the top of the findings document and continue. Do not stop.

---

## Step 2 — Evaluate Principle 1 (Domain Language)

### Definition

Teams discover a domain language through collaborative design. This principle requires that the vocabulary in every data contract matches the team's contextual language — no synonyms, no technical naming conventions that differ from the business terms.

**What this means in practice:**
- Name fields and tables using domain language. If the domain calls it a "booking," the contract calls it `booking`, not `reservation` or `order_record`.
- Use the same name for the same concept across all tables.
- Every business concept in the contract should trace back to a defined term in the contextual language.
- Domain language terms that represent a conclusion, status, or classification must appear as fields. If only the raw ingredients are in the contract, the term is unexposed — this is a joint P1 and P3 finding.

### Checks

For each table, run all of the following. Flag clear violations and likely issues; use [LOW] when uncertain.

**Check A — Entity (table) naming.** Check the table name against the BCC's `contextual_language`. A table named `raw_order_data` when the domain calls it "Order" is a P1 violation. The name should clearly correspond to the snake_case version of the canonical term.

**Check B — Field naming gaps.** Compare every field name against `contextual_language`. Flag fields whose names have no plausible match to any term in the BCC. Do not flag `_id`, `_at`, `_date` suffix conventions if the root concept is covered.

**Check C — Synonym usage.** Flag fields that express a concept already defined in the contextual language but under a different name (e.g. the domain calls it "booking" but the field says `reservation`).

**Check D — Cross-table inconsistency.** If the same concept appears under different names across tables within the same context, flag every instance that deviates from the canonical term.

**Check E — Unexposed domain language conclusions (P1/P3 overlap).** Review `contextual_language` for terms that represent a status, classification, or outcome. Also review `business_capabilities` for capabilities that represent a conclusion. If such a term is defined but has no corresponding field, flag it here as a P1 finding (term defined but not reflected in contract) and note it is also a P3 concern. One entry — do not duplicate under P3.

### Severity (P1)
- **HIGH** — a field or table name actively misleads consumers: wrong synonym, cross-table inconsistency on a core concept, missing field for a defined domain conclusion.
- **MEDIUM** — a field name not in the BCC but not causing clear harm; minor naming divergence.
- **LOW** — genuine uncertainty — flag the issue and state the doubt.

Number P1 findings as `P1-001`, `P1-002`, etc.

---

## Step 3 — Evaluate Principle 2 (Ownership)

### Definition

A source-aligned data product contains your domain's data, and only your domain's data. When teams republish data from other bounded contexts, they create duplicate sources of truth, stale snapshots, and ambiguous ownership.

**What this means in practice:**
- Only expose data your bounded context creates or captures as part of its own business processes.
- Reference foreign entities by ID only.
- **Captured snapshots are yours.** If your bounded context stores a snapshot of foreign data as part of its own process (e.g. the price at the moment of ordering), that snapshot is owned by your context.
- **Fetched state is not yours.** If you would need to query another system to populate a field, it does not belong here.

**The two-question test** for any field that is not a clearly local concept:
1. Does your system create this data? → yours, share it
2. Does your system capture this data as part of its own process? → yours, share it (but name it as a snapshot)
3. Would you need to query or pull from another system to get it? → not yours, reference by ID only

### Checks

**Check A — Fields that are attributes of another bounded context.** Apply the two-question test to each field that is not a clearly local concept. For any field that appears to be an attribute of a foreign bounded context, flag it using this format:

```
[HIGH] [FIELD: field_name] This field appears to be an attribute of another bounded context. Clarify intent before acting:
- If this is foreign state fetched from another context — it does not belong here.
- If this is a point-in-time snapshot captured during a local business action — it is owned by this context and correct to share, but the field name should reflect that clearly (e.g. `price_at_order`, `address_at_shipment`).
The team must confirm which case applies.
```

Severity is HIGH in both cases. Where the BCC `relationships` list includes an entry whose `context` matches the likely owner, reference that context name when flagging.

**Check B — Reference field naming.** For every confirmed foreign key field, check whether the field name matches the term used by the owning context's contextual language.

**Check C — Reference field descriptions.** For every confirmed foreign key field, check whether the field has a description that identifies the owning bounded context. If absent, flag as [MEDIUM].

**Check D — Fields not flagged.** Do not flag confirmed foreign key fields with correct naming and descriptions, or fields that clearly belong to this context.

### Severity (P2)
- **HIGH** — a field carries foreign state that does not belong here, or its ownership is ambiguous.
- **MEDIUM** — a reference field's description does not identify the owning context; minor naming mismatch on a foreign key.
- **LOW** — possible ownership concern but genuine uncertainty.

Number P2 findings as `P2-001`, `P2-002`, etc.

---

## Step 4 — Evaluate Principle 3 (Knowledge vs Data)

### Definition

Bounded contexts sometimes leak their internal state — exposing raw fields and leaving consumers to reconstruct business logic that should never have left the domain.

**What this means in practice:**
- If your domain knows the answer, share the answer. Expose the conclusions your bounded context draws, not just the raw ingredients.
- Raw fields are fine and expected. The problem is when they are exposed *instead of* the conclusions, not *alongside* them.

### Checks

**Check A — Business capabilities that imply unexposed knowledge.** For each capability in `business_capabilities`, ask: does this capability involve a computation, classification, or judgement that produces an outcome a consumer would want? Read the capability `description` — if it implies computing a result, and the raw inputs are in the contract but the result is not, that is a likely P3 finding.

**Check B — Contextual language conclusions not exposed (P1/P3 overlap).** Review `contextual_language` for terms that represent a status, classification, or outcome. If such a term is defined but has no corresponding field, raise it. **If you already raised this as a P1 Check E finding, do not duplicate — reference its ID.**

**Check C — Raw-only exposure.** Look for tables that expose only raw technical data (unprocessed JSON blobs, API response logs, status transition records) with no derived or interpreted output alongside them.

### Severity (P3)
- **HIGH** — a derived fact explicitly listed in `business_capabilities` is absent and the raw ingredients to compute it are present.
- **MEDIUM** — a contextual language term that represents a conclusion is absent; a table that is heavily raw-weighted but does have some interpreted fields.
- **LOW** — genuine uncertainty about whether a conclusion should be exposed.

Number P3 findings as `P3-001`, `P3-002`, etc.

---

## Step 5 — Evaluate Principle 4 (Business Events)

### Definition

Entity state tells you that something changed. Domain events tell you what happened in business terms.

**Two types of finding under P4 — treat them separately:**
1. **Active violations:** Tables that record only CRUD-level state with no business context.
2. **Backlog notes (not findings):** Domain events the BCC implies but that have no corresponding table or field — list separately, not counted in severity totals.

### Building the events list

Derive the events candidate list from `business_capabilities` — for each capability, read its `description` and look for business moments, outcomes, or state changes the domain cares about (e.g. "The ability to confirm when a customer places an order" hints at an `OrderPlaced` event). Note clearly in the output that this list is inferred from capabilities, not a declared event list. If `business_capabilities` is absent or yields no event-like candidates, note this and proceed — Check A can still run.

### Checks

**Check A — CRUD audit log anti-pattern (active violations).** Flag tables whose structure amounts to a generic CRUD audit log:
- Combinations of fields like `table_name`, `old_json`, `new_json`, `crud_operation`, `operation_type`, `raw_payload_json`, `record_id`
- Table names suggesting generic entity tracking: `entity_change_log`, `system_audit_trail`, `generic_record_tracker`

The violation is when only CRUD-style state exists and no domain events are present or planned for events the context explicitly identifies.

**Check B — Domain events vs contracts (backlog notes).** For each candidate domain event inferred from `business_capabilities`, check whether a corresponding table or field exists in the contracts. If no match exists, record it as a **backlog note** — not a finding. Backlog notes do not count toward severity totals.

### Severity (P4)
- **HIGH** — a table is purely a generic CRUD audit log: `old_json`/`new_json`/`crud_operation` combinations, or generic entity tracker table names.
- **MEDIUM** — a table mixes some business fields with heavy CRUD-log characteristics.
- **LOW** — possible concern but genuine uncertainty.

Number P4 active findings as `P4-001`, `P4-002`, etc.

---

## Step 6 — Evaluate Field Description Quality

### Purpose

Well-documented contracts reduce consumer confusion and prevent misuse. This step checks whether each field has a description and whether that description is specific enough to be useful.

### Checks

**Check A — Missing descriptions.** For each field in each contract, check whether a `description` value is present. If absent, flag it.

**Check B — Thin descriptions.** For each field that has a description, assess whether it is specific and actionable. A description is considered thin if it:
- Restates the field name with minor rephrasing (e.g. `booking_id` → "The ID of the booking")
- Is a single generic word such as "identifier", "timestamp", or "value"
- Does not tell a consumer what the field means in business terms or when to use it

### Output

Record all fields with missing or thin descriptions as **documentation backlog items** — not principle findings. These do not count toward severity totals and do not require Team Review blocks.

---

## Step 7 — Write the combined findings document

**File output:** If you have file-write tools available (e.g. Claude Code, Gemini Code Assist, Cursor), save the findings to `findings-[context-name]-all-principles.md` before presenting inline. If not, present inline only.

---

```
# Data Product Alignment Findings — All Four Principles

> Evaluated against: [list of BCC names] / [list of contract table names]

[Note any absent BCC keys here]

---

## Combined Summary

| Principle | HIGH | MEDIUM | LOW | Total |
|---|---|---|---|---|
| P1 — Domain Language | N | N | N | N |
| P2 — Ownership | N | N | N | N |
| P3 — Knowledge vs Data | N | N | N | N |
| P4 — Business Events | N | N | N | N |
| **Total** | **N** | **N** | **N** | **N** |

(P4 backlog notes are listed separately and not counted.)

---

## Principle 1 — Domain Language

### Findings

#### [P1-001] `field_name` — table: `table_name`

**Severity:** HIGH / MEDIUM / LOW
**Check:** A / B / C / D / E
**Issue:** [clear description, citing the relevant contextual language term and its definition]
**Suggested Fix:** [concrete recommendation]

**Team Review** — fill in before running the Agent 3 improvement-plan prompt:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(explain reasoning, or describe your own implementation plan)_

---

[repeat for each P1 finding]

### Tables with no P1 findings
[list any tables that passed all P1 checks cleanly]

---

## Principle 2 — Ownership

### Findings

#### [P2-001] `field_name` — table: `table_name`

**Severity:** HIGH / MEDIUM / LOW
**Check:** A / B / C / D
**Issue:** [clear description — name the bounded context that is likely the owner where known]
**Suggested Fix:** [concrete recommendation]

**Team Review** — fill in before running the Agent 3 improvement-plan prompt:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(for ambiguous P2 fields, confirm whether this is fetched foreign state or a captured snapshot, and describe what you will do)_

---

[repeat for each P2 finding]

### Tables with no P2 findings
[list any tables that passed all P2 checks cleanly]

---

## Principle 3 — Knowledge vs Data

### Findings

#### [P3-001] Missing derived field `suggested_field_name` — table: `table_name`

**Severity:** HIGH / MEDIUM / LOW
**Check:** A / B / C
**Issue:** [describe which business capability or domain conclusion is unexposed, and what raw ingredients are present]
**Suggested Fix:** [concrete recommendation]

**Team Review** — fill in before running the Agent 3 improvement-plan prompt:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(describe what field you will add and how you plan to compute it)_

---

[repeat for each P3 finding]

### Tables with no P3 findings
[list any tables that passed all P3 checks cleanly]

---

## Principle 4 — Business Events

### Active Findings

#### [P4-001] `table_name` — generic CRUD audit log

**Severity:** HIGH / MEDIUM / LOW
**Check:** A
**Issue:** [describe the CRUD-log pattern observed — list the specific fields that indicate this]
**Suggested Fix:** [concrete recommendation]

**Team Review** — fill in before running the Agent 3 improvement-plan prompt:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(describe the domain events you plan to add, or explain why this table is appropriate as-is)_

---

[repeat for each P4 active finding]

### P4 Backlog Notes

> These domain events were inferred from business_capabilities but have no corresponding table or field in the contracts. Not active violations — add to your backlog when implemented.

- `EventName` — [brief note]

[omit if no backlog items]

### Tables with no P4 findings
[list any tables that passed all P4 checks cleanly]

---

## Documentation Quality Backlog

> Fields with missing or thin descriptions. Not active violations — add to your backlog to improve contract documentation.

- `table_name.field_name` — [missing description / thin: explain why it is not specific enough]

[omit section if all fields have adequate descriptions]
```

---

## After presenting the combined findings

**Output the following closing message exactly:**

> These are the findings across all four Data Product Principles. Your producer team should review each finding and fill in the **Team Review** block:
> - Write **Accept** to implement the suggested fix as described
> - Write **Reject** if no change is needed (include a brief reason)
> - Write **Propose Alternative** and describe your own approach if you want to do something different
>
> The P4 backlog notes list domain events inferred from your capabilities but not yet implemented — no action required now, but keep them on your backlog.
>
> The Documentation Quality Backlog lists fields with missing or thin descriptions — no action required now, but work through them when improving contract documentation.
>
> Once all Team Review blocks are filled in, those annotated findings — along with the output of the Agent 2 (consumer requests) template — are the inputs to the Agent 3 (improvement plan) template.

---

## Rules across all four principles

- Apply reasonable judgement. Flag clear violations and likely issues; use [LOW] when uncertain.
- Do **not** duplicate the P1/P3 overlap finding. Raise it once under P1 with a note that it has a P3 dimension, and reference its ID under P3.
- Do **not** stop on missing inputs — note the gap and continue.
- Findings must be numbered per principle (`P1-001`, `P2-001`, `P3-001`, `P4-001`...).
- Keep "Team Review" blocks intact on every finding.
