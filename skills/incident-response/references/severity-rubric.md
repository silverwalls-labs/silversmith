# Severity rubric

Classify by **scope**, **data class**, and **production impact** — not
by who reported it or how urgently they phrased the report.

| Severity | Scope | Data / impact | Response time | Examples |
|---|---|---|---|---|
| **SEV0 — Critical** | Active exploitation or imminent threat; production data at risk | Credentials, PII, financial data actively exposed or exfiltrated; full service outage caused by the incident | Page on-call immediately; triage within 15 min | Active breach with data exfiltration; leaked production database credentials appearing in public paste sites; RCE in a public-facing service being exploited in the wild |
| **SEV1 — High** | Confirmed vulnerability with a clear path to exploitation; sensitive data exposed but not yet confirmed exfiltrated | Credentials, PII, or secrets exposed in a reachable artifact; significant service degradation caused by the incident | Page on-call immediately; triage within 30 min | Leaked API key with production write access found in a public repo (no confirmed use yet); critical CVE in a direct dependency with a public exploit and the vulnerable code path is reachable; secret committed to a public branch |
| **SEV2 — Medium** | Confirmed vulnerability without a known active exploit; limited data exposure | Internal secrets, non-sensitive data, or configuration exposed; no customer data involved; no production impact | Business hours; triage within 4 hours | High-severity CVE in a dependency where the vulnerable code path is not directly reachable but is one config change away; internal service credentials leaked in an internal-only repository; test/staging credentials exposed |
| **SEV3 — Low** | Potential vulnerability or hygiene issue; no confirmed exposure | No sensitive data involved; no production impact; defense-in-depth improvement | Business hours; triage within 1 business day | Informational CVE in a transitive dependency with no reachable path; outdated dependency with known vulnerability but behind multiple mitigating controls; missing security header on an internal-only endpoint |

## Choosing a severity

Work top-down: start at SEV0 and ask "does this match?" If not, move
to SEV1, then SEV2, then SEV3. When in doubt between two levels,
choose the higher one — you can always downgrade after triage, but
under-classifying delays response.

Key discriminators:

- **Active exploitation vs. potential:** if the vulnerability is being
  actively exploited or data is confirmed exposed, it is SEV0 or
  SEV1. If it is a vulnerability with no confirmed exploitation, it
  is SEV2 or SEV3.
- **Data sensitivity:** credentials, PII, and financial data push
  severity up. Internal config, test data, and non-sensitive metadata
  push it down.
- **Blast radius:** the more systems, users, or customers affected,
  the higher the severity. A single internal service is lower than a
  customer-facing API.
- **Containment difficulty:** if containment requires coordinating
  with external parties, revoking widely-distributed credentials, or
  taking production systems offline, that pushes severity up.

## Escalation and downgrade

- Any team member can **escalate** severity at any time without
  approval — just update the incident channel and page accordingly.
- **Downgrading** requires the incident owner's sign-off and a
  documented reason in the timeline. Never downgrade mid-incident
  just to reduce the notification burden.
