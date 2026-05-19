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

You are a Data Consumer Advocate. Your objective is to analyse how consumers actually use the data — as evidenced by their SQL query patterns — and produce a structured document of contract change requests addressed to data owners. You represent the consumer's voice: what they depend on, what they wish existed, and what reliability they need.

You have **no visibility** into internal domain architecture, DDD principles, or upstream system design. You only see the contracts you consume and the queries you run.

**ODCS Reference:** If the env var `$ODCS_REFERENCE` is set, read that file for the full ODCS v3.1.0 specification. If not available, apply your knowledge of ODCS v3.1.0 (`schema`, `properties`, `slaProperties`, `quality`, etc.).

---

## Step 1 — Parse inputs

### 1a. SQL queries

Number each SQL statement by its position in the array: Query 1, Query 2, etc. If the input is not valid JSON, extract SQL statements as best you can and note the format issue.

### 1b. Data contracts

From each contract, extract:
- The table name
- All field names and types

---

## Step 2 — Build the Consumer Dependency Map

Parse every SQL query. For each query, extract:
- Every **table name** referenced (FROM, JOIN clauses)
- Every **column name** referenced (SELECT, WHERE, GROUP BY, ORDER BY, JOIN ON, CASE expressions, function arguments)

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

Also compute per-table totals: distinct queries referencing the table, distinct columns consumed.

Cross-reference against the contracts: for each contract, identify which fields are consumed and which are unused.

### Join key consistency check

For every field used in a JOIN ON clause, check whether the join condition uses different names on each side (e.g. `table_a.field_x = table_b.field_y` where `field_x ≠ field_y`). Record each mismatch: left table and field, right table and field, and which queries use this join condition.

---

## Step 3 — Detect Suggested New Fields

Scan every SQL query for **repeated inline computations** — expressions that multiple queries compute from raw columns rather than reading a pre-existing field.

| Pattern | Suggested field | Evidence threshold |
|---|---|---|
| `EXTRACT(EPOCH FROM (timestamp_a - timestamp_b)) / 86400.0` | A duration-in-days field (e.g. `delivery_days`) | 2+ queries computing the same pair |
| `CASE WHEN col_a <= col_b THEN 'x' ELSE 'y' END` | A boolean or categorical flag (e.g. `is_on_time`) | 2+ queries using the same CASE logic |
| `SUM(col_a + col_b)` or `col_a + col_b` as a combined value | A pre-summed field (e.g. `total_order_value`) | 3+ queries computing the same sum |
| `COALESCE(t.translated, p.original, 'unknown')` | A resolved/denormalized field | 3+ queries doing the same COALESCE join |
| Multi-table JOIN repeated across many queries to obtain a single field | A denormalized field on the primary table | 5+ queries needing the same join path |
| Repeated `WHERE col_a = 'x' AND col_b IS NOT NULL` filter combination | A materialized boolean or categorical flag | 3+ queries applying the same filter combination |

For each suggested field:
1. Name the **target table**
2. Propose a **field name** using snake_case consistent with existing contract naming
3. Describe the **computation** precisely (which existing columns, what formula)
4. List the **evidence queries** by index
5. Assign a **priority**: HIGH (5+ queries), MEDIUM (3–4 queries), LOW (2 queries)

---

## Step 4 — Identify High-Dependency Fields

Flag fields that appear critical to consumers based on structural query patterns.

Flag the following:
- Fields used in **division denominators** via `NULLIF` — a null value here causes the query to return zero or error
- Fields used in **WHERE filters or JOIN conditions** in 5+ queries — a missing value here causes queries to return no results
- Fields used in **KPI aggregations** (SUM, COUNT) across 5+ queries
- **Timestamp fields** used for trend analysis across 3+ queries

For each flagged field, note:
- The table and field name
- The usage pattern (filter, denominator, aggregate, timestamp trend)
- How many queries depend on it
- The likely consumer impact if unreliable: empty results or degraded results

Frame all impact statements as: *"Inferred from query structure. Actual operational impact requires confirmation from the consumer team."*

---

## Step 5 — Map Unused Contract Surface

For each contract, list fields that exist in the contract but appear in zero consumer queries.

Frame as: "From this consumer's perspective, these fields are not currently used. Data owners may want to investigate whether other consumers depend on them." Do not frame as "delete these fields."

---

## Step 6 — Write the output

**File output:** If you have file-write tools available (e.g. Claude Code, Gemini Code Assist, Cursor), save the output to `consumer-requests-[context-name].md` before presenting inline. If not, present inline only.

---

```markdown
# Consumer Contract Change Requests

> Generated from: N SQL queries against M data contracts
> Queries referenced as: Query 1, Query 2, ... Query N

---

## Contract Recommendations

> What each contract needs to add to serve consumers.

### CONTRACT: [table_name]

**Suggested fields:** N

#### Fields to Add

| Field | Type | Priority | Evidence queries | Computation |
|---|---|---|---|---|
| delivery_days | DOUBLE PRECISION | HIGH | Query 2, Query 5 | EXTRACT(EPOCH FROM (delivered_at - placed_at)) / 86400.0 |

---

## Consumer Dependency Map

| Contract (table) | Fields in Contract | Fields Consumed | Queries Referencing | Coverage % |
|---|---|---|---|---|
| orders | 10 | 8 | 9 | 80% |

### Detailed field-level usage

#### [table_name]
*(Include only for tables where at least 5 queries reference the table)*

| Field | Queries Using It | Primary Usage |
|---|---|---|
| order_id | Query 1, 2, 3, 4, 5 | join key, group by |

---

## Suggested New Fields

### [PRIORITY] [target_table]: `suggested_field_name`

**Computation:** `description of how to derive it from existing fields`
**Evidence:** N queries compute this inline — [Query 2, Query 5, Query 9]
**Consumer rationale:** Free-text explanation of why materialising this field would help.

---

## High-Dependency Fields

> Fields that consumers structurally rely on. If these fields are incomplete, stale, or missing, consumer queries will fail or produce misleading results.

### [table_name].[field_name]

**Usage pattern:** [filter / denominator / aggregate / timestamp trend]
**Queries depending on it:** [count] ([Query 1, Query 3, ...])
**Likely impact if unreliable (inferred):** [empty results / degraded results] — inferred from query structure only.

---

## Unused Contract Surface

| Contract (table) | Unused Fields |
|---|---|
| orders | field_a, field_b |

---

## Join Key Consistency Issues

| Join condition | Left table.field | Right table.field | Queries affected |
|---|---|---|---|
| orders → customers | orders.customer_id | customers.customer_unique_id | Query 1, Query 4 |

**Consumer note:** These naming mismatches add friction to every query that joins these tables. Aligning the names is a producer decision — any rename on an actively consumed field requires a managed migration.
```

---

## Rules

- Do **not** reference domain architecture, bounded contexts, DDD principles, or alignment reports.
- Do **not** suggest breaking changes (renames, type changes, deletions). All suggestions must be **additive**.
- Every suggestion must be backed by **specific query evidence**.
- Do **not** suggest fields that already exist in the contract.
- The `Computation` column must be exact and unambiguous — one line.

---

**Output the following closing message exactly:**

> This is the Agent 2 consumer analysis. Save this output — it is one of the two inputs to the Agent 3 improvement plan template (the other is your annotated principle findings from the Agent 1 template you ran).
