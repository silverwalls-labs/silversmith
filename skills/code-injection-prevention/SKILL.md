---
name: code-injection-prevention
description: Injection-prevention guardrails for code that handles untrusted input. Use when writing or reviewing code that builds SQL, shells out, opens user-supplied file paths, makes outbound HTTP requests, or renders templates with external data.
---

# code-injection-prevention

Prevent injection vulnerabilities in code that processes external input:
SQL injection, command injection, path traversal, SSRF, and server-side
template injection (SSTI). The governing principle is simple — treat all
external input (request parameters, headers, file contents, environment
variables, data from another trust domain) as untrusted and enforce the
safe pattern at the trust boundary, not downstream.

The scope is the five injection classes above. Adjacent concerns —
authentication, authorization, cryptography, XSS output encoding — are
out of scope and belong to their own skills or the project's own policy.

## Rules

Normative. Each MUST/NEVER doubles as a review check: to review code
that handles untrusted input, walk this list top to bottom and flag
every violation, even ones you were not asked about.

- **MUST treat every external input as untrusted.** Request parameters,
  headers, cookies, file uploads, environment variables, messages from
  queues, and data read from a database populated by another trust domain
  are all untrusted. Validate, parse, and constrain at each trust
  boundary — both at ingestion and again at consumption when data
  crosses between components (see Notes: "Stored input").
- **MUST use parameterized queries or prepared statements for every SQL
  query.** The query text is a static string with placeholders; values
  are bound separately by the driver. This applies to all SQL dialects
  and all access methods — raw connections, ORMs' raw-query escape
  hatches, and query builders alike.
- **NEVER build SQL via string concatenation, f-strings, `format()`,
  `%`-formatting, or template literals with untrusted values.** Even
  "just an integer" is not safe to interpolate — the input may not be
  what it claims.
- **MUST pass external commands as argument lists**, not shell strings.
  Use `subprocess.run(["cmd", arg1, arg2])` (Python),
  `execFile("cmd", [arg1, arg2])` (Node.js), or the equivalent in the
  target language — the OS receives each argument as a discrete value,
  bypassing shell parsing entirely.
- **NEVER combine shell invocation with user-derived input.** `shell=True`
  (Python), `child_process.exec()` (Node.js), `system()` (C/Ruby), or
  backtick interpolation turns an argument into an attacker-controlled
  command. If a shell feature is genuinely required (pipes, globs),
  hardcode the shell fragment and pass untrusted values only as
  arguments — never as part of the command string.
- **MUST canonicalize user-supplied paths and verify containment.**
  Resolve the path to its canonical absolute form — `os.path.realpath`
  (Python), `fs.realpathSync` (Node.js), `filepath.EvalSymlinks` (Go) —
  and confirm it starts with the allowed root directory. Rejecting `..`
  segments or checking for path separators alone is insufficient — symlinks
  and encoding tricks bypass string-level checks.
- **NEVER serve, read, write, or delete a file at a path derived from
  untrusted input without canonicalization and containment checks.** This
  includes paths assembled from multiple inputs — canonicalize the final
  path, not the individual segments.
- **MUST validate the resolved IP address of outbound requests before
  connecting** when the destination is influenced by external input.
  Resolve the hostname, then check the resulting address against the
  blocked ranges — do not trust the hostname alone. Re-validate on
  redirect; a safe hostname can redirect to an internal IP.
- **NEVER allow outbound requests to private (RFC 1918 / RFC 4193),
  loopback (`127.0.0.0/8`, `::1`), link-local (`169.254.0.0/16`,
  `fe80::/10`), or cloud metadata endpoints (`169.254.169.254`,
  `fd00:ec2::254`).** Prefer an allowlist of permitted destinations
  over a denylist of blocked ranges — a denylist is always one range
  behind.
- **MUST pass user input as template data (variables), never as template
  source.** The template string itself is authored code; external input
  fills its placeholders. This applies to every template engine — Jinja2,
  Handlebars, ERB, Thymeleaf, Twig, Go `html/template`, etc.
- **NEVER render, compile, or evaluate a user-supplied string as template
  code.** `Template(user_input).render()` or
  `Handlebars.compile(user_input)` hands the attacker the full power of
  the template engine — from reading server variables to remote code
  execution.

## Injection classes

Open only the reference the current code touches.

| Class | Reference | One-line summary |
|---|---|---|
| SQL injection | [references/sql-injection.md](references/sql-injection.md) | Parameterized queries; never interpolate |
| Command injection | [references/command-injection.md](references/command-injection.md) | Argument lists; never shell-parse untrusted input |
| Path traversal | [references/path-traversal.md](references/path-traversal.md) | Canonicalize then confine to an allowed root |
| SSRF | [references/ssrf.md](references/ssrf.md) | Validate resolved IPs; block private/link-local/metadata |
| Template injection | [references/template-injection.md](references/template-injection.md) | Data goes in variables; user input is never template source |

## Notes and edge cases

- **Defense in depth:** a single layer can fail. Combine input validation,
  parameterized APIs, and least privilege — a database user without DDL
  rights, a process without a shell, network egress rules blocking
  internal ranges — so a missed check is not the whole story.
- **ORM raw-query escape hatches:** ORMs parameterize by default, but
  `raw()`, `execute()`, `$queryRaw`, and similar methods reintroduce
  injection risk. Apply the same parameterization rules to every raw-SQL
  call — the ORM wrapper does not protect it.
- **Stored input is still untrusted:** data written by one component and
  read by another crosses a trust boundary at the read. A value that was
  "validated on write" can be stale, re-encoded, or from a compromised
  writer — validate at the point of consumption, not only at ingestion.
- **Encoding is not sanitization:** URL-encoding, HTML-encoding, and
  escaping serve specific output contexts (browser rendering, URL
  construction). They do not make input safe for SQL, commands, or file
  paths — use the right safe API for each context.
- **Multi-step resolution (SSRF):** a hostname can resolve to a private IP,
  or a first request can redirect to an internal target. Validate the
  resolved address after DNS resolution and re-validate on every redirect
  — checking the hostname string is not sufficient.
- **Allowlists beat denylists:** for both SSRF destinations and command
  arguments, an allowlist of known-good values is more durable than a
  denylist of known-bad patterns — the next bypass is always one step
  ahead of the denylist.
