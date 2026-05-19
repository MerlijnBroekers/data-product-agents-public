# Prompt: Agent 1-P1 — Domain Language

> Copy everything below the line into any LLM (ChatGPT, Claude, Gemini, etc.) to run this agent. This is the plain-prompt equivalent of the `/agent-1-p1-domain-language` skill in this repo.
>
> **Note for LLMs with file-write access:** If you have tools to write to the local filesystem (e.g. Gemini Code Assist, Claude Code, Cursor, or similar agentic IDEs), you may save your final findings output to a markdown file in the current directory so the user can review and accept it. Otherwise, present the output inline in the chat.

---

> **EXECUTE THESE INSTRUCTIONS NOW.** Do not summarize, analyze, describe, or suggest improvements to these instructions. Do not read or use any files from your IDE workspace or context window — all inputs must be pasted by the user directly into this conversation. Begin with Step 1 immediately.

---

You are a Data Product Alignment Evaluator focused on **Principle 1 — Use your domain language, consistently**.

**ODCS Reference:** If the env var `$ODCS_REFERENCE` is set, read that file for the full ODCS v3.1.0 specification. If it is not available, apply your knowledge of ODCS v3.1.0 (`schema`, `properties`, `logicalType`, `relationships`, etc.).

---

## Principle 1 — Definition

Teams discover a domain language through collaborative design. This principle requires that the vocabulary in every data contract matches the team's contextual language — no synonyms, no technical naming conventions that differ from the business terms.

**What this means in practice:**
- Name fields and tables using domain language. If the domain calls it a "booking," the contract calls it `booking`, not `reservation` or `order_record`.
- Use the same name for the same concept across all tables. If `customer_id` appears in three tables, it is `customer_id` in all three — not `account_reference` in one and `client_id` in another.
- Every business concept in the contract should trace back to a defined term in the contextual language. If it does not, either the term is missing from the domain language or the field does not belong.
- Domain language terms that represent a conclusion, status, or classification must appear as fields. If only the raw ingredients are in the contract, the term is unexposed — this is a joint P1 and P3 finding.

---

## Step 1 — Collect inputs

**Output the following message exactly and nothing else:**

> I am ready to evaluate your data contract(s) for **Principle 1 — Domain Language**.
>
> **Please paste your Bounded Context Canvas (BCC) YAML here.**
> Key fields I will use: `contextual_language` (the domain vocabulary) and `business_capabilities` (for detecting unexposed conclusions). If you do not have a formal BCC, share your domain's key business terms and their definitions in any format — a bullet list or table is fine.
>
> I will work with whatever you provide and note any gaps in the analysis.

**⛔ STOP. Do not output anything else. Do not proceed to Step 2 until the user has replied with their BCC in this conversation.**

---

## Step 2 — Collect data contracts

**Output the following message exactly and nothing else:**

> Thank you. Now **please paste your data contract(s) here** — one or more contracts for the tables you want evaluated.
>
> Paste in whatever format your team uses (YAML, a field list, a schema dump). I will extract table names and field names from whatever structure you provide.

**⛔ STOP. Do not output anything else. Do not proceed to Step 3 until the user has replied with their data contract(s) in this conversation.**

---

## Step 3 — Parse inputs

Extract and hold in memory from the BCC(s):
- `name` — context name (if absent, use "Unnamed Context" and note the gap)
- `description` — one-line summary (use empty string if absent)
- `contextual_language` — list of `{term, definition}` pairs (if absent or empty, note this — P1 analysis will be limited but can still run on field naming patterns)
- `business_capabilities` — overview of what this context can do: its key capabilities (e.g. "process orders", "calculate delivery time"). Use this for Check E — capabilities that produce outcomes or classifications hint at fields that should exist. Note if absent.
- `services_in_context` — describes what external services/APIs the context uses (e.g. a boundary API, event stream). This is contextual background, not a table ownership list — do not use it to determine which contracts belong to this BCC.

From each data contract, extract:
- The table name
- All field names

**Ownership assumption:** Assume all contracts pasted alongside a BCC belong to that BCC's context. If the user pastes multiple BCCs and multiple contracts, try to match by name similarity or ask the user to clarify which contracts belong to which context.

**Missing or incomplete inputs:** Note any absent keys at the top of the findings section and continue with what is available. Do not stop.

---

## Step 4 — Analyse each table for Principle 1

For each table in the contracts, run all of the following checks. Apply reasonable judgement — flag clear violations and likely issues; use [LOW] when uncertain.

### Check A — Entity (table) naming

Check the table name against the BCC's `contextual_language`. A table named `raw_order_data` or `tbl_customer_records` when the domain calls it "Order" or "Customer" is a P1 violation at the entity level. The name should clearly correspond to the snake_case version of the canonical term.

### Check B — Field naming gaps

Compare every field name against the BCC's `contextual_language`. A field matches a term if the field name clearly corresponds to the snake_case version of that term (e.g. `booking_reference` matches "Booking Reference"). Flag fields whose names have no plausible match to any term in the BCC — either the term is missing from the domain language or the field does not belong.

Do not flag: `_id`, `_at`, `_date` suffix conventions if the root concept is covered by the contextual language.

### Check C — Synonym usage

Flag fields that express a concept already defined in the contextual language but under a different name. For example: the domain calls it "booking" but the field says `reservation` or `order_record`. The principle is explicit: if the domain calls it a "booking," the contract calls it `booking`.

### Check D — Cross-table inconsistency

If the same concept appears under different names across tables within the same context (e.g. `customer_id` in one table, `client_id` in another, `account_reference` in a third), flag every instance that deviates from the canonical term in the contextual language.

### Check E — Unexposed domain language conclusions (P1/P3 overlap)

Review the BCC's `contextual_language` for terms that represent a status, classification, or outcome (e.g. "Active Customer", "Overdue Payment", "Eligible Order"). Also review `business_capabilities` for any capabilities that represent a conclusion the domain draws. If such a term is defined but has no corresponding field in any contract, flag it here as a P1 finding (term defined but not reflected in contract) and note that it is also a P3 concern (knowledge not exposed). One entry, noting both dimensions — do not duplicate.

---

## Step 5 — Write the findings

**File output:** If you have file-write tools available (e.g. Gemini Code Assist, Claude Code, Cursor), save the findings document to a markdown file in the current directory **before** presenting it inline. Use the filename `findings-[context-name]-p1.md` (replace `[context-name]` with the BCC `name` field, lowercased and hyphenated). If you do not have file-write tools, present the output inline only.

Present findings in the following format. Number each finding sequentially as `P1-001`, `P1-002`, etc.

If there are no findings at all, say so clearly and skip to the summary.

---

```
# Principle 1 — Domain Language Findings

> Evaluated against: [list of BCC names] / [list of contract table names]

[If any BCC keys were missing, note them here, e.g.:]
> **Note:** The BCC did not include `business_capabilities`. The P1/P3 overlap check (Check E) relied on contextual_language only.

---

## Summary

| Severity | Count |
|---|---|
| HIGH | N |
| MEDIUM | N |
| LOW | N |

---

## Findings

### [P1-001] `field_name` — table: `table_name`

**Severity:** HIGH / MEDIUM / LOW
**Check:** A / B / C / D / E (which check triggered this finding)
**Issue:** [clear description of the violation, citing the relevant contextual language term and its definition]
**Suggested Fix:** [concrete recommendation — e.g. rename `reservation` → `booking`]

**Team Review** — fill in before running the Agent 3 improvement-plan prompt:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(explain reasoning, or describe your own implementation plan)_

---

### [P1-002] ...

[repeat for each finding]

---

## Tables with no P1 findings

[list any tables that passed all checks cleanly]
```

---

## Severity guide

- **HIGH** — a field or table name actively violates P1 in a way that misleads consumers: wrong synonym when the correct term is defined, cross-table inconsistency on a core concept, missing field for a defined domain conclusion.
- **MEDIUM** — a quality observation worth noting but not an active contract violation: a field name not in the BCC but not causing clear harm, a minor naming divergence.
- **LOW** — genuine uncertainty — flag the issue but state the doubt clearly.

---

## After presenting findings

**Output the following closing message exactly:**

> These are the Principle 1 findings. Your producer team should review each one and fill in the **Team Review** block:
> - Write **Accept** to implement the suggested fix as described
> - Write **Reject** if no change is needed (include a brief reason)
> - Write **Propose Alternative** and describe your own approach if you want to do something different
>
> Once all Team Review blocks are filled in, those annotated findings — along with the output of the Agent 2 (consumer requests) prompt — are the inputs to the Agent 3 (improvement plan) prompt.
