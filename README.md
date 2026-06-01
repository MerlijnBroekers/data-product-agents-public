# Data Product Alignment Prompts

---

## Step 1 — Analyse your principles

Open `prompts/agent-1-all-principles-template.md`.

Paste in your **Bounded Context Canvas** and **data contract(s)** where indicated at the top of the file.

Copy the entire file and paste it into your LLM (Claude, ChatGPT, Gemini, or any other).

The LLM will produce a findings document. Each finding has a **Team Review block** — go through them as a team and fill in your decision: Accept, Reject, or Propose Alternative.

---

## Step 2 — Analyse your consumers (can run in parallel with Step 1)

Open `prompts/agent-2-consumer-requests-template.md`.

Paste in your **SQL queries** and **data contract(s)** where indicated at the top of the file.

Copy the entire file and paste it into your LLM.

The LLM will produce a consumer analysis. Each recommendation has a **Team Review block** — go through them as a team and fill in your decision.

---

## Step 3 — Build your improvement plan

Open `prompts/agent-3-improvement-plan-template.md`.

Paste in the **completed output from Step 1** and the **completed output from Step 2** (with all Team Review blocks filled in) where indicated at the top of the file.

Copy the entire file and paste it into your LLM.

The LLM will produce a phased improvement plan: quick wins, consumer enhancements, managed migrations, and deferred items.

---

```
Step 1 — Principle Analysis            Step 2 — Consumer Analysis
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
              Step 3 — Improvement Plan
              (annotated findings from 1 + 2)
                           │
                           ▼
           Phased, risk-ordered improvement plan
```

See `FACILITATOR-GUIDE.md` for guidance on how to run these prompts in a workshop setting.
