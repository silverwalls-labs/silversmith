# Report template

The output contract behind flow step 5. Copy the skeletons, fill them,
keep the order: findings sorted severity-descending, `confirmed` before
`plausible` within a severity. The report is tracker-agnostic — file it
wherever the project tracks review output, or emit it inline.

## Report skeleton

```markdown
# Security review: <target>

- **Scope:** <ref/PR/diff range> against <base>; <N> files;
  <diff- or tree-scoped>
- **Blast radius:** <callers of changed code, configuration it reads,
  dependencies added or bumped>
- **Reviewed:** <paths/areas covered>
- **Not reviewed:** <explicit gaps and why>
- **Date:** <YYYY-MM-DD>

## Dimension status

| Dimension | Mode | Tool (version) | Notes |
|---|---|---|---|
| Static analysis | tool-backed | semgrep 1.x | |
| Dependency audit | degraded | — | no network; lockfile-only |
| Secrets | tool-backed | gitleaks 8.24 | range `<base>..HEAD` + full history |
| Manual review | manual | — | always manual |

## Findings

| ID | Title | Severity | Verdict | Dimension | Location |
|---|---|---|---|---|---|
| SCR-1 | ... | high | confirmed | ... | file:line |

<one finding record per entry, template below>

## Dismissed

| ID | Source | Location | Reasoning |
|---|---|---|---|
| SCR-7 | semgrep <rule-id> | file:line @ ref | match is in a test fixture; sink not in shipped code |
```

## Finding record

```markdown
### SCR-<n>: <title>

- **Severity / verdict:** <severity> / <verdict>
- **Dimension / source:** <dimension>; <tool + rule/advisory id, or "manual">
- **Location:** <file:line @ ref, or commit for history hits>

**Description.** <what the flaw is, one short paragraph>

**Evidence.** <raw scanner excerpt and/or quoted code and the traced
path from input to sink — the load-bearing section; quotes, not
paraphrase>

**Impact.** <what an attacker gets, under which preconditions>

**Remediation.** <the concrete change; for secrets: rotation steps,
then history cleanup, then the code change>

**References.** <CWE-xxx, CVE/GHSA/OSV id, rule docs>
```

## Filled example

```markdown
### SCR-1: SQL built by string interpolation in order lookup

- **Severity / verdict:** high / confirmed
- **Dimension / source:** static analysis; semgrep
  `python.lang.security.audit.formatted-sql-query.formatted-sql-query`
  + manual trace
- **Location:** `api/orders.py:84,91` @ `feat/order-filters` — the
  `sort` interpolation at :91 is the same root cause, merged

**Description.** The new `status` filter is interpolated into the SQL
string instead of being passed as a bound parameter.

**Evidence.**
    query = f"SELECT * FROM orders WHERE status = '{status}'"
`status` comes from `request.args["status"]` (`api/orders.py:78`) with
no validation; the route is registered unauthenticated
(`api/routes.py:12`). Reachability traced end to end.

**Impact.** Unauthenticated SQL injection: full read of the orders
table, and write if the driver allows stacked queries.

**Remediation.** Bind the parameter:
`cursor.execute("SELECT * FROM orders WHERE status = %s", (status,))`.
The merged `sort` location (`api/orders.py:91`) is a column name, so
binding does not apply — validate it against an allowlist. Fix both.

**References.** CWE-89.
```
