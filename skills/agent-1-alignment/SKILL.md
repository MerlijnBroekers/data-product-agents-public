---
name: agent-1-alignment
description: "Agent 1 — Alignment (redirects to the four individual principle agents)"
version: 2.0.0
---

The alignment evaluation has been split into four focused agents — one per principle. Tell the user the following:

---

The `/agent-1-alignment` skill has been replaced by four individual principle agents. Run the ones that are relevant to your evaluation:

| Skill | Principle |
|---|---|
| `/agent-1-p1-domain-language` | Principle 1 — Use your domain language, consistently |
| `/agent-1-p2-ownership` | Principle 2 — Do not share what you do not own |
| `/agent-1-p3-knowledge` | Principle 3 — Share knowledge, not data |
| `/agent-1-p4-business-events` | Principle 4 — Capture business meaning, not just state changes |

Each agent will ask you to paste your Bounded Context Canvas(es) and data contract(s) directly into the conversation — no file paths needed. It produces numbered findings, each with a **Team Review** block where your producer team records their decision (Accept / Reject / Propose Alternative) and implementation notes.

After annotating the findings from the agents you ran, proceed to:
- `/agent-2-consumer-requests` — consumer demand analysis (independent, can run in parallel)
- `/agent-3-improvement-plan` — phased improvement plan (requires annotated findings + Agent 2 output)

If you are not sure where to start, run `/agent-0-setup` for a full walkthrough.
