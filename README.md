# silversmith
Collection of skills and agentic config

## Skills

Each skill is a folder under [`skills/`](skills/) following the
[Agent Skills](https://agentskills.io) open standard (`SKILL.md` +
optional `references/`), usable by any tool that supports the standard.

| Skill | Description |
|---|---|
| [code-injection-prevention](skills/code-injection-prevention/SKILL.md) | Injection-prevention guardrails: parameterized SQL, safe command execution, path-traversal confinement, SSRF target validation, template-injection separation. |
| [npm-publish](skills/npm-publish/SKILL.md) | Secure npm package publishing: trusted publishing (OIDC), provenance, staged publish, dist-tag promotion. |
| [security-code-review](skills/security-code-review/SKILL.md) | Tool-backed security review: parallel scan dimensions (SAST, dependencies, secrets, manual review), adversarial verification of findings, severity-ranked report. |
