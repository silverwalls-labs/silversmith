# Disclosure checklist

Work through this checklist in order. Do not skip steps — each one
gates the next. Do not disclose externally before containment and legal
sign-off.

## 1. Vulnerability in your code

- [ ] **Confirm the vulnerability** — reproduce it, identify the
  affected versions, and determine the attack surface.
- [ ] **Develop the fix** in a private branch or fork. Do not push to
  a public branch before disclosure.
- [ ] **File a security advisory (GHSA)** on GitHub: repository →
  Security → Advisories → New draft advisory. Include:
  - Affected product and versions
  - Vulnerability type (CWE if applicable)
  - Severity (CVSS score)
  - Description of the vulnerability and its impact
  - Remediation (patched version, workaround if available)
- [ ] **Request a CVE** through the GHSA — GitHub is a CNA and can
  assign one directly from the advisory draft. If not using GitHub,
  request through [cveform.mitre.org](https://cveform.mitre.org).
- [ ] **Coordinate disclosure timing** — 90 days is the standard
  window from initial report to public disclosure. Agree on a date
  with the reporter (if externally reported).
- [ ] **Publish the fix** — merge the fix, tag the release, publish
  the patched version.
- [ ] **Publish the advisory** — make the GHSA public on the agreed
  disclosure date. The CVE record updates automatically if assigned
  through GitHub.
- [ ] **Notify affected users** — changelog entry, release notes,
  and direct notification if the vulnerability is critical.
- [ ] **Update the incident timeline** with disclosure actions and
  dates.

## 2. Vulnerability in a dependency

- [ ] **Identify the upstream advisory** — check the dependency's
  security advisories, CVE databases
  ([github.com/advisories](https://github.com/advisories)), and
  `npm audit` / `gh api /advisories`.
- [ ] **Assess your exposure** — is the vulnerable code path
  reachable in your usage? What data does it handle?
- [ ] **Patch or pin** — update to the patched version; if no patch
  exists, pin to the last safe version or remove the dependency.
  If neither is possible, document the mitigating controls.
- [ ] **Disclose to your users** if you were affected — a changelog
  entry and, for critical exposure, a security advisory of your own
  explaining your exposure and remediation.
- [ ] **Update the incident timeline.**

## 3. Compliance clocks

These are hard deadlines. Missing them is a separate compliance
failure independent of the incident itself.

### GDPR (personal data breaches)

- [ ] **72 hours** from becoming aware of a personal data breach to
  notify the supervisory authority (Art. 33 GDPR). "Becoming aware"
  is when you have a reasonable degree of certainty — not when the
  investigation is complete.
- [ ] **Without undue delay** to notify affected data subjects if the
  breach is likely to result in a high risk to their rights and
  freedoms (Art. 34 GDPR).
- [ ] **Document the breach** regardless of whether notification is
  required — facts, effects, remedial actions (Art. 33(5) GDPR).

### SOC 2 / contractual

- [ ] Review customer agreements for notification clauses — many
  require notification within 24–72 hours.
- [ ] Notify affected customers per the agreed schedule and channel.
- [ ] Retain evidence of notification (timestamp, recipient, content).

### Other frameworks

- [ ] **HIPAA:** 60 days for breach notification to individuals; for
  breaches affecting 500+ individuals, notify HHS within 60 days;
  for fewer than 500, log and submit to HHS within 60 days of the
  end of the calendar year in which the breach was discovered.
- [ ] **PCI DSS:** notify the payment card brands and acquiring bank
  immediately upon suspicion of cardholder data compromise.
- [ ] Check for sector-specific or jurisdiction-specific requirements
  (state breach notification laws, NIS2, etc.).

## 4. External communication

- [ ] **Legal sign-off** before any external disclosure — legal
  reviews the advisory text, customer notification, and public
  statement.
- [ ] **Single spokesperson** — one person or team drafts and sends
  all external communication. No ad-hoc statements.
- [ ] **Factual, no speculation** — disclose what happened, what
  data was affected, what you did, and what the customer should do.
  Do not speculate on attacker identity or motive.
- [ ] **Support readiness** — brief the support team before the
  public disclosure so they can handle inbound questions.
