---
name: subpane
description: Publish and maintain live Subpane dashboards over MCP. Use when the user wants to create or update a dashboard, status board, monitor, briefing, or KPI report in Subpane, or shares a Subpane /boards/{board_id} link.
---

# Subpane

Subpane hosts live dashboards that the user opens at `board_url`. A board stores your `data_json` (a JSON object) and a `definition`: widgets bound to that data by JSON Pointer, arranged in a layout. Values render as stored, so compute totals, rates, and deltas yourself.

The connected server's tool schemas, `get_board_guide`, and `get_limits` are authoritative; follow them where they differ from this skill.

## Connect

- The MCP endpoint is `https://<host>/mcp` for the host in the user's board link, otherwise `https://subpane.app/mcp`.
- If Subpane tools are missing or fail authorization, help the user add that URL in their client's MCP or Connect Apps settings and sign in, then retry.

## Workflow

1. Target: use the board_id the user gave or the UUID in a `/boards/<id>` URL. Otherwise find it with `list_boards` (ask if several match) or call `create_board`.
2. First publish: call `publish_board` with `definition` and `data_json` together. Start from a template in [layout-blueprints.md](references/layout-blueprints.md) or the widget examples in [widget-and-repair-cookbook.md](references/widget-and-repair-cookbook.md).
3. Later changes: use the narrowest write tool (see below). On a board you did not build in this session, call `get_definition` first.
4. Done when: the write returns no `warnings` and you have shared `board_url` with the user, noting when the data was captured.

## Content rules

- Localize all visible copy to the user's language: board name (up to 120 chars) and description (up to 200), section, tab, widget, item, and column labels, and `empty_text`. Widget, section, and tab names stay lowercase ASCII matching `^[a-z][a-z0-9_]{0,63}$`.
- Write numbers as JSON numbers (`18420000`, `14.8`), not strings, so formats, charts, and sorting work. Large integers and exact decimals keep full precision.
- Show when the data was captured, e.g. a `/meta/as_of` field in a `kv` or `stat` subtext.
- Keep API keys, bearer tokens, passwords, connection strings, and secret query parameters out of data, labels, and URLs.

## Write tools

| Goal | Tool | Note |
|---|---|---|
| First publish, or a change to both widgets and data | `publish_board` | `definition` plus one of `data_json` or `ops_json`. |
| Routine refresh of specific values | `update_board_data` | Object paths only; set arrays whole (`/rows`, not `/rows/0/x`). |
| Replace all data | `replace_board_data` | Omitted keys are deleted; widgets are kept. |
| Add, edit, or remove widgets | `change_definition` | When adding or removing, include `set_layout` in the same call. |
| Replace all widgets and the layout | `replace_definition` | Data is kept. |

Definition writes (`publish_board`, `replace_definition`, `change_definition`) share `max_definition_writes_per_minute` (default 30), so keep routine refreshes data-only. `get_limits` reports this instance's caps.

## Layout

- Each widget appears exactly once in `layout`: in a section's `items`, or in one of its `tabs[].items` (2-6 tabs).
- Put 3-4 stat cards (or one text alert) on top, trends and composition in the middle, and detailed tables or timelines at the bottom.
- Without `tone`, negative numbers and `trend: "down"` render neutral; set `tone` when color should carry meaning.

Templates (full JSON in [layout-blueprints.md](references/layout-blueprints.md), or `get_board_guide({ blueprint })`):

- `agent_golden_signals`: Agent & Service Golden Signals. Autonomous agent fleets, background workers, API gateways, and LLM token/latency monitoring.
- `executive_briefing`: Executive & Daily Intelligence Briefing. Morning news/research digests, competitive intelligence, engineering standups, and PR/issue triage.
- `saas_business_pulse`: Personal Business & SaaS Revenue Pulse. Indie SaaS revenue, e-commerce orders, subscription retention, and personal finance tracking.
- `release_war_room`: Release Rollout & Incident War Room. Canary rollouts, database migrations, live incident response, and SLA burn tracking.

## Warnings and errors

- `warnings` means the write was saved but a binding does not match the data. Fix the data with `update_board_data` or the binding with `change_definition`, and repeat until none remain.
- On `isError: true`, read `error.details`. [widget-and-repair-cookbook.md](references/widget-and-repair-cookbook.md) lists each warning and error code with its fix.
