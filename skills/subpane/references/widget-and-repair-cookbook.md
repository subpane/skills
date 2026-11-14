# Widget Rules and Repair Cookbook

Widgets reject unknown fields with `VALIDATION_FAILED`. Where a rule offers alternatives (items or source, path or value), use exactly one. If the server's `get_board_guide` or `get_limits` differs, follow the server.

## 1. Shared primitives

- JSON Pointer: paths start with `/` (e.g. `/summary/revenue`); escape `~` as `~0` and `/` as `~1`. Scalar widgets use root paths; `source` widgets take a root array and row-relative field paths.
- `NumberFormat`: `{ digits?: 0-12, prefix?, suffix?, grouping?: boolean, notation?: "standard" | "compact" }`, affixes up to 200 chars. `compact` abbreviates to K/M/B/T (`18420000` shows as `18.4M`).
- `ToneInput`: a tone (`"default" | "info" | "success" | "warn" | "danger"`), or `{ path, map?, fallback? }` with up to 32 map entries to choose one from data. Paths resolve from the root, or from the current row on `source` widgets.
- `DisplayValue`: a non-empty string, `{ path, format?, link? }`, or up to 20 fragments such as `["Workers: ", { path: "/online", format: { grouping: true } }]`.
- `Link` (up to 2048 bytes, no credentials): `{ kind: "static", url: "https://..." }` or `{ kind: "path", path: "/url" }`. Widget header links are static only.

## 2. Widget rules

| Widget | Rules |
|---|---|
| `stat` | `path` reads a root value: a number when `format` is set, otherwise a string or number. Optional `badge` (literal text max 64 chars), `delta`, `subtext`, `value_link`. |
| `kv` | `items` (1–24), optional `columns` (1, 2, or 3). Each item has `label` plus either `path` (with optional `format`) or `value`, not both. `cell`: `"auto"`, `"badge"`, or `"code"`. |
| `meter` | `variant`: `"progress"` (default, up to 24 rows) or `"segmented"` (up to 8 segments). Fixed `items` or `source` + `fields`, not both. In `items`, paths are objects (`{ path: "/used" }`); in `fields`, strings (`value: "/used"`). Segmented items accept only `label`, `value`, and `format`. |
| `list` | `variant`: `"plain"` (default), `"feed"`, or `"timeline"`; `marker` applies to plain only. Fixed `items` (1–100) or `source`, not both. With `source`, use `fields` for object rows, `item` for one field per row, or neither for scalars. |
| `table` | `source` and `fields.columns` (1–50 unique `path`s). `cell`: `"auto"`, `"badge"`, `"progress"`, or `"code"`; `max` only with `"progress"`. `sortable: true` lets viewers sort; numbers sort numerically. |
| `line` | `source` (up to 1,000 points) and `fields: { x, series }` (1–12 unique series paths). `x` is a string or number; series values are numbers or `null` gaps. Optional `area`, `y_format`, `reference_lines` (1–4). |
| `bar` | `source` and `fields: { x, series }` (1–12 unique series paths). Optional `stacked`, `orientation` (`"vertical"` or `"horizontal"`), `y_format`. |
| `text` | `content: DisplayValue`. Optional `markdown: true` (headings, lists, emphasis, code, `http(s)` links) and `tone` (callout banner). |

## 3. Warnings and errors

### `structuredContent.warnings`

The write is saved, but a binding does not match the data (hidden tabs included). Each warning is `{ widget, path, code, expected, actual? }`. Fix and write again until none remain.

- `missing`: the path does not exist. Add the value with `update_board_data`, or fix the widget's `path` or `source` with `change_definition`.
- `type_mismatch`: the value has the wrong JSON type. Unquote numbers (`"42"` becomes `42`), give chart `x` a string or number, or drop `format` from a stat that shows text. An empty `source` array is valid and shows `empty_text`.
- `warnings_truncated: true`: more than 12 warnings (or 2 KiB) exist; fix these and check the next response.

### `isError: true` (`structuredContent.error`)

- `VALIDATION_FAILED` with `reason: "widget_not_in_layout"`: a widget is missing from `layout`. When `change_definition` adds or removes widgets, include `set_layout` in the same `changes`.
- `VALIDATION_FAILED` with `reason: "widget_missing"`: `change_definition` removed a widget that does not exist; check `get_definition`.
- `VALIDATION_FAILED` with `reason: "url_must_be_absolute_http"`: a static `link.url` is not an `http(s)` URL, or contains credentials.
- `VALIDATION_FAILED` with `reason: "path_parent_not_object"`: `ops_json` indexed an array (`/daily/0/count`) or passed through a scalar; set the parent array (`/daily`) whole.
- `VALIDATION_FAILED` with `reason: "path_missing"`: `remove` targeted a key that does not exist; check `get_board_data` or drop the op.
- `VALIDATION_FAILED` with `reason: "expected_object" | "invalid_operation" | "missing_value"`: `data_json` must encode an object, and each op must be `{ "op": "set", "path": "/...", "value": ... }` or `{ "op": "remove", "path": "/..." }`.
- `INVALID_JSON`: `data_json` or `ops_json` is malformed or has duplicate keys.
- `PAYLOAD_TOO_LARGE`: data exceeds the limit in `error.details.limit`; aggregate or trim rows (see `get_limits`).
- `CAPACITY_EXCEEDED`: the board limit is reached; reuse a board from `list_boards`.
- `RATE_LIMITED`: too many definition writes this minute (default 30); wait, and refresh values with `update_board_data` instead.
- `BOARD_ARCHIVED`: the board is read-only; ask the user to restore it, or use an active board.
