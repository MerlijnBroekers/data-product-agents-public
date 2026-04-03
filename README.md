# Data Product Agents

A Claude Code plugin with three AI agents that evaluate data contracts against DDD-based Data Product Principles and produce an actionable improvement plan.

---

## The three agents

```
Agent 1 — Alignment Evaluator    /agent-1-alignment
Agent 2 — Consumer Advocate      /agent-2-consumer-requests
Agent 3 — Improvement Planner    /agent-3-improvement-plan
```

Run them in order. Each agent feeds into the next.

```
Bounded Context Canvases  ──┐
Data Contracts            ──┴──► Agent 1 ──► alignment_report.md ──┐
                                                                     │
                                                                     ├──► Agent 3 ──► improvement_plan.md
Analytics Queries ──► Agent 2 ──► contract_recommendations.md ──────┘
Data Contracts    ──┘
```

### Agent 1 — Alignment Evaluator

Reads your Bounded Context Canvases (BCCs) and data contracts, then evaluates every table against the 4 Data Product Principles. Outputs a structured report with HIGH / MEDIUM / LOW findings per principle, per table.

**Inputs:** BCCs (`domain_contexts_dir`), contracts (`datacontracts_dir`)
**Output:** `$ALIGNMENT_REPORT`

### Agent 2 — Consumer Advocate

Reads your analytics query library and data contracts. Identifies fields consumers repeatedly compute inline (suggesting materialisation), flags SLA gaps, maps unused contract surface, and detects join key naming mismatches.

**Inputs:** Query library (`analytics_queries`), contracts (`datacontracts_dir`)
**Output:** `$CONTRACT_RECOMMENDATIONS`

### Agent 3 — Improvement Planner

Mediates between Agent 1 findings and Agent 2 demands. Classifies every change by consumer risk, sequences it into phases (Phase 0 quick wins → Phase 1 additive → Phase 2 managed migrations → Phase 3 deferred), and produces ODCS-compatible SLA recommendations.

**Inputs:** `alignment_report`, `contract_recommendations`, `analytics_queries`, `datacontracts_dir`, `domain_contexts_dir`
**Output:** `$IMPROVEMENT_PLAN`

---

## Prerequisites

Before running the agents, your project needs:

| File / folder | What it is | How to produce it |
|---|---|---|
| `domain_contexts_dir/*.yaml` | Bounded Context Canvas files — the gold standard for evaluation | Author manually; see the BCC format below |
| `datacontracts_dir/*.yaml` | ODCS v3.1.0 data contracts, one per table | Generate from your schema using `datacontract-cli` or equivalent |
| `analytics_queries` | JSON array of queries with `name`, `sql`, and optional `sla` fields | Author manually; see the query format below |

Agent 3 additionally requires the outputs of Agents 1 and 2 to be present and complete.

---

## Installation

### 1. Clone this repository

```bash
git clone https://gitlab.rewirenow.com/merlijnbroekers/data-product-agents.git
```

### 2. Open the plugin manager

In Claude Code, run:

```
/plugins
```

Navigate to the **Marketplaces** tab and click **Add Marketplace**.

### 3. Enter the local path

When prompted for a marketplace source, enter the absolute path to where you cloned the repo:

- macOS/Linux: `/home/yourname/data-product-agents`
- Windows (WSL): `/mnt/c/Users/yourname/Desktop/data-product-agents`
- Windows (native): `C:\Users\yourname\Desktop\data-product-agents`

### 4. Install the plugin

Claude Code will show a plugin details screen for **data-product-agents**. Choose your install scope:

- **Install for you (user scope)** — available in all your projects
- **Install for all collaborators on this repository (project scope)** — shared via the repo

### 5. Reload plugins

```
/reload-plugins
```

Confirm the skills loaded — they should appear when you type `/` in the prompt. If the skills don't show up, close and re-open your terminal and try again.

### 6. Run the setup wizard

```
/agent-0-setup
```

This interactive wizard checks which input files exist in your project, asks for their paths, and writes `.claude/settings.local.json` so the agents know where to find your data.

---

## Configuration

Agent 0 writes a `data-product-config.json` file to your project root during setup. This is the single source of truth for all file paths. Edit it directly at any time to change where the agents look for inputs or write outputs.

```json
{
  "domain_contexts_dir":      "docs/domain_contexts",
  "datacontracts_dir":        "docs/datacontracts",
  "analytics_queries":        "docs/analytics_queries.json",
  "alignment_report":         "docs/alignment_report.md",
  "contract_recommendations": "docs/contract_recommendations.md",
  "improvement_plan":         "docs/improvement_plan.md"
}
```

If `data-product-config.json` is absent, all agents fall back to the defaults shown above.

---

## Input formats

### Bounded Context Canvas (`$DOMAIN_CONTEXTS_DIR/*.yaml`)

One file per bounded context. Agent 1 will halt if any of the five required keys are missing or empty.

```yaml
name: Orders Context
description: Manages the full lifecycle of a customer order from placement to delivery.
tables_included:
  - orders
  - order_items
ubiquitous_language:
  - term: Order
    definition: A confirmed purchase intent placed by a customer
  - term: Order Item
    definition: A single product line within an order, including quantity and price at time of purchase
  - term: Delivery Days
    definition: The number of calendar days between order placement and delivery to the customer
  - term: On-Time Delivery
    definition: An order delivered on or before the estimated delivery date
derived_knowledge:
  - Whether an order was delivered on time (delivered_at <= estimated_delivery_date)
  - Total number of calendar days from placement to delivery
domain_events:
  - OrderPlaced
  - OrderShipped
  - OrderDelivered
  - OrderCancelled
```

---

### Data contract (`$DATACONTRACTS_DIR/*.yaml`)

One file per table, following [ODCS v3.1.0](https://github.com/bitol-io/open-data-contract-standard). A full spec reference is bundled at `reference/odcs_reference.md`. Each contract must have a `schema` block with at least one object containing `name` and `properties`.

```yaml
apiVersion: v3.1.0
kind: DataContract
id: a3f1c2d4-8e56-4b2a-9f10-123456789abc
name: orders
version: 1.0.0
status: active
domain: orders

servers:
  - server: production-db
    type: postgresql
    host: db.example.com
    port: 5432
    database: shop
    schema: public
    environment: prod

schema:
  - name: orders
    physicalType: table
    logicalType: object
    properties:
      - name: order_id
        logicalType: string
        physicalType: TEXT
        primaryKey: true
        primaryKeyPosition: 1
        required: true
      - name: customer_id
        logicalType: string
        physicalType: TEXT
        required: true
        description: Reference to the owning customer. Owned by the Customer context.
        relationships:
          - to: customers.customer_id
            type: foreignKey
      - name: status
        logicalType: string
        physicalType: TEXT
        required: true
      - name: placed_at
        logicalType: timestamp
        physicalType: TIMESTAMP
        required: true
      - name: delivered_at
        logicalType: timestamp
        physicalType: TIMESTAMP
      - name: estimated_delivery_date
        logicalType: timestamp
        physicalType: TIMESTAMP
```

---

### Analytics queries (`analytics_queries`)

A JSON array of consumer queries. Each entry must have `name` and `sql`. The optional `sla` object signals that the query owner considers the underlying fields contractable.

```json
[
  {
    "section": "1. Order Funnel",
    "name": "1.1 Daily order volume",
    "purpose": "Track daily order volume for trend analysis and anomaly detection",
    "sql": "SELECT DATE_TRUNC('day', placed_at) AS day, COUNT(order_id) AS orders FROM orders WHERE placed_at IS NOT NULL GROUP BY 1 ORDER BY 1",
    "sla": {
      "metric": "daily_order_volume_freshness",
      "suggested_target": "Data for the previous day available by 06:00 UTC",
      "sla_type": "freshness"
    }
  },
  {
    "section": "1. Order Funnel",
    "name": "1.2 Order status distribution",
    "purpose": "Show how orders progress through statuses to identify processing bottlenecks",
    "sql": "SELECT status, COUNT(order_id) AS orders FROM orders GROUP BY status ORDER BY orders DESC"
  },
  {
    "section": "2. Delivery Performance",
    "name": "2.1 Average delivery time",
    "purpose": "Measure average days from order placement to delivery",
    "sql": "SELECT AVG(EXTRACT(EPOCH FROM (delivered_at - placed_at)) / 86400.0) AS avg_delivery_days FROM orders WHERE delivered_at IS NOT NULL",
    "sla": {
      "metric": "delivery_completeness",
      "suggested_target": "delivered_at non-null rate above 99% for completed orders",
      "sla_type": "completeness"
    }
  },
  {
    "section": "2. Delivery Performance",
    "name": "2.2 On-time delivery rate",
    "purpose": "Percentage of orders delivered on or before the estimated date",
    "sql": "SELECT ROUND(SUM(CASE WHEN delivered_at <= estimated_delivery_date THEN 1 ELSE 0 END)::NUMERIC / NULLIF(COUNT(*), 0) * 100, 2) AS on_time_pct FROM orders WHERE delivered_at IS NOT NULL AND estimated_delivery_date IS NOT NULL"
  }
]
```
