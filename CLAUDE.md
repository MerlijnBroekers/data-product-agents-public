# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Plugin Does

This is a **Claude Code plugin** implementing a three-agent pipeline that evaluates data contracts (ODCS v3.1.0) against Domain-Driven Design principles and produces a phased improvement plan. The agents are invoked as skills:

0. `/agent-0-setup` — Interactive setup wizard: checks what input files exist, asks for paths, and writes `data-product-config.json`
1. `/agent-1-alignment` — Evaluates data contracts against 4 DDD-based Data Product Principles using Bounded Context Canvases as ground truth
2. `/agent-2-consumer-requests` — Analyzes SQL query patterns to surface consumer demands and gaps
3. `/agent-3-improvement-plan` — Reconciles Agents 1 and 2 findings into a phased, risk-ordered migration plan

Run Agent 0 first on a new project. Agents 1 and 2 can run independently; Agent 3 depends on both their outputs.

## Installation

```bash
/plugin marketplace add ./path/to/data-product-agents
```

## Configuration

All file paths are configured in `data-product-config.json` in the user's project root. Agent 0 creates this file during setup. Agents 1, 2, and 3 read it at startup and fall back to defaults if it is absent.

| Key | Default |
|---|---|
| `domain_contexts_dir` | `docs/domain_contexts` |
| `datacontracts_dir` | `docs/datacontracts` |
| `analytics_queries` | `docs/analytics_queries.json` |
| `alignment_report` | `docs/alignment_report.md` |
| `contract_recommendations` | `docs/contract_recommendations.md` |
| `improvement_plan` | `docs/improvement_plan.md` |

`ODCS_REFERENCE` remains an env var in `.claude/settings.json` — it points into the plugin itself, not the user's project.

## Architecture

### Plugin Structure

```
skills/agent-0-setup/SKILL.md           — Agent 0 instructions (setup wizard)
skills/agent-1-alignment/SKILL.md       — Agent 1 instructions (~330 lines)
skills/agent-2-consumer-requests/SKILL.md — Agent 2 instructions (~310 lines)
skills/agent-3-improvement-plan/SKILL.md  — Agent 3 instructions (~365 lines)
reference/odcs_reference.md             — Bundled ODCS v3.1.0 spec (offline reference)
.claude/settings.json                   — Default env var configuration
.claude-plugin/plugin.json              — Plugin metadata
```

Each skill is entirely defined in its `SKILL.md` file — there is no separate runtime code. When a skill is invoked, Claude Code reads that file as the instruction set.

### The 4 Data Product Principles (core evaluation criteria)

1. **Use your domain language, consistently** — Field/table names must match the Bounded Context's ubiquitous language; same concept always uses the same name
2. **Do not share what you do not own** — Only expose data the bounded context creates or captures; reference foreign entities by ID only
3. **Share knowledge, not data** — Expose derived, business-meaningful facts (e.g., `is_on_time`, `delivery_days`) rather than raw technical data
4. **Capture business meaning, not just state changes** — Model past-tense domain events (e.g., `OrderPlaced`) rather than generic CRUD audit logs

### Input Formats

**Bounded Context Canvas (BCC):** YAML files in `$DOMAIN_CONTEXTS_DIR/`. Required keys: `name`, `description`, `tables_included`, `ubiquitous_language` (list of `{term, definition}`), `derived_knowledge`, `domain_events`.

**Data Contract:** ODCS v3.1.0 YAML files in `datacontracts_dir/`. One file per table with `schema.objects[].properties` defining fields. FK relationships are declared via the `relationships` block on properties.

**Analytics Queries:** JSON array at `analytics_queries` with entries having `{section, name, purpose, sql, sla?}`.

### Agent 3 Change Classification

Agent 3 classifies all findings into four phases:
- **Phase 0:** Quick wins — zero consumer risk (rename internal fields, add missing domain terms with no existing consumers)
- **Phase 1:** Consumer enhancements — additive changes, zero breaking risk
- **Phase 2:** Managed migrations — breaking changes requiring deprecation paths
- **Phase 3:** Deferred — needs product/domain decision before acting

## Modifying Skills

Each agent's behavior is entirely in its `SKILL.md`. The files follow a numbered step structure (Step 0 = input validation through Step 5/6 = output). Step 0 validation in each skill halts processing on missing or malformed inputs — keep this pattern when adding new agents.

When editing skills, the ODCS spec in `reference/odcs_reference.md` is the authoritative reference for contract format compliance.
