# Scanners: detect, install, run

The tool matrix behind flow step 2, run once by the orchestrating
agent — dimension passes receive the resolved mode and never re-probe
or propose installs. For every tool: probe first; if missing, propose
the install command and **ask before installing** — never install
unprompted. If the tool stays unavailable (declined, no network,
unsupported platform), the dimension runs manually per its charter in
[review-dimensions.md](review-dimensions.md) and the report records
the mode.

Every run has one of two scopes — pick per the review target:

- **diff-scoped** — a PR or branch review: scan what changed against
  the base ref. Faster, and findings map 1:1 to the change.
- **tree-scoped** — a codebase audit or when the diff touches
  manifests/config whose effects are global: scan the whole tree.

## Static analysis (SAST)

| | semgrep |
|---|---|
| Detect | `command -v semgrep` |
| Install | `brew install semgrep` · `pipx install semgrep` |
| Tree | `semgrep scan --config auto --json` |
| Diff | `semgrep scan --config auto --json --baseline-commit <base>` |

`--config auto` pulls community rules for the detected languages,
needs network, and **sends metrics to semgrep.dev** (machine id,
pseudoanonymized project hash) — registry-fetched configs require
them. On a private repo, ask before running it (same treatment as
trufflehog below) or use a local ruleset with `--metrics=off`.
Offline, point `--config` at a vendored ruleset or run the dimension
manually. Without `--error` these invocations exit 0 even when
findings exist, so a nonzero exit means the scan itself failed
(2 fatal, 7 bad config) — fall back to manual mode instead of
treating the failure as output.

## Dependency audit — known vulnerabilities

Default, ecosystem-agnostic:

| | osv-scanner |
|---|---|
| Detect | `command -v osv-scanner` |
| Install | `brew install osv-scanner` · `go install github.com/google/osv-scanner/v2/cmd/osv-scanner@latest` |
| Tree | `osv-scanner scan source -r .` |
| Lockfile | `osv-scanner scan source --lockfile <path>` |

Ecosystem natives — prefer them when a lockfile pins the ecosystem,
since they resolve the actual installed graph:

| Ecosystem | Detect | Install | Run |
|---|---|---|---|
| npm | `command -v npm` | ships with Node | `npm audit` (add `--omit=dev` to see the shipped surface) |
| Python | `command -v pip-audit` | `pipx install pip-audit` | `pip-audit` or `pip-audit -r requirements.txt` |
| Rust | `cargo audit --version` | `cargo install cargo-audit` | `cargo audit` |
| Go | `command -v govulncheck` | `go install golang.org/x/vuln/cmd/govulncheck@latest` | `govulncheck ./...` — call-level reachability, carry it into the finding |

## Dependency audit — freshness and maintenance

The second axis has no single scanner; combine:

- **Outdated:** `npm outdated`, `pip list --outdated`,
  `cargo outdated` (plugin), `go list -m -u all`. Flag majors behind,
  not every patch.
- **Deprecated:** `npm view <pkg> deprecated` prints the deprecation
  message if any; npm lockfiles ≥ v3 may already carry a `deprecated`
  field — grep the lockfile offline.
- **Unmaintained:** last release date (`npm view <pkg> time.modified`,
  registry/API equivalents elsewhere) and an archived upstream repo are
  the two strongest signals. Years-stale + open CVEs = finding.
- **Typosquats:** for every dependency *added* in the diff, compare the
  name character-by-character against the popular package it resembles
  (`lodash`/`1odash`, `requests`/`request`), and check the package's
  age and download count — a days-old package shadowing a famous name
  is a finding regardless of its code.

## Secrets

| | gitleaks (preferred) |
|---|---|
| Detect | `gitleaks version` — needs **≥ 8.19.0**: the `git`/`dir` subcommands landed there; older 8.x only has the deprecated `detect` |
| Install | `brew install gitleaks` |
| Diff range | `gitleaks git --log-opts="<base>..HEAD" --report-format json --report-path <out>` — the commits being merged, only |
| Full history | `gitleaks git . --report-format json --report-path <out>` |
| Working tree | `gitleaks dir . --report-format json --report-path <out>` — files on disk, no history at all |

The diff-range run covers nothing at or before `<base>`, and `dir`
covers no history: when the review's scope includes pushed history
(the secrets note in the main skill), run the full-history command —
tree-scoped audits always do. The report states the coverage actually
run.

Alternate: trufflehog (`brew install trufflehog`;
`trufflehog git file://. --since-commit <base> --json`) — its verified
results (live credential check) upgrade a finding's evidence, but
verification phones home: ask before using it on a private repo.

Manual fallback — grep the diff and reachable history for:

- key-shaped literals: `AKIA[0-9A-Z]{16}` (AWS), `ghp_`/`github_pat_`,
  `xox[baprs]-` (Slack), `sk-[A-Za-z0-9_-]{20,}`/`sk_live_` (API
  keys), `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`;
- assignments with a non-placeholder literal on the right:
  `(password|passwd|secret|token|api_key|apikey|private_key)\s*[:=]`;
- high-entropy strings in config/env files. Hashes, UUIDs, and lorem
  placeholders are the classic false positives — verification decides.

## Reading the output

Carry into each candidate finding, verbatim: the tool name and version,
the rule/advisory id (semgrep rule id, OSV/CVE/GHSA id, gitleaks rule),
the location (`file:line`, commit for history hits), and the raw
matched snippet or advisory summary. Evidence that survives into the
report must be quotable — a paraphrase of scanner output is not
evidence. Severity fields in tool output (CVSS, semgrep severity) ride
along as input to the rubric in
[verification.md](verification.md), never as the final severity.
