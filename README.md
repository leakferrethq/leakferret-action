# leakferret — GitHub Action

> MCP-native secret scanner — verified findings, agent-applied rewrites.

Run [`leakferret`](https://github.com/leakferrethq/leakferret) on every push and
pull request. This is a composite action: it downloads the prebuilt,
statically-linked binary (written in Rust) from GitHub Releases, caches it on
the runner, runs the scan, and uploads SARIF to GitHub Code Scanning so findings
appear in the **Security** tab and inline on the PR "Files changed" view. The
action contains no scanning logic of its own — all the work happens in the
binary.

## What leakferret does

leakferret finds hardcoded secrets and API keys and confirms which ones are
actually live: it regex-scans files (respecting `.gitignore`), marks documented
public examples as `FIXTURE` via a signed catalog, classifies each candidate as
`REAL` / `FIXTURE` / `UNKNOWN`, and **verifies** real findings with a harmless
API call to the provider (AWS SigV4, GitHub, GitLab, Stripe, OpenAI, Anthropic,
Slack, Twilio, SendGrid, Mailgun, Datadog, Heroku, npm, PyPI, DigitalOcean),
plus a trufflehog fallback.

**Privacy invariant:** the full secret value never leaves the runner. Only a
redacted first-4-plus-last-4 preview (e.g. `AKIA...4XYZ`) is ever written to the
SARIF, logs, or any output. Verification calls go straight from the runner to
the provider — leakferret has no servers.

## Quickstart

```yaml
# .github/workflows/leakferret.yml
name: leakferret
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write   # required for SARIF upload

jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: leakferrethq/leakferret-action@v1
        with:
          path: .
          verify-mode: only-verified
          fail-on: any
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `version` | `latest` | leakferret version to install (e.g. `0.1.0`). `latest` resolves via the GitHub Releases API. |
| `path` | `.` | Path to scan, relative to the repo root. |
| `verify-mode` | `best-effort` | One of `none`, `best-effort`, `only-verified`, `ever-verified`. |
| `format` | `sarif` | Output format: `pretty`, `json`, or `sarif`. SARIF is what Code Scanning ingests. |
| `fail-on` | `any` | Severity threshold to fail the run: `low`, `medium`, `high`, `critical`, or `any`. `any` fails on any REAL finding. |
| `baseline-path` | `.leakferret-baseline.json` | Baseline file relative to the repo root. |
| `upload-sarif` | `true` | Upload SARIF to GitHub Code Scanning. |
| `category` | `leakferret` | SARIF category (lets multiple leakferret runs coexist in Code Scanning). |
| `github-token` | `${{ github.token }}` | Token for fetching binaries on rate-limited runners. |

## Outputs

| Output | Description |
|---|---|
| `findings-count` | Total number of findings. |
| `verified-count` | Findings verified live by a provider. |
| `sarif-path` | Path to the generated SARIF file. |

## SARIF upload

When `upload-sarif: true` (the default), the action calls
`github/codeql-action/upload-sarif@v3` for you, so the workflow needs
`security-events: write` permission (shown in the Quickstart). Findings then
surface in the Security tab and as annotations on changed lines. Use distinct
`category` values if you run leakferret more than once in a repo so the results
don't overwrite each other.

To gate merges without publishing to Code Scanning, set `upload-sarif: false`
and rely on the `fail-on` policy and the action outputs.

## Baselines

leakferret stores one-way HMAC fingerprints of known findings (never the raw
secret) in the baseline file, so CI can fail only on **new** leaks. Commit
`.leakferret-baseline.json` to your repo, or point `baseline-path` at a custom
location.

## How it works

1. Detect the runner platform and resolve the requested `version`.
2. Cache the binary in the runner tool-cache, keyed by version and target triple.
3. Download the matching `tar.gz` from GitHub Releases if not cached.
4. Run `leakferret verify` to JSON (for counts) and to SARIF (for upload); both
   runs share fingerprints because the engine is deterministic for a given salt.
5. Apply the `fail-on` policy.
6. Upload the SARIF via `github/codeql-action/upload-sarif@v3`.

Hosted runners (`ubuntu-latest`, `macos-latest`, `windows-latest`) are all
supported.

## Using a local binary

Like every leakferret wrapper, the underlying binary honors the `LEAKFERRET_BIN`
environment variable. On a self-hosted runner with the binary pre-positioned,
set it on the step to skip the download:

```yaml
      - uses: leakferrethq/leakferret-action@v1
        env:
          LEAKFERRET_BIN: /opt/leakferret/leakferret
        with:
          path: .
```

## License

MIT for this action and the bundled binary. The fixture catalog **data** is
CC-BY-SA-4.0 — see [`leakferret-catalog`](https://github.com/leakferrethq/leakferret-catalog).

---

Part of [leakferret](https://github.com/leakferrethq/leakferret) ·
[leakferret.com](https://leakferret.com) ·
maintained by Maria Khan &lt;missusk@protonmail.com&gt;.
