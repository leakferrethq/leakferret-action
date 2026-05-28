# leakferret — GitHub Action

Run `leakferret` on every push and PR. Uploads SARIF to GitHub Code
Scanning so findings show up in the Security tab and on the PR
"Files changed" view.

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

| input | default | meaning |
|---|---|---|
| `version` | `latest` | leakferret version to install |
| `path` | `.` | Path to scan |
| `verify-mode` | `best-effort` | `none` / `best-effort` / `only-verified` / `ever-verified` |
| `format` | `sarif` | `pretty` / `json` / `sarif` |
| `fail-on` | `any` | `critical` / `high` / `medium` / `low` / `any` |
| `baseline-path` | `.leakferret-baseline.json` | Baseline file relative to repo root |
| `upload-sarif` | `true` | Upload SARIF to Code Scanning |
| `category` | `leakferret` | SARIF category |

## Outputs

| output | meaning |
|---|---|
| `findings-count` | Total findings |
| `verified-count` | Findings verified live by a provider |
| `sarif-path` | Path to the generated SARIF file |

## How it works

1. Resolve the requested version (`latest` hits the GitHub Releases API).
2. Cache the binary under the runner tool-cache keyed by version+triple.
3. Download the matching `tar.gz` from GitHub Releases if not cached.
4. Run `leakferret verify` twice — once to JSON (for counts), once to
   SARIF (for upload). Both runs share the same fingerprints because
   the engine is deterministic for a given salt.
5. Apply the `fail-on` policy.
6. Upload SARIF via `github/codeql-action/upload-sarif@v3`.

## License

MIT.
