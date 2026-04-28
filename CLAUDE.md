# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Plugin Does

This is a **Claude Code plugin** implementing a three-stage pipeline that evaluates data contracts against Domain-Driven Design principles and produces a phased improvement plan. Compatible with Claude and Gemini models. No ODCS dependency.

0. `/agent-0-setup` — Onboarding guide: explains the workflow and what inputs to prepare
1. `/agent-1-p1-domain-language` — Evaluates contracts against Principle 1 (domain language)
2. `/agent-1-p2-ownership` — Evaluates contracts against Principle 2 (ownership)
3. `/agent-1-p3-knowledge` — Evaluates contracts against Principle 3 (knowledge vs data)
4. `/agent-1-p4-business-events` — Evaluates contracts against Principle 4 (business events)
5. `/agent-2-consumer-requests` — Analyses SQL query patterns to surface consumer demands and gaps
6. `/agent-3-improvement-plan` — Reconciles annotated principle findings and consumer recommendations into a phased migration plan

The legacy `/agent-1-alignment` skill redirects users to the four individual principle agents.

Agents 1a–1d and Agent 2 can run independently. Agent 3 depends on annotated findings from Stage 1 and Agent 2 output.

## Installation

```bash
/plugins
```

Navigate to Marketplaces → Add Marketplace → enter the local path to this repo.

## Architecture

### Plugin Structure

```
skills/agent-0-setup/SKILL.md               — Onboarding guide
skills/agent-1-alignment/SKILL.md           — Redirect to individual principle agents
skills/agent-1-p1-domain-language/SKILL.md  — Principle 1 check
skills/agent-1-p2-ownership/SKILL.md        — Principle 2 check
skills/agent-1-p3-knowledge/SKILL.md        — Principle 3 check
skills/agent-1-p4-business-events/SKILL.md  — Principle 4 check
skills/agent-2-consumer-requests/SKILL.md   — Consumer advocate
skills/agent-3-improvement-plan/SKILL.md    — Improvement planner
reference/odcs_reference.md                 — Bundled ODCS spec (retained for reference; not used by agents)
.claude-plugin/plugin.json                  — Plugin metadata
```

Each skill is entirely defined in its `SKILL.md` file — there is no separate runtime code.

### Key Design Decisions (v2.1)

**No file paths.** All agents prompt the user to paste their input directly into the conversation. Agents work with whatever is provided and note any gaps — they do not hard-fail on missing keys.

**ODCS reference is optional but preferred.** The `$ODCS_REFERENCE` env var points to `reference/odcs_reference.md`. When set (Claude Code resolves it automatically), agents read it for the full ODCS spec. When not set (e.g. Gemini API usage), agents fall back to built-in knowledge. Data quality recommendations in Agent 3 output are ODCS-compatible YAML.

**SQL-only query input.** Agent 2 accepts a JSON array of SQL strings. Queries are referenced by position (Query 1, Query 2, etc.).

**Team annotation loop.** Stage 1 agents output findings with a Team Review block per finding. The producer team fills in Decision (Accept / Reject / Propose Alternative) and Notes before running Agent 3. Agent 3 only plans for accepted/alternative findings; rejected findings are excluded; unannotated findings are deferred.

**The team owns the fixes.** The principle agents flag issues and suggest approaches. The producer team decides what to do and how — Agent 3 uses the team's proposed approach directly in the plan.

### The 4 Data Product Principles

1. **Use your domain language, consistently** — field/table names must match the BCC's `contextual_language`; same concept always uses the same name
2. **Do not share what you do not own** — only expose data the bounded context creates or captures; reference foreign entities by ID only
3. **Share knowledge, not data** — expose derived facts from `business_capabilities`; don't leave consumers to reconstruct logic
4. **Capture business meaning, not just state changes** — model domain events from `business_capabilities`; avoid generic CRUD logs

### BCC Field Names

| Field | Purpose | Used by |
|---|---|---|
| `name` | Context name | All agents |
| `description` | One-line summary | All agents |
| `contextual_language` | Domain vocabulary — list of `{term, definition}` pairs | P1, P2, P3 |
| `business_capabilities` | What this context can do — its capabilities. Hints at derived fields (P3) and domain events (P4) but is not a direct list of either. | P3, P4 |
| `domain_events` | Authoritative list of past-tense business events (e.g. `OrderPlaced`) | P4 |
| `relationships` | Upstream and downstream context dependencies | P2 |
| `services_in_context` | External services/APIs the context uses (e.g. boundary API, event stream). Contextual background only — not a table ownership list, not used for contract matching. | (background context) |

**Table ownership:** Since `services_in_context` is not a table list, agents assume that all contracts pasted alongside a BCC belong to that BCC's context. When multiple BCCs are involved, agents match by name or ask the user.

### SQL Query Input

Agent 2 accepts a JSON array of SQL strings. Nothing else — no name, section, purpose, or SLA annotation fields.

### Agent 3 Change Classification

Agent 3 classifies all accepted/alternative findings into four phases:
- **Phase 0:** Quick wins — zero consumer risk
- **Phase 1:** Consumer enhancements — additive changes, zero breaking risk
- **Phase 2:** Managed migrations — breaking changes requiring deprecation paths
- **Phase 3:** Deferred — unannotated findings, team-deferred items, or findings needing product/domain decision

## Modifying Skills

Each agent's behavior is entirely in its `SKILL.md`. The files follow this pattern:
- Introduction and principle definition
- Step 1: prompt user for first input (BCC or SQL queries)
- Step 2: prompt user for second input (contracts)
- Step 3: parse inputs (graceful on missing keys)
- Step 4: analysis (principle-specific logic)
- Step 5: write findings with Team Review annotation blocks

For Gemini compatibility: keep instructions explicit and numbered, avoid hard dependencies on env vars, do not use model-specific features.
