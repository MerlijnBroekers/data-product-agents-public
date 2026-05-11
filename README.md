# Data Product Agents

AI agents that evaluate data contracts against DDD-based Data Product Principles and produce an actionable, phased improvement plan.

Works with **any LLM** via copy-pasted prompts, and is also packaged as a plugin for **Claude Code** and **Gemini** — see the setup guides below.

---

## How it works

```
Stage 1 — Principle checks (run any or all)          Stage 2 — Consumer analysis
─────────────────────────────────────────────        ──────────────────────────────
/agent-1-p1-domain-language                          /agent-2-consumer-requests
/agent-1-p2-ownership                                    │
/agent-1-p3-knowledge                                    │
/agent-1-p4-business-events                              │
        │                                                │
        ▼                                                │
Producer team reviews findings                           │
Fills in: Accept / Reject / Propose Alternative          │
        │                                                │
        └──────────────────┬──────────────────────────────┘
                           ▼
                    Stage 3 — Improvement plan
                    /agent-3-improvement-plan
                           │
                           ▼
              Phased, risk-ordered improvement plan
```

Stage 1 and Stage 2 run independently and in parallel. Stage 3 requires the annotated output of Stage 1 and the output of Stage 2.

Each agent asks you to paste your input directly into the conversation — no file paths or configuration required. If input is incomplete, the agent notes the gaps and continues with what is available.

---

## The agents

### /agent-0-setup

An onboarding guide. Explains the pipeline, the four principles, and what inputs to prepare. Start here.

### /agent-1-p1-domain-language

Evaluates your data contract(s) against **Principle 1 — Use your domain language, consistently**. Checks entity and field naming against your BCC's `contextual_language`, detects synonyms and cross-table inconsistencies, and flags domain language terms that represent conclusions but have no corresponding field.

### /agent-1-p2-ownership

Evaluates against **Principle 2 — Do not share what you do not own**. Checks for fields that are attributes of another bounded context, verifies reference field naming, and flags ambiguous cases (fetched foreign state vs captured snapshot). Uses the BCC `relationships` block to identify upstream context dependencies.

### /agent-1-p3-knowledge

Evaluates against **Principle 3 — Share knowledge, not data**. Checks `business_capabilities` for capabilities that imply unexposed derived fields, flags tables that expose only raw technical data.

### /agent-1-p4-business-events

Evaluates against **Principle 4 — Capture business meaning, not just state changes**. Detects generic CRUD audit log patterns. Uses the BCC's `domain_events` field as the primary source for backlog notes; uses `business_capabilities` as supplementary inspiration when `domain_events` is absent.

### /agent-2-consumer-requests

Analyses SQL query patterns against data contracts. Identifies fields consumers repeatedly compute inline (materialisation candidates), maps high-dependency fields, surfaces unused contract surface, and detects join key naming mismatches.

**Input:** JSON array of SQL statements. Nothing else.

### /agent-3-improvement-plan

Reconciles annotated Stage 1 findings and Stage 2 consumer recommendations into a phased, risk-ordered improvement plan. Only plans for findings the team accepted or proposed an alternative for. Produces ODCS-compatible data quality recommendations.

**Input:** Annotated principle findings (from Stage 1) + Agent 2 output (from Stage 2).

---

## The four Data Product Principles

1. **Use your domain language, consistently** — field and table names must match the contextual language your team discovered during collaborative design.
2. **Do not share what you do not own** — expose only data the bounded context creates or captures as part of its own business processes.
3. **Share knowledge, not data** — expose derived, business-meaningful facts rather than raw data that forces consumers to reconstruct business logic.
4. **Capture business meaning, not just state changes** — model past-tense domain events, not generic CRUD audit logs.

---

## How the Team Review works

Each Stage 1 finding includes a Team Review block. The **producer team** fills it in before running Stage 3:

```
**Team Review** — fill in before running `/agent-3-improvement-plan`:
- **Decision:** [ ] Accept suggested fix   [ ] Reject — no change needed   [ ] Propose Alternative
- **Notes / Proposed Approach:** _(explain reasoning or describe your own implementation plan)_
```

- **Accept** — implement the suggested fix as described
- **Reject** — no change needed; include a brief reason
- **Propose Alternative** — describe your own approach in the Notes field

Agent 3 uses the team's decisions and proposed approaches directly. **The producer team owns the fixes.**

---

## Input formats

### Bounded Context Canvas

One YAML document per domain context. None of the keys are strictly required — agents will note gaps and continue.

```yaml
name: Orders Context
description: Manages the full lifecycle of a customer order from placement to delivery.
contextual_language:
  - term: Order
    definition: A confirmed purchase intent placed by a customer
  - term: Delivery Days
    definition: The number of calendar days between order placement and delivery
  - term: On-Time Delivery
    definition: An order delivered on or before the estimated delivery date
business_capabilities:
  - Process customer orders from placement through to delivery confirmation
  - Determine whether an order was delivered on time
  - Calculate total calendar days from placement to delivery
domain_events:
  - OrderPlaced
  - OrderShipped
  - OrderDelivered
  - OrderCancelled
relationships:
  upstream:
    - context: Customer Context
      description: We receive the customer reference at time of order placement
  downstream:
    - context: Analytics Context
      description: We provide order and delivery data for reporting
services_in_context:
  - Order Management API
  - Delivery tracking event stream
```

### Data contract

One file per table in whatever format your team uses. Should list tables and their fields with names and types.

```yaml
name: orders
schema:
  - name: orders
    properties:
      - name: order_id
        type: TEXT
        required: true
      - name: customer_id
        type: TEXT
        required: true
        description: Reference to the owning customer. Owned by the Customer context.
      - name: placed_at
        type: TIMESTAMP
        required: true
      - name: delivered_at
        type: TIMESTAMP
      - name: estimated_delivery_date
        type: TIMESTAMP
```

### SQL query library

A JSON array of SQL strings. Nothing else.

```json
[
  "SELECT DATE_TRUNC('day', placed_at) AS day, COUNT(order_id) AS orders FROM orders WHERE placed_at IS NOT NULL GROUP BY 1 ORDER BY 1",
  "SELECT AVG(EXTRACT(EPOCH FROM (delivered_at - placed_at)) / 86400.0) AS avg_delivery_days FROM orders WHERE delivered_at IS NOT NULL",
  "SELECT ROUND(SUM(CASE WHEN delivered_at <= estimated_delivery_date THEN 1 ELSE 0 END)::NUMERIC / NULLIF(COUNT(*), 0) * 100, 2) AS on_time_pct FROM orders WHERE delivered_at IS NOT NULL"
]
```

Queries are referenced by position: Query 1, Query 2, Query 3.

---

## Setup: copy-paste prompts (any LLM)

The simplest way to run any agent — no install, no plugin, no clone required. Works with ChatGPT, Claude, Gemini, or any other chat-style LLM.

### 1. Open the prompt file

Browse to the [`prompts/`](./prompts) folder in this repo and open the file for the agent you want to run:

| Agent | Prompt file |
|---|---|
| Start here | [`prompts/agent-0-setup.md`](./prompts/agent-0-setup.md) |
| Principle 1 — Domain language | [`prompts/agent-1-p1-domain-language.md`](./prompts/agent-1-p1-domain-language.md) |
| Principle 2 — Ownership | [`prompts/agent-1-p2-ownership.md`](./prompts/agent-1-p2-ownership.md) |
| Principle 3 — Knowledge | [`prompts/agent-1-p3-knowledge.md`](./prompts/agent-1-p3-knowledge.md) |
| Principle 4 — Business events | [`prompts/agent-1-p4-business-events.md`](./prompts/agent-1-p4-business-events.md) |
| All four principles at once | [`prompts/agent-1-all-principles.md`](./prompts/agent-1-all-principles.md) |
| Consumer requests | [`prompts/agent-2-consumer-requests.md`](./prompts/agent-2-consumer-requests.md) |
| Improvement plan | [`prompts/agent-3-improvement-plan.md`](./prompts/agent-3-improvement-plan.md) |

### 2. Copy everything below the horizontal rule

Each prompt file has a short header explaining what it does, followed by a `---` line. Copy everything below that line.

### 3. Paste into your LLM and follow its instructions

The agent will greet you and ask for your inputs (BCC, contracts, SQL queries) step by step. Paste them in when prompted.

---

## Setup: Claude Code

For interactive use with slash commands like `/agent-0-setup`.

### 1. Clone this repository

```bash
git clone https://github.com/your-org/data-product-agents.git
```

### 2. Open the plugin manager

In Claude Code, run `/plugins`. Navigate to the **Marketplaces** tab and click **Add Marketplace**.

### 3. Enter the local path

Enter the absolute path to where you cloned the repo:

- macOS/Linux: `/home/yourname/data-product-agents`
- Windows (WSL): `/mnt/c/Users/yourname/Desktop/data-product-agents`
- Windows (native): `C:\Users\yourname\Desktop\data-product-agents`

### 4. Install and reload

Install the plugin, then run `/reload-plugins`. The skills should appear when you type `/` in the prompt.

### 5. Start

```
/agent-0-setup
```

---

## Setup: Gemini

### Option A — Gemini CLI

### 1. Clone this repository

```bash
git clone https://github.com/your-org/data-product-agents.git
```

### 2. Install the plugin

In the Gemini CLI, load the plugin from the local path the same way you would for any skill directory:

```bash
gemini skills add ./data-product-agents
```

### 3. Start

The skills are immediately available. Run any agent by name:

```
/agent-0-setup
/agent-1-p1-domain-language
```

---

### Option B — gemini agent window

### 1. Clone this repository

```bash
git clone https://github.com/your-org/data-product-agents.git
```

### 2. Open a skill file

In the gemini agent window, tell the agent to open the skill you want to run:

```
Open skills/agent-1-p1-domain-language/SKILL.md and follow the instructions in it.
```

The agent reads the file and starts the skill immediately — it will ask you to paste your BCC and contracts step by step.

To run a different agent, tell it to open the corresponding file:

```
Open skills/agent-3-improvement-plan/SKILL.md and follow the instructions.
```

---

## Workflow example

```
1. (parallel) Run /agent-1-p1-domain-language  →  paste BCC + contracts  →  P1 findings
2. (parallel) Run /agent-1-p2-ownership         →  paste BCC + contracts  →  P2 findings
3. (parallel) Run /agent-2-consumer-requests    →  paste SQL array + contracts  →  consumer map

4. Producer team fills in Team Review blocks on P1 and P2 findings

5. Run /agent-3-improvement-plan  →  paste annotated findings + Agent 2 output  →  phased plan
```

You can run any subset of the principle agents. Agent 3 works with whatever findings you provide.
