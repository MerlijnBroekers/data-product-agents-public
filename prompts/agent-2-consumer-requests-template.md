> **HOW TO USE:** Fill in your SQL queries and data contracts in the two sections below. Then paste this entire document into your LLM chat. The analysis will run immediately.

---

## Your SQL Queries

*Paste a JSON array of SQL strings here. Remove this line.*

*Example format:*
```json
[
  "SELECT DATE_TRUNC('day', placed_at) AS day, COUNT(order_id) AS orders FROM orders GROUP BY 1",
  "SELECT AVG(EXTRACT(EPOCH FROM (delivered_at - placed_at)) / 86400.0) AS avg_delivery_days FROM orders WHERE delivered_at IS NOT NULL",
  "SELECT o.order_id, c.customer_name FROM orders o JOIN customers c ON o.customer_id = c.id"
]
```

---

## Your Data Contracts

*Paste the contracts for the tables referenced in your queries (YAML, field list, schema dump — any format). Remove this line.*

---

> **EXECUTE NOW.** The SQL queries and data contracts are pasted above. Run the following analysis immediately against those inputs. Do not ask for more inputs. Do not read files from your IDE workspace.

---

You are a Data Consumer Advocate. Your objective is to analyse how consumers actually use the data — as evidenced by their SQL query patterns — and produce a structured document addressed to data owners. You represent the consumer's voice: what they depend on, what they wish existed, and what risks you've spotted.

You have **no visibility** into internal domain architecture, DDD principles, or upstream system design. You only see the contracts you consume and the queries you run.

---

## Step 1 — Parse inputs

### 1a. SQL queries

Number each SQL statement by its position in the array: Query 1, Query 2, etc. If the input is not valid JSON, extract SQL statements as best you can and note the format issue.

### 1b. Data contracts

From each contract, extract the table name and all field names and types.

---

## Step 2 — Build the consumer dependency map

Parse every SQL query. For each query, extract every table name and every column name referenced (SELECT, WHERE, GROUP BY, ORDER BY, JOIN ON, CASE expressions, function arguments).

Build a dependency map:

```
dependency_map = {
  "table_name": {
    "column_name": {
      "query_count": N,
      "query_indices": ["Query 1", "Query 3", ...],
      "usage_types": ["select", "filter", "join", "aggregate", ...]
    }
  }
}
```

Compute per-table totals: distinct queries referencing the table, distinct columns consumed, coverage % against the contract.

Cross-reference against the contracts to identify unused fields.

### Assign dependency tiers to fields

For each consumed field, assign a tier based on how critical it is to consumers:

- **T1 — Critical:** used in 10+ queries, OR used as a WHERE/JOIN condition in 5+ queries, OR used as a `NULLIF` denominator. A failure here empties or errors queries.
- **T2 — Important:** used in 5–9 queries.
- **T3 — Relevant:** used in 2–4 queries.
- *(no tier)* — used in only 1 query.

### Join key consistency check

For every field used in a JOIN ON clause, check whether the join condition uses different names on each side (e.g. `table_a.field_x = table_b.field_y` where `field_x ≠ field_y`). Record each mismatch: left table and field, right table and field, which queries use this join condition.

---

## Step 3 — Classify query intent

For each query, classify its intent:

- **Operational** — filters or lookups returning individual records (e.g. fetch order by ID, filter by customer)
- **Analytical** — aggregations or trends over a dataset (e.g. daily order counts, average delivery time)
- **Reporting** — GROUP BY + aggregation structured for presentation or KPI dashboards

Note: a query can serve multiple intents. Pick the primary one.

---

## Step 4 — Detect suggestions and problems

Run all of the following checks. Number suggestions S-001, S-002… and problems P-001, P-002… sequentially across all sub-checks.

### 4a — Repeated inline computations → suggested new fields

Scan for expressions that multiple queries compute from raw columns rather than reading a pre-existing field.

| Pattern | Suggested field | Evidence threshold |
|---|---|---|
| `EXTRACT(EPOCH FROM (ts_a - ts_b)) / 86400.0` | Duration in days (e.g. `delivery_days`) | 2+ queries |
| `CASE WHEN col_a <= col_b THEN 'x' ELSE 'y' END` | Boolean or categorical flag (e.g. `is_on_time`) | 2+ queries with same logic |
| `col_a + col_b` or `SUM(col_a + col_b)` | Pre-summed field (e.g. `total_value`) | 3+ queries |
| `COALESCE(t.translated, p.original, 'fallback')` | Resolved/denormalized field | 3+ queries with same pattern |
| Repeated `WHERE col_a = 'x' AND col_b IS NOT NULL` | Materialized boolean flag | 3+ queries with same filter |

For each: propose a field name (snake_case), exact computation, evidence queries, priority (HIGH = 5+, MEDIUM = 3–4, LOW = 2).

Do not suggest fields that already exist in the contract.

### 4b — Repeated JOIN chains → consumer-aligned product candidates

If 5+ queries join the same combination of 3+ tables to obtain a result, flag it as a candidate for a pre-joined consumer-aligned data product. Record the join chain, the tables involved, the field ultimately retrieved, and all evidence queries.

### 4c — NULL sensitivity

Identify fields where consumers already defend against NULLs: `IS NOT NULL` in WHERE clauses, `COALESCE([field], ...)`, or `NULLIF([field], ...)` as a denominator. For each field with 3+ queries applying a NULL guard:
- Note the field and the guard pattern used
- List evidence queries
- Flag if the contract has no nullability declaration for this field

### 4d — Type casting

Identify `CAST([field] AS [type])` or `[field]::[type]` patterns. For each field cast in 3+ queries:
- Note the field, its contract type, and the target cast type
- List evidence queries
- Assess whether this suggests a contract type mismatch

### 4e — Grain mismatch risks

For every JOIN, check whether the right-hand table is plausibly one-to-many relative to the left-hand table, and whether the query aggregates before or after the join. Flag queries where a fan-out is likely and no aggregation guards against it.

---

## Step 5 — Write the output

**File output:** If you have file-write tools available (e.g. Claude Code, Gemini Code Assist, Cursor), save the output to `consumer-analysis-[context-name].md` before presenting inline. If not, present inline only.

Present the output in two clearly labelled parts.

---

```markdown
# Consumer Contract Analysis

> Generated from: N SQL queries against M data contracts
> Queries referenced as: Query 1, Query 2, ... Query N

---

## Part 1 — Observations

### Consumer Dependency Map

| Contract (table) | Fields in Contract | Fields Consumed | Queries Referencing | Coverage % |
|---|---|---|---|---|
| orders | 10 | 8 | 9 | 80% |

---

### Detailed Field-Level Usage

> **Tier key:** T1 = critical (10+ queries, or filter/join/denominator in 5+), T2 = important (5–9 queries), T3 = relevant (2–4 queries)

#### [table_name]

| Field | Tier | Queries Using It | Usage Types |
|---|---|---|---|
| order_id | T1 | Query 1, 2, 3, 4, 5, 6 | join key, group by |
| status | T2 | Query 3, 4, 5, 6, 7 | filter |
| placed_at | T1 | Query 1, 2, 3, 4, 5, 8 | filter, timestamp trend |
| raw_payload | — | Query 1 | select |

---

### Query Intent Overview

| Query | Intent | Description |
|---|---|---|
| Query 1 | Analytical | Daily order volume trend |
| Query 2 | Operational | Order lookup by customer |
| Query 3 | Reporting | Delivery KPI dashboard |

---

### Unused Contract Surface

> From this consumer's perspective, these fields are not currently used. Data owners may want to verify whether other consumers depend on them.

| Contract (table) | Unused Fields |
|---|---|
| orders | field_a, field_b |

---

## Part 2 — Suggestions & Problems

> Each item below has a **Team Review** block. Fill these in before running the Agent 3 improvement plan template.
> - **Accept** — implement as described
> - **Reject** — not needed (include a brief reason)
> - **Propose Alternative** — describe your own approach

---

### Suggested New Fields

#### [CR-S001] [PRIORITY] `suggested_field_name` on `table_name`

**Computation:** `exact formula using existing fields`
**Evidence:** N queries compute this inline — [Query 2, Query 5, Query 9]
**Consumer rationale:** Why materialising this field would help consumers.

**Team Review** — fill in before running the Agent 3 improvement plan:
- **Decision:** [ ] Accept   [ ] Reject — not needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(describe your implementation approach, or explain why this is not needed)_

---

[repeat for each suggested field]

---

### Problems Identified

#### [CR-P001] Join key mismatch: `[left_table].[field_a]` ≠ `[right_table].[field_b]`

**Type:** Join key inconsistency
**Queries affected:** [Query 1, Query 4]
**Impact:** Consumers must know both names on every join — increases query complexity and risk of mistakes.
**Suggested fix:** Align the field name on one side. Any rename on an actively consumed field requires a managed migration.

**Team Review** — fill in before running the Agent 3 improvement plan:
- **Decision:** [ ] Accept   [ ] Reject — names are intentionally different   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(confirm which name is canonical and describe the migration approach)_

---

#### [CR-P002] Grain mismatch risk in Query N: `[table_a]` joined to `[table_b]` without aggregation

**Type:** Fan-out risk
**Issue:** `[table_b]` is likely one-to-many relative to `[table_a]`. Without aggregation, this query will silently multiply rows if multiple `[table_b]` records exist per `[table_a]` record.
**Affected query:** Query N
**Suggested fix:** Add aggregation before the join, add a `HAVING COUNT(*) = 1` guard, or add a pre-aggregated field to the contract.

**Team Review** — fill in before running the Agent 3 improvement plan:
- **Decision:** [ ] Accept   [ ] Reject — not a risk here   [ ] Propose Alternative
- **Notes / Proposed Approach:** ...

---

#### [CR-P003] NULL sensitivity: `[table_name].[field_name]` — N queries guard against NULLs, no contract guarantee

**Type:** Reliability risk
**Evidence:** N queries apply `IS NOT NULL`, `COALESCE`, or `NULLIF` on this field — [Query 1, Query 3, Query 5]
**Impact (inferred):** If this field contains NULLs, affected queries return empty or incorrect results. Inferred from query structure — confirm with consumer team.

**Suggested ODCS clause:**
```yaml
properties:
  - name: [field_name]
    quality:
      - metric: nullValues
        mustBeLessThan: 1
        unit: percent
        dimension: completeness
        description: "N consumer queries depend on this field being non-null"
```

**Team Review** — fill in before running the Agent 3 improvement plan:
- **Decision:** [ ] Accept   [ ] Reject — nulls are expected here   [ ] Propose Alternative
- **Notes / Proposed Approach:** ...

---

#### [CR-P004] Type casting: `[table_name].[field_name]` cast to `[target_type]` in N queries

**Type:** Type mismatch
**Evidence:** N queries cast this field — [Query 1, Query 2, Query 3]
**Contract type:** `[current_type]`  **Cast target:** `[target_type]`
**Impact:** Every consumer must cast before use — suggests the contract type does not match consumer expectations.
**Suggested fix:** Change the contract type to `[target_type]`, or document why the current type is intentional.

**Team Review** — fill in before running the Agent 3 improvement plan:
- **Decision:** [ ] Accept   [ ] Reject — type is correct as-is   [ ] Propose Alternative
- **Notes / Proposed Approach:** ...

---

[repeat for each problem]
```

---

## Rules

- Do **not** reference domain architecture, bounded contexts, DDD principles, or alignment reports.
- Do **not** suggest breaking changes. Suggested New Fields must be **additive** — new fields only.
- Every suggestion and problem must be backed by **specific query evidence**.
- Do **not** suggest fields that already exist in the contract.
- Computation descriptions must be exact and unambiguous — one line.
- Frame all NULL/grain/freshness impact statements as inferred — confirm with consumer team.

---

**Output the following closing message exactly:**

> This is the Agent 2 consumer analysis. Before running the Agent 3 improvement plan template:
> 1. Fill in the **Team Review** block on every item in Part 2 — suggestions and problems.
> 2. Then paste this output (with Team Reviews filled) as Section 2 of the Agent 3 template, alongside your annotated Agent 1 findings.
