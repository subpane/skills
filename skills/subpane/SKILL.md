---
name: subpane
description: Design, publish, and maintain live Subpane dashboards over MCP. Use when the user asks to create or update a dashboard, status board, agent/service telemetry monitor, daily/executive briefing, or business KPI report in Subpane, or provides a Subpane `/boards/{board_id}` link.
---

# Subpane Dashboard Workflow

Turn structured JSON into calm, scannable Subpane dashboards over Streamable HTTP MCP.

> **Source of truth**: Treat the connected MCP server's live tool schemas, `get_board_guide`, and `get_limits` as authoritative whenever they differ from these local reference files (such as on newer server releases or custom-quota instances).

## 1. Connect & Resolve the Target Board

- **Resolve the MCP endpoint**: When the user provides a board link (`https://<host>/boards/{board_id}`) or an explicit server origin, target `https://<host>/mcp`; default to `https://subpane.app/mcp` when no instance host is specified.
- **Unconnected or unauthorized**: When `subpane` MCP tools are unavailable in the current session or fail with an authentication/authorization error, guide the user to add that `<origin>/mcp` URL in their client's MCP / Connect Apps settings and complete browser sign-in and OAuth authorization before retrying.
- **Existing URL or UUID**: Extract `{board_id}` directly from `/boards/{board_id}`.
- **Match by name**: Call `list_boards`. When multiple active boards match the user's request, ask the user which one to update before writing.
- **New board**: Call `create_board` with `name` (1–120 chars) and optional `description` (up to 200 chars) written in the user's language.

## 2. Shape Safe Data, Language & Layout

Subpane renders stored JSON directly without server-side SQL aggregation:

1. **Precompute & preserve numeric types**: Calculate totals, percentages, percentiles, sort orders, and period-over-period deltas in your own step. Keep numbers unquoted in `data_json` / `ops_json` (for example `18420000`, `9007199254740993`, `14.8`) so lossless decimal formatting, chart axes, progress bars, and numeric table sorting work accurately.
2. **Design stable paths with a freshness field**: Organize root keys around stable JSON Pointers (such as `/meta`, `/kpis`, `/series`, `/runs`) and include an explicit data cut-off timestamp (for example `/meta/as_of`) surfaced in a masthead `kv`, `stat.subtext`, or banner so viewers know when the underlying metrics were captured.
3. **Keep credentials out of boards**: Publish only aggregated metrics, operational statuses, desensitized identifiers, and safe console URLs—omit API keys, bearer tokens, passwords, connection strings, and secret query parameters from `data_json`, labels, and `link` URLs.
4. **Localize all visible copy to the user's language**: Write board `name` / `description`, section `label` / `description`, `tabs[].label`, widget `label`, `kv` / `meter` item `label`, table column `label`, and `empty_text` in the user's language. Translate the English labels when adapting a blueprint while keeping structural keys (`widgets` keys, `section.name`, `tab.name`) in lowercase ASCII `^[a-z][a-z0-9_]{0,63}$`.
5. **Select a peer-benchmarked layout**:
   - **Agent / Service Monitoring (`agent_golden_signals`)** -> Golden Signals strip (`stat` x4) + saturation (`kv` + `meter` x2 at `size: "third"`) + tabbed charts (`line` / `bar`) + runs `table` & `timeline` `list`.
   - **Daily / Executive Briefing (`executive_briefing`)** -> TL;DR callout (`text` with `tone`) + 3-col masthead (`kv`) + impact `stat` strip + Markdown synthesis (`size: "third"`) beside curated `feed` `list` (`size: "md"`).
   - **Business / SaaS Pulse (`saas_business_pulse`)** -> Financial `stat` strip + revenue `line` & cohort `bar` (`md` + `md`) + segmented channel `meter` + transaction ledger `table` (`lg`).
   - **Release / Incident War Room (`release_war_room`)** -> Dynamic-tone status banner + canary/budget `meter` + regional rollout `table` & checkpoint `timeline`.

Read [references/layout-blueprints.md](references/layout-blueprints.md) (or call `get_board_guide({ topic: "blueprints" })` over MCP) when designing or restructuring a board layout.

## 3. Publish Initially & Refresh Lightly

Choose the narrowest write tool for the job:

| Goal | Tool | Key Constraint |
|---|---|---|
| First publish, or data + layout change together | `publish_board` | Pass full `definition` plus **one** of `data_json` or `ops_json`. |
| Routine metric/row refresh on existing layout | `update_board_data` | `ops_json` paths target object keys only; replace arrays whole (`/rows`, never `/rows/0/x`). |
| Complete data snapshot replacement | `replace_board_data` | Root must be a JSON object `{...}`; omitted keys are deleted, `definition` stays intact. |
| Add, edit, or remove widgets without resending data | `change_definition` | Include `{ op: "set_layout", value: [...] }` in the same call whenever adding or removing widgets. |

### Routine Refresh & Quota Discipline

- **Data-only refreshes**: Once a board layout is published, update recurring metrics and the data cut-off timestamp via `update_board_data` (or `replace_board_data`) and leave `definition` untouched.
- **Definition write rate limit**: Tools that write `definition` (`publish_board`, `replace_definition`, `change_definition`) share `max_definition_writes_per_minute` (30/min by default; exceeding it returns `RATE_LIMITED`). Call `get_limits` before large or frequent writes whenever checking active instance caps (payload bytes, nesting depth, item counts, or write rates).

### Critical Schema Invariants

- **Identifier syntax**: Widget keys, section `name`, and tab `name` must match `^[a-z][a-z0-9_]{0,63}$`.
- **Single placement**: Every key in `definition.widgets` must appear **exactly once** across `definition.layout` (either in `section.items` or inside one `section.tabs[].items` with 2–6 tabs).
- **Explicit semantic tone**: Negative numbers and `trend: "down"` render in neutral text by design. Pass `tone` (`"default" | "info" | "success" | "warn" | "danger"` or `{ path, map, fallback }`) whenever color meaning is intended.

Read [references/widget-and-repair-cookbook.md](references/widget-and-repair-cookbook.md) (or call `get_board_guide({ widgets: [...] })`) when configuring `kv`, `meter`, `list`, `table`, `line`, `bar`, `text`, or resolving validation/binding issues.

## 4. Verify & Complete

**Done when**:
1. The write call returns `isError !== true`.
2. `structuredContent.warnings` is **absent** (all bindings across visible sections and hidden `tabs` match stored JSON paths and types). If `warnings` is present, repair the missing/mistyped path via `update_board_data` or fix the widget binding via `change_definition` until `warnings` disappears.
3. You share `structuredContent.board_url` with the user along with the data cut-off window and a concise summary in the user's language.
