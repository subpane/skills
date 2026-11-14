# Widget Schema Rules & Repair Cookbook

All Subpane schemas enforce `strictObject` (unknown fields fail immediately with `VALIDATION_FAILED`). When connected to a server with newer schemas or custom quotas, treat `get_board_guide` and `get_limits` as authoritative.

## 1. Shared Primitives

- **JSON Pointer**: Starts with `/` (for example `/summary/revenue`). Escape `~` as `~0` and `/` as `~1`.
- **`NumberFormat`**: `{ digits?: 0..12, prefix?: string, suffix?: string, grouping?: boolean, notation?: "standard" | "compact" }` (affixes up to 200 chars).
  - `notation: "compact"` formats `18420000` -> `18.4M` (or `999950` -> `1M` with cross-tier rounding) and places the unit before `suffix`.
- **`ToneInput`**:
  - Static string: `"default" | "info" | "success" | "warn" | "danger"`
  - Dynamic mapping (up to 32 entries): `{ path: "/status", map?: { ok: "success", warn: "warn", fail: "danger" }, fallback?: "default" }` (resolves from root on root-bound widgets, or from the current row on `source`-bound widgets).
- **`DisplayValue`**:
  - Non-empty string `"literal"`
  - `{ path: "/field", format?: NumberFormat, link?: Link }`
  - Fragment array (1–20 items): `["Workers: ", { path: "/online", format: { grouping: true } }]`
- **`Link`** (up to 2048 UTF-8 bytes; never embed secret tokens or credentials in URLs):
  - `{ kind: "static", url: "https://..." }` (required for widget header `link`)
  - `{ kind: "path", path: "/url" }` (allowed on `stat.value_link`, `kv` items, `table` columns, `list` items/fields, and `DisplayValue` path objects; unsafe runtime protocols fall back safely to plain text).

## 2. Mutually Exclusive Widget Branches

| Widget | Branch Rules & Gotchas |
|---|---|
| `stat` | Root `path` required. If `format` is set, target JSON value must be a `number` (otherwise `string` or `number`). Optional `badge` (literal `text` max 64 chars), `delta`, `subtext`, `value_link`. |
| `kv` | Fixed `items` (1–24), optional `columns` (`1 \| 2 \| 3`). Each item must use **either** `path` (+ optional `format`) **or** `value: DisplayValue`—never both on the same item. `cell`: `"auto" \| "badge" \| "code"`. |
| `meter` | `variant`: `"progress"` (default, 1–24 rows) or `"segmented"` (1–8 segments). Choose **either** fixed `items` **or** array `source` + `fields`. In fixed `items`, path refs are objects (`{ path: "/used" }`); in `source` `fields`, paths are strings (`value: "/used"`). `segmented` items only allow `label`, non-negative `value`, and `format` (omit `max`, `tone`, `subtext`). |
| `list` | `variant`: `"plain"` (default), `"feed"`, or `"timeline"`. `marker` (`"bullet" \| "number" \| "none"`) is valid **only** when `variant` is `"plain"` or omitted. Choose **either** fixed `items` (1–100) **or** `source`. With `source`, pass at most **one** of `item` or `fields` (omit both for scalar arrays). |
| `table` | Requires `source` and `fields.columns` (1–50 unique `path`s). `cell`: `"auto" \| "badge" \| "progress" \| "code"`. Pass `max` **only** when `cell: "progress"`. Set `sortable: true` for interactive lossless numeric sorting. |
| `line` | Requires `source` (up to 1,000 points) and `fields: { x, series }` (1–12 unique series `path`s). `x` must be non-null string/number; `series` values must be numbers (or `null` gaps). Optional `area: true`, `y_format`, `reference_lines` (1–4). |
| `bar` | Requires `source` and `fields: { x, series }` (1–12 unique series `path`s). Optional `stacked: true`, `orientation: "vertical" \| "horizontal"`, `y_format`. |
| `text` | Requires `content: DisplayValue` (up to 20 fragments). Optional `markdown: true` (safe headings, lists, emphasis, code, `http(s)` links) and `tone: ToneInput` (callout banner). |

## 3. Resolving Warnings & Errors

### `structuredContent.warnings` (`isError: false`)

Writes still commit when `warnings` appear, and Subpane scans **all** sections including inactive `section.tabs`. Always clear warnings before finishing:

- `code: "missing"` -> Add the missing field via `update_board_data` (`op: "set"`) or fix the typo in the widget's `path` / `source` via `change_definition`.
- `code: "type_mismatch"` -> Convert stringified numbers (`"42"` -> `42`), replace `null` on chart `x` labels with a string/number, or remove `format` on a `stat` intended to show strings. Note: an empty array `[]` on `source` is valid and renders `empty_text` with zero warnings.
- `warnings_truncated: true` -> More than 12 warnings (or >2 KiB) exist; fix the reported batch and inspect the next write response until `warnings` is absent.

### `isError: true` (`structuredContent.error`)

- `VALIDATION_FAILED` + `reason: "widget_not_in_layout"`: Every key in `widgets` must appear once in `layout[]`. When adding/removing widgets in `change_definition`, include `{ op: "set_layout", value: [...] }` in the same `changes` array.
- `VALIDATION_FAILED` + `reason: "widget_missing"`: `change_definition` tried to `remove` a widget name not present in `definition.widgets`; inspect `get_definition` first.
- `VALIDATION_FAILED` + `reason: "url_must_be_absolute_http"`: A static `link.url` is not a valid `http://` or `https://` URL without credentials.
- `VALIDATION_FAILED` + `reason: "path_parent_not_object"`: `ops_json` attempted to index into an array (like `/daily/0/count`) or traverse through a scalar. Replace the parent array whole (`/daily`).
- `VALIDATION_FAILED` + `reason: "path_missing"`: `ops_json` tried to `remove` a key (or traverse a parent path on `remove`) that does not exist in stored data; inspect `get_board_data` first or omit the `remove`.
- `VALIDATION_FAILED` + `reason: "expected_object" | "invalid_operation" | "missing_value"`: Ensure `data_json` encodes a root JSON object `{...}` and every `ops_json` entry is `{ "op": "set", "path": "/...", "value": ... }` or `{ "op": "remove", "path": "/..." }`.
- `INVALID_JSON`: `data_json` or `ops_json` is malformed JSON or contains duplicate keys in an object.
- `PAYLOAD_TOO_LARGE` / `CAPACITY_EXCEEDED`: Exceeded `error.details.limit`; call `get_limits` and aggregate/trim rows or reuse an active board via `list_boards`.
- `RATE_LIMITED`: Exceeded `max_definition_writes_per_minute` (30/min by default); send routine metric refreshes via `update_board_data` without rewriting `definition`.
- `BOARD_ARCHIVED`: Board is read-only until restored by the user in Web UI; ask the user to restore it or create/select an active board.
