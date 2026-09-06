# silversmith
Collection of skills and agentic config

## Skills

Each skill is a folder under [`skills/`](skills/) following the
[Agent Skills](https://agentskills.io) open standard (`SKILL.md` +
optional `references/`), usable by any tool that supports the standard.

| Skill | Description |
|---|---|
| [npm-publish](skills/npm-publish/SKILL.md) | Secure npm package publishing: trusted publishing (OIDC), provenance, staged publish, dist-tag promotion. |

## Using a skill

Symlink or copy the skill folder into your harness's discovery path —
`.claude/skills/<name>` (Claude Code, opencode) or `.agents/skills/<name>`
(opencode, Mistral Vibe):

```sh
ln -s "$(pwd)/skills/npm-publish" /path/to/project/.claude/skills/npm-publish
```
