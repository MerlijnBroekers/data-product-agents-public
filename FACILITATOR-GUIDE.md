# Facilitator Guide — Data Product Agent Prompts

This guide explains how to run the three agent prompt templates across the workshop series.

---

## Overview

| When | Who | Action |
|---|---|---|
| End of preparation workshop | Facilitator | Send **Email 1** — request BCC + data contract from team |
| After preparation workshop | Teams | Generate data contract (BOLD CLI) + send BCC metadata to facilitator |
| Before workshop 1 | Facilitator | Run **Agent 1** on team inputs |
| Workshop 1 | Facilitator + team | Review Agent 1 findings; team fills in review blocks |
| End of workshop 1 | Facilitator | Send **Email 2** — request consumer SQL queries from team |
| End of workshop 1 | Teams | Provide consumer SQL queries as a JSON list |
| Before workshop 2 | Facilitator | Run **Agent 2** on SQL queries + contracts |
| Workshop 2 | Facilitator + team | Review Agent 2 findings; team fills in review blocks on both outputs |
| End of workshop 2 | Facilitator | Send **Email 3** — share annotated findings + Agent 3 template with team |
| After workshop 2 | Teams | Run **Agent 3** independently to generate their improvement plan |

---

## Before Workshop 1 — Run Agent 1

**What you need from the team:**
- Bounded Context Canvas (BCC) metadata file
- Data contract(s) — generated using the BOLD CLI after the preparation workshop

**Steps:**

1. Open `prompts/agent-1-all-principles-template.md`
2. Copy the entire file
3. Paste into your LLM chat (Claude, Gemini, ChatGPT, etc.)
4. In the chat, replace the placeholder text with:
   - The team's BCC under **Your Bounded Context Canvas (BCC)**
   - The team's data contract(s) under **Your Data Contracts**
5. Send the prompt — the analysis runs immediately
6. If the LLM did not automatically save a `.md` file, ask it to do so:
   > *"Please save the findings to a markdown file called `findings-[team-name]-all-principles.md`"*
7. Save the output file — you will use it in workshop 1

---

## Before Workshop 2 — Run Agent 2

**What you need from the team (collected at the end of workshop 1):**
- A JSON array of SQL queries — ask teams to provide unique queries only, numbered by position

Example format to share with teams:
```json
[
  "SELECT order_id, status FROM orders WHERE placed_at > '2024-01-01'",
  "SELECT AVG(delivery_days) FROM orders WHERE delivered_at IS NOT NULL"
]
```

**Steps:**

1. Open `prompts/agent-2-consumer-requests-template.md`
2. Copy the entire file
3. Paste into your LLM chat
4. In the chat, replace the placeholder text with:
   - The team's SQL queries (JSON array) under **Your SQL Queries**
   - The same data contract(s) used in Agent 1 under **Your Data Contracts**
5. Send the prompt
6. If the LLM did not automatically save a `.md` file, ask it to do so:
   > *"Please save this analysis to a markdown file called `consumer-analysis-[team-name].md`"*
7. Save the output file — you will use it in workshop 2

---

## During Workshop 2 — Team Review

Share both output files with the team. For every finding, the team fills in the **Team Review** block:

- **Accept** — implement the suggested fix as described
- **Reject** — no change needed (ask them to add a brief reason)
- **Propose Alternative** — they have a different approach (ask them to describe it)

The team can do this directly in the `.md` files or on printed copies.

---

## After Workshop 2 — Run Agent 3 (Team)

Tell teams they can now generate their phased improvement plan using Agent 3. Share the template with them.

**What they need:**
- The annotated Agent 1 findings file (Team Review blocks filled in)
- The annotated Agent 2 consumer analysis file (Team Review blocks filled in)

**Steps for the team:**

1. Open `prompts/agent-3-improvement-plan-template.md`
2. Copy the entire file
3. Paste into an LLM chat
4. Replace the placeholder text with:
   - Annotated Agent 1 findings under **Section 1**
   - Annotated Agent 2 output under **Section 2**
5. Send the prompt
6. If the LLM did not automatically save a `.md` file, ask it to do so:
   > *"Please save the improvement plan to a markdown file called `improvement-plan-[team-name].md`"*

The output is a phased improvement plan (Phase 0–3) the team can use to implement their data contract improvements after the workshops.

---

## Email Templates

Use these templates to communicate with teams at each handoff point. Adapt names and dates as needed.

---

### Email 1 — After Preparation Workshop
*Send to: team*
*Trigger: end of preparation workshop*

> **Subject: Data Product Workshop — Please send us your BCC and data contract**
>
> Hi [team name],
>
> Thank you for today's preparation workshop. To get ready for Workshop 1, we need two files from you:
>
> 1. **Your Bounded Context Canvas (BCC)** — the metadata file describing your bounded context
> 2. **Your data contract** — generated using the BOLD CLI
>
> If you haven't generated your data contract yet, please do so using the BOLD CLI and export it as a YAML file.
>
> **Please send both files as a reply to the Workshop 1 meeting invite by [date — suggested: 3 working days before Workshop 1].** This ensures the facilitator running Workshop 1 receives your inputs directly — note that this may be a different person than today's facilitator.
>
> If you run into any issues generating the contract, reply to this email and we'll help you.
>
> See you at Workshop 1 on [date].
>
> Kind regards,
> [Facilitator name]

---

### Email 2 — After Workshop 1
*Send to: team*
*Trigger: end of workshop 1 (or same-day follow-up)*

> **Subject: Data Product Workshop — Please send us your consumer SQL queries**
>
> Hi [team name],
>
> Great session today. As discussed, we need your team's consumer SQL queries before Workshop 2.
>
> Please collect the SQL queries that your consumers run against your data contracts and send them as a **JSON array** — one query per entry, no duplicates. Use the format below:
>
> ```json
> [
>   "SELECT order_id, status FROM orders WHERE placed_at > '2024-01-01'",
>   "SELECT AVG(delivery_days) FROM orders WHERE delivered_at IS NOT NULL"
> ]
> ```
>
> A few tips:
> - Include only unique queries — remove duplicates
> - Copy them directly from your query tool or BI layer if possible
> - There is no minimum or maximum — include what you have
>
> **Please send the JSON as a reply to the Workshop 2 meeting invite by [date — suggested: 3 working days before Workshop 2].** This ensures the facilitator running Workshop 2 receives your input directly — note that this may be a different person than today's facilitator.
>
> See you at Workshop 2 on [date].
>
> Kind regards,
> [Facilitator name]

---

### Email 3 — After Workshop 2
*Send to: team*
*Trigger: end of workshop 2*

> **Subject: Data Product Workshop — Your improvement plan tool**
>
> Hi [team name],
>
> Thank you for Workshop 2. Attached are your completed findings documents with your team's decisions filled in:
>
> - `findings-[team-name]-all-principles.md` — principle findings with your Team Review annotations
> - `consumer-analysis-[team-name].md` — consumer analysis with your Team Review annotations
>
> You can now generate your phased improvement plan using the Agent 3 template. Here's how:
>
> 1. Open the attached `agent-3-improvement-plan-template.md`
> 2. Copy the entire file and paste it into an LLM chat (Claude, Gemini, ChatGPT, etc.)
> 3. Paste your annotated findings from the two files above into **Section 1** and **Section 2**
> 4. Send the prompt — the plan will be generated immediately
> 5. If the LLM does not save a file automatically, ask it: *"Please save the improvement plan to a markdown file called `improvement-plan-[team-name].md`"*
>
> The output is a phased plan (Phase 0–3) you can use to prioritise and implement your data contract improvements.
>
> Feel free to reach out if you have any questions.
>
> Kind regards,
> [Facilitator name]
