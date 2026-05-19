# Prompt: Agent 1-P4 — Business Events — Part 2: Evaluate

> **This is Part 2.** Paste everything below this line into the **same conversation** after providing your BCC and data contracts in Part 1. Do not start a new conversation — your inputs are already in this conversation's context.

---

> **You now have the BCC and data contract(s) from this conversation. Run the following evaluation immediately. Do not ask for inputs again.**

---

## Step 3 — Parse inputs

Extract and hold in memory from the BCC(s):
- `name` — context name
- `business_capabilities` — list of capabilities, each with a `name` (2–4 words, starts with a verb) and `description` (starts with "The ability to..."). Scan the `description` of each capability for business moments and outcomes that hint at domain events.
- `services_in_context` — applications making up this bounded context, each classified as `internal` or `boundary_api`. Not relevant to P4 analysis.

**Building the events list:** Derive the events candidate list from `business_capabilities` — for each capability, read its `description` and look for business moments, outcomes, or state changes the domain cares about (e.g. "The ability to confirm when a customer places an order" hints at an `OrderPlaced` event). Note clearly in the output that this list is inferred from capabilities, not a declared event list. If `business_capabilities` is absent or yields no event-like candidates, note this and proceed — Check A can still run.

**Ownership assumption:** Assume all contracts pasted alongside a BCC belong to that BCC's context.

From each data contract, extract table names and all field names.

**Missing or incomplete inputs:** Note any absent keys and continue with what is available.

---

## Step 4 — Analyse each table for Principle 4

### Check A — CRUD audit log anti-pattern (active violations)

Flag tables whose structure amounts to a generic CRUD audit log. Indicators:
- Combinations of fields like `table_name`, `old_json`, `new_json`, `crud_operation`, `operation_type`, `raw_payload_json`, `record_id`
- Table names suggesting generic entity tracking: `entity_change_log`, `system_audit_trail`, `generic_record_tracker`, `entity_state_changes`

The violation is not the presence of an entity state table — it is when only CRUD-style state exists and no domain events are present or planned for events the context explicitly identifies.

### Check B — Domain events in the BCC vs events in the contracts (backlog notes)

For each candidate domain event inferred from `business_capabilities`, check whether a corresponding table or field exists in the contracts. If no match exists, record it as a **backlog note** — not a finding. The principle allows teams to add domain events incrementally.

List backlog notes separately from active violations. They do not count toward HIGH/MEDIUM/LOW totals.

---

## Step 5 — Write the findings

**File output:** If you have file-write tools available (e.g. Gemini Code Assist, Claude Code, Cursor), save the findings document to a markdown file in the current directory **before** presenting it inline. Use the filename `findings-[context-name]-p4.md` (replace `[context-name]` with the BCC `name` field, lowercased and hyphenated). If you do not have file-write tools, present the output inline only.

Present findings in the following format. Number each active finding sequentially as `P4-001`, `P4-002`, etc.

If there are no active findings, say so clearly and proceed to backlog notes.

---

```
# Principle 4 — Business Events Findings

> Evaluated against: [list of BCC names] / [list of contract table names]

[If any BCC keys were missing, note them here]

---

## Summary

| Severity | Count |
|---|---|
| HIGH | N |
| MEDIUM | N |
| LOW | N |

(Backlog notes are listed below and are not counted here)

---

## Active Findings

### [P4-001] `table_name` — generic CRUD audit log

**Severity:** HIGH / MEDIUM / LOW
**Check:** A (which check triggered this finding)
**Issue:** [describe the CRUD-log pattern observed — list the specific fields that indicate this]
**Suggested Fix:** [concrete recommendation — e.g. add a domain event table for the relevant events alongside the audit log; or restructure the table to capture business events instead of raw state transitions]

**Team Review** — fill in before running the Agent 3 improvement-plan prompt:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(describe the domain events you plan to add, or explain why this table is appropriate as-is)_

---

### [P4-002] ...

[repeat for each active finding]

---

## P4 Backlog Notes

> These domain events were identified in the BCC but have no corresponding table or field in the contracts. They are not active violations — add them to your backlog and update the contracts when implemented.

- `EventName` — [brief note on what this event represents]
- `EventName` — ...

[omit this section entirely if all identified domain events are already represented in the contracts, or if no domain events could be identified from the BCC]

---

## Tables with no P4 findings

[list any tables that passed all checks cleanly]
```

---

## Severity guide

- **HIGH** — a table is purely a generic CRUD audit log with no business meaning: `old_json`/`new_json`/`crud_operation` combinations, or generic entity tracker table names.
- **MEDIUM** — a table mixes some business fields with heavy CRUD-log characteristics.
- **LOW** — a possible concern but genuine uncertainty — flag it and state the doubt.

---

## After presenting findings

**Output the following closing message exactly:**

> These are the Principle 4 findings. Your producer team should review each active finding and fill in the **Team Review** block.
>
> The backlog notes list domain events your team has identified but not yet implemented — no action required now, but keep them on your backlog.
>
> Once all Team Review blocks are filled in, those annotated findings feed into the Agent 3 (improvement plan) prompt.
