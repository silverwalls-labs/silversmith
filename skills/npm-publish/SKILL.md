---
name: npm-publish
description: Secure npm package publishing. Use when setting up, editing, or reviewing an npm publish or release flow, or preparing a package for publication.
---

# npm-publish

Enforce current good practices for publishing npm packages: supply-chain
integrity (trusted publishing, provenance) and a safe release flow (staged
publish, dist-tag promotion). The scope is the publish itself — project
quality gates (lint, tests, build, audits) are the project's own concern;
this skill only requires that the publish is sequenced behind them.

Version floors: **npm ≥ 11.15.0**, **Node ≥ 22.14.0** (staged publishing;
trusted publishing needs npm ≥ 11.5.1). These are minimums — target the
current Node release (26 as of late 2026). If the CI toolchain is older,
upgrading is step zero.

## Rules

Normative. Each MUST/NEVER doubles as a review check: to pre-flight a
release or review an existing setup, walk this list top to bottom and
flag every violation, even ones you were not asked about.

- **MUST publish via trusted publishing (OIDC)** from CI (GitHub Actions
  or GitLab CI). NEVER publish from a local machine, and NEVER keep a
  registry token in CI — no `NPM_TOKEN`/`NODE_AUTH_TOKEN` secrets, no
  `_authToken` lines in any `.npmrc`. Classic tokens were revoked
  registry-wide on 2025-12-09.
- **MUST NOT bypass staged publishing.** `npm stage publish` from CI,
  human review of the staged tarball, then `npm stage approve` with 2FA.
  Do not grant `--allow-publish` (direct publish) without a documented
  reason.
- **MUST keep provenance on** (automatic with trusted publishing): never
  set `NPM_CONFIG_PROVENANCE=false` or `provenance=false` anywhere, and
  `package.json` `repository.url` must match the repository the workflow
  runs in — the publish fails on mismatch.
- **MUST require 2FA** on every account with publish or maintainer
  access, using WebAuthn/passkeys (new TOTP enrollment is disabled). Set
  the package's publishing access to **"Require two-factor authentication
  and disallow tokens"**.
- **MUST stage releases through a dist-tag channel:** prereleases under
  `next`/`beta`; promotion to `latest` is a separate human command
  (`npm dist-tag add`), never a side effect of CI on a feature branch and
  never implicit by publishing a prerelease without `--tag`.
- **Versions are immutable.** NEVER republish, force-publish, or
  unpublish-then-republish. A fix is a new version plus `npm deprecate`
  on the bad one.
- **MUST sequence the publish behind the project's quality gates** in the
  same workflow (`needs:`), none of them `continue-on-error`. Gate
  contents are the project's choice and out of scope here.
- **MUST isolate the publish from code execution.** The build job
  installs (`npm ci --ignore-scripts`, committed lockfile), runs the
  gates, packs, and uploads the tarball artifact; the publish job
  downloads and publishes **that tarball only** — no checkout, no
  dependency install, no project scripts next to the OIDC credential.
  Publishing a tarball also runs no lifecycle scripts.
- **MUST control the shipped file set** with a `files` allowlist in
  `package.json` (preferred over `.npmignore`) and review the `npm pack`
  ship list: no secrets, tests, `.env`, CI config, or unintended source
  maps — the published tarball is that exact artifact.
- **MUST publish from a clean, tagged commit** on the default branch:
  semver bump via `npm version`, changelog entry, workflow triggered by
  the tag push — never by branch pushes.
- **MUST harden the workflow itself:** top-level `permissions: {}`, build
  job `contents: read`, publish job `id-token: write` only; every action
  pinned to a full commit SHA; publish job gated by a GitHub Environment
  with required reviewers.

## Publish flow

1. **One-time setup:** on npmjs.com, set publishing access to "Require
   two-factor authentication and disallow tokens" and add a trusted
   publisher (org/user, repository, workflow filename, environment name —
   keep the stage-publish-only default). On the repo, create that
   environment with required reviewers; protect `main` and the `v*` tag
   pattern.
2. **Cut the release:** `npm version <patch|minor|major|pre*>`, update the
   changelog, `git push --follow-tags`.
3. **CI stages the tarball** via the two-job (build → publish) workflow:
   [references/publish-workflow.md](references/publish-workflow.md).
4. **Review, approve (2FA), verify, promote** — commands, verification,
   and failure handling:
   [references/release-flow.md](references/release-flow.md).

## Notes and edge cases

- **First publish of a new package:** staged publishing and trusted
  publishers require the package to already exist on the registry, so the
  very first publish is the one sanctioned exception to the no-local
  rule: a maintainer with 2FA publishes from a clean tagged commit
  (`--access public` or `publishConfig.access` for a scoped public
  package), then immediately applies the one-time setup above.
- **Private repositories:** provenance is unavailable. Everything else
  still applies — note the gap in the release docs instead of faking it.
- **GitLab CI** is the other supported trusted publisher; same rules.
  Unsupported CI (Jenkins, Buildkite, self-hosted): use a thin GitHub
  Actions or GitLab publish job that runs after the main CI builds the
  artifact.
- **Interim token fallback / migration:** if a token is truly unavoidable,
  use a granular write token (90-day cap, 2FA by default) scoped to the
  single package — and configure the trusted publisher and verify a
  staged publish works *before* revoking tokens and tightening access,
  so the pipeline never goes dark.
