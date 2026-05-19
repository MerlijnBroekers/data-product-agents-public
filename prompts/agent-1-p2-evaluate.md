# Prompt: Agent 1-P2 — Ownership — Part 2: Evaluate

> **This is Part 2.** Paste everything below this line into the **same conversation** after providing your BCC and data contracts in Part 1. Do not start a new conversation — your inputs are already in this conversation's context.

---

> **You now have the BCC and data contract(s) from this conversation. Run the following evaluation immediately. Do not ask for inputs again.**

---

## Step 3 — Parse inputs

Extract and hold in memory from the BCC(s):
- `name` — context name (if absent, use "Unnamed Context")
- `contextual_language` — list of `{term, definition}` pairs (if absent, note the gap — ownership checks can still run but reference naming checks will be limited)
- `relationships` — upstream and downstream context dependencies (if present, use this to name likely owners when flagging foreign attributes; if absent, rely on field-level analysis alone)
- `services_in_context` — describes what external services/APIs this context uses (e.g. boundary APIs, event streams). This is contextual background, not a table ownership list — useful for understanding what external systems this context interacts with, which may inform ownership judgements.

From each data contract, extract:
- The table name
- All field names
- Any foreign key declarations (e.g. `relationships` blocks on properties, or `references` annotations)
- Field `description` values

**Using `relationships`:** If the BCC lists upstream contexts by name, reference them when flagging potential foreign attributes (e.g. "This field appears to belong to the Customer Context listed as an upstream dependency").

**Ownership assumption:** Assume all contracts pasted alongside a BCC belong to that BCC's context. If multiple BCCs are pasted, try to match contracts to contexts by name or ask the user for clarification.

**Missing or incomplete inputs:** Note any absent keys and continue.

---

## Step 4 — Analyse each table for Principle 2

### Check A — Fields that are attributes of another bounded context

Apply the two-question test to each field that is not a clearly local concept (identifiers, timestamps, amounts, or fields clearly derived from the context's own process are fine).

For any field that appears to be an attribute of a foreign bounded context (a descriptive value, classification, or state that another context would own), flag it using this format:

```
[HIGH] [FIELD: field_name] This field appears to be an attribute of another bounded context. Clarify intent before acting:
- If this is foreign state fetched from another context — it does not belong here. Consumers should access it via the owning context's data product using the reference ID.
- If this is a point-in-time snapshot captured during a local business action — it is owned by this context and correct to share, but the field name should reflect that clearly (e.g. `price_at_order`, `address_at_shipment`) so consumers understand they are reading a captured value, not a live foreign attribute.
The team must confirm which case applies.
```

Severity is HIGH in both cases. Ambiguity at this level always warrants a HIGH flag.

Where the BCC `relationships` block identifies upstream contexts by name, reference the specific context when naming the likely owner (e.g. "This field appears to belong to the Customer Context listed as an upstream dependency").

### Check B — Reference field naming

For every confirmed foreign key field, check whether the field name matches the term used by the owning context's contextual language. A mismatch (e.g. using `order_reference` when the owning context calls it `shop_order_id`) is a P2 finding. The reference exists but uses the wrong ownership vocabulary.

### Check C — Reference field descriptions

For every confirmed foreign key field, check whether the field has a description that identifies the owning bounded context. If the description is absent or does not mention ownership, flag it as [MEDIUM] — the reference is technically correct but leaves consumers without ownership context.

### Check D — Fields not flagged

Do not flag:
- Confirmed foreign key fields that have correct naming and descriptions.
- Fields ending in `_id` not confirmed as FKs but whose name clearly matches the primary key of a known table — treat as likely reference pointers; flag at [LOW] at most.
- Fields that clearly belong to this context (the context creates or captures them as part of its process).

---

## Step 5 — Write the findings

**File output:** If you have file-write tools available (e.g. Gemini Code Assist, Claude Code, Cursor), save the findings document to a markdown file in the current directory **before** presenting it inline. Use the filename `findings-[context-name]-p2.md` (replace `[context-name]` with the BCC `name` field, lowercased and hyphenated). If you do not have file-write tools, present the output inline only.

Present findings in the following format. Number each finding sequentially as `P2-001`, `P2-002`, etc.

If there are no findings, say so clearly and skip to the summary.

---

```
# Principle 2 — Ownership Findings

> Evaluated against: [list of BCC names] / [list of contract table names]

[If any BCC keys were missing, note them here]

---

## Summary

| Severity | Count |
|---|---|
| HIGH | N |
| MEDIUM | N |
| LOW | N |

---

## Findings

### [P2-001] `field_name` — table: `table_name`

**Severity:** HIGH / MEDIUM / LOW
**Check:** A / B / C / D (which check triggered this finding)
**Issue:** [clear description — name the bounded context that is likely the owner where known, referencing the BCC relationships if relevant]
**Suggested Fix:** [concrete recommendation]

**Team Review** — fill in before running the Agent 3 improvement-plan prompt:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(for ambiguous P2 fields, confirm whether this is fetched foreign state or a captured snapshot, and describe what you will do)_

---

### [P2-002] ...

[repeat for each finding]

---

## Tables with no P2 findings

[list any tables that passed all checks cleanly]
```

---

## Severity guide

- **HIGH** — a field carries foreign state that does not belong here, or its ownership is ambiguous and cannot be resolved without team input. Both directions of the ambiguity (fetched vs captured) are high-risk.
- **MEDIUM** — a reference field exists but its description does not identify the owning context; a minor naming mismatch on a foreign key field.
- **LOW** — a possible ownership concern but genuine uncertainty — flag it and state the doubt.

---

## After presenting findings

**Output the following closing message exactly:**

> These are the Principle 2 findings. Your producer team should review each one and fill in the **Team Review** block.
>
> For ambiguous P2 findings (marked "Clarify intent before acting"), the notes field is especially important — please confirm whether the field is **fetched foreign state** (should be removed) or a **point-in-time snapshot** (should be renamed to make the captured nature clear) and describe your plan.
>
> Once all Team Review blocks are filled in, those annotated findings feed into the Agent 3 (improvement plan) prompt.

Note to yourself as the model: if consumers are consistently combining this context's data with another context's data, mention that the right architectural solution is a consumer-aligned data product, not duplicating foreign data in a source-aligned product.
