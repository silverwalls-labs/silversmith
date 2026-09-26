# Incident timeline template

Copy this template at incident declaration. Every action gets a row —
the timeline is the postmortem's spine and the compliance record. Use
UTC timestamps. The incident owner is responsible for keeping it
current; anyone on the response team can add rows.

## Incident: [TITLE]

**Severity:** SEV[0-3]
**Incident owner:** [NAME]
**Declared:** [YYYY-MM-DD HH:MM UTC]
**Resolved:** [YYYY-MM-DD HH:MM UTC]

| Timestamp (UTC) | Actor | Action | Notes |
|---|---|---|---|
| YYYY-MM-DD HH:MM | [who] | **Detection** — incident reported / alert fired | Source: [alert name / reporter / CVE ID] |
| YYYY-MM-DD HH:MM | [who] | **Triage** — severity classified as SEV[N] | Rationale: [one line] |
| YYYY-MM-DD HH:MM | [who] | **Owner declared** — [name] assigned as incident owner | |
| YYYY-MM-DD HH:MM | [who] | **Paged** — on-call / stakeholders notified | Channel: [link] |
| YYYY-MM-DD HH:MM | [who] | **Evidence preserved** — [what was captured] | Snapshots, logs, artifacts |
| YYYY-MM-DD HH:MM | [who] | **Containment** — [action taken] | e.g., secret rotated, host isolated, IP blocked |
| YYYY-MM-DD HH:MM | [who] | **Eradication** — [vulnerability removed] | e.g., patch applied, package removed, config changed |
| YYYY-MM-DD HH:MM | [who] | **Verification** — attack vector confirmed closed | How verified: [method] |
| YYYY-MM-DD HH:MM | [who] | **Recovery** — service restored, monitoring confirmed | |
| YYYY-MM-DD HH:MM | [who] | **Disclosure** — advisory filed / customers notified | GHSA/CVE ID, notification sent |
| YYYY-MM-DD HH:MM | [who] | **Postmortem scheduled** — [date] | |

## How to maintain the timeline

- **Add rows in real time**, not after the fact. If you forget, add
  the row with your best-effort timestamp and mark it `(approximate)`.
- **Never delete rows.** If an action was reversed or a classification
  changed, add a new row — the history of changes is the record.
- **One action per row.** "Rotated secret and blocked IP" is two rows.
- **Actor is a person**, not a system. If an automated system took the
  action, name the person who triggered or approved it.
- **Include links** to PRs, commits, advisories, and alerts in the
  Notes column — the postmortem author will thank you.
