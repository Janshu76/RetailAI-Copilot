# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RetailAI-Copilot is a hackathon project that builds a Snowflake-based retail analytics layer with a semantic model for natural-language querying (the "Copilot" experience). The project is centered on a single denormalized source table `STORE_DATA` in the `RETAIL_AI.RETAIL` database/schema, with views layered on top to expose business entities and metrics, plus a YAML semantic model describing the canonical schema for an LLM/query engine.

## Repository Structure

- `data/raw/` — landing zone for raw input data (currently empty)
- `sql/01_create_views.sql` — creates four entity views over `STORE_DATA` (customers, products, sales, outlets)
- `sql/02_buisness_metrics.sql` — creates business metric views (total sales, profit, AOV, profit margin, category/region/outlet breakdowns, monthly trend). Note the intentional typo in the filename (`buisness`).
- `sql/03_validation.sql` — smoke-test queries against the base table and views. Note: this file currently contains syntax errors (see "Known Issues" below).
- `semantic_models/semantic_model.yaml` — single-table semantic model for the `sales` table, mapping logical names (e.g. `category`, `sales_date`, `total_profit`) to physical column expressions. This is the contract an LLM uses to translate natural-language questions to SQL.
- `docs/architecture.md` — high-level architecture note (currently a one-line stub)

## Architecture

Three layers, all in Snowflake under database `RETAIL_AI`, schema `RETAIL`:

1. **Raw layer** — single wide table `STORE_DATA` containing customers, products, sales transactions, and outlet attributes combined.
2. **Entity views** (`VW_*`) — normalizes `STORE_DATA` into four logical entities: `VW_CUSTOMERS`, `VW_PRODUCTS`, `VW_SALES`, `VW_OUTLETS`. The semantic model currently only references one of these (`sales`).
3. **Metric views** (`METRIC_*`) — pre-aggregated business metrics grouped by category, region, outlet type, and year-month.
4. **Semantic layer** — `semantic_models/semantic_model.yaml` exposes dimensions and measures to an LLM/agent, with the `sales` base table mapping to physical columns. Column names in the warehouse are upper-case; the YAML preserves one quoted identifier (`"Sub-Category"`) that contains a hyphen and is therefore case-sensitive in Snowflake.

## Development Commands

There is no build system, test runner, or linter configured — this is a SQL + YAML project. Standard development flow is to execute the SQL files sequentially against a Snowflake account:

```sql
-- run in order, each in its own worksheet or via SnowSQL:
!source sql/01_create_views.sql
!source sql/02_buisness_metrics.sql
!source sql/03_validation.sql
```

Raw data ingestion target is `RETAIL_AI.RETAIL.STORE_DATA` (place CSVs in `data/raw/` then load via `PUT` + `COPY INTO`).

## Known Issues

`sql/03_validation.sql` is broken as committed and will not execute:

- Line 1: `SELECT DATABASE RETAIL_AI;` — invalid syntax; should be `USE DATABASE RETAIL_AI;`
- Line 4: `SEELCT COUNT(*) FROM STORE_DATA;` — typo (`SEELCT` instead of `SELECT`)
- Line 11–12: `SUM(PROFIT) AS TOTAL_PROFIT,` — trailing comma before `FROM` is invalid SQL
- Line 1 also re-issues `USE SCHEMA RETAIL;` correctly, so the `SELECT DATABASE` line is the only one needing the `USE` fix

If asked to fix or run this file, correct these four issues first.

## Conventions

- File naming in `sql/` uses a numeric prefix (`01_`, `02_`, `03_`) to enforce execution order. Follow this pattern when adding new SQL files.
- Metric views use a `METRIC_<NAME>` prefix; entity views use a `VW_<NAME>` prefix. Match these when adding new views.
- All identifiers in the semantic model are written in `snake_case` (logical) mapping to `UPPER_CASE` (physical). Keep this convention when extending the YAML.
- The semantic model targets only the `sales` entity today. Entity views for customers/products/outlets exist in SQL but are not yet described in the YAML — extending the semantic model to cover them is a likely next step.
