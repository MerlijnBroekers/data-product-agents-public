# Prompt: Agent 1-P3 — Knowledge vs Data — Part 2: Evaluate

> **This is Part 2.** Paste everything below this line into the **same conversation** after providing your BCC and data contracts in Part 1. Do not start a new conversation — your inputs are already in this conversation's context.

---

> **You now have the BCC and data contract(s) from this conversation. Run the following evaluation immediately. Do not ask for inputs again.**

---

## Step 3 — Parse inputs

Extract and hold in memory from the BCC(s):
- `name` — context name
- `contextual_language` — list of `{term, definition}` pairs (if absent, note and continue; Check B will be skipped)
- `business_capabilities` — overview of what this context can do: its key capabilities (e.g. "process customer orders", "calculate delivery time", "determine on-time status"). This is not a direct list of derived fields, but capabilities that imply computation or judgement are strong signals for Check A. Note if absent; Check A will be limited to raw-only table detection.
- `services_in_context` — contextual background about external services the context uses. Not relevant to P3 analysis.

From each data contract, extract all field names per table.

**Missing or incomplete inputs:** Note any absent keys and continue with what is available.

---

## Step 4 — Analyse each table for Principle 3

### Check A — Business capabilities that imply unexposed knowledge

For each entry in the BCC's `business_capabilities`, ask: does this capability involve a computation, classification, or judgement that produces an outcome a consumer would want? (e.g. "calculate delivery time", "determine on-time status", "assess customer eligibility"). If yes, check whether a corresponding derived field plausibly exists in the contract.

This requires judgement — `business_capabilities` describes what the context *can do*, not directly what fields it *should expose*. Use it as a signal: if a capability clearly involves computing a result, and the raw inputs for that computation are in the contract but the result is not, that is a likely P3 finding.

Example: a capability like "determine whether an order was delivered on time" combined with the presence of `delivered_at` and `estimated_delivery_date` in the contract but no `is_on_time` field — the domain can compute the answer; it should share it.

### Check B — Contextual language conclusions not exposed (P1/P3 overlap)

Review the BCC's `contextual_language` for terms that represent a status, classification, or outcome. If such a term is defined but has no corresponding field in the contract, raise it here. This is the same check as P1 Check E — do not duplicate a finding if it was already raised in the P1 analysis. Raise it here only if running this agent independently.

### Check C — Raw-only exposure

Look for tables that expose only raw technical data: unprocessed JSON blobs, API response logs, status transition records — with no derived or interpreted output alongside them. Flag these as leaking internal state without adding knowledge.

Indicators: fields named `raw_payload`, `response_json`, `api_log`, `old_value`, `new_value` with no business-level fields alongside them; tables whose entire field set consists of generic audit columns.

---

## Step 5 — Write the findings

**File output:** If you have file-write tools available (e.g. Gemini Code Assist, Claude Code, Cursor), save the findings document to a markdown file in the current directory **before** presenting it inline. Use the filename `findings-[context-name]-p3.md` (replace `[context-name]` with the BCC `name` field, lowercased and hyphenated). If you do not have file-write tools, present the output inline only.

Present findings in the following format. Number each finding sequentially as `P3-001`, `P3-002`, etc.

If there are no findings, say so clearly and skip to the summary.

---

```
# Principle 3 — Knowledge vs Data Findings

> Evaluated against: [list of BCC names] / [list of contract table names]

[If any BCC keys were missing, note them here, e.g.:]
> **Note:** The BCC did not include `business_capabilities`. Check A was skipped.

---

## Summary

| Severity | Count |
|---|---|
| HIGH | N |
| MEDIUM | N |
| LOW | N |

---

## Findings

### [P3-001] Missing derived field `suggested_field_name` — table: `table_name`

**Severity:** HIGH / MEDIUM / LOW
**Check:** A / B / C (which check triggered this finding)
**Issue:** [describe which business capability or domain conclusion is unexposed, and what raw ingredients are present that could compute it]
**Suggested Fix:** [concrete recommendation — e.g. add a boolean field `is_on_time` computed from `delivered_at <= estimated_delivery_date`]

**Team Review** — fill in before running the Agent 3 improvement-plan prompt:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(describe what field you will add and how you plan to compute it, or explain why this knowledge should not be exposed)_

---

### [P3-002] ...

[repeat for each finding]

---

## Tables with no P3 findings

[list any tables that passed all checks cleanly]
```

---

## Severity guide

- **HIGH** — a derived fact that the BCC explicitly lists in `business_capabilities` is absent from the contract and the raw ingredients to compute it are present. The domain knows the answer and is not sharing it.
- **MEDIUM** — a contextual language term that represents a conclusion is absent; a table that is heavily raw-weighted but does have some interpreted fields.
- **LOW** — genuine uncertainty about whether a conclusion should be exposed, or the capability description is vague.

---

## After presenting findings

**Output the following closing message exactly:**

> These are the Principle 3 findings. Your producer team should review each one and fill in the **Team Review** block.
>
> For "Propose Alternative" entries, describe what field you plan to add and how you will compute it. Agent 3 will use your proposed approach directly in the improvement plan.
>
> Once all Team Review blocks are filled in, those annotated findings feed into the Agent 3 (improvement plan) prompt.
