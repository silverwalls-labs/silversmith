# silversmith
Collection of skills and agentic config

## Skills

Each skill is a folder under [`skills/`](skills/) following the
[Agent Skills](https://agentskills.io) open standard (`SKILL.md` +
optional `references/`), usable by any tool that supports the standard.

| Skill | Description |
|---|---|
| [code-injection-prevention](skills/code-injection-prevention/SKILL.md) | Injection-prevention guardrails: parameterized SQL, safe command execution, path-traversal confinement, SSRF target validation, template-injection separation. |
| [incident-response](skills/incident-response/SKILL.md) | Security incident response: severity triage, containment, evidence preservation, disclosure, blameless postmortems. |
| [npm-publish](skills/npm-publish/SKILL.md) | Secure npm package publishing: trusted publishing (OIDC), provenance, staged publish, dist-tag promotion. |
| [owasp-overview](skills/owasp-overview/SKILL.md) | Global OWASP secure-coding reference: Top 10 (2025), ASVS 5.0 tiered checklist, API Security Top 10, STRIDE threat-modeling prompt. |
| [security-code-review](skills/security-code-review/SKILL.md) | Tool-backed security review: parallel scan dimensions (SAST, dependencies, secrets, manual review), adversarial verification of findings, severity-ranked report. |
| [threat-modeling](skills/threat-modeling/SKILL.md) | Per-feature threat modeling: when to model, STRIDE over data flows, tracked threat register. |
| [vulnerability-triage](skills/vulnerability-triage/SKILL.md) | Evidence-based vulnerability triage: reachability over CVSS, urgency ranking, patch/pin/mitigate/accept, tracked findings with SLAs. |
