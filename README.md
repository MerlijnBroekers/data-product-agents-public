# Data Product Alignment Prompts

Three prompts that help teams evaluate and improve their data contracts.

---

**Prompt 1 — Principle Analysis** (`prompts/agent-1-all-principles-template.md`)
- **What it does:** Evaluates a team's data contract against the four Data Sharing Principles.
- **Input:** Fill in your Bounded Context Canvas and data contract(s) at the top of the file.
- **Output:** A findings document with a Team Review block on each finding for the team to fill in.

**Prompt 2 — Consumer Analysis** (`prompts/agent-2-consumer-requests-template.md`)
- **What it does:** Analyses how consumers use the data based on their SQL query patterns.
- **Input:** Fill in your SQL queries and data contract(s) at the top of the file.
- **Output:** A dependency map, suggested new fields, and flagged risks — each with a Team Review block.

**Prompt 3 — Improvement Plan** (`prompts/agent-3-improvement-plan-template.md`)
- **What it does:** Turns the team's annotated findings into a phased improvement plan.
- **Input:** Paste the completed outputs of Prompts 1 and 2 (with Team Review blocks filled in) at the top of the file.
- **Output:** A sequenced plan across four phases — quick wins, consumer enhancements, managed migrations, and deferred items.

---

```
Prompt 1 — Principle Analysis          Prompt 2 — Consumer Analysis
──────────────────────────────         ──────────────────────────────
BCC + data contracts                   SQL queries + data contracts
        │                                          │
        ▼                                          ▼
  Principle findings                       Consumer findings
  (Team Review blocks)                     (Team Review blocks)
        │                                          │
        ▼                                          ▼
Team fills in each finding:            Team fills in each finding:
Accept / Reject / Propose Alternative  Accept / Reject / Propose Alternative
        │                                          │
        └──────────────────┬───────────────────────┘
                           ▼
              Prompt 3 — Improvement Plan
              (annotated findings from 1 + 2)
                           │
                           ▼
           Phased, risk-ordered improvement plan
```

Prompts 1 and 2 run independently and can run in parallel. Prompt 3 requires the annotated output of both.

See `FACILITATOR-GUIDE.md` for instructions on how and when to run each prompt.
