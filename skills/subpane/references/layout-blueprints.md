# Layout Blueprints

Four complete boards to adapt. Each block pairs a valid `definition` with matching `sample_data`. If the server's `get_board_guide({ topic: "layout" })` or `get_board_guide({ topic: "blueprints" })` differs, follow the server.

## Layout rules

- Each widget appears exactly once in `layout`: in a section's `items`, or in one of its `tabs[].items`.
- Section names, and tab names within a section, are unique identifiers.
- A section has either `items: [{ widget, size? }]` or `tabs: [{ name, label, items }]` (2-6 tabs, switched without reloading). Every section and tab holds at least one widget.
- Sizes on the 12-column desktop grid: `sm` = 3, `third` = 4, `md` = 6, `lg` = 12. `auto` (default) uses the widget's default size: stat `sm`, table `lg`, others `md`. Narrower screens reflow automatically (8 columns at 560-899px, 2 below).
- Stat cards never share a row with content widgets and split into balanced rows (5 become 3 + 2). In other rows, leftover columns go to the widget with the largest base span (`third` + `md` becomes 4 + 8), and cards in a row share one height.

## Design principles

- Put 3-4 stat cards (or one text alert) on top, trends and composition in the middle, and detailed tables or timelines at the bottom.
- Group related charts into one tabbed section rather than stacking more than four sections.
- Use `third` x3 for small cards, `md` x2 for side-by-side charts, and `lg` for wide tables.
- Templates use English copy: translate labels and `empty_text`, keep identifiers.

---

## Blueprint 1: Agent & Service Golden Signals (`agent_golden_signals`)

Best for: Autonomous agent fleets, background workers, API gateways, and LLM token/latency monitoring.

- Row 1 (4 `stat` cards): service state, throughput, error rate, and P95 latency.
- Row 2 (3x `size:'third'`): runtime details (`kv`), quota and budget usage (`meter`, progress), and token mix (`meter`, segmented).
- Row 3 (`section.tabs`): latency over time (`line` with an SLA reference line) and per-tool latency (`bar`, horizontal, stacked).
- Row 4 (`md` + `md`): sortable runs (`table` with `code`, `badge`, `progress` cells) beside an alert log (`list` timeline).

```json
{
  "blueprint": "agent_golden_signals",
  "definition": {
    "widgets": {
      "health_state": {
        "type": "stat",
        "label": "Service State",
        "path": "/kpis/state",
        "badge": {
          "text": { "path": "/kpis/env" },
          "tone": { "path": "/kpis/health", "map": { "healthy": "success", "degraded": "warn", "down": "danger" } }
        },
        "subtext": ["Workers online: ", { "path": "/kpis/workers_online" }, "/", { "path": "/kpis/workers_total" }]
      },
      "throughput": {
        "type": "stat",
        "label": "Throughput",
        "path": "/kpis/rpm",
        "format": { "grouping": true, "suffix": " rpm" },
        "delta": {
          "value": { "path": "/kpis/rpm_delta", "format": { "digits": 1, "prefix": "+", "suffix": "%" } },
          "trend": "up",
          "tone": "info"
        }
      },
      "error_rate": {
        "type": "stat",
        "label": "Error Rate",
        "path": "/kpis/error_pct",
        "format": { "digits": 2, "suffix": "%" },
        "badge": {
          "text": "SLO < 1%",
          "tone": { "path": "/kpis/error_level", "map": { "ok": "success", "warn": "warn", "breach": "danger" } }
        }
      },
      "p95_latency": {
        "type": "stat",
        "label": "P95 Latency",
        "path": "/kpis/p95_ms",
        "format": { "grouping": true, "suffix": " ms" },
        "subtext": "Target <= 1,500 ms"
      },
      "runtime_profile": {
        "type": "kv",
        "label": "Coordinator Profile",
        "columns": 2,
        "items": [
          { "label": "Instance", "path": "/profile/id", "cell": "code" },
          { "label": "Model", "path": "/profile/model", "cell": "badge", "tone": "info" },
          { "label": "Heartbeat", "path": "/profile/heartbeat" },
          { "label": "Trace", "value": "Open Console", "link": { "kind": "path", "path": "/profile/trace_url" } }
        ]
      },
      "quota_bars": {
        "type": "meter",
        "label": "Saturation & Quotas",
        "variant": "progress",
        "items": [
          {
            "label": "Concurrency",
            "value": { "path": "/quotas/active" },
            "max": { "path": "/quotas/limit" },
            "tone": "info"
          },
          {
            "label": "Daily Token Budget",
            "value": { "path": "/quotas/budget_pct" },
            "max": 100,
            "format": { "digits": 1, "suffix": "%" },
            "tone": "warn"
          }
        ]
      },
      "token_mix": {
        "type": "meter",
        "label": "Token Composition",
        "variant": "segmented",
        "format": { "notation": "compact" },
        "items": [
          { "label": "Uncached Input", "value": { "path": "/tokens/input" } },
          { "label": "Cache Hit", "value": { "path": "/tokens/cached" } },
          { "label": "Output", "value": { "path": "/tokens/output" } }
        ]
      },
      "latency_trend": {
        "type": "line",
        "label": "Latency Percentiles",
        "area": true,
        "y_format": { "grouping": true, "suffix": " ms" },
        "reference_lines": [{ "value": 1500, "label": "SLA 1.5s", "tone": "warn" }],
        "source": "/series",
        "fields": { "x": "/slot", "series": [{ "path": "/p50", "label": "P50" }, { "path": "/p95", "label": "P95" }] }
      },
      "tool_breakdown": {
        "type": "bar",
        "label": "Slowest Tools",
        "orientation": "horizontal",
        "stacked": true,
        "y_format": { "grouping": true, "suffix": " ms" },
        "source": "/tools",
        "fields": { "x": "/tool", "series": [{ "path": "/compute_ms", "label": "Compute" }, { "path": "/io_ms", "label": "I/O" }] }
      },
      "recent_runs": {
        "type": "table",
        "label": "Recent Runs",
        "source": "/runs",
        "empty_text": "No runs in window",
        "fields": {
          "columns": [
            { "path": "/id", "label": "Run", "cell": "code", "link": { "kind": "path", "path": "/url" } },
            {
              "path": "/status",
              "label": "Status",
              "cell": "badge",
              "tone": { "path": "/status", "map": { "succeeded": "success", "running": "info", "failed": "danger" } }
            },
            {
              "path": "/pct",
              "label": "Progress",
              "cell": "progress",
              "max": 100,
              "format": { "digits": 0, "suffix": "%" },
              "sortable": true
            },
            { "path": "/tokens", "label": "Tokens", "format": { "notation": "compact" }, "sortable": true }
          ]
        }
      },
      "event_log": {
        "type": "list",
        "variant": "timeline",
        "label": "Dispatch & Alert Log",
        "source": "/events",
        "empty_text": "No anomalies recorded",
        "fields": {
          "title": "/title",
          "meta": "/time",
          "badge": "/tag",
          "description": "/detail",
          "tone": { "path": "/level", "map": { "ok": "success", "warn": "warn", "error": "danger" }, "fallback": "default" }
        }
      }
    },
    "layout": [
      {
        "name": "golden_signals",
        "label": "Golden Signals",
        "description": "Rolling 15-minute window · refreshed every 30s",
        "items": [{ "widget": "health_state" }, { "widget": "throughput" }, { "widget": "error_rate" }, { "widget": "p95_latency" }]
      },
      {
        "name": "saturation",
        "label": "Runtime & Capacity",
        "items": [
          { "widget": "runtime_profile", "size": "third" },
          { "widget": "quota_bars", "size": "third" },
          { "widget": "token_mix", "size": "third" }
        ]
      },
      {
        "name": "telemetry",
        "label": "Performance Telemetry",
        "tabs": [
          { "name": "latency", "label": "Latency Trend", "items": [{ "widget": "latency_trend", "size": "lg" }] },
          { "name": "tools", "label": "Tool Breakdown", "items": [{ "widget": "tool_breakdown", "size": "lg" }] }
        ]
      },
      {
        "name": "drilldown",
        "label": "Runs & Events",
        "items": [{ "widget": "recent_runs", "size": "md" }, { "widget": "event_log", "size": "md" }]
      }
    ]
  },
  "sample_data": {
    "kpis": {
      "state": "Operational",
      "env": "PROD",
      "health": "healthy",
      "workers_online": 12,
      "workers_total": 12,
      "rpm": 1840,
      "rpm_delta": 6.4,
      "error_pct": 0.18,
      "error_level": "ok",
      "p95_ms": 1240
    },
    "profile": {
      "id": "agent-worker-us1",
      "model": "frontier-v4",
      "heartbeat": "14:30 UTC",
      "trace_url": "https://trace.example.com/agent-worker-us1"
    },
    "quotas": { "active": 38, "limit": 64, "budget_pct": 62.5 },
    "tokens": { "input": 6400000, "cached": 11200000, "output": 1950000 },
    "series": [{ "slot": "14:00", "p50": 380, "p95": 1120 }, { "slot": "14:15", "p50": 410, "p95": 1240 }],
    "tools": [
      { "tool": "search.index", "compute_ms": 190, "io_ms": 620 },
      { "tool": "sql.query", "compute_ms": 85, "io_ms": 310 }
    ],
    "runs": [
      {
        "id": "run_401",
        "status": "succeeded",
        "pct": 100,
        "tokens": 84200,
        "url": "https://trace.example.com/runs/401"
      },
      {
        "id": "run_402",
        "status": "running",
        "pct": 60,
        "tokens": 41000,
        "url": "https://trace.example.com/runs/402"
      }
    ],
    "events": [
      {
        "time": "14:22",
        "title": "Autoscaled worker pool +2",
        "tag": "SCALE",
        "level": "ok",
        "detail": "Queue wait stayed below 200 ms."
      }
    ]
  }
}
```

---

## Blueprint 2: Executive & Daily Intelligence Briefing (`executive_briefing`)

Best for: Morning news/research digests, competitive intelligence, engineering standups, and PR/issue triage.

- Row 1 (`lg` + `lg`): a one-sentence summary banner (`text` with `tone`) above 3-column edition details (`kv`).
- Row 2 (3 `stat` cards): high-priority alerts, shipped milestones, and open follow-ups.
- Row 3 (`third` + `md`): Markdown takeaways (`text`) beside a curated feed (`list`, `variant:'feed'`).
- Row 4 (`section.tabs`): topic counts (`bar`, horizontal) and a source index (`table`).

```json
{
  "blueprint": "executive_briefing",
  "definition": {
    "widgets": {
      "tldr_banner": { "type": "text", "tone": "info", "markdown": true, "content": ["**TL;DR**: ", { "path": "/meta/headline" }] },
      "masthead": {
        "type": "kv",
        "columns": 3,
        "items": [
          { "label": "Edition", "path": "/meta/edition", "cell": "code" },
          { "label": "Sources Scanned", "path": "/meta/sources", "format": { "grouping": true } },
          { "label": "Curator", "path": "/meta/curator", "cell": "badge", "tone": "default" }
        ]
      },
      "high_priority": {
        "type": "stat",
        "label": "Action Required",
        "path": "/stats/action_required",
        "badge": { "text": "P0/P1", "tone": "warn" }
      },
      "shipped_count": {
        "type": "stat",
        "label": "Key Releases",
        "path": "/stats/releases",
        "delta": { "value": "+3 vs yesterday", "trend": "up", "tone": "success" }
      },
      "saved_hours": { "type": "stat", "label": "Signal Ratio", "path": "/stats/signal_pct", "format": { "digits": 1, "suffix": "%" } },
      "synthesis": { "type": "text", "label": "Key Takeaways", "markdown": true, "content": { "path": "/digest/markdown" } },
      "top_stories": {
        "type": "list",
        "variant": "feed",
        "label": "Curated Stories",
        "source": "/stories",
        "empty_text": "No stories in this edition",
        "fields": {
          "title": "/title",
          "meta": "/published",
          "badge": "/topic",
          "description": "/summary",
          "tone": { "path": "/impact", "map": { "high": "warn", "critical": "danger" }, "fallback": "info" },
          "link": { "kind": "path", "path": "/url" }
        }
      },
      "topic_volume": {
        "type": "bar",
        "label": "Mentions by Topic",
        "orientation": "horizontal",
        "source": "/topics",
        "fields": { "x": "/topic", "series": [{ "path": "/mentions", "label": "Mentions" }] }
      },
      "watchlist_table": {
        "type": "table",
        "label": "Tracked Signals",
        "source": "/signals",
        "fields": {
          "columns": [
            { "path": "/entity", "label": "Entity", "link": { "kind": "path", "path": "/url" } },
            { "path": "/category", "label": "Category", "cell": "badge", "tone": "info" },
            {
              "path": "/score",
              "label": "Relevance",
              "cell": "progress",
              "max": 100,
              "format": { "digits": 0, "suffix": "%" },
              "sortable": true
            }
          ]
        }
      }
    },
    "layout": [
      {
        "name": "masthead_section",
        "items": [{ "widget": "tldr_banner", "size": "lg" }, { "widget": "masthead", "size": "lg" }]
      },
      {
        "name": "pulse_kpis",
        "items": [{ "widget": "high_priority" }, { "widget": "shipped_count" }, { "widget": "saved_hours" }]
      },
      {
        "name": "narrative",
        "label": "Briefing & Feed",
        "items": [{ "widget": "synthesis", "size": "third" }, { "widget": "top_stories", "size": "md" }]
      },
      {
        "name": "radar",
        "label": "Topic Radar & Watchlist",
        "tabs": [
          { "name": "topics", "label": "Topic Volume", "items": [{ "widget": "topic_volume", "size": "lg" }] },
          { "name": "signals", "label": "Watchlist", "items": [{ "widget": "watchlist_table", "size": "lg" }] }
        ]
      }
    ]
  },
  "sample_data": {
    "meta": {
      "headline": "2 protocol releases and 1 upstream rate-limit change require review today.",
      "edition": "2026-11-14-AM",
      "sources": 42,
      "curator": "daily-curator"
    },
    "stats": { "action_required": 2, "releases": 7, "signal_pct": 16.7 },
    "digest": {
      "markdown": "### Focus Today\n- **MCP OAuth 2.1**: Verify connector scopes in production.\n- **Cost Watch**: Prompt caching cut morning batch spend by `34%`."
    },
    "stories": [
      {
        "title": "Agent Plugins 1.0 stabilizes portable skill manifests",
        "published": "08:40",
        "topic": "Ecosystem",
        "impact": "high",
        "summary": "Unified plugin.json and .mcp.json layout works across CLI and desktop agent hosts.",
        "url": "https://example.com/news/agent-plugins-1"
      }
    ],
    "topics": [{ "topic": "MCP & OAuth", "mentions": 18 }, { "topic": "Inference Latency", "mentions": 11 }],
    "signals": [{ "entity": "Subpane v2", "category": "Release", "score": 94, "url": "https://subpane.app" }]
  }
}
```

---

## Blueprint 3: Personal Business & SaaS Revenue Pulse (`saas_business_pulse`)

Best for: Indie SaaS revenue, e-commerce orders, subscription retention, and personal finance tracking.

- Row 1 (4 `stat` cards): net revenue, active subscribers, conversion rate, and average order value.
- Row 2 (`md` + `md`): daily revenue (`line` with area) beside new vs. renewal revenue (`bar`, stacked).
- Row 3 (`third` + `md`): acquisition channel split (`meter`, segmented) beside numbered follow-ups (`list`, plain).
- Row 4 (`lg` full-width `table`): recent transactions with invoice links, status badges, and sortable amounts.

```json
{
  "blueprint": "saas_business_pulse",
  "definition": {
    "widgets": {
      "net_revenue": {
        "type": "stat",
        "label": "Net Revenue (MTD)",
        "path": "/kpis/net_revenue",
        "format": { "prefix": "$", "grouping": true },
        "delta": {
          "value": { "path": "/kpis/rev_delta", "format": { "digits": 1, "prefix": "+", "suffix": "%" } },
          "trend": "up",
          "tone": "success"
        }
      },
      "active_customers": {
        "type": "stat",
        "label": "Active Customers",
        "path": "/kpis/customers",
        "format": { "grouping": true },
        "delta": { "value": "+19 net", "trend": "up", "tone": "success" }
      },
      "conversion_rate": {
        "type": "stat",
        "label": "Trial Conversion",
        "path": "/kpis/conversion_pct",
        "format": { "digits": 2, "suffix": "%" }
      },
      "arpu": { "type": "stat", "label": "Avg Order Value", "path": "/kpis/aov", "format": { "prefix": "$", "digits": 2 } },
      "revenue_curve": {
        "type": "line",
        "label": "Daily Net Revenue",
        "area": true,
        "y_format": { "prefix": "$", "grouping": true },
        "source": "/daily",
        "fields": {
          "x": "/date",
          "series": [{ "path": "/revenue", "label": "Net Revenue", "format": { "prefix": "$", "grouping": true } }]
        }
      },
      "cohort_bars": {
        "type": "bar",
        "label": "New vs Expansion",
        "stacked": true,
        "y_format": { "prefix": "$", "notation": "compact" },
        "source": "/daily",
        "fields": {
          "x": "/date",
          "series": [{ "path": "/new_mrr", "label": "New" }, { "path": "/expansion_mrr", "label": "Expansion" }]
        }
      },
      "channel_split": {
        "type": "meter",
        "label": "Revenue by Channel",
        "variant": "segmented",
        "format": { "prefix": "$", "grouping": true },
        "items": [
          { "label": "Organic Search", "value": { "path": "/channels/organic" } },
          { "label": "Referrals", "value": { "path": "/channels/referral" } },
          { "label": "Outbound", "value": { "path": "/channels/outbound" } }
        ]
      },
      "action_queue": {
        "type": "list",
        "variant": "plain",
        "marker": "number",
        "label": "Today's Follow-ups",
        "items": [
          ["Review enterprise annual quote for ", { "path": "/focus/top_account" }],
          "Follow up on 2 expiring card retries"
        ]
      },
      "recent_orders": {
        "type": "table",
        "label": "Recent Transactions",
        "source": "/orders",
        "empty_text": "No transactions yet",
        "fields": {
          "columns": [
            { "path": "/invoice", "label": "Invoice", "cell": "code", "link": { "kind": "path", "path": "/url" } },
            { "path": "/customer", "label": "Customer" },
            {
              "path": "/status",
              "label": "Status",
              "cell": "badge",
              "tone": { "path": "/status", "map": { "paid": "success", "pending": "warn", "refunded": "default" } }
            },
            {
              "path": "/amount",
              "label": "Amount",
              "format": { "prefix": "$", "digits": 2, "grouping": true },
              "sortable": true
            }
          ]
        }
      }
    },
    "layout": [
      {
        "name": "headline",
        "label": "Commercial Summary",
        "items": [{ "widget": "net_revenue" }, { "widget": "active_customers" }, { "widget": "conversion_rate" }, { "widget": "arpu" }]
      },
      {
        "name": "growth",
        "label": "Revenue Velocity",
        "items": [{ "widget": "revenue_curve", "size": "md" }, { "widget": "cohort_bars", "size": "md" }]
      },
      {
        "name": "attribution",
        "label": "Channels & Actions",
        "items": [{ "widget": "channel_split", "size": "third" }, { "widget": "action_queue", "size": "md" }]
      },
      { "name": "ledger", "label": "Order Ledger", "items": [{ "widget": "recent_orders", "size": "lg" }] }
    ]
  },
  "sample_data": {
    "kpis": { "net_revenue": 48250, "rev_delta": 14.8, "customers": 612, "conversion_pct": 4.65, "aov": 78.84 },
    "daily": [
      { "date": "2026-11-12", "revenue": 3100, "new_mrr": 2100, "expansion_mrr": 1000 },
      { "date": "2026-11-13", "revenue": 3650, "new_mrr": 2400, "expansion_mrr": 1250 }
    ],
    "channels": { "organic": 26400, "referral": 14200, "outbound": 7650 },
    "focus": { "top_account": "Acme Robotics" },
    "orders": [
      {
        "invoice": "INV-2048",
        "customer": "Acme Robotics",
        "status": "paid",
        "amount": 1200,
        "url": "https://billing.example.com/inv/2048"
      }
    ]
  }
}
```

---

## Blueprint 4: Release Rollout & Incident War Room (`release_war_room`)

Best for: Canary rollouts, database migrations, live incident response, and SLA burn tracking.

- Row 1 (`lg`): a severity banner (`text` with tone mapped from `/incident/severity`).
- Row 2 (3x `size:'third'`): rollout and error-budget bars (`meter`), release details (`kv`), and 5xx rate (`line` with a rollback threshold).
- Row 3 (`md` + `md`): regional rollout (`table` with `progress` cells) beside a mitigation log (`list` timeline).

```json
{
  "blueprint": "release_war_room",
  "definition": {
    "widgets": {
      "status_banner": {
        "type": "text",
        "markdown": true,
        "tone": {
          "path": "/incident/severity",
          "map": { "clear": "success", "watch": "warn", "critical": "danger" },
          "fallback": "info"
        },
        "content": ["**Rollout Status**: ", { "path": "/incident/summary" }]
      },
      "rollout_meters": {
        "type": "meter",
        "label": "Canary & Error Budget",
        "variant": "progress",
        "items": [
          {
            "label": "Traffic Shifted",
            "value": { "path": "/rollout/traffic_pct" },
            "max": 100,
            "format": { "digits": 0, "suffix": "%" },
            "tone": "info"
          },
          {
            "label": "Error Budget Remaining",
            "value": { "path": "/rollout/budget_remaining_pct" },
            "max": 100,
            "format": { "digits": 1, "suffix": "%" },
            "tone": "success"
          }
        ]
      },
      "release_kv": {
        "type": "kv",
        "label": "Release Metadata",
        "columns": 1,
        "items": [
          { "label": "Build SHA", "path": "/rollout/sha", "cell": "code" },
          { "label": "Commander", "path": "/rollout/owner", "cell": "badge", "tone": "info" },
          { "label": "Runbook", "value": "Open Runbook", "link": { "kind": "path", "path": "/rollout/runbook_url" } }
        ]
      },
      "error_curve": {
        "type": "line",
        "label": "5xx Count per Minute",
        "reference_lines": [{ "value": 25, "label": "Auto-Rollback", "tone": "danger" }],
        "source": "/errors_per_min",
        "fields": { "x": "/minute", "series": [{ "path": "/count", "label": "5xx" }] }
      },
      "region_table": {
        "type": "table",
        "label": "Regional Targets",
        "source": "/regions",
        "fields": {
          "columns": [
            { "path": "/region", "label": "Region", "cell": "code" },
            {
              "path": "/state",
              "label": "Health",
              "cell": "badge",
              "tone": { "path": "/state", "map": { "healthy": "success", "baking": "info", "halted": "danger" } }
            },
            {
              "path": "/shifted_pct",
              "label": "Traffic",
              "cell": "progress",
              "max": 100,
              "format": { "digits": 0, "suffix": "%" },
              "sortable": true
            }
          ]
        }
      },
      "mitigation_timeline": {
        "type": "list",
        "variant": "timeline",
        "label": "Checkpoint Timeline",
        "source": "/checkpoints",
        "fields": {
          "title": "/step",
          "meta": "/at",
          "badge": "/actor",
          "tone": { "path": "/tone", "map": { "ok": "success", "warn": "warn" }, "fallback": "info" }
        }
      }
    },
    "layout": [
      { "name": "banner", "items": [{ "widget": "status_banner", "size": "lg" }] },
      {
        "name": "vitals",
        "label": "Rollout Vitals",
        "items": [
          { "widget": "rollout_meters", "size": "third" },
          { "widget": "release_kv", "size": "third" },
          { "widget": "error_curve", "size": "third" }
        ]
      },
      {
        "name": "execution",
        "label": "Regions & Checkpoints",
        "items": [{ "widget": "region_table", "size": "md" }, { "widget": "mitigation_timeline", "size": "md" }]
      }
    ]
  },
  "sample_data": {
    "incident": { "severity": "clear", "summary": "v2.4.0 canary baking cleanly at 50% traffic with zero budget burn." },
    "rollout": {
      "traffic_pct": 50,
      "budget_remaining_pct": 99.2,
      "sha": "41b1231",
      "owner": "oncall-sre",
      "runbook_url": "https://runbooks.example.com/deploy-v2"
    },
    "errors_per_min": [{ "minute": "14:25", "count": 2 }, { "minute": "14:26", "count": 1 }],
    "regions": [
      { "region": "ap-east-1", "state": "healthy", "shifted_pct": 100 },
      { "region": "us-west-2", "state": "baking", "shifted_pct": 50 }
    ],
    "checkpoints": [{ "at": "14:20", "step": "Promoted ap-east-1 to 100%", "actor": "deploy-bot", "tone": "ok" }]
  }
}
```
