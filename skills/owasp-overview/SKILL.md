---
name: owasp-overview
description: Global OWASP secure-coding reference. Use when designing or reviewing an application's security posture, threat-modeling a feature, or applying a structured secure-coding checklist beyond a single vulnerability class.
---

# owasp-overview

Map an application against the OWASP standards: Top 10 (web), API
Security Top 10, and ASVS, plus a lightweight threat-modeling pass for
new designs. The scope is assessment and routing — naming the risk,
mapping it to a standard identifier, and pointing at the authoritative
fix reference (the OWASP Cheat Sheet Series). Concrete remediation is
the linked cheat sheet's concern; this skill does not restate it.

Editions covered (as of 2026-09): **OWASP Top 10:2025**, **ASVS 5.0.0**,
**API Security Top 10 2023**. Each reference file states its edition
and the date it was checked.

## Rules

Normative. Each MUST/NEVER doubles as a review check: to assess a
codebase or review a design, walk this list top to bottom and flag
every violation, even ones you were not asked about.

- **MUST state the edition assessed against** (e.g., "Top 10:2025",
  "ASVS 5.0.0") in every assessment output, and MUST flag when OWASP
  has published a newer edition than the ones in
  [references/](references/) — then treat the newer edition as
  authoritative and note that this skill needs updating.
- **MUST map every finding to a standard identifier** — a Top 10
  category (A01–A10), an API Top 10 category (API1–API10), or an ASVS
  requirement ID (Vx.y.z). An unmapped finding is an opinion, not an
  assessment.
- **MUST pick and justify an ASVS level (L1/L2/L3) before mapping a
  codebase** against [references/asvs.md](references/asvs.md): the
  target level is a risk decision (data sensitivity, user
  expectations, regulatory context), not a default. L1 is the floor
  for anything in production.
- **NEVER present the condensed ASVS excerpt as a full audit.** It is
  a triage checklist (~60 of 345 requirements); a compliance claim
  requires the full standard, linked from the reference.
- **MUST assess API surfaces against the API Top 10 as well** — the
  general Top 10 alone misses API-specific failure modes (object-level
  authorization, mass assignment, unrestricted business flows).
- **MUST run the threat-modeling prompt in
  [references/threat-modeling.md](references/threat-modeling.md) on
  any new feature or architectural change** before reviewing its code
  — a design flaw found in code review is found late.
- **MUST cite the relevant OWASP cheat sheet for each finding's fix**
  rather than inventing remediation guidance; each reference table
  carries the per-topic link.
- **NEVER downgrade a finding because it is not in a Top 10.** The
  Top 10 lists are awareness rankings, not an exhaustive taxonomy;
  ASVS is the closer-to-exhaustive net.

## Routing

Open only the reference the task needs:

| Task | Reference |
|---|---|
| Security posture review of a web application | [references/top10.md](references/top10.md), then [references/asvs.md](references/asvs.md) |
| Codebase mapping against a verification level | [references/asvs.md](references/asvs.md) |
| API design or review (REST, GraphQL, service-to-service) | [references/api-top10.md](references/api-top10.md) |
| Threat-modeling a new feature or design | [references/threat-modeling.md](references/threat-modeling.md) |

Sequencing for a full posture review: threat model first (what matters
here?), then a Top 10 pass (are the big risk classes handled?), then
ASVS at the chosen level (systematic gaps), plus the API Top 10
wherever an API surface exists.

## Notes and edge cases

- **Edition provenance (as of 2026-09):** Top 10:2025 supersedes 2021.
  SSRF (A10:2021) was folded into A01 Broken Access Control,
  Vulnerable and Outdated Components (A06:2021) was expanded into A03
  Software Supply Chain Failures, and A10 Mishandling of Exceptional
  Conditions is new. ASVS 5.0.0 (May 2025) restructured 4.0 into 17
  chapters with re-leveled requirements — 4.0 requirement IDs do not
  map 1:1 to 5.0. API Security Top 10 2023 is the current API edition.
- **Keeping current:** each reference file carries an
  "edition / checked" header. When OWASP publishes a new edition,
  update the file and its header in the same change — never mix
  editions silently within one assessment.
- **Self-contained:** this skill references only its own files and
  official OWASP sources; it does not depend on any other skill.
- **Not a scanner:** these are review checklists for human/agent
  judgment. They complement SAST/DAST/SCA tooling, they do not
  replace it — nor does a green scan replace them.
