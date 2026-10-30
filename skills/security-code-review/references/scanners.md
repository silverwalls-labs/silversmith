# Scanners: detect, install, run

The tool matrix behind flow step 2, run once by the orchestrating
agent — dimension passes receive the resolved mode and never re-probe
or propose installs. For every tool: probe first; if missing, propose
the install command and **ask before installing** — never install
unprompted. If the tool stays unavailable (declined, no network,
unsupported platform), the dimension runs manually per its charter in
[review-dimensions.md](review-dimensions.md), recorded as `manual` —
or `degraded` when even the manual procedure can only partially run
(e.g. offline advisory lookups).

Every run has one of two scopes — pick per the review target:

- **diff-scoped** — a PR or branch review: scan what changed against
  the base ref. Faster, and findings map 1:1 to the change.
- **tree-scoped** — a codebase audit. A diff that touches a manifest
  escalates only the dependency dimension's scan (the touched lockfile
  resolves the whole installed graph), never the entire review.

## Static analysis (SAST)

| | semgrep |
|---|---|
| Detect | `command -v semgrep` |
| Install | `brew install semgrep` · `pipx install semgrep` |
| Tree | `semgrep scan --config auto --json` |
| Diff | `semgrep scan --config auto --json --baseline-commit <base>` |

`--config auto` pulls community rules for the detected languages,
needs network, and **sends metrics to semgrep.dev** (machine id,
pseudoanonymized project hash) — it refuses `--metrics=off`. Named
registry rulesets do not: on a private repo prefer a named ruleset
such as `--config p/security-audit --metrics=off` (fetches rules,
sends no metrics), and treat `auto` as off-host data (the consent
rule in the main skill). Offline, point `--config` at a ruleset
outside the reviewed tree or run the dimension manually. Without
`--error` these invocations exit 0 even when findings exist, so a
nonzero exit means the scan itself failed (2 fatal, 7 bad config) —
fall back to manual mode instead of treating the failure as output.
One failure is recoverable: `--baseline-commit` needs a git checkout
and a locally resolvable `<base>` — it handles dirty worktrees fine —
so when `<base>` does not resolve (shallow clone) or there is no
checkout, run the tree scan and keep only findings in changed files
instead of dropping to manual.

Scan policy is reviewer-owned, never diff-owned: a ruleset vendored
into the repo, a local ruleset path, and `.semgrepignore` are all
files the PR under review can edit — a diff that weakens a rule or
adds an ignore entry suppresses exactly the code it introduces. So
never point `--config` at a ruleset inside the reviewed tree. The
same goes for target selection: semgrep consults the project's
`.semgrepignore` when picking targets, and the only flags that
disable it are internal and unstable — the control is review-side.
Every `.semgrepignore` or ruleset change in the diff is itself a
candidate finding (config surface), and when it excluded code, the
coverage note records degraded coverage — never a clean pass.

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
| npm | `command -v npm` | ships with Node | `npm audit` on the full graph — a second `npm audit --omit=dev` run only *labels* the shipped surface, never replaces the full run (dev-only advisories are lowered in verification, not dropped at collection) |
| Python | `command -v pip-audit` | `pipx install pip-audit` | `pip-audit` only with the project's own venv active (bare, it audits the reviewer's environment); for manifests, `pip-audit --no-deps -r requirements.txt` — without `--no-deps` it resolves pip-style and can build/execute sdist code, breaking the read-only review. Prefer osv-scanner on a lockfile |
| Rust | `cargo audit --version` | `cargo install cargo-audit` | `cargo audit` |
| Go | `command -v govulncheck` | `go install golang.org/x/vuln/cmd/govulncheck@latest` | `govulncheck ./...` — call-level reachability, carry it into the finding |

## Dependency audit — freshness and maintenance

The second axis has no single scanner; combine:

- **Outdated:** `npm outdated`, `pip list --outdated`,
  `cargo outdated` (plugin), `go list -m -u all`. Flag majors behind,
  not every patch.
- **Deprecated:** grep the lockfile first — npm lockfiles ≥ v3 carry
  `deprecated` fields for the resolved versions; for spot checks,
  `npm view <pkg>@<installed-version> deprecated` — the bare `<pkg>`
  form reports only the latest version and misses deprecations of the
  pinned one.
- **Unmaintained:** last release date — the newest *per-version*
  timestamp in `npm view <pkg> time --json` (NOT `time.modified`,
  which deprecation edits and metadata changes bump: a dead package
  can look weeks-fresh), registry equivalents elsewhere — and an
  archived upstream repo are the two strongest signals. Years-stale +
  open CVEs = finding.
- **Typosquats:** for every dependency *added* in the diff (every
  direct dependency, on a tree-scoped review), compare the name
  character-by-character against the popular package it resembles
  (`lodash`/`1odash`, `requests`/`request`), and check the package's
  age and download count — a days-old package shadowing a famous name
  is a finding regardless of its code.

## Secrets

| | gitleaks (preferred) |
|---|---|
| Detect | `gitleaks version` — needs **≥ 8.19.1**: the `git`/`dir` subcommands landed in 8.19.0 but only work from 8.19.1; older 8.x has only the deprecated `detect` |
| Install | `brew install gitleaks` |
| Full history | `gitleaks git . --config <trusted> --gitleaks-ignore-path <tmp> --ignore-gitleaks-allow --redact --report-format json --report-path <out>` |
| Working tree | `gitleaks dir . --config <trusted> --gitleaks-ignore-path <tmp> --ignore-gitleaks-allow --redact --report-format json --report-path <out>` — files on disk, no history at all |

Scan policy is reviewer-owned here too. Gitleaks resolves config in
the order `--config`, `GITLEAKS_CONFIG`, `GITLEAKS_CONFIG_TOML`,
`(target path)/.gitleaks.toml`, then the embedded default — the
target-tree step means a `.gitleaks.toml` the diff adds or edits can
disable the rule that would catch the diff's own secret. Pass
`--config` pointing outside the reviewed tip: when the repo keeps
its own policy, extract the base ref's copy into `<tmp>`
(`git show <base>:.gitleaks.toml`); when neither base nor tip has
one, the embedded default governs untouched — but a
`.gitleaks.toml` the diff introduces is itself a candidate finding,
and the scan may not run against it: use a reviewer-owned config
(the gitleaks default config fetched at a pinned ref) or the manual
patterns below. Point `--gitleaks-ignore-path` at `<tmp>` (outside
the repo) so a diff-added `.gitleaksignore` cannot baseline-suppress
its own fingerprints, and pass `--ignore-gitleaks-allow` (probe
`gitleaks git --help`; older builds lack it) so `# gitleaks:allow`
comments in the target cannot suppress findings — without the flag,
grep the diff for them and record the dimension degraded if any
appear. Never record a tool-backed pass whose policy the diff
controlled.

Exit codes: 0 is clean; **1 means leaks found _or_ the scan errored**
(126 is only an unknown flag). On exit 1 read the report and stderr
together: entries in the report are results; an empty report plus an
error log is a failed scan — fall back to manual mode, never record a
clean pass. Write `<out>` to a temp directory outside the repo
(`mktemp -d`), never into the working tree, and keep `--redact` on —
even a redacted report left in the repo seeds the next scan. The
secrets scan is the full-history run — it subsumes a
`--log-opts="<base>..HEAD"` range run (attribute in-range hits by
commit SHA), and a range run alone misses secrets at or before
`<base>` that the main skill declares compromised. Check
`git rev-parse --is-shallow-repository` first: on a shallow clone,
`git fetch --unshallow` (a network call — consent rule applies) or
mark the dimension degraded; never report full-history coverage the
clone did not contain. Every review also runs the working-tree scan
(`dir`): uncommitted or staged diff content exists only in the
working tree — history cannot contain it — so the full-history run
alone never covers the diff of an uncommitted-change review.

Alternate: trufflehog (`brew install trufflehog`;
`trufflehog git file://. --json` — full history, same mandate and
shallow-clone check as above) — its verified results upgrade a
finding's evidence, but verification sends each
candidate secret to the issuing provider's live API (AWS, GitHub, …)
to test it: discovered secret material leaves the machine. That is
off-host data under the main skill's consent rule; pass
`--no-verification` to skip it.

Manual fallback — `grep -E` over the diff and working tree, and
`git grep -nE <pattern> $(git rev-list --all)` for history (`-n` is
required: without it git grep prints `commit:path:<match>` with no
line number; plain `grep` cannot read history at all). Full history
matches the tool mandate; any bounding — a large repo scanned only
over the review range plus a recent window — records the dimension
degraded, whatever the repo size. Guards: confirm `git rev-list`
printed at least one commit (an empty or failed substitution makes
`git grep` silently scan only the worktree and exit 0) and run the
same shallow-clone check as the tool path. The patterns are POSIX
ERE — the brace/alternation ones match nothing under BRE `grep`, and
`git grep -E` has no `\s` (use `[[:space:]]`) — and the PEM pattern
must follow `--` so its leading hyphens are not parsed as options.
Patterns:

- key-shaped literals: `AKIA[0-9A-Z]{16}` (AWS), `ghp_`/`github_pat_`,
  `xox[baprs]-` (Slack), `sk-[A-Za-z0-9_-]{20,}`/`sk_live_` (API
  keys), `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`;
- assignments with a non-placeholder literal on the right:
  `(password|passwd|secret|token|api_key|apikey|private_key)[[:space:]]*[:=]`;
- high-entropy strings in config/env files. Hashes, UUIDs, and lorem
  placeholders are the classic false positives — verification decides.

## Reading the output

Carry into each candidate finding, verbatim: the tool name and version,
the rule/advisory id (semgrep rule id, OSV/CVE/GHSA id, gitleaks rule),
the location (`file:line`, commit for history hits), and the raw
matched snippet or advisory summary. Evidence that survives into the
report must be quotable — a paraphrase of scanner output is not
evidence. One exception: secret material itself is never carried or
quoted — a secrets candidate holds the rule id, the location, the
secret's length, and at most the detector-identifying prefix (`AKIA`,
`ghp_`), never any other characters of the value. The `--redact`
report plus the location is the whole evidence; never re-open the
commit to extract the value. Severity fields in tool output (CVSS, semgrep severity) ride
along as input to the rubric in
[verification.md](verification.md), never as the final severity.
