---
name: npm-publish
description: Secure npm package publishing — trusted publishing (OIDC), automatic provenance, staged publish with 2FA approval, dist-tag promotion, pre-publish gates. Load when setting up or editing an npm publish workflow, a release flow, or a package.json/.npmrc for publication, and when reviewing an existing npm release setup.
---

# npm-publish

Enforce current good practices for publishing npm packages: supply-chain
integrity (trusted publishing, provenance), a safe release flow (staged
publish, dist-tag promotion), and hard gates before anything reaches the
registry. Apply these rules when authoring or reviewing publish workflows,
release scripts, `package.json`, or `.npmrc`.

Version floors (verify before anything else): **npm ≥ 11.15.0** and
**Node ≥ 22.14.0** — required for staged publishing; trusted publishing
needs npm ≥ 11.5.1. If the project's CI uses older versions, upgrading is
step zero.

## Rules

Normative. MUST/NEVER items are non-negotiable; flag every violation you
find, even ones you were not asked about.

- **MUST publish via trusted publishing (OIDC)** from CI (GitHub Actions or
  GitLab CI). NEVER publish from a local machine, and NEVER store a
  long-lived registry token in CI secrets. Classic tokens were revoked
  registry-wide on 2025-12-09; a lingering `NPM_TOKEN` secret is dead
  weight at best and an exfiltration target at worst — remove it.
- **MUST NOT bypass staged publishing.** Publish with `npm stage publish`
  from CI, review the staged tarball, then approve with 2FA
  (`npm stage approve`). Trusted-publisher configs created after
  2026-09-03 default to stage-publish-only; do not loosen that with
  `--allow-publish` without a documented reason.
- **MUST keep provenance on.** With trusted publishing, provenance
  attestations are generated automatically — do not set
  `NPM_CONFIG_PROVENANCE=false` or `provenance=false` anywhere. The
  explicit `--provenance` flag is only a migration/debugging aid.
- **MUST require 2FA** on every account with publish or maintainer access,
  using WebAuthn/passkeys (new TOTP enrollment is disabled). Set the
  package's publishing access to **"Require two-factor authentication and
  disallow tokens"**.
- **MUST stage releases through a dist-tag channel.** Prereleases go out
  under `next`/`beta` (`npm publish --tag next`); promote to `latest` only
  after verification (`npm dist-tag add pkg@x.y.z latest`). NEVER publish
  straight to `latest` from a CI run on a feature branch, and never let a
  prerelease version become `latest` implicitly by publishing without
  `--tag`.
- **Versions are immutable.** NEVER attempt to republish, force-publish, or
  unpublish-then-republish a version. A bad release is fixed by publishing
  a new version and `npm deprecate`-ing the bad one.
- **MUST gate the publish** on lint, tests, build, and a vulnerability
  audit (`npm audit` and/or `osv-scanner`) — any failure aborts the
  publish. Gates run in the same workflow, before the publish step.
- **MUST control the shipped file set** with a `files` allowlist in
  `package.json` (preferred over `.npmignore`) and inspect it with
  `npm pack --dry-run` before releasing. No secrets, `.env` files, tests,
  CI config, or unintended source maps in the tarball.
- **MUST publish from a clean, tagged commit** on the default branch:
  semver bump via `npm version`, changelog entry, annotated tag, workflow
  triggered by the tag push — not by every push to `main`.
- **MUST harden the publish workflow itself:** top-level
  `permissions: {}` with the publish job scoped to `contents: read` +
  `id-token: write`; every action pinned to a full commit SHA; install
  with `npm ci --ignore-scripts` from a committed lockfile; publish job
  gated by a GitHub Environment with required reviewers.

## Publish flow

1. **One-time setup (npmjs.com):**
   - Package settings → *Publishing access* → "Require two-factor
     authentication and disallow tokens".
   - Package settings → *Trusted publisher* → select GitHub Actions; set
     organization/user, repository, workflow filename (e.g.
     `publish.yml`), and the environment name (e.g. `npm-publish`). Leave
     the default permission (stage publish only).
   - Repo side: create the `npm-publish` GitHub Environment with required
     reviewers; protect `main` and the release tag pattern (`v*`).
2. **Cut the release:** on a clean default branch, `npm version <patch|minor|major>`
   (or `premajor`/`prerelease --preid=beta` for a prerelease), update the
   changelog, push the commit and tag: `git push --follow-tags`.
3. **CI stages the package:** the tag push triggers the workflow
   (see [references/publish-workflow.yml](references/publish-workflow.yml)),
   which runs the gates and ends with `npm stage publish` via OIDC — no
   token anywhere. Prereleases stage with `--tag next`.
4. **Human approves:** a maintainer reviews the staged tarball
   (`npm stage list` / `npm stage view <stage-id>` /
   `npm stage download <stage-id>`), then `npm stage approve <stage-id>`
   — prompts for 2FA. Reject with `npm stage reject <stage-id>`.
5. **Verify, then promote:** install the released version from the `next`
   tag in a scratch project, smoke-test it, check
   `npm view <pkg> --json` shows `dist.attestations`, then
   `npm dist-tag add <pkg>@<x.y.z> latest`.

Worked walkthrough with commands and failure handling:
[references/release-flow.md](references/release-flow.md).

## Pre-publish gate checklist

Every item passes before `npm stage publish` runs (automate all of it in
the workflow):

- [ ] Lint clean.
- [ ] Tests pass.
- [ ] Build succeeds; artifacts are what `main` produces (no local edits).
- [ ] `npm audit --omit=dev --audit-level=high` (and/or `osv-scanner`)
      reports nothing at the failing threshold.
- [ ] Lockfile committed; install used `npm ci --ignore-scripts`.
- [ ] `npm pack --dry-run` output reviewed: only intended files, no
      secrets/tests/`.env`/unintended source maps.
- [ ] `package.json` `repository.url` matches the repository the workflow
      runs in — provenance validates this and the publish fails on mismatch.
- [ ] Version bumped with `npm version`, follows semver for the change set.
- [ ] Changelog updated for this version.
- [ ] Publishing commit is clean and tagged; workflow was triggered by the
      tag, not a branch push.
- [ ] `engines`/CI use npm ≥ 11.15.0 and Node ≥ 22.14.0.

## Release-flow review checklist

Use when reviewing an existing publish setup (PR review, audit, or
migration):

- [ ] Publishing uses trusted publishing (OIDC), not tokens. No `NODE_AUTH_TOKEN`
      / `NPM_TOKEN` secrets, no `//registry.npmjs.org/:_authToken` lines
      in any `.npmrc`.
- [ ] Publish job has `id-token: write` and nothing beyond
      `contents: read`; top-level `permissions` is empty or read-only.
- [ ] Workflow triggers on release tags only — not `push: branches`.
- [ ] Publish job is gated by an environment with required reviewers.
- [ ] All actions pinned to full commit SHAs (not tags or branches).
- [ ] `npm ci --ignore-scripts` (or equivalent lifecycle-script
      suppression) is used for install.
- [ ] Gates (lint/test/audit/build) run in the same workflow before
      publish and are not `continue-on-error`.
- [ ] Publish step is `npm stage publish` (or documented justification for
      direct publish); trust config does not grant `--allow-publish`
      needlessly.
- [ ] Provenance not disabled anywhere (`NPM_CONFIG_PROVENANCE`,
      `.npmrc`, `publishConfig`).
- [ ] Package access is "Require two-factor authentication and disallow
      tokens"; maintainers use WebAuthn/passkey 2FA.
- [ ] Prereleases publish under a dist-tag (`next`/`beta`); promotion to
      `latest` is a separate, human step.
- [ ] `files` allowlist present in `package.json`.
- [ ] No workflow or script ever republishes or unpublishes an existing
      version.

## Notes and edge cases

- **First publish of a new package:** staged publishing and trusted
  publishers both require the package to already exist on the registry, so
  the very first publish is the one sanctioned exception to the no-local
  rule. Bootstrap: a maintainer with WebAuthn 2FA publishes the initial
  version from a clean tagged commit (`npm publish`, with
  `--access public` or `publishConfig.access` for a scoped public
  package), then *immediately* configures the trusted publisher and sets
  "Require two-factor authentication and disallow tokens". Every
  subsequent release follows the normal flow.
- **Private repositories:** provenance is not available when the source
  repo is private. Everything else here still applies; note the gap in
  the release docs instead of faking it.
- **GitLab CI:** trusted publishing also supports GitLab CI/CD — same
  rules, with the trust configured for the GitLab project/pipeline.
  Other CI systems (Jenkins, Buildkite, self-hosted) are not supported;
  the accepted pattern is a thin GitHub Actions or GitLab publish job
  that runs after the main CI builds the artifact.
- **If a token is truly unavoidable** (interim migration only): use a
  granular access token — write tokens are capped at 90 days and require
  2FA by default — scoped to the single package, and plan the OIDC
  migration that removes it. `npm login` sessions now expire after two
  hours; that is deliberate, don't script around it.
- **Migrating from token-based publishing:** configure the trusted
  publisher and verify a staged publish works *before* revoking tokens
  and tightening publishing access, so the pipeline never goes dark.
