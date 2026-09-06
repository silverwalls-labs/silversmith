# Provider setup — GitHub Actions

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
    outputs:
      dist-tag: ${{ steps.tag.outputs.tag }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
        with:
          node-version: 26 # current release; hard floor is 22.14.0
      - run: npm ci --ignore-scripts
      # project quality gates — your choice, out of scope for this skill
      - run: npm run lint && npm test && npm run build
      # pack once; the log prints the exact ship list for reviewer eyes,
      # and this tarball is the only thing handed to the publish job.
      # Pack OUTSIDE the workspace: dist/ and friends are typically
      # shipped package content — don't drop tarballs inside them.
      - run: npm pack --pack-destination "$RUNNER_TEMP"
      # choose the channel from package.json (authoritative), not the tag
      # name: prerelease -> next; a stable superseding dist-tags.latest ->
      # latest; anything else (a maintenance/backport release) ->
      # legacy-<major> — an untagged publish would happily move `latest`
      # backwards (the registry does not stop that), so the channel must
      # be explicit
      - name: Choose dist-tag
        id: tag
        run: |
          TAG="$(node -e '
            const { execSync } = require("child_process");
            const pkg = require("./package.json");
            const [core, pre] = pkg.version.split("-");
            if (pre) { console.log("next"); process.exit(0); }
            const cur = core.split(".").map(Number);
            let latest = "";
            try { latest = execSync(`npm view ${pkg.name} dist-tags.latest`).toString().trim(); } catch {}
            const lat = (latest || "0.0.0").split("-")[0].split(".").map(Number);
            let newer = false;
            for (let i = 0; i < cur.length; i++) {
              if (cur[i] > (lat[i] ?? 0)) { newer = true; break; }
              if (cur[i] < (lat[i] ?? 0)) break;
            }
            console.log(newer ? "latest" : `legacy-${cur[0]}`);
          ')"
          echo "tag=$TAG" >> "$GITHUB_OUTPUT"
      - uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
        with:
          name: package-tarball
          path: ${{ runner.temp }}/*.tgz

  publish:
    needs: build # publish cannot run unless build + gates passed
    runs-on: ubuntu-latest
    environment: npm-publish # human gate: required reviewers
    permissions:
      id-token: write # OIDC for trusted publishing — no NPM_TOKEN
    steps:
      # NO checkout, NO npm ci, NO npm run: nothing project-controlled
      # may execute in the job that holds the publish credential.
      # Node 26.3+ bundles npm >= 11.15 (the `npm stage` floor) — no npm
      # upgrade step needed.
      - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
        with:
          node-version: 26
      - uses: actions/download-artifact@3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c # v8.0.1
        with:
          name: package-tarball
      # the build output crosses the trust boundary via env, NEVER via
      # ${{ }} inside the script: a compromised build job controls its
      # outputs, and expression interpolation in `run:` would let it
      # inject shell into this credentialed job. Validate it too.
      - env:
          DIST_TAG: ${{ needs.build.outputs.dist-tag }}
        run: |
          if [[ ! "$DIST_TAG" =~ ^(next|latest|legacy-[0-9]+)$ ]]; then
            echo "unexpected dist-tag: $DIST_TAG" >&2; exit 1
          fi
          # publish the built tarball only, under the channel chosen in build
          set -- *.tgz
          if [ ! -f "$1" ]; then echo "no tarball found in artifact" >&2; exit 1; fi
          if [ "$#" -ne 1 ]; then echo "expected exactly one tarball, found $#" >&2; exit 1; fi
          npm stage publish "$1" --tag "$DIST_TAG"
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
| `dist-tag` job output | Prerelease → `next`, maintenance → `legacy-<major>`; only a superseding stable targets `latest` — an untagged publish would move `latest` backwards, the registry does not stop it |
| `env:` handoff + allowlist | Build outputs cross into the credentialed job as env vars, never `${{ }}` in `run:` — blocks script injection from a compromised build job |
| `needs: build` | Publish is sequenced behind the project's quality gates |
| `environment: npm-publish` | Required reviewers must approve before the job runs |
| `npm stage publish` | Nothing goes live — a maintainer must `npm stage approve` with 2FA |
| No token anywhere | The npm CLI exchanges the OIDC token itself; provenance is attested automatically |

A green workflow is **not** a release: review, approve, and verify per
[release-flow.md](../release-flow.md).
