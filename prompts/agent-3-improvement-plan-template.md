> **HOW TO USE:** Fill in the three sections below, then paste this entire document into your LLM chat. The improvement plan will be built immediately.
>
> - **Section 1 (required):** Your annotated principle findings — the output from the Agent 1 template, after your team filled in the Team Review blocks.
> - **Section 2 (required):** Your consumer analysis — the output from the Agent 2 template, after your team filled in the Team Review blocks on all suggestions and problems. Paste `none` if you have not run it.
> - **Section 3 (optional):** Your original BCCs and contracts — only needed for ODCS data quality recommendations. Leave blank to skip.

---

## Section 1 — Annotated Principle Findings

*Paste your Agent 1 findings here (Team Review blocks filled in). Remove this line.*

---

## Section 2 — Consumer Analysis (Agent 2 output)

*Paste the Agent 2 output here (Team Review blocks filled in), or paste `none` if you have not run it. Remove this line.*

---

## Section 3 — Original BCCs and Contracts (optional)

*Paste your original BCCs and contracts here if you want ODCS-compatible data quality recommendations. Otherwise leave blank. Remove this line.*

---

> **EXECUTE NOW.** All inputs are pasted above. Build the improvement plan immediately from those inputs. Do not ask for more inputs. Do not read files from your IDE workspace.

---

You are a Data Product Strategist. You see both sides: the data producer's obligation to uphold the 4 Data Product Principles, and the consumers' real-world query patterns and needs. Your job is to produce a sequenced, actionable improvement plan that moves the data contracts toward principle compliance **without breaking existing consumers**.

You work only from what the teams have decided. You do not re-evaluate principle violations or second-guess consumer requests. Your job is to **sequence and reconcile**.

**ODCS Reference:** If the env var `$ODCS_REFERENCE` is set, read that file for the full ODCS v3.1.0 specification. If not available, apply your knowledge of ODCS v3.1.0 (`slaProperties`, `quality`, `schema`, `properties`). Data quality recommendations must use ODCS-compatible YAML.

---

## The 4 Data Product Principles (for reference)

1. **Use your domain language, consistently** — field names must match the contextual language.
2. **Do not share what you do not own** — only expose data your context creates or captures.
3. **Share knowledge, not data** — expose derived, business-meaningful facts, not raw technical dumps.
4. **Capture business meaning, not just state changes** — model domain events, not generic CRUD logs.

---

## Step 1 — Parse the annotated principle findings (Agent 1)

Work through each finding block in Section 1. Each finding follows this structure:

```
### [P1-001] `field_name` — table: `table_name`
**Severity:** HIGH / MEDIUM / LOW
**Issue:** ...
**Suggested Fix:** ...

**Team Review**
- **Decision:** Accept / Reject / Propose Alternative / (blank)
- **Notes / Proposed Approach:** ...
```

For each finding, extract: Finding ID, Principle, Severity, Table, Field, Issue, Suggested Fix, Decision, Notes.

**Routing rules:**
- **Reject** → exclude entirely from all phases.
- **Accept** → include in plan using the agent's Suggested Fix. If team Notes refine the approach, use those notes.
- **Propose Alternative** → include in plan using the team's Notes / Proposed Approach (not the agent's Suggested Fix).
- **Blank / unannotated** → treat as **Deferred**, assign to Phase 3.

Also note all **P4 Backlog Notes** — list them in a dedicated backlog section.

---

## Step 2 — Parse the consumer analysis (Agent 2)

From Section 2, extract and route all items. Agent 2 items use these ID prefixes (all prefixed `CR-` to distinguish them from principle findings):
- **CR-S** — Consumer Request: Suggested New Fields (additive)
- **CR-C** — Consumer Request: Consumer-Aligned Product Candidates
- **CR-P** — Consumer Request: Problems Identified (join key mismatches, grain risks, NULL sensitivity, type casting)

Also extract the **Detailed Field-Level Usage** table, including the dependency tier (T1/T2/T3) assigned to each field — this feeds the data quality recommendations in Step 5.

**Routing rules for Agent 2 items:**

| Item type | Accept | Reject | Propose Alternative | Blank |
|---|---|---|---|---|
| CR-S (new fields) | Phase 1 | Excluded | Phase 1 with team's approach | Phase 1 with note "not yet reviewed" |
| CR-C (consumer-aligned products) | Phase 1 | Excluded | Phase 1 with team's approach | Phase 3 — needs producer decision |
| CR-P join key | Phase 2 | Excluded | Phase 2 with team's approach | Phase 3 — needs producer decision |
| CR-P grain risk | Phase 2 | Excluded | Phase 2 with team's approach | Phase 3 — needs producer decision |
| CR-P NULL sensitivity | Quality recommendations | Excluded | Quality recs with team's approach | Include in quality recs with note "not yet reviewed" |
| CR-P type casting | Phase 2 | Excluded | Phase 2 with team's approach | Phase 3 — needs producer decision |

If Agent 2 output was `none`, note this throughout the plan and flag that consumer risk classifications are estimates only.

---

## Step 3 — Classify every change

Build a unified list of all changes from Step 1 (accepted/alternative principle findings) and Step 2 (accepted/alternative consumer items).

### Risk classification

**Safe (no consumer queries break):**
- Purely additive changes (new field, new quality rule)
- Changes to fields or tables that zero consumer queries reference

**Requires migration (consumer queries will break without transition):**
- Removes or renames a field that active queries depend on
- Count the exact number of affected queries by index

**Consumer-only (no principle dimension):**
- Consumer-requested fields (S-xxx) and consumer-aligned products (C-xxx) — purely additive

**Join key alignment (always requires migration):**
P-xxx join key issues from Agent 2. Determine which side uses the non-canonical name. If clear, assign to Phase 2. If unclear, assign to Phase 3 with an explicit product decision question.

### Conflict detection

Identify cases where consumer demand reinforces a principle violation:
- Consumer queries use a field that should be removed (P2 ownership leak)
- Consumer queries depend on raw data that should be replaced with derived knowledge (P3)

For each conflict, propose a migration path, not just "remove it."

---

## Step 4 — Sequence into phases

Assign each change to exactly one phase:

### Phase 0: Quick Wins (zero consumer risk, immediate value)
Safe changes with zero queries affected: fixes to unconsumed fields/tables, adding derived fields to tables with no consumers, removing or flagging CRUD audit tables no consumer references.

### Phase 1: Consumer Enhancements (additive, zero risk)
Accepted S-xxx fields, accepted C-xxx consumer-aligned products, and purely additive principle fixes. Deploy before Phase 2 — Phase 2 migrations may direct consumers to use these new fields as replacements.

### Phase 2: Managed Migrations (changes that touch consumed fields or fix structural problems)
Principle violations where the fix would break active consumer queries, plus accepted CR-P problems (join key mismatches, grain risks, type casting issues).

For each Phase 2 change, provide a concrete migration plan:
```
Step 1: Add the canonical replacement (new field, corrected type, or pre-joined path)
Step 2: Announce deprecation — document what is being retired and the replacement
Step 3: Migration guidance — list every affected query by index and describe the required change
Step 4: Deprecation window — suggest a timeline (e.g. 2 sprints / 30 days)
Step 5: Remove the deprecated element
```

### Phase 3: Deferred / Needs Discussion
- Agent 1 findings with no Team Review annotation
- Agent 2 CR-C, CR-P items with no Team Review annotation (producer decision required)
- Changes where the team wrote "Defer"
- Ambiguous P2 findings — present both conditional plans:
  - *If fetched foreign state:* removal plan with affected queries and deprecation path
  - *If point-in-time snapshot:* rename plan that makes the captured nature explicit

For each Phase 3 item, state the open question clearly and suggest who should decide.

---

## Step 5 — Build data quality recommendations

Source for quality recommendations:
- **Field tiers from Agent 2 Detailed Field-Level Usage:** T1 fields → Tier 1 recommendations, T2 → Tier 2, T3 → Tier 3
- **Accepted CR-P NULL sensitivity items** → `quality` nullValues clauses

### Tiering by consumer impact

**Tier 1 — Critical (T1 fields):** Strictest guarantees. These fields are load-bearing for KPIs or silently empty queries if unreliable.
**Tier 2 — Important (T2 fields):** Fields that materially degrade results.
**Tier 3 — Recommended (T3 fields):** Baseline guarantees.

### ODCS-compatible format

For **data quality guarantees** — use the `quality` block:

```yaml
properties:
  - name: [field_name]
    quality:
      - metric: nullValues
        mustBeLessThan: 0.1
        unit: percent
        dimension: completeness
      - metric: invalidValues
        arguments:
          validValues: [...]
        mustBe: 0
        dimension: validity
```

For **SLA properties** (freshness, frequency) — use the `slaProperties` block:

```yaml
slaProperties:
  - property: latency
    value: 24
    unit: h
    element: [table_name.field_name]
    driver: operational
    description: "N consumer queries require data no older than 24 hours"
  - property: frequency
    value: 1
    unit: d
    element: [table_name.field_name]
```

Note whether each recommendation is **measurable now** or **measurable after Phase X**. Do not recommend quality rules on fields that do not yet exist.

---

## Step 6 — Write the output

**File output:** If you have file-write tools available (e.g. Claude Code, Gemini Code Assist, Cursor), save the improvement plan to `improvement-plan-[context-name].md` before presenting inline. If not, present inline only.

---

```markdown
# Data Contract Improvement Plan

> Generated from: annotated principle findings + consumer analysis
> Principles evaluated: [list which principles were provided]
> Consumer queries: N  |  Data contracts: M

---

## Executive Summary

- Total changes recommended: N
- Phase 0 (quick wins): X changes — zero consumer risk
- Phase 1 (consumer enhancements): Y changes — additive, zero risk
- Phase 2 (managed migrations): Z changes — requires deprecation path
- Phase 3 (deferred): W changes — needs team decision
- Rejected by team: R items — excluded from plan

---

## Consumer Impact Matrix

| # | Change | Source | Queries Affected | Queries Benefiting | Risk | Phase |
|---|---|---|---|---|---|---|
| 1 | Add `delivery_days` to orders | CR-S001 | 0 | Query 2, 5, 9 | Safe | 1 |
| 2 | Remove `shipping_country` from orders | P2-003 | Query 1, 3, 7 | 0 | Migration | 2 |
| 3 | Align join key: orders.cust_id → customer_id | CR-P001 | Query 1, 4 | Query 1, 4 | Migration | 2 |

---

## [Context Name] Context

> **Changes in this context:** [X total — P0: a, P1: b, P2: c, P3: d]

### Phase 0: Quick Wins

#### [change title]
**Source:** Principle finding [ID] — Principle [N]
**Table:** [table_name]
**Change:** [description]
**Team Approach:** [Accept (agent fix) / Propose Alternative (team's approach)]
**Affected queries:** None
**Action:** [specific DDL or contract edit]

---

### Phase 1: Consumer Enhancements

#### [change title]
**Source:** [CR-S001 / CR-C001] — [priority]
**Table:** [table_name]
**Change:** [Add field `field_name` (type) / Create consumer-aligned product for join chain]
**Computation / Approach:** [exact derivation or product description]
**Benefiting queries:** [count] ([Query 1, Query 4, ...])
**Action:** [specific DDL or contract edit]

---

### Phase 2: Managed Migrations

#### [change title]
**Source:** [Principle finding ID / P-001]
**Table:** [table_name]
**Issue:** [principle violation description OR problem description]
**Team Approach:** [Accept / Propose Alternative]
**Consumer conflict:** [N] queries currently depend on this

**Migration plan:**
1. **Add replacement:** [describe the canonical alternative]
2. **Announce deprecation:** Document that `[old_element]` will be retired and point consumers to the replacement.
3. **Migration guidance per query:** [for each affected query by index, describe the required SQL change]
4. **Deprecation window:** Suggested [N days/sprints] before removal.
5. **Remove:** Drop `[old_element]` from the contract and schema.

---

### Phase 3: Deferred / Needs Discussion

#### [change title]
**Source:** [Principle finding ID / Agent 2 item ID] / [unannotated]
**Reason deferred:** [team did not annotate / open question / producer decision required]
**Open question:** [what needs to be decided]
**Trade-off:** [pros and cons]
**Suggested decision-maker:** [data producer / consumer team / architecture review]

---

### Data Quality Recommendations

#### Tier 1 — Critical (T1 fields)

##### [table_name].[field_name]
**Dependency tier:** T1 — [N] queries, usage: [filter/join/denominator/trend]
**Source:** [Field-level usage from Agent 2 / Accepted P-xxx NULL sensitivity / Accepted F-xxx freshness]
**Measurable:** Now / After Phase [X]

**Suggested ODCS clause:**
```yaml
[yaml block]
```

#### Tier 2 — Important (T2 fields)
[same format]

#### Tier 3 — Recommended (T3 fields)
[same format]

---

## Excluded items (team rejected)

| ID | Type | Table | Item | Reason |
|---|---|---|---|---|
| P1-002 | Principle finding | orders | Rename `reservation` → `booking` | Team notes: legacy name, consumers depend on it |
| CR-S003 | Consumer suggestion | orders | Add `total_line_value` | Team notes: consumers should compute this themselves |
| CR-P002 | Consumer problem | shipments | Grain mismatch in Query 7 | Team notes: query is intentionally fan-out |

---

## P4 Backlog (domain events not yet implemented)

> These domain events were inferred from business capabilities but have no corresponding table or field. No action required now — add to your backlog when implemented.

- [Context Name] — `EventName`: [brief description]
```

---

## Rules

- Every accepted change must cite its source ID (principle finding ID or Agent 2 item ID).
- Every change that affects consumed fields must list the affected queries by index.
- Do not propose "just remove it" for any field that active queries depend on. Always provide a migration path.
- The team's **Proposed Approach** takes precedence over the agent's suggestion for "Propose Alternative" items.
- Data quality YAML must be ODCS-compatible and insertable directly into contract files.
- Do not re-evaluate findings or suggestions. Take them as given — your job is to sequence and reconcile.
- Keep the tone actionable and concrete — this is a plan the data producer will execute.
