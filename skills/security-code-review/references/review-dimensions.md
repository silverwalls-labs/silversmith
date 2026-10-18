# Review dimensions

The four passes behind flow step 3. Each section below is a charter:
a spawned subagent's brief is the Common contract plus its own
section, verbatim; when running sequentially, execute the charters
inline.

## Common contract (every dimension)

- **Inputs:** the scope header from flow step 1 (ref/diff, scan scope
  — diff- or tree-scoped — file list, manifests, blast radius) and the
  dimension's resolved mode from flow step 2 (tool-backed, manual, or
  degraded, with the tool name if any) — tool probing and install
  proposals already happened there; never redo them in the pass.
- **Output:** a list of *candidate* findings — each with location,
  description, and raw evidence (scanner excerpt, code quote, or traced
  flow). **No severity, no verdict** — the verification pass owns both.
  Also report the dimension's coverage: what was scanned, what was not.
- **No silent skips:** if part of the charter could not run (tool
  failed, path unreadable, timeout), say so in the coverage note.
- Candidates are cheap and dismissals are recorded, so when in doubt,
  **emit the candidate** — verification filters, collection does not.

## 1. Static analysis

Run the SAST tool per [scanners.md](scanners.md) at the right scope;
carry every hit forward as a candidate with its rule id and snippet.

Manual mode: walk the diff against the dangerous-sink list and flag
every flow from external input into one of them:

- code execution: `eval`, `exec`, `Function()`, `child_process`/
  `subprocess`/`os.system` with built strings, deserialization of
  untrusted bytes (`pickle`, `yaml.load`, Java native, `unserialize`);
- SQL/NoSQL built by string concatenation or interpolation;
- template rendering of user input outside auto-escaping;
- path operations joining user input (`path.join`, `open`, archive
  extraction) without canonicalization checks;
- outbound requests to URLs derived from user input (SSRF);
- XML parsers with external entities left enabled;
- HTML sinks (`innerHTML`, `dangerouslySetInnerHTML`, `v-html`).

## 2. Dependency audit

Both axes, every manifest and lockfile the diff touches (all of them on
a tree-scoped review):

- **Known vulnerabilities:** run osv-scanner and/or the ecosystem
  native per [scanners.md](scanners.md). Each advisory hit becomes a
  candidate carrying the advisory id, the affected version range, the
  installed version, and — when the advisory names vulnerable
  functions — whether the project calls them. Manual mode (tool
  unavailable but network up): look up each direct dependency at its
  pinned version against the advisory sources (osv.dev, GitHub
  advisories, the registry's advisory page), diff-touched manifests
  first; offline, inspect the lockfile instead (deprecation markers,
  ancient pins) and mark the axis degraded.
- **Maintenance health:** outdated majors, deprecated packages,
  archived or years-stale upstreams, and unmaintained packages in the
  transitive tree that sit on a security boundary (parsers, crypto,
  auth middleware).
- **New dependencies get extra scrutiny:** a dependency *added* in the
  diff (every direct dependency, on a tree-scoped review) is checked
  for typosquatting, age, download count, install scripts
  (`preinstall`/`postinstall`), and whether the need justifies the
  tree it brings in. "Could this be twenty lines of code instead?"
  is a legitimate review question.

## 3. Secrets

Run the secrets tool on the diff range *and* the reachable history per
[scanners.md](scanners.md); manual mode uses the fallback patterns
there.

- Candidates: credentials, API keys and tokens, private keys,
  signed/presigned URLs, session cookies, cloud account ids paired
  with keys.
- Judgment calls to pass to verification rather than drop: internal
  hostnames and IPs, employee emails in fixtures, example keys that
  look too real.
- For every hit, record the commit that introduced it. Rotation is the
  remediation in every case — any committed secret is compromised (the
  secrets rule in the main skill); the commit's location changes only
  the cleanup step: unpushed, amend/rebase it out before pushing;
  pushed, history cleanup on top of the rotation.

## 4. Manual security review

Always runs, even when every scanner is available — tools do not see
missing authorization checks or business-logic flaws. Walk the diff
class by class; on a tree-scoped audit, walk the security-sensitive
surfaces instead (auth, crypto, input handling, CI and workflow
config, dependency manifests) and read change-relative checks below as
"anywhere present":

- **Injection:** any user-controlled value reaching the sinks in
  dimension 1 via a path a SAST rule might not model (ORM `raw()`
  escapes, shell-outs behind helpers, log injection).
- **Authentication:** new endpoints/routes without the auth middleware
  the rest of the module uses; weakened checks (`== null` vs role
  check); credentials compared with non-constant-time equality.
- **Authorization / IDOR:** object ids from the request used without an
  ownership check; privilege boundaries crossed in changed code paths;
  mass assignment of privileged fields.
- **Crypto misuse:** home-rolled primitives, ECB mode, static IVs or
  salts, `MD5`/`SHA-1` for passwords (want argon2/bcrypt/scrypt),
  `random` where `secrets`/CSPRNG is needed, disabled TLS verification.
- **Input validation:** length/type/range checks missing at trust
  boundaries; canonicalization after validation instead of before;
  client-side-only validation.
- **Error handling and logging:** secrets or PII in logs; stack traces
  or internal paths returned to the client; `catch` blocks that
  swallow security failures.
- **Config and CI:** workflow files gaining write permissions or
  secrets access; `curl | sh` installs; containers running as root
  where they did not before.

### AI-generated code

Apply with extra weight when the diff is known or suspected to be
AI-authored (including your own output). Fluent and plausible is not
correct:

- **Hallucinated APIs:** for every unfamiliar call, grep the actual
  dependency source or docs — does the function exist, with that
  signature, in the pinned version? Deprecated-presented-as-current
  counts.
- **Plausible-but-wrong logic:** the code reads right but handles the
  wrong edge case. Verify boundary behavior against the real API, not
  the comment that claims it.
- **Phantom tests:** would the test fail if the code broke? Tests that
  restate the implementation, mock the behavior under test, or assert
  nothing are candidates, not coverage.
- **Fabricated security claims:** comments and docstrings asserting
  "sanitized upstream", "constant-time", "safe because X" are claims to
  verify, never evidence. If the claim is false, that is itself a
  finding.
