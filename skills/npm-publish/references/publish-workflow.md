# Publish workflow — GitHub Actions example

Minimal correct shape: tag-triggered, OIDC trusted publishing, staged
publish. Assumes the one-time setup from SKILL.md (trusted publisher →
`publish.yml` + environment `npm-publish` with required reviewers,
package access = "Require 2FA and disallow tokens").

```yaml
name: publish

on:
  push:
    tags: ["v*.*.*"] # tags only — never branch pushes

permissions: {} # deny-all; jobs opt in below

jobs:
  gates:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
        with:
          node-version: 26 # current release; hard floor is 22.14.0
      - run: npm ci --ignore-scripts
      # project quality gates — your choice, out of scope for this skill
      - run: npm run lint && npm test && npm run build
      # ship-list into the job log (lists only, never fails on content)
      - run: npm pack --dry-run

  publish:
    needs: gates # publish cannot run unless gates passed
    runs-on: ubuntu-latest
    environment: npm-publish # human gate: required reviewers
    permissions:
      contents: read
      id-token: write # OIDC for trusted publishing — no NPM_TOKEN
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
        with:
          node-version: 26 # current release; hard floor is 22.14.0
      - run: npm install -g npm@^11.15.0 # floor for `npm stage`
      - run: npm ci --ignore-scripts
      - run: npm run build
      - run: |
          # prerelease tags (v1.2.3-beta.1) stage under `next`
          if [[ "$GITHUB_REF_NAME" == *-* ]]; then
            npm stage publish --tag next
          else
            npm stage publish
          fi
```

## The load-bearing lines

| Element | Why it matters |
|---|---|
| `tags: ["v*.*.*"]` | A release is a deliberate tag push, never a branch push |
| `permissions: {}` + per-job blocks | Least privilege; only the publish job gets `id-token: write` |
| SHA-pinned `uses:` | Tags are mutable — pin the commit you audited (SHAs above verified 2026-09) |
| `npm ci --ignore-scripts` | Dependency lifecycle scripts never execute on the publish runner |
| `needs: gates` | Publish is sequenced behind the project's quality gates |
| `environment: npm-publish` | Required reviewers must approve before the job runs |
| `npm stage publish` | Nothing goes live — a maintainer must `npm stage approve` with 2FA |
| No token anywhere | The npm CLI exchanges the OIDC token itself; provenance is attested automatically |

A green workflow is **not** a release: review, approve, and promote per
[release-flow.md](release-flow.md).
