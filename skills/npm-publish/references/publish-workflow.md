# Publish workflow — GitHub Actions example

Minimal correct shape: tag-triggered, OIDC trusted publishing, staged
publish, **build/publish separation** — the credentialed job never
executes project code; it only publishes the tarball built by the
previous job. Assumes the one-time setup from SKILL.md (trusted
publisher → `publish.yml` + environment `npm-publish` with required
reviewers, package access = "Require 2FA and disallow tokens").

```yaml
name: publish

on:
  push:
    tags: ["v*.*.*"] # tags only — never branch pushes

permissions: {} # deny-all; jobs opt in below

jobs:
  build:
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
      # pack once; the log prints the exact ship list for reviewer eyes,
      # and this tarball is the only thing handed to the publish job
      - run: npm pack
      - uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
        with:
          name: package-tarball
          path: "*.tgz"

  publish:
    needs: build # publish cannot run unless build + gates passed
    runs-on: ubuntu-latest
    environment: npm-publish # human gate: required reviewers
    permissions:
      id-token: write # OIDC for trusted publishing — no NPM_TOKEN
    steps:
      # NO checkout, NO npm ci, NO npm run: nothing project-controlled
      # may execute in the job that holds the publish credential.
      - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
        with:
          node-version: 26
      - run: npm install -g npm@^11.15.0 # floor for `npm stage`
      - uses: actions/download-artifact@3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c # v8.0.1
        with:
          name: package-tarball
      - run: |
          # publish the built tarball only; prerelease tags stage under `next`
          TARBALL="$(ls ./*.tgz)"
          if [[ "$GITHUB_REF_NAME" == *-* ]]; then
            npm stage publish "$TARBALL" --tag next
          else
            npm stage publish "$TARBALL"
          fi
```

## The load-bearing lines

| Element | Why it matters |
|---|---|
| `tags: ["v*.*.*"]` | A release is a deliberate tag push, never a branch push |
| `permissions: {}` + per-job blocks | Least privilege; only the publish job gets `id-token: write` |
| SHA-pinned `uses:` | Tags are mutable — pin the commit you audited (SHAs above verified 2026-09) |
| `npm ci --ignore-scripts` | Dependency lifecycle scripts never execute during install |
| Build/publish job split | Project code (deps, build, scripts) runs only in `build`; a compromise there cannot reach the publish credential |
| Tarball artifact handoff | The publish job publishes the exact built bytes — and publishing a tarball runs no lifecycle scripts |
| `needs: build` | Publish is sequenced behind the project's quality gates |
| `environment: npm-publish` | Required reviewers must approve before the job runs |
| `npm stage publish` | Nothing goes live — a maintainer must `npm stage approve` with 2FA |
| No token anywhere | The npm CLI exchanges the OIDC token itself; provenance is attested automatically |

A green workflow is **not** a release: review, approve, and promote per
[release-flow.md](release-flow.md).
