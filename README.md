# QAless — binaries

**AI-powered E2E test scenario generator for small dev teams.** Feed it a feature description, OpenAPI spec, or git diff and get back a prioritized graph of E2E scenarios and edge cases, exported to Playwright, Postman, or Markdown. In 30 seconds instead of a day.

This repo hosts **compiled binaries** for QAless. The source code lives in a private repo; the binaries are free to download. Running paid commands (`analyze`, `verify`, `report`, `bench`) requires a $5/month license key from [polar.sh/qaless](https://polar.sh/qaless).

- **Landing page:** https://qaless.vercel.app
- **Buy a license ($5/mo):** https://polar.sh/qaless
- **Releases:** https://github.com/rwrrioe/qaless-releases/releases

---

## Install

### macOS (Apple Silicon · arm64)

```bash
sudo mkdir -p /usr/local/bin
curl -L https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.12.1_darwin_arm64.tar.gz \
  | sudo tar xz -C /usr/local/bin qaless
qaless --version
```

The trailing `qaless` on `tar` filters the archive so only the binary lands in `/usr/local/bin` (the `LICENSE` and `README.md` from the archive stay out of your PATH).

### Windows 11 (amd64)

In PowerShell:

```powershell
Invoke-WebRequest "https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.12.1_windows_amd64.zip" -OutFile qaless.zip
Expand-Archive qaless.zip C:\qaless -Force
$env:Path += ";C:\qaless"
qaless --version
```

To persist the PATH change across sessions, use `setx PATH "$env:Path"` (or add `C:\qaless` via System Properties → Environment Variables).

### Linux / Intel macOS

Not yet shipped. Open an issue if you need one — most-requested platform ships first.

---

## Configure (one-time, ~30 seconds)

QAless needs two keys: your **LLM API key** (Anthropic or Gemini — you pay the LLM provider directly for tokens) and a **QAless license key** ($5/mo via Polar).

```bash
# Your LLM bill
qaless config set ANTHROPIC_API_KEY sk-ant-...
# or use Gemini:
qaless config set QALESS_LLM_PROVIDER gemini
qaless config set GEMINI_API_KEY AIza...

# Your QAless license — buy at https://polar.sh/qaless ($5/mo)
qaless config set LICENSE_KEY qal_...

# Or, for a trial without payment:
qaless config set LICENSE_KEY demo
# → short-circuits Polar, returns synthetic valid status,
#   prints a yellow stderr warning on first use ("not for production")

# Verify
qaless config show         # secrets auto-redacted
qaless license status      # cached license state + grace remaining
```

Config lives at `~/.qaless/config.yaml`. License cache at `~/.qaless/license.json` (7-day offline grace). History DB at `~/.qaless/history.db`.

---

## Wire QAless into Claude Desktop / Cursor / Zed

QAless ships as an MCP server. One command writes the right entry into each installed client's config:

```bash
qaless mcp install
#   ✓ wired claude-desktop
#   ✓ wired cursor
#   ✓ wired zed
#   restart each updated client app for the new tools to appear.
```

After restarting the client, in any chat: *"use qaless to analyze: users can reset password via email."* QAless's 8 tools (`qaless_analyze_feature`, `qaless_export_scenarios`, `qaless_list_history`, `qaless_get_analysis`, `qaless_show_scenario`, `qaless_trace_scenario`, `qaless_list_incidents`, `qaless_related_incidents`) appear in the client's tool catalog.

```bash
qaless mcp doctor          # see what's wired where + binary path per client
qaless mcp uninstall       # remove the qaless entry (preserves your other MCP servers)
```

The env block (`LICENSE_KEY`, `ANTHROPIC_API_KEY`, etc.) is baked into the client config at install time so the spawned MCP server has what it needs. Pass `--no-copy-env` if you'd rather set those via shell rc.

---

## Use it

### Generate scenarios from a feature description

```bash
# Plain text
qaless analyze --input "Users can reset password via email link"

# From a Markdown / text file
qaless analyze --file feature.md

# From an OpenAPI spec
qaless analyze --openapi ./api/openapi.yaml

# From a single source file
qaless analyze --code ./src/auth/reset.go

# From the current git diff (staged by default)
qaless analyze --diff
qaless analyze --diff --diff-ref origin/main
```

Output is a P × C × R-prioritized table of scenarios with IDs you can drill into. Each scenario has a precondition, steps, and an expected result.

### Explore — free, no license check

```bash
qaless history                         # list past analyses
qaless history --id <analysis-id>      # one in detail

qaless show <analysis-id> <node-id>    # call-path frame + steps + related incidents
qaless tui                             # interactive picker; ↵ to drill in, b to open browser
qaless trace <analysis-id> <node-id>   # HTML view; --open auto-launches browser
```

### Export

```bash
qaless export --format markdown                            # stdout
qaless export --format playwright --output ./tests/        # TypeScript test skeletons
qaless export --format postman --output ./postman/         # collection v2.1 JSON
qaless export --format html                                # → qaless-report.html (interactive)
```

### Teach QAless about *your* project (memory layer)

```bash
# Project context — auto-detect language / frameworks / db / auth from manifests
qaless context detect

# Or set explicitly:
qaless context set \
  --project "acme-checkout" \
  --stack "Go 1.22, Postgres, Stripe" \
  --context "B2B checkout, idempotency keys required on all writes"

# Service topology — drives the call-path frame in `show` / `tui` / `trace`
qaless services init                   # scaffold hand-edit template
qaless services detect                 # auto-generate from docker-compose.yml
qaless services detect --llm           # infer from source code via your configured LLM
qaless services detect --llm --dry-run # preview without writing

# Production incident memory — drives history-uplift weighting on related scenarios
qaless incident record \
  --title "stale token cache after rotation" \
  --tag auth --tag redis \
  --ref PROD-412 \
  --date 2026-04-18
qaless incident list

# Per-scenario feedback — drives few-shot examples in the next analyze
qaless review <analysis-id>            # k/r/e/s per node (kept / removed / edited / skip)
```

After feedback is collected, the next `qaless analyze` for the same `cwd` weights kept scenarios as positive few-shot examples and removed scenarios as negative — the same project gets smarter over time.

### LLM-based topology detection (`qaless services detect --llm`)

For repos without a `docker-compose.yml`, the LLM detector reads your source code and infers `qaless.services.yml` directly. Best for: Go / Node / Python / Rust monoliths and microservice repos where routes live in dedicated handler files.

```bash
qaless services detect --llm --dry-run    # preview YAML on stdout, no write
qaless services detect --llm              # write qaless.services.yml in cwd
qaless services detect --llm --force      # overwrite existing
```

**How it works**

1. Walks `cwd` up to **50 files / 200 KB total / 32 KB per file** (bounded — predictable token cost)
2. Skips `node_modules`, `vendor`, `.git`, `dist`, `build`, `target`, `.next`, `__pycache__`, `.venv`, etc.
3. Collects three signal types:
   - **File tree** — up to 200 paths so the model sees the shape
   - **Manifests** — `go.mod`, `package.json`, `requirements.txt`, `pyproject.toml`, `Cargo.toml`, `Gemfile`, `composer.json`, `Dockerfile`, `docker-compose.yml`
   - **Entry points + router files** — `main.go`, `cmd/*/main.go`, `server.{ts,js}`, `app.py`, `main.py`, `src/main.rs`, **plus** by basename anywhere in the tree: `routes.go`, `router.go`, `handlers.go`, `handler.go`, `routes.ts`, `router.ts`, `routes.js`, `router.js`, `urls.py`, `routes.py`, `router.py`, `routes.rs`, `router.rs`
4. Sends one prompt to your configured LLM (`QALESS_LLM_PROVIDER`)
5. Parses the YAML response (strips fences if present), drops services with empty names
6. Writes `qaless.services.yml` with a provenance header

**Cost**: ~5–30¢ per detection on Sonnet 4.6 (depends on repo size). Same license gate as `qaless analyze`.

**Output format the matcher expects**

QAless's path matcher does prefix matching with `*` wildcards. The detector is instructed to emit:

```yaml
routes:
  - "GET /cars/*"                  # NOT /cars/{id}
  - "POST /api/pictures/*"         # one wildcard route > three leaf paths
  - "grpc:ResetPassword"
```

**Common gotchas**

- **`{id}` / `:param` placeholders in routes** — won't match scenario steps with concrete IDs. v0.12.1+ prompts the LLM to emit `*` instead; if you're on an older binary, re-run with v0.12.1 or hand-edit.
- **Routes registered in unusual files** — the walker covers the 13 most common router/handler basenames. If your routes live in `internal/api/v2/endpoints.go` or similar non-standard names, the LLM may infer empty `routes:` arrays. Hand-edit the YAML after detection.
- **Hallucinated `calls` edges** — the prompt instructs the LLM to only emit edges defensible from source, but it can still over-reach. Review with `--dry-run` first.
- **Use compose when you have it** — `docker-compose.yml` is more deterministic than LLM inference. The plain `qaless services detect` (no `--llm`) reads compose and is free. Use `--llm` only when you don't have compose, OR when compose is missing route data.

**Verifying the topology actually drives the call-path frame**

```bash
# 1. Run an analyze that produces scenarios with HTTP routes in the steps
qaless analyze --input "Users can update their payment method"

# 2. Drill in
qaless tui
# ↑↓ to select a scenario, ↵ to open

# The detail pane should show a "call-path across services" frame
# above the steps box. If it doesn't, the scenario's first HTTP line
# isn't matching any service's `routes:` — check qaless.services.yml.
```

### Execute scenarios against staging

```bash
qaless verify --against https://staging.example.com               # every node in latest analysis
qaless verify --analysis-id <id> --against https://staging.example.com
qaless verify --scenario s12 --against https://staging.example.com  # one scenario only
```

QAless parses each scenario's first HTTP line out of the steps, fires the request, and reports confirm / refute / skipped / error. Yellow warning if the URL contains "prod" (doesn't block).

### Release report (accountability artifact)

```bash
qaless report --release v1.2.3                              # auto-derives --since from git tags
qaless report --release v1.2.3 --since v1.2.2
qaless report --release v1.2.3 --output ./reports/release.html
```

Content-addressed HTML report (sha256 footer) covering analyses + verifications + feedback + production incidents between tags.

### Mock mode (free, instant, no LLM tokens)

```bash
QALESS_LLM_PROVIDER=mock qaless analyze --input "anything"
# → full P×C×R table, no real API call. Perfect for demos / screenshots / screen recordings.
```

### Reset state cleanly

```bash
qaless config reset                            # drop config.yaml (prompts)
qaless config reset --key LICENSE_KEY          # unset one key (no prompt)
qaless config reset --license-cache            # force fresh Polar validation
qaless config reset --history                  # wipe past analyses
qaless config reset --all --yes                # nuke ~/.qaless/
```

---

## Wire QAless into your CI/CD

Drop this in your repo as `.github/workflows/qaless.yml`. Every PR gets a sticky comment with the prioritized scenario checklist for the changed code.

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
- `ANTHROPIC_API_KEY` — your LLM bill
- `QALESS_LICENSE_KEY` — from [polar.sh/qaless](https://polar.sh/qaless)

Cost per PR: $5/mo flat (QAless) + ~5–15¢ in LLM tokens + ~30–60s of runner time.

### Non-GitHub CI (GitLab, CircleCI, Buildkite, Jenkins)

Same shape, no Action needed:

```bash
# Install (Linux binaries not shipped yet — open an issue if you need one)
curl -L https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.12.1_linux_amd64.tar.gz \
  | sudo tar xz -C /usr/local/bin qaless

export ANTHROPIC_API_KEY=sk-ant-...
export LICENSE_KEY=qal_...

qaless analyze --diff --diff-ref origin/main
qaless export --format markdown --output scenarios.md
```

---

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| `error: license required: run qaless config set LICENSE_KEY=...` | No license key configured | Buy at polar.sh/qaless, or `qaless config set LICENSE_KEY=demo` for a trial |
| `error: license invalid or expired` | Polar marked the key revoked, or 7-day grace expired offline | `qaless license status` and re-check on Polar |
| `error: unknown flag: --version` or `--llm` | Stale binary — older than v0.12.0 | Re-run the install snippet above; `hash -r` to drop shell PATH cache |
| `tar: could not chdir to '/usr/local/bin'` | Fresh Apple Silicon Mac without Homebrew (no `/usr/local/bin`) | The install snippet above includes `sudo mkdir -p /usr/local/bin` — make sure you ran that line |
| `qaless tui` shows steps but no call-path frame | `qaless.services.yml` missing, or routes use `{id}` placeholders | Run `qaless services detect --llm`, or edit `routes:` to use `*` wildcards (`POST /api/users/*`, not `POST /api/users/{id}`) |
| `git diff origin/main` fails inside a CI runner | Missing `fetch-depth: 0` on `actions/checkout` | Add it to the checkout step |
| MCP tool returns license error in Claude Desktop | Same gate as CLI | `qaless config set LICENSE_KEY=...`, restart Claude Desktop |

---

## License

The QAless binaries in this repo are **proprietary**. The source code is visible in a separate private repo for transparency only; no rights are granted to copy, modify, redistribute, or create derivative works.

Running gated commands (`qaless analyze`, `qaless verify`, `qaless report`, `qaless bench`, and the equivalent MCP tool `qaless_analyze_feature`) requires a **$5/month license key** from [polar.sh/qaless](https://polar.sh/qaless). Read-only commands (`history`, `show`, `tui`, `trace`, `services`, `context`, `incident`, `review`, `config`, `license`, `mcp`) run without a key.

Set `QALESS_DEV=1` to bypass the license gate during local development. Set `LICENSE_KEY=demo` for a payment-free trial (prints a stderr warning).

---

## Versioning

QAless follows semver. The current minor (v0.12.x) is the first public release line.

| Version | Notable |
|---|---|
| **v0.12.1** | LLM topology detector now emits `*`-wildcard routes (matches scenario steps) + scans router/handler files outside the entry point |
| v0.12.0 | First public release: Polar.sh license gate, MCP auto-install, LLM topology detector, friendlier intent rejection, binary distribution |

See [Releases](https://github.com/rwrrioe/qaless-releases/releases) for binaries + release notes.

---

## Support

- **Bug / install issue:** open an issue at https://github.com/rwrrioe/qaless-releases/issues
- **Pricing / license questions:** see https://polar.sh/qaless
- **Feature requests:** open an issue with the `feature` label
