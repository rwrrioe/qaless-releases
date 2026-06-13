<div align="center">

# QAless

**AI-powered E2E test scenario generator for small dev teams without a dedicated QA engineer.**

Feed it a feature description, OpenAPI spec, or git diff — get back a prioritized graph of E2E scenarios with call-path traces, ready to export to Playwright, Postman, or Markdown. In 30 seconds instead of a day.

[![Release](https://img.shields.io/github/v/release/rwrrioe/qaless-releases?label=release&color=0ea5e9)](https://github.com/rwrrioe/qaless-releases/releases)
[![Platforms](https://img.shields.io/badge/platforms-macOS%20%C2%B7%20Windows-1f2937)](https://github.com/rwrrioe/qaless-releases/releases)
[![License](https://img.shields.io/badge/license-Proprietary%20%C2%B7%20%2420%2Fmo-22c55e)](https://qaless.lemonsqueezy.com)
[![MCP](https://img.shields.io/badge/MCP-Claude%20Desktop%20%C2%B7%20Cursor%20%C2%B7%20Zed-8b5cf6)](#wire-qaless-into-claude-desktop--cursor--zed)
[![Issues](https://img.shields.io/github/issues/rwrrioe/qaless-releases?color=64748b)](https://github.com/rwrrioe/qaless-releases/issues)

**[Landing →](https://qaless.vercel.app)** &nbsp; · &nbsp; **[Buy a license ($20/mo) →](https://qaless.lemonsqueezy.com)** &nbsp; · &nbsp; **[Releases →](https://github.com/rwrrioe/qaless-releases/releases)**

</div>

---

> **What QAless answers:** not *"how do I test this?"* but *"what can break here?"* — in seconds, grounded in your codebase, prioritized by impact, drill-able to the exact call path across your services.

```
  scenarios
  s11  token expired before use                       27   critical
▸ s12  token reused after password set                18   critical   ◀ open
  s13  concurrent reset under load                    18   critical

  selected  s12 · token reused after password set   p:3 c:3 r:2 = 18   critical

  ╭─ s12 · call path across services ────────────────────────────────────────╮
  │                                                                          │
  │   [client]                                                               │
  │       │  https post /auth/reset                                          │
  │       ▼                                                                  │
  │   [api-gateway · 2 of 2]                                                 │
  │       │  grpc resetpassword()                                            │
  │       ▼                                                                  │
  │   [auth-svc]                                                             │
  │       ├──▶ [postgres-primary]  select users where token=?                │
  │       │                                                                  │
  │       ▼  get token:{tok}                                                 │
  │   [redis-cache] ⚠  stale read · culprit on prod-412                      │
  │       │  post /notify                                                    │
  │       ▼                                                                  │
  │   [notification-svc · 0 of 2]                                            │
  ╰──────────────────────────────────────────────────────────────────────────╯
```

That's `qaless tui` against a real scenario. The frame above is derived from a declarative `qaless.services.yml` and (in v0.13.0) live observability signals from Sentry / Vercel / Grafana Loki / Datadog. See it animated at [qaless.vercel.app](https://qaless.vercel.app).

---

## Table of Contents

- [Why QAless](#why-qaless)
- [Quick start (60 seconds)](#quick-start-60-seconds)
- [Install](#install)
- [Configure](#configure)
- [Use it](#use-it)
- [Claude Desktop / Cursor / Zed integration](#wire-qaless-into-claude-desktop--cursor--zed)
- [Observability — Sentry / Vercel / Grafana / Datadog](#observability-v0130) `v0.13.0+`
- [Dashboard — local analytics](#dashboard-v0130) `v0.13.0+`
- [Production memory — incidents + feedback](#production-memory)
- [Service topology](#service-topology)
- [CI/CD integration](#wire-qaless-into-your-cicd)
- [Troubleshooting](#troubleshooting)
- [License + pricing](#license--pricing)
- [Versioning + changelog](#versioning--changelog)
- [Support](#support)

---

## Why QAless

| You have | You don't have | QAless gives you |
|---|---|---|
| A growing codebase shipping weekly | A dedicated QA engineer | A senior-QA-equivalent scenario list per feature, in seconds |
| Anthropic or Gemini API key already | A budget for a QA SaaS contract | A local-first CLI that bills your existing LLM provider directly |
| Claude Desktop / Cursor / Zed in your team | A GUI test-management tool | Native MCP integration — `"use qaless to analyze this"` works in any tool that speaks MCP |
| A real production incident history | A way to feed that history back into testing | An incident-replay benchmark + history-uplift weighting on related scenarios |
| Git, PRs, CI | A QA gate on every PR | A GitHub Action that comments scenarios on every PR |

QAless is **not**:

- Not a test runner — it generates scenarios; you decide where they execute (Playwright / Postman / staging via `qaless verify`).
- Not an OTel ingest — it reads observability signals (Sentry / Vercel / Loki / Datadog) but doesn't replace your observability stack. See [our anti-move stance](#why-not-otel).
- Not a multi-tenant SaaS — it's a standalone CLI binary. Your data stays local in `~/.qaless/`.

---

## Quick start (60 seconds)

```bash
# 1. Install (macOS Apple Silicon — full install table below)
sudo mkdir -p /usr/local/bin
curl -L https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.13.0_darwin_arm64.tar.gz \
  | sudo tar xz -C /usr/local/bin qaless

# 2. Configure (one-time)
qaless config set ANTHROPIC_API_KEY sk-ant-...
qaless config set LICENSE_KEY demo         # or your $20/mo key from qaless.lemonsqueezy.com

# 3. Analyze something
qaless analyze --input "Users can reset password via email link"
qaless tui                                  # interactive drill-in
```

That's the entire flow. Everything else in this README is depth on the parts you already saw above.

---

## Install

### macOS · Apple Silicon (arm64)

```bash
sudo mkdir -p /usr/local/bin
curl -L https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.13.0_darwin_arm64.tar.gz \
  | sudo tar xz -C /usr/local/bin qaless
qaless --version
```

The trailing `qaless` after `tar xz` filters the archive so only the binary lands in `/usr/local/bin` — the bundled `LICENSE` and `README.md` stay out of your PATH.

> **Heads-up:** fresh Apple Silicon Macs without Homebrew don't have `/usr/local/bin`. The `sudo mkdir -p` line above creates it; skip that line and `tar` errors with `could not chdir`.

> **Replacing an older install:** previous QAless installs may belong to a different uid. If `cp` over `/usr/local/bin/qaless` fails, run `sudo rm /usr/local/bin/qaless` first, then `sudo install -m 0755 -o "$(id -u)" -g "$(id -g)" qaless /usr/local/bin/`.

### Windows 11 · amd64

In PowerShell:

```powershell
Invoke-WebRequest "https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.13.0_windows_amd64.zip" -OutFile qaless.zip
Expand-Archive qaless.zip C:\qaless -Force
$env:Path += ";C:\qaless"
qaless --version
```

> **Persistent PATH:** the `$env:Path` line above only updates the current session. For permanent, use `setx PATH "$env:Path"` or add `C:\qaless` via System Properties → Environment Variables.

### Linux · Intel macOS

Not yet shipped. Open an issue at [rwrrioe/qaless-releases/issues](https://github.com/rwrrioe/qaless-releases/issues) if you need one — most-requested platform ships first.

### Always-latest install (any version)

GitHub's `/releases/latest/download/` redirects to the newest tag. To always grab the current Apple Silicon binary regardless of version:

```bash
LATEST=$(curl -s https://api.github.com/repos/rwrrioe/qaless-releases/releases/latest | grep tag_name | cut -d '"' -f 4)
curl -L "https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_${LATEST#v}_darwin_arm64.tar.gz" \
  | sudo tar xz -C /usr/local/bin qaless
```

### From source

The source is in a private repo. If you have access, `make build` produces the binary; `make install` puts it in `$GOPATH/bin`. Otherwise, use the pre-built binaries above.

---

## Configure

QAless needs two things to run gated commands: an **LLM API key** (you pay your LLM provider directly for tokens) and a **QAless license key** ($20/mo via LemonSqueezy).

```bash
# 1. LLM key — Anthropic (default)
qaless config set ANTHROPIC_API_KEY sk-ant-...

# Or use Google Gemini:
qaless config set QALESS_LLM_PROVIDER gemini
qaless config set GEMINI_API_KEY AIza...

# 2. QAless license — buy at https://qaless.lemonsqueezy.com ($20/mo)
qaless config set LICENSE_KEY lic_xxxxxxxxxxxx

# 3. Verify
qaless config show         # secrets auto-redacted to lic_xxxx****
qaless license status      # cached license state + remaining grace
```

### Demo key (payment-free trial)

```bash
qaless config set LICENSE_KEY demo
```

Short-circuits the LemonSqueezy HTTP call and returns a synthetic valid status. Prints a one-line stderr warning on first use (*"warn: using demo license key — not for production use"*) so you notice it's not a real subscription. Use for meetup demos, CI smoke tests, concierge trials.

### Dev bypass

```bash
QALESS_DEV=1 qaless analyze --input "..."
```

Skips the license gate entirely. Intended for local development against the source. Same env var registers the internal `qaless bench` command.

### File locations

| What | Path | Notes |
|---|---|---|
| Config | `~/.qaless/config.yaml` | Lookup order: CLI flag → env → file → default |
| License cache | `~/.qaless/license.json` | 7-day offline grace, atomic write |
| History | `~/.qaless/history.db` | SQLite — every analyze writes here |
| Observability cache `v0.13.0+` | `~/.qaless/observability-cache.json` | 1h freshness window |

### Environment variables

| Variable | Required? | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | one of LLM keys | Anthropic API access |
| `GEMINI_API_KEY` | one of LLM keys | Google Gemini API access |
| `QALESS_LLM_PROVIDER` | optional | `anthropic` (default) · `gemini` · `mock` |
| `QALESS_MODEL` | optional | Override default model (e.g. `claude-sonnet-4-6`) |
| `LICENSE_KEY` | yes (or demo) | QAless license — buy at qaless.lemonsqueezy.com |
| `QALESS_DEV` | optional | `=1` bypasses the license gate (dev only) |
| `QALESS_HISTORY_DB` | optional | Override SQLite path |
| `LOG_LEVEL` | optional | `debug` · `info` (default) · `warn` · `error` |
| `SENTRY_AUTH_TOKEN` `v0.13.0+` | optional | Sentry PAT with `project:read` scope |
| `SENTRY_ORG` `v0.13.0+` | optional | Sentry organization slug |
| `VERCEL_TOKEN` `v0.13.0+` | optional | Vercel account/team PAT |
| `VERCEL_TEAM_SLUG` `v0.13.0+` | optional | Team-scoped token only |
| `GRAFANA_LOKI_ENDPOINT` `v0.13.0+` | optional | e.g. `https://logs-prod-us-central1.grafana.net` |
| `GRAFANA_LOKI_INSTANCE_ID` `v0.13.0+` | optional | Numeric Loki instance ID |
| `GRAFANA_LOKI_TOKEN` `v0.13.0+` | optional | Cloud Access Policy token with `logs:read` |
| `DATADOG_API_KEY` `v0.13.0+` | optional | Datadog API key |
| `DATADOG_APP_KEY` `v0.13.0+` | optional | Datadog application key |
| `DATADOG_REGION` `v0.13.0+` | optional | `us1` (default) · `eu` · `us3` · `us5` · `ap1` |

> **Never commit `.env`.** QAless looks at process env first, then `~/.qaless/config.yaml`, then defaults — set production keys via your CI provider's secret store.

---

## Use it

### Generate scenarios

```bash
# Plain text
qaless analyze --input "Users can reset password via email link"

# Markdown / text file
qaless analyze --file feature.md

# OpenAPI 3.x schema
qaless analyze --openapi ./api/openapi.yaml

# Single source file
qaless analyze --code ./src/auth/reset.go

# Current git diff (staged by default)
qaless analyze --diff
qaless analyze --diff --diff-ref origin/main
```

Each scenario gets a P × C × R priority score:

| Score | Label | What it means |
|---|---|---|
| 18–27 | **CRITICAL** | Test before this ships. Touches money, auth, or data integrity. |
| 9–17 | **IMPORTANT** | Test before the next minor release. |
| 1–8 | LOW | Backlog. |

### Browse (free, no license check)

```bash
qaless history                          # list all analyses
qaless history --id <analysis-id>       # one analysis in detail

qaless show <analysis-id> <node-id>     # call-path frame + steps + related incidents
qaless tui                              # interactive picker — ↵ opens detail; b opens browser
qaless trace <analysis-id> <node-id>    # self-contained HTML — --open auto-launches browser
```

`qaless tui` is the closest match to the landing-page animation. `qaless trace` produces the same content as an HTML file (great for sharing in Slack / Notion).

### Verify against staging

```bash
qaless verify --against https://staging.example.com               # every node in latest analysis
qaless verify --analysis-id <id> --against https://staging.example.com
qaless verify --scenario s12 --against https://staging.example.com  # one scenario
```

QAless parses each scenario's first HTTP line from the steps, fires the request, and reports **confirm** / **refute** / **skipped** / **error**. Yellow warning if the URL contains "prod" — doesn't block, but flags it.

### Export

| Format | Command | Output |
|---|---|---|
| Markdown checklist | `qaless export --format markdown` | stdout (pipe to file) |
| Playwright | `qaless export --format playwright --output ./tests/` | TypeScript `describe`/`it` skeletons |
| Postman | `qaless export --format postman --output ./postman/` | Collection v2.1 JSON |
| Interactive HTML | `qaless export --format html` | `qaless-report.html` with table + graph view |

### Mock mode (free, instant)

```bash
QALESS_LLM_PROVIDER=mock qaless analyze --input "anything"
# → full P×C×R table, no real LLM call, no token cost.
```

Perfect for demos, screenshots, screen recordings, and the QAless self-CI smoke test.

### Reset state

```bash
qaless config reset                            # drop ~/.qaless/config.yaml (prompts)
qaless config reset --key LICENSE_KEY          # unset one key (no prompt)
qaless config reset --license-cache            # force fresh LemonSqueezy validation
qaless config reset --history                  # wipe past analyses
qaless config reset --all --yes                # nuke ~/.qaless/ — fresh start
```

---

## Wire QAless into Claude Desktop / Cursor / Zed

QAless ships as an MCP server. One command writes the entry into every installed client's config:

```bash
qaless mcp install
#   ✓ wired claude-desktop
#   ✓ wired cursor
#   ✓ wired zed
#   restart each updated client for the new tools to appear.
```

After restarting, in any client chat: *"use qaless to analyze: users can reset password via email."*

### Tool catalog `v0.13.0`

| Tool | What it does | License-gated? |
|---|---|---|
| `qaless_analyze_feature` | Generate the scenario graph | yes |
| `qaless_export_scenarios` | Export latest as markdown / playwright / postman | no |
| `qaless_list_history` | List past analyses | no |
| `qaless_get_analysis` | One analysis in detail | no |
| `qaless_show_scenario` | Plain-text drill-in for one node | no |
| `qaless_trace_scenario` | HTML drill-in for one node (with live observability links) | no |
| `qaless_list_incidents` | List recorded prod incidents | no |
| `qaless_related_incidents` | Incidents matching a scenario's topics | no |
| `qaless_dashboard` `v0.13.0+` | Six-panel HTML analytics over your local history | no |

```bash
qaless mcp doctor          # see what's wired where + binary path per client
qaless mcp uninstall       # remove qaless (preserves your other MCP servers)
```

The env block (`LICENSE_KEY`, `ANTHROPIC_API_KEY`, observability vars) is baked into the client config at install time so the spawned MCP server has what it needs. Pass `--no-copy-env` to skip that and set them via your shell rc.

---

## Observability `v0.13.0+`

QAless integrates four observability providers behind a single port. Set any subset — unconfigured providers are silently skipped, so you can start with one and add more later.

| Provider | What QAless does | Required env |
|---|---|---|
| **Sentry** | Pulls issues for the scenario's primary service; parses structured stack frames | `SENTRY_AUTH_TOKEN` + `SENTRY_ORG` |
| **Vercel** | Pulls deployment error events; parses stack frames from `text` field | `VERCEL_TOKEN` (+ optional `VERCEL_TEAM_SLUG`) |
| **Grafana Loki** | Issues LogQL query for error-level events; parses frames from log lines | `GRAFANA_LOKI_ENDPOINT` + `GRAFANA_LOKI_INSTANCE_ID` + `GRAFANA_LOKI_TOKEN` |
| **Datadog** | POSTs to events/search for error-status events; parses frames from `attributes.message` | `DATADOG_API_KEY` + `DATADOG_APP_KEY` (+ optional `DATADOG_REGION`) |

### What lands where

Once configured, `qaless trace <analysis-id> <node-id>` and `qaless tui` (on `↵ → b`) render the call-path frame with **real permalinks**:

- *Open in dashboard* button → deep-link into the first non-Sentry provider that returned signals
- *Jump to Sentry* button → deep-link into the Sentry issue

When no provider returns a signal for the scenario's primary service, the buttons stay as honest disabled stubs with a tooltip pointing at the config step. No fake URLs, ever.

### Stack-frame → service mapping

QAless's accuracy lever is the **stack-frame matcher**: when a Sentry/Loki/Vercel/Datadog signal includes frames like `at /var/task/api/auth/reset.js:42`, QAless resolves the frame to a service by matching against the `paths:` glob declarations in your `qaless.services.yml`:

```yaml
services:
  - name: api-gateway
    paths:
      - "src/api/**"
      - "internal/auth/*"
    routes:
      - "POST /auth/reset"
    calls:
      - { name: auth-svc, via: "grpc resetpassword()" }
```

Globs: `*` matches one segment, `**` matches multi-segment, no-glob means prefix. First match wins, one service per frame. This is what drives the priority uplift — scenarios touching a service with live error signals get boosted in the next analyze.

### Why not OTel?

QAless deliberately does not ingest OpenTelemetry traces. The constraint is product, not technical: QAless is **scenarios first, observability second**. Becoming Datadog-by-the-back-door would obscure the actual wedge.

### Cache

Each `FetchSignals` call is keyed by `(cwd, service)` and persisted at `~/.qaless/observability-cache.json` for 1 hour. Reduces external API pressure during `qaless tui` browsing.

---

## Dashboard `v0.13.0+`

```bash
qaless dashboard                            # writes ~/qaless-dashboard-<ts>.html
qaless dashboard --output ./dash.html
qaless dashboard --open                     # also launches browser
```

Free surface — no LLM cost, no license check. Six panels over your local SQLite history:

| Panel | What it shows |
|---|---|
| Analyses / week | 8-week sparkline of generation volume |
| Top critical scenarios | 10 highest P×C×R across all analyses |
| Incident hit rate | Monthly % of recorded prod incidents matched by ≥18-priority scenarios |
| Verify pass/fail trend | 8-week stacked-bar of `qaless verify` outcomes |
| Feedback kept ratio | 8-week kept / (kept+removed) from `qaless review` |
| Observability coverage | % scenarios with ≥1 linked observability signal `v0.14.0+` |

> **Panel 6 is provisional in v0.13.0** — it renders a `"not yet measurable"` placeholder pointing at the ADR. The persistence schema for signal-linkage lands in v0.14.0.

The MCP tool `qaless_dashboard` returns the same content as a JSON envelope for use inside Claude Desktop / Cursor / Zed.

---

## Production memory

QAless gets smarter on **your** codebase as you give it signal. Three mechanisms compose:

### Project context

```bash
qaless context detect        # auto-detect from go.mod / package.json / requirements.txt / etc.
qaless context show          # see what's detected vs hand-set
qaless context set \
  --project "acme-checkout" \
  --stack "Go 1.22, Postgres, Stripe" \
  --context "B2B checkout, idempotency keys required on all writes"
```

User-supplied values always win over detected ones. Detected fields cover: language, frameworks, database, auth library, domain glossary, conventions.

### Incidents

```bash
qaless incident record \
  --title "stale token cache after rotation" \
  --tag auth --tag redis \
  --ref PROD-412 \
  --date 2026-04-18
qaless incident list
```

Recorded incidents drive **history-uplift weighting**: the next `qaless analyze` for the same project pushes scenarios that touch services with prior incidents up in priority. The `qaless show` drill-in surfaces related incidents inline.

### Feedback

```bash
qaless review <analysis-id>
# Interactive: k (kept) · r (removed) · e (edited) · s (skip) per node
```

After feedback is collected, the next `qaless analyze` for the same `cwd` weights kept scenarios as positive few-shot examples and removed scenarios as negative ("the user previously rejected these — don't repeat this pattern"). Same project gets sharper over time.

---

## Service topology

Service topology drives the **call-path frame** in `qaless show` / `qaless tui` / `qaless trace`, the stack-frame matcher in observability, and (in v0.13.0+) the priority uplift on services with live incidents.

```bash
qaless services init                       # scaffold qaless.services.yml with 5 example services
qaless services detect                     # auto-generate from docker-compose.yml (deterministic, free)
qaless services detect --llm               # infer from source code via your configured LLM (~5-30¢)
qaless services detect --llm --dry-run     # preview without writing
qaless services detect --force             # overwrite an existing file
```

### Format

```yaml
services:
  - name: api-gateway
    routes:
      - "POST /auth/reset"
      - "POST /auth/login"
      - "grpc:ResetPassword"
    paths:
      - "internal/api/**"
      - "cmd/gateway/*"
    calls:
      - { name: auth-svc, via: "grpc resetpassword()" }
      - { name: redis-cache, via: "get token:{tok}" }
    tags: ["auth", "edge"]
    incident_blame:
      ref: PROD-412
      note: stale read after rotation

  - name: auth-svc
    routes: ["grpc:ResetPassword"]
    paths: ["services/auth/**"]
    calls:
      - { name: postgres-primary, via: "select users where token=?" }
    tags: ["auth"]
```

### LLM topology detection — how it works

For repos without `docker-compose.yml`, `qaless services detect --llm` reads source and infers `qaless.services.yml`. Best fit: Go / Node / Python / Rust monoliths and microservice repos where routes live in dedicated handler files.

1. Walks `cwd` up to **50 files / 200 KB total / 32 KB per file** (bounded — predictable token cost).
2. Skips `node_modules`, `vendor`, `.git`, `dist`, `build`, `target`, `.next`, `__pycache__`, `.venv`, etc.
3. Collects three signal types:
   - **File tree** — up to 200 paths so the model sees shape.
   - **Manifests** — `go.mod`, `package.json`, `requirements.txt`, `pyproject.toml`, `Cargo.toml`, `Gemfile`, `composer.json`, `Dockerfile`, `docker-compose.yml`.
   - **Entry points + router files** — `main.go`, `cmd/*/main.go`, `server.{ts,js}`, `app.py`, `main.py`, `src/main.rs`, plus by basename anywhere in the tree: `routes.go`, `router.go`, `handlers.go`, `handler.go`, `routes.ts`, `router.ts`, `routes.js`, `router.js`, `urls.py`, `routes.py`, `router.py`, `routes.rs`, `router.rs`.
4. Sends one prompt to your configured LLM.
5. Parses YAML (strips fences), drops services with empty names, **normalizes placeholder routes** (`{id}` / `:param` / `<id>` → `*` so the matcher gets prefix-matchable patterns).
6. Writes `qaless.services.yml` with a provenance header.

> **Routes use `*` wildcards, not `{id}` placeholders.** QAless's path matcher does prefix matching with `*`. The detector emits `GET /cars/*` not `GET /cars/{id}`. If you hand-edit, follow the same rule — placeholder routes silently fail to match scenario steps.

### Verifying topology drives the frame

```bash
# 1. Generate an analysis that produces scenarios with HTTP routes in steps
qaless analyze --input "Users can update their payment method"

# 2. Drill in
qaless tui
# ↑↓ select scenario, ↵ open detail pane

# The detail pane should render "call-path across services" above the steps.
# If not: the scenario's first HTTP line isn't matching any service's routes.
# Check qaless.services.yml and re-run with --llm or hand-edit.
```

When no scenario route matches the topology, QAless falls back to a **declared topology overview** (renders the topology graph with `0 of N` counters) so you still see the service map.

---

## Wire QAless into your CI/CD

### GitHub Actions

Drop this into `.github/workflows/qaless.yml`. Every PR gets a sticky comment with the prioritized scenario checklist for the changed code.

```yaml
name: qa-scenarios
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  scenarios:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }   # required so --diff resolves origin/<base>
      - uses: rwrrioe/qaless@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          license-key:       ${{ secrets.QALESS_LICENSE_KEY }}
```

Two secrets in **Settings → Secrets and variables → Actions**:

| Secret | Where it comes from | Why |
|---|---|---|
| `ANTHROPIC_API_KEY` | https://console.anthropic.com | Pays the LLM tokens (your bill) |
| `QALESS_LICENSE_KEY` | https://qaless.lemonsqueezy.com ($20/mo) | Unlocks gated commands |

### Cost per PR

| Item | Who pays | Per-PR cost |
|---|---|---|
| QAless subscription | You → LemonSqueezy | Flat **$20/month**, not per-PR |
| LLM tokens | You → Anthropic / Google | ~5–15¢ depending on diff size |
| GitHub Actions minutes | You → GitHub | ~30–60 s of runner time |

The license-key cache is ephemeral on fresh CI runners, so every PR triggers one LemonSqueezy API validation — well inside their rate limits.

### Non-GitHub CI (GitLab, CircleCI, Buildkite, Jenkins)

Same shape, no Action required:

```bash
# Install (Linux binaries forthcoming — issue if you need one)
curl -L https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.13.0_linux_amd64.tar.gz \
  | sudo tar xz -C /usr/local/bin qaless

export ANTHROPIC_API_KEY=sk-ant-...
export LICENSE_KEY=lic_...

qaless analyze --diff --diff-ref origin/main
qaless export --format markdown --output scenarios.md
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `error: license required: run qaless config set LICENSE_KEY=...` | No license key configured | Buy at qaless.lemonsqueezy.com or `qaless config set LICENSE_KEY=demo` |
| `error: license invalid or expired` | LemonSqueezy marked the key revoked, or 7-day offline grace expired | `qaless license status` and re-check on LemonSqueezy |
| `error: store mismatch` | Key was minted on a different LemonSqueezy store | Buy at the official storefront (qaless.lemonsqueezy.com) |
| `error: unknown flag: --version` or `--llm` | Stale binary — older than v0.12.0 | Re-run install snippet; `hash -r` to drop shell PATH cache |
| `tar: could not chdir to '/usr/local/bin'` | Fresh Apple Silicon Mac without Homebrew | Run `sudo mkdir -p /usr/local/bin` first |
| `qaless tui` shows steps but no call-path frame | `qaless.services.yml` missing, or routes use `{id}` placeholders | `qaless services detect --llm`, or edit `routes:` to use `*` wildcards |
| `git diff origin/main` fails inside CI | Missing `fetch-depth: 0` on `actions/checkout` | Add it to the checkout step |
| MCP tool returns license error in Claude Desktop | Same gate as CLI | `qaless config set LICENSE_KEY=...`, restart Claude Desktop |
| `qaless trace` buttons stay disabled with config tooltip | No observability provider configured | Set env vars from the [observability section](#observability-v0130) |
| Observability shows stale data | 1h cache window | `rm ~/.qaless/observability-cache.json` for an immediate refresh |
| `qaless services detect --llm` emits empty routes | Routes registered in unusual file names | Hand-edit the YAML or open an issue with the framework's convention |

---

## License + pricing

**$20/month** via [qaless.lemonsqueezy.com](https://qaless.lemonsqueezy.com). LemonSqueezy is merchant of record (handles tax, receipts, payment methods).

The binaries in this repo are **proprietary**. The source is visible in a separate private repo for transparency only; no rights are granted to copy, modify, redistribute, or create derivative works.

### What requires a license

| Surface | Gated? | Why |
|---|---|---|
| `qaless analyze` | yes | Spends LLM tokens |
| `qaless verify --against` | yes | Hits staging URLs the LLM constructed |
| `qaless report --release` | yes | Aggregates LLM-generated artifacts |
| `qaless bench` (dev only) | yes | Multi-call LLM iteration |
| MCP `qaless_analyze_feature` | yes | Same as CLI |
| `qaless history` / `show` / `tui` / `trace` / `dashboard` | no | Read-only over local SQLite |
| `qaless services` / `context` / `incident` / `review` | no | Reads + writes local files only |
| `qaless config` / `license` / `mcp` | no | Tooling |

### Trial paths

- `LICENSE_KEY=demo` — payment-free, prints a yellow stderr warning, fine for evaluation.
- `QALESS_DEV=1` — dev-loop bypass, intended for contributing to the source.

For commercial licensing beyond the standard $20/month tier (team-shared keys, on-premise, redistribution) contact the copyright holder via [GitHub issues](https://github.com/rwrrioe/qaless-releases/issues).

---

## Versioning + changelog

QAless follows semver. Major versions reserved for breaking CLI / config / SQLite-schema changes.

| Version | Released | Notable |
|---|---|---|
| **v0.13.0** *(in flight — PRs #26-29)* | — | LemonSqueezy license gate (\$5/mo → \$20/mo, replaces ADR-021 Polar.sh) · Observability foundation (Sentry / Vercel / Grafana Loki / Datadog adapters + fanout + 1h file cache) · Stack-frame → service mapping with `paths:` globs · `qaless dashboard` six-panel analytics + MCP tool · Incident-replay benchmark mode (`qaless bench --mode=replay --held-out`) · Intent classifier cache + Haiku critic downshift (\~3–5% pipeline savings) |
| v0.12.3 | 2026-06-12 | TUI scenario-list alignment fix · `qaless show` / `trace` overview fallback when no scenario route matches the declared topology |
| v0.12.2 | 2026-06-11 | LLM topology detector — adapter-level route normalization (`{id}` → `*`) |
| v0.12.1 | 2026-06-11 | LLM topology detector — `*`-wildcard rule in prompt + scans router/handler files outside entry points |
| v0.12.0 | 2026-06-11 | First public release — Polar.sh license gate, MCP auto-install, LLM topology detector, intent-classifier relaxation, binary distribution |

See [Releases](https://github.com/rwrrioe/qaless-releases/releases) for binaries, release notes, and SHA-256 checksums.

---

## Support

| What | Where |
|---|---|
| Bug / install issue | [github.com/rwrrioe/qaless-releases/issues](https://github.com/rwrrioe/qaless-releases/issues) |
| Feature request | Same — tag with `feature` |
| Pricing / license question | [qaless.lemonsqueezy.com](https://qaless.lemonsqueezy.com) |
| Live walkthrough / concierge onboarding | Open an issue with `concierge` and we'll set up a 30-min Zoom |

---

<div align="center">

**QAless** — *Not how to test. What can break.*

Made for small teams that ship weekly. Built on [Anthropic Claude](https://www.anthropic.com/claude) + [Model Context Protocol](https://modelcontextprotocol.io).

</div>
