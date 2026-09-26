# Blameless postmortem template

Copy this template and fill it in within **one week** of incident
resolution. The postmortem is the artifact that makes the next
incident shorter — skip nothing.

**Blameless means system-focused:** if a human made an error, the root
cause is the system that allowed that error — the missing guard rail,
the unclear runbook, the absent automation. "X clicked the wrong
button" is not a root cause; "the button was next to the delete button
with no confirmation dialog" is.

---

## Postmortem: [INCIDENT TITLE]

**Date:** YYYY-MM-DD
**Severity:** SEV[0-3]
**Incident owner:** [NAME]
**Postmortem author:** [NAME]
**Postmortem attendees:** [NAMES]

---

### Summary

_One paragraph: what happened, when, how long it lasted, and the
bottom-line impact. A reader who reads only this paragraph should
understand the incident._

### Timeline

_Copy from the incident timeline
([timeline-template.md](timeline-template.md)). Add any entries
discovered during the postmortem investigation. Keep UTC timestamps._

| Timestamp (UTC) | Actor | Action | Notes |
|---|---|---|---|
| | | | |

### Impact

_Quantify what was affected:_

- **Duration:** total time from detection to resolution.
- **Users / customers affected:** number and segment.
- **Data affected:** what data types, how many records, sensitivity
  classification.
- **Service impact:** downtime, degraded performance, feature
  unavailability.
- **Financial impact:** if quantifiable or estimable.
- **Compliance impact:** missed notification clocks, regulatory
  exposure.

### Root cause

_System cause, not person cause. Use the "5 whys" or a causal tree to
get past the proximate trigger to the systemic gap._

**Proximate cause:** _what directly caused the incident (the event)._

**Contributing factors:** _what allowed the proximate cause to have
the impact it did (the gaps)._

**Root cause:** _the deepest systemic issue — the one where a fix
would have prevented this class of incident, not just this instance._

### What went well

_What worked during the response — detection, containment,
communication, tooling, team coordination. Recognizing what worked
is as important as fixing what didn't; it prevents regressing._

- _e.g., "Alert fired within 2 minutes of the anomaly."_
- _e.g., "Secret rotation was automated and completed in under 5
  minutes."_

### What didn't go well

_What failed, was slow, or was missing — and the systemic reason, not
a person. Each item here should map to at least one action item below._

- _e.g., "Triage took 45 minutes because the on-call runbook didn't
  cover this alert type."_
- _e.g., "Customer notification was delayed because the disclosure
  template didn't exist yet."_

### Action items

_Every item has an owner and a due date. Track to closure. An action
item without an owner is a wish._

| # | Action | Owner | Due date | Status |
|---|---|---|---|---|
| 1 | _e.g., Add runbook entry for [alert type]_ | [NAME] | YYYY-MM-DD | Open |
| 2 | _e.g., Automate secret rotation for [service]_ | [NAME] | YYYY-MM-DD | Open |
| 3 | _e.g., Create disclosure notification template_ | [NAME] | YYYY-MM-DD | Open |
| 4 | _e.g., Add confirmation dialog to [dangerous action]_ | [NAME] | YYYY-MM-DD | Open |

### Appendix

_Links to supporting artifacts: incident channel, timeline, alerts,
PRs, advisories, logs, forensic snapshots. The appendix is the
evidence locker._

- Incident channel: [link]
- Timeline: [link]
- Security advisory: [GHSA / CVE link]
- Fix PR: [link]
- Relevant logs: [link / location]

---

### Postmortem review

- [ ] Reviewed by incident owner
- [ ] Reviewed by engineering lead
- [ ] Action items entered into issue tracker with owners and due dates
- [ ] Published internally
- [ ] Published externally (SEV0/SEV1 only, if applicable)
