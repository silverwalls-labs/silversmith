# Verification pass

The adversarial step behind flow step 4. Every candidate from every
dimension goes through it; nothing reaches the report without a
verdict, and nothing is reported as a finding without a severity
(dismissed `false-positive` entries carry none). The stance is
adversarial in both directions: try to
*break* the finding (prove it unreachable) and try to *land* it (trace
the exploit path) — whichever succeeds decides.

## Verdicts

- **`confirmed`** — exploitability demonstrated, or reachability traced
  end to end with the preconditions shown to hold in this codebase and
  configuration.
- **`plausible`** — the pattern is real but reachability could not be
  fully determined (dynamic dispatch, config outside the repo, time
  box). Fails toward reporting: a `plausible` finding ships, with what
  could not be determined stated explicitly.
- **`false-positive`** — dismissed, with the reasoning recorded in the
  report's dismissed appendix. Never deleted: the dismissal itself is
  reviewable, and "it's a test key" written down can be challenged.

## Procedure, per finding

1. **Shipped or not?** Is the flagged code in the artifact that ships,
   or only in tests, examples, or build tooling? Test-only lowers
   severity one notch (a modifier, below the rubric) — except secrets
   (compromised anywhere) and CI code (the pipeline *is* an attack
   surface). Generated and vendored code: code-quality findings are
   dismissed as `false-positive` (out of scope by policy, recorded in
   the appendix); secrets, dependency, and CI/config findings are
   untouched — a vendored library's known CVE stays reportable.
2. **Reachable?** Trace the actual call path from attacker-influenced
   input to the sink. Do not assume a route is exposed — find the
   registration; do not assume input is attacker-controlled — find
   where it enters.
3. **Preconditions real?** Framework auto-escaping, a parameterized
   driver, sandboxing, or an upstream allowlist may neutralize the
   pattern. Verify the mitigation is actually active in this config —
   a mitigation that *could* be enabled is not one that *is*.
4. **Dependency findings: is the vulnerable code called?** When the
   advisory names vulnerable functions, check whether the project
   reaches them (govulncheck does this natively). Not-called lowers
   severity, it does not dismiss — the dependency is still carrying
   the vuln into the tree.
5. **Dedupe and merge:** same root cause flagged by several tools or
   in several places becomes one finding with all sources and
   locations listed.

## False-positive heuristics, per source

- **SAST (semgrep):** hits in test fixtures and documentation examples;
  flows already sanitized upstream of the match; sinks the framework
  mitigates by default (auto-escaping templates, ORM parameterization).
  Each of these is a *reason to check*, not an auto-dismiss — confirm
  the sanitizer/mitigation on the actual path.
- **Secrets scanners:** documented example/placeholder keys, high
  entropy non-secrets (hashes, UUIDs, base64 assets), dummy test
  credentials. "It's a test key" requires proof — a comment saying so,
  a `test`/`example` path, *and* the key not resembling a live format;
  when provable, still note it, since real keys get pasted into test
  files constantly.
- **Dependency audits:** advisory withdrawn; affected-range misparse;
  dev-only dependency flagged against the shipped surface (lower, not
  dismiss — the dev machine and CI are surfaces too).

## Severity rubric

Severity = impact × exploitability, decided here — a tool's CVSS or
severity label is an input, never the output.

| | Trivially exploitable (unauth, no preconditions) | Exploitable with effort (auth, timing, chaining) | Hard (local access, unlikely config) |
|---|---|---|---|
| **Full compromise** (RCE, auth bypass, secret with broad access) | critical | high | medium |
| **Serious** (data exposure, privilege escalation, SSRF into internal net) | high | medium | low |
| **Limited** (info leak, DoS of one endpoint, missing hardening) | medium | low | info |

Modifiers — applied in this order: lowering adjusters, then the
verdict drop, floors last.

- Lowering adjusters — **one notch each**, apply every one that
  matches, never below `info`: test-only code (step 1) and vulnerable
  function not called (step 4). The maintenance-health rule is a
  ceiling, not a notch: outdated, deprecated, or unmaintained with no
  live advisory lands at **low** or **info** — real, reportable, not
  urgent.
- Verdict drop: a `plausible` finding then drops one further notch; at
  `info` it stays `info` — there is no lower notch, and `info` already
  means report-only. `false-positive` has no severity.
- Floors win: a committed secret starts at **high** (critical if the
  credential is broad) regardless of where it sits — compromise is
  assumed — and overrides every adjuster and drop above.
