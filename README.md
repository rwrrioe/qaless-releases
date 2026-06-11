# QAless

**AI-powered E2E test scenario generator for small dev teams.** Give it a feature description, OpenAPI spec, or git diff — get back a prioritized graph of E2E scenarios and edge cases, exported to Playwright, Postman, or Markdown. In 30 seconds instead of a day.

This repo hosts **compiled binaries** for QAless. The source code lives in a private repo; the binaries are free to download. Running gated commands (`analyze`, `verify`, `report`, `bench`) requires a $5/month license key from [polar.sh/qaless](https://polar.sh/qaless).

- **Landing page:** https://qaless.vercel.app
- **Buy a license ($5/mo):** https://polar.sh/qaless
- **Releases:** https://github.com/rwrrioe/qaless-releases/releases

---

## Install

### macOS (Apple Silicon)

Paste these in your terminal:

```bash
sudo mkdir -p /usr/local/bin
curl -L https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.12.0_darwin_arm64.tar.gz \
  | sudo tar xz -C /usr/local/bin qaless
qaless --version
```

The trailing `qaless` on `tar` filters the archive so only the binary lands in `/usr/local/bin` (the `LICENSE` and `README.md` from the archive stay out of your PATH).

### Windows 11

In PowerShell:

```powershell
Invoke-WebRequest "https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.12.0_windows_amd64.zip" -OutFile qaless.zip
Expand-Archive qaless.zip C:\qaless -Force
$env:Path += ";C:\qaless"
qaless --version
```

To persist the PATH change across sessions, use `setx PATH "$env:Path"` (or add `C:\qaless` via System Properties → Environment Variables).

### Linux / Intel macOS

Not yet shipped. Open an issue if you need one — we'll prioritise the most-requested platform.

---

## Configure (one-time, 30 seconds)

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

Config lives at `~/.qaless/config.yaml`. License cache at `~/.qaless/license.json` (7-day offline grace after validation). History DB at `~/.qaless/history.db`.

---

## Wire QAless into Claude Desktop / Cursor / Zed

```bash
qaless mcp install
#   ✓ wired claude-desktop
#   ✓ wired cursor
#   ✓ wired zed
#   restart each updated client app for the new tools to appear.
```

After restarting the client, in any chat: *"use qaless to analyze: users can reset password via email."* QAless's 8 tools (analyze, export, history, show, trace, incident list, related-incidents, get-analysis) appear in the client's tool catalog.

```bash
qaless mcp doctor          # see what's wired where + binary path per client
qaless mcp uninstall       # remove the qaless entry (preserves your other MCP servers)
```

The env block (`LICENSE_KEY`, `ANTHROPIC_API_KEY`, etc.) is baked into the client config at install time so the spawned MCP server has what it needs. `--no-copy-env` skips that if you'd rather set env via shell rc.

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

Output is a P × C × R-prioritized table of scenarios with IDs you can drill into.

### Explore (free — no license check)

```bash
qaless history                         # list past analyses
qaless history --id <analysis-id>      # one in detail

qaless show <analysis-id> <node-id>    # call-path frame + steps + related incidents
qaless tui                             # interactive picker; ↵ to drill in
qaless trace <analysis-id> <node-id>   # HTML view, browser-opens with --open
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
qaless context detect                  # reads go.mod / package.json / etc. for stack signal
qaless context set \
  --project "acme-checkout" \
  --stack "Go 1.22, Postgres, Stripe" \
  --context "B2B checkout, idempotency keys required on all writes"

# Service topology — drives the call-path frame in `show` / `tui` / `trace`
qaless services init                   # hand-edit template
qaless services detect                 # auto-generate from docker-compose.yml
qaless services detect --llm           # infer from source code via your configured LLM
qaless services detect --llm --dry-run # preview without writing

# Prod-incident memory — drives the history-uplift weighting
qaless incident record \
  --title "stale token cache after rotation" \
  --tag auth --tag redis \
  --ref PROD-412 \
  --date 2026-04-18
qaless incident list

# Per-analysis feedback — drives few-shot examples in the next analyze
qaless review <analysis-id>            # k/r/e/s per node
```

### Execute scenarios against staging

```bash
qaless verify --against https://staging.example.com
qaless verify --analysis-id <id> --against https://staging.example.com
qaless verify --scenario s12 --against https://staging.example.com   # single scenario
```

### Release report (accountability)

```bash
qaless report --release v1.2.3                  # auto-derives --since from git tags
qaless report --release v1.2.3 --since v1.2.2
qaless report --release v1.2.3 --output ./reports/release.html
```

Content-addressed HTML; covers analyses + verifications + feedback + incidents between tags.

### Mock mode (free, instant, no LLM tokens)

```bash
QALESS_LLM_PROVIDER=mock qaless analyze --input "anything"
# → full P×C×R table, no real API call, perfect for demos / screenshots
```

### Reset state cleanly

```bash
qaless config reset                            # drop config.yaml (prompts)
qaless config reset --key LICENSE_KEY          # unset one key
qaless config reset --license-cache            # force fresh Polar validation
qaless config reset --history                  # wipe past analyses
qaless config reset --all --yes                # nuke ~/.qaless/
```

---

## Wire QAless into your CI/CD

Drop this into your repo as `.github/workflows/qaless.yml`. Every PR gets a sticky comment with the prioritized scenario checklist for the changed code.

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

Two secrets:
- `ANTHROPIC_API_KEY` — your LLM bill
- `QALESS_LICENSE_KEY` — from [polar.sh/qaless](https://polar.sh/qaless)

Cost per PR: $5/mo flat (QAless) + ~5–15¢ in LLM tokens + ~30–60s of runner time.

### Non-GitHub CI (GitLab, CircleCI, Buildkite, Jenkins)

Same three steps, no Action needed:

```bash
curl -L https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_0.12.0_linux_amd64.tar.gz \
  | sudo tar xz -C /usr/local/bin qaless
# (Linux binaries not yet shipped — open an issue)

export ANTHROPIC_API_KEY=sk-ant-...
export LICENSE_KEY=qal_...

qaless analyze --diff --diff-ref origin/main
qaless export --format markdown --output scenarios.md
```

---

## License

The QAless binaries in this repo are **proprietary**. The source code is visible in a separate private repo for transparency only; no rights are granted to copy, modify, redistribute, or create derivative works.

Running gated commands (`qaless analyze`, `qaless verify`, `qaless report`, `qaless bench`, and the equivalent MCP tool `qaless_analyze_feature`) requires a **$5/month license key** from [polar.sh/qaless](https://polar.sh/qaless). Read-only commands (`history`, `show`, `tui`, `trace`, `services`, `context`, `incident`, `review`, `config`, `license`, `mcp`) run without a key.

Set `QALESS_DEV=1` to bypass the license gate during local development. Set `LICENSE_KEY=demo` for a payment-free trial (prints a stderr warning).

---

## Versioning

QAless follows semver. The current minor (v0.12.x) is the first public release.

| Version | Notable |
|---|---|
| v0.12.0 | First public release: license gate, MCP auto-install, LLM topology detector, friendlier error surfaces, signed builds |

See [Releases](https://github.com/rwrrioe/qaless-releases/releases) for binaries + release notes.

---

## Support

- **Bug / install issue:** open an issue at https://github.com/rwrrioe/qaless-releases/issues
- **Pricing or license questions:** see https://polar.sh/qaless
- **Feature requests:** open an issue with the `feature` label
