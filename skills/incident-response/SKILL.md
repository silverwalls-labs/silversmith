---
name: incident-response
description: Security incident response. Use when triaging or responding to a security incident — a CVE landing, a secret leaked, a breach suspected, a service compromised.
---

# incident-response

Enforce a structured response to security incidents: severity-based
triage, containment before root cause, evidence preservation,
compliance-driven communication, coordinated disclosure, and blameless
postmortems. This is the operational counterpart to supply-chain and
secrets-handling skills — those prevent, this one responds.

## Rules

Normative. Each MUST/NEVER doubles as a review check: when helping
respond to an incident, walk this list top to bottom and flag every
violation, even ones you were not asked about.

### Severity & triage

- **MUST classify by scope, data class, and production impact** — not
  by who reported it or how loudly. Severity rubric:
  [references/severity-rubric.md](references/severity-rubric.md).
- **MUST declare an incident owner immediately.** An incident without
  a declared owner drifts. The owner drives the timeline, not the fix.
- **MUST page by severity:** SEV0/SEV1 page on-call immediately;
  SEV2/SEV3 can wait for business hours. Know the escalation path
  before you need it.
- **MUST triage in minutes, not hours:** confirm the report, assess
  scope, declare (or stand down) fast. A slow triage is the first
  failure.

### Contain & eradicate

- **MUST contain before root-causing.** Stop the bleeding first:
  revoke the leaked token, rotate the secret, isolate the host, block
  the IP, disable the compromised account. Containment buys time;
  analysis without containment leaks data.
- **MUST preserve evidence before eradicating:** snapshots, logs,
  memory dumps, the vulnerable artifact. Eradicating without
  preservation destroys the root-cause investigation.
- **MUST eradicate the vulnerability, not just the symptom:** patch,
  config change, remove the package — not just block the IP.
- **MUST document every action with a timestamp and an actor.** The
  timeline is the postmortem's spine. Template:
  [references/timeline-template.md](references/timeline-template.md).

### Communication

- **MUST use a single incident channel** as the source of truth —
  one channel, one status thread. No side channels, no DMs with
  partial updates.
- **MUST update the status on a cadence:** every 30 minutes for
  SEV0/SEV1, every 2 hours for SEV2, daily for SEV3. Silence is
  scarier than bad news.
- **MUST know the compliance clocks before the incident:** GDPR
  requires notification to the supervisory authority within 72 hours
  of becoming aware of a personal data breach; SOC 2 requires timely
  notification per customer agreements. Missing the clock is a
  separate compliance failure.
- **MUST NOT disclose externally before containment and legal
  sign-off.** Premature disclosure can tip off an attacker and
  create legal exposure. Internal awareness first, external
  disclosure on the compliance schedule.
- **MUST brief stakeholders (exec, legal, support) on a schedule,
  not on demand.** One writer, one channel, one cadence.

### Disclosure — CVE & advisories

- **For a vulnerability in your code:** file a **security advisory**
  (GHSA) and request a CVE; coordinate a fix before public disclosure
  (coordinated disclosure). Checklist:
  [references/disclosure-checklist.md](references/disclosure-checklist.md).
- **For a vulnerability in a dependency:** track the upstream
  advisory, assess your exposure, patch or pin, and disclose to your
  users if you were affected.
- **MUST NOT silently patch a disclosed vulnerability without a
  record.** The changelog entry and security advisory are the evidence
  of response.
- **MUST keep a disclosure template ready.** The middle of an incident
  is the wrong time to draft one — use the template in
  [references/disclosure-checklist.md](references/disclosure-checklist.md).

### Postmortem

- **MUST run a blameless postmortem within one week** of resolution.
  Template:
  [references/postmortem-template.md](references/postmortem-template.md).
- **Root cause MUST be a system cause** — the check didn't run, the
  alert didn't fire, the rotation wasn't automated — not a person
  cause ("X clicked the wrong button"). If a human error caused it,
  the system that allowed that error is the root cause.
- **Every action item MUST have an owner and a due date.** Track to
  closure. An action item without an owner is a wish.
- **MUST publish the postmortem internally.** For major incidents
  (SEV0/SEV1), publish a public write-up. The postmortem is the
  artifact that makes the next incident shorter.

## Incident flow

1. **Detect** — a report arrives: CVE advisory, alert, customer
   report, automated scan, leaked credential detected. Log the
   detection timestamp — the compliance clock may start here.
2. **Triage** — classify severity
   ([references/severity-rubric.md](references/severity-rubric.md)),
   declare an incident owner, open the incident channel, page by
   severity. If the report is a false positive, document the
   assessment and stand down.
3. **Contain** — stop the bleeding: revoke, rotate, isolate, block.
   Start the timeline
   ([references/timeline-template.md](references/timeline-template.md)).
   Preserve evidence before any destructive action.
4. **Eradicate** — remove the vulnerability: patch, config change,
   dependency update. Verify the fix: the attack vector must no longer
   work.
5. **Recover** — restore service, monitor for recurrence, confirm
   data integrity. Lift containment measures only after eradication
   is verified.
6. **Disclose** — file advisories, notify customers, meet compliance
   clocks
   ([references/disclosure-checklist.md](references/disclosure-checklist.md)).
7. **Postmortem** — blameless review within one week. Timeline,
   impact, root cause, action items with owners
   ([references/postmortem-template.md](references/postmortem-template.md)).

## Notes and edge cases

- **False positives:** a triage that concludes "not an incident" is
  still worth documenting — a one-paragraph stand-down note prevents
  the same report from consuming another triage cycle.
- **Multi-org incidents:** when the incident spans your code and a
  third party's, coordinate containment and disclosure timelines.
  Don't publish details of a vulnerability in someone else's system
  before they've had a chance to patch — coordinated disclosure
  applies outward too.
- **Legal holds:** if litigation or regulatory investigation is
  possible, legal may require preserving all artifacts (logs, chat
  history, artifacts) beyond normal retention. Confirm with legal
  before any cleanup.
- **Coordinated disclosure timing:** the standard window is 90 days
  from initial report to public disclosure. If the vendor is
  unresponsive, escalate through a CERT or disclose on your timeline
  — but document the escalation path.
- **Leaked secrets in git history:** containment means rotating the
  secret, not just removing it from the current tree. The secret is
  in the reflog, in forks, and in CI caches. Rotate first, clean
  history second.
- **Recurring incidents:** if the same class of incident happens
  twice, the second postmortem must reference the first and explain
  why the prior action items didn't prevent recurrence.
