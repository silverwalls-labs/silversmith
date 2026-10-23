---
name: security-code-review
description: Tool-backed, multi-pass security review of code changes. Use when reviewing a PR, diff, branch, or codebase with a security lens, or when asked for a security review or audit.
---

# security-code-review

Enforce evidence-backed security review of code: the review fans out
into focused dimensions (static analysis, dependency audit, secrets,
manual review), grounds findings in real scanner output where tooling
exists, adversarially verifies every candidate finding for reachability
and impact, and reports them severity-ranked with evidence and
remediation. The scope is finding and reporting security issues in the
code under review — review-process etiquette (PR sizing, comment tone,
approval flow), applying the fixes, and ongoing triage of a scanner
backlog are out of scope. The review itself is read-only: scanners run,
the code does not change.

Input is a PR, diff, branch, or whole tree; output is one structured
report ([references/report-template.md](references/report-template.md)).

## Rules

Normative. Each MUST/NEVER doubles as a review check: to run a security
review or audit one that was already done, walk this list top to bottom
and flag every violation, even ones you were not asked about.

- **MUST state the scope before reviewing:** the exact ref, diff, or
  paths under review plus the blast radius — callers of changed code,
  configuration it reads, dependencies it adds or bumps. A report with
  a missing or vague scope section is a violation.
- **MUST run every dimension** (static analysis, dependency audit,
  secrets, manual review). A dimension with no usable tool runs as a
  manual pass; a dimension may be skipped only explicitly in the report
  with a reason, NEVER silently.
- **MUST ground findings in evidence:** scanner output, a quoted code
  excerpt, or a traced data flow. A finding backed only by "this looks
  unsafe" is reported as `plausible` at best, never `confirmed`.
- **MUST detect tools before relying on them, and NEVER install
  anything unprompted:** probe for each scanner, propose the install
  command for missing ones
  ([references/scanners.md](references/scanners.md)), and ask before
  installing; on decline, fall back to manual analysis for that
  dimension, recording the mode as `degraded` when even the manual
  procedure can only partially run (e.g. offline).
- **MUST ask before any command that transmits project data off-host
  from a private repo** — the class, not a fixed list: scanner
  telemetry (semgrep `--config auto`), advisory queries carrying
  dependency names (`npm audit`, osv-scanner, pip-audit, govulncheck),
  registry freshness lookups (`npm outdated`/`npm view`,
  `pip list --outdated`, `go list -m -u all`), live credential
  verification (trufflehog), manual advisory lookups. Consent is
  gathered once, at flow step 2; a declined category runs
  offline-degraded, and dimension passes never ask or exceed it.
- **MUST treat scanner output as leads, not conclusions:** every raw
  hit passes through the verification pass
  ([references/verification.md](references/verification.md)) before it
  may appear in the report.
- **MUST attach a verdict to every finding** — `confirmed` /
  `plausible` / `false-positive` — **and a severity to every reported
  one** (dismissed `false-positive` entries carry no severity), from
  the impact-and-exploitability rubric, never from a tool's default
  score alone.
- **MUST NOT drop a finding silently.** A dismissed hit becomes a
  `false-positive` entry with the reasoning recorded in the report's
  dismissed appendix, so the dismissal itself is reviewable.
- **MUST audit dependencies on two axes:** known vulnerabilities
  (OSV/advisory data) AND maintenance health — outdated majors,
  deprecated or archived packages, unmaintained transitive
  dependencies on security boundaries — in every touched manifest and
  lockfile, plus a typosquat check on every dependency the diff adds
  (every direct dependency, on a tree-scoped review).
- **MUST treat any committed secret as compromised.** The remediation
  is rotation plus history cleanup; deleting the line in a follow-up
  commit is not a fix and does not lower the severity.
- **MUST apply heightened scrutiny to AI-generated or AI-assisted
  code:** verify that every unfamiliar API exists with that signature,
  that the logic matches real behavior rather than its comments, that
  the tests would fail if the code broke, and that security claims in
  comments are verified rather than trusted — the AI section of
  [references/review-dimensions.md](references/review-dimensions.md).

## Review flow

1. **Scope the target.** Resolve what is under review (PR number, diff
   range, branch vs base, or full tree) and fix the scan scope —
   diff-scoped or tree-scoped; enumerate changed files, languages, and
   manifests; sketch the blast radius. This becomes the report's scope
   header.
2. **Detect tooling.** Probe for each dimension's scanners, propose
   installs for missing ones (and ask), gather consent for anything
   that sends project data off-host, and record each dimension's
   mode — `tool-backed`, `manual`, or `degraded` (partial coverage,
   e.g. offline), with `skipped` plus a reason as the explicit last
   resort: [references/scanners.md](references/scanners.md). Probing,
   install proposals, and consent happen here, once — dimension passes
   receive the resolved modes and commands and never re-probe,
   propose, or ask.
3. **Fan out the dimensions.** If the harness supports spawning
   subagents, run the four passes in parallel, one focused agent per
   dimension; otherwise run the same passes sequentially in order.
   Each pass follows its charter in
   [references/review-dimensions.md](references/review-dimensions.md)
   — a subagent's brief carries the charter plus its dimension's
   resolved commands and output-reading notes, since the skill's files
   are not in the reviewed repo — and returns raw candidate findings
   with evidence, no severity yet.
4. **Verify adversarially.** Re-examine every candidate — reachability,
   exploit preconditions, known false-positive patterns — and assign
   verdict and severity:
   [references/verification.md](references/verification.md).
5. **Report.** Emit the severity-ranked structured report, including
   each dimension's mode (tool-backed / manual / degraded /
   skipped-with-reason) and the dismissed-findings appendix:
   [references/report-template.md](references/report-template.md).

## Notes and edge cases

- **No network:** skip install proposals and online advisory lookups;
  the dependency dimension degrades to manifest/lockfile inspection
  (pinned-version hygiene, deprecation markers already in the lockfile,
  typosquats by eye) and the report marks it degraded instead of faking
  coverage.
- **Harness without subagents:** nothing changes but concurrency — the
  same four passes, same order, same outputs. Never collapse the
  dimensions into one undifferentiated pass to save time.
- **Huge diffs and monorepos:** partition by package or top-level
  directory and review security-sensitive surfaces first (auth, crypto,
  input handling, CI and workflow config, dependency manifests). State
  in the report what was deprioritized or not reviewed — an unstated
  gap is worse than a stated one.
- **Generated and vendored code:** exempt from code-quality findings
  in the manual pass — never from secrets or dependency scanning, and
  never from the config-and-CI checks: a generated workflow gaining
  secrets access is in scope and its candidate is still emitted
  (verification filters, collection does not). A vendored library
  with a known CVE is a finding; its code style is not.
- **Reviewing your own AI-authored diff:** treat it as third-party
  code. The AI-code scrutiny applies to your own output, and the
  verification pass may not be skipped on the grounds that you just
  wrote it.
- **Secrets in history:** a secret reachable anywhere in the pushed
  history is compromised even if the tip is clean — the rotation rule
  applies to history, not just the diff under review.
- **Scanner disagreement:** two tools flagging the same location merge
  into one finding listing both sources; when they disagree, the
  verification pass decides — tool consensus is corroboration, not a
  verdict.
