# Release flow walkthrough

End-to-end example for `@acme/widget`, combining the two staging layers:

- **Staged publishing** (`npm stage`) — a registry-level approval gate:
  nothing goes live until a maintainer approves with 2FA.
- **Dist-tag channels** (`next`, `legacy-<major>`, `latest`) — a
  release-channel gate: a live version is not the default install unless
  it is a superseding stable approved onto `latest`.

They are complementary: `npm stage` protects the *registry write*,
dist-tags protect the *default consumer experience*.

## 1. Prerelease to the `next` channel

```sh
# clean default branch, all gates green locally
git switch main && git pull && git status   # must be clean

npm version preminor --preid=beta           # 1.4.0 -> 1.5.0-beta.0, commits + tags
                                            # (further betas: npm version prerelease)
git push --follow-tags                      # tag push triggers the publish pipeline
```

CI runs the gates, packs, and stages the built tarball under `next` via
trusted publishing (provider setups: [providers/](providers/)). Then a
maintainer:

```sh
npm stage list                              # -> stage-id for 1.5.0-beta.0
npm stage view <stage-id>                   # metadata: version, integrity, provenance
npm stage download <stage-id>               # pull the exact tarball
tar -tzf acme-widget-1.5.0-beta.0.tgz       # audit contents: files allowlist honored?
npm stage approve <stage-id>                # 2FA prompt -> version goes live on `next`
```

Verify from a consumer's seat:

```sh
mkdir /tmp/widget-smoke && cd /tmp/widget-smoke && npm init -y
npm install @acme/widget@next
node -e "require('@acme/widget')"           # or the package's real smoke test
npm view @acme/widget --json | jq '.dist.attestations'   # provenance present —
                                                          # confirms the tarball handoff kept it
```

`latest` still points at 1.4.0 — beta users opt in, nobody else is affected.

## 2. Stable release

```sh
npm version minor                           # 1.5.0, commit + tag v1.5.0
git push --follow-tags
```

The build job verified that 1.5.0 supersedes the current `latest`, so CI
stages it under `latest` — the one channel a superseding stable may
target. Maintainer reviews and approves exactly as above; on approval,
`latest` moves to 1.5.0. `next` still points at 1.5.0-beta.0 —
harmless, but you can retire it:

```sh
npm dist-tag ls @acme/widget                # latest -> 1.5.0, next -> 1.5.0-beta.0
npm dist-tag rm @acme/widget next           # optional cleanup
```

You never promote the beta itself: the stable release is a new version
through the same flow. `latest` only moves at approval or by a
deliberate human `npm dist-tag add` (rollback, legacy promotion) —
never as a side effect of CI on a feature branch.

## 3. Maintenance release of an old major

Publishing 1.4.1 while `latest` is 2.x: the registry refuses to move
`latest` backwards, so the pipeline stages it under `legacy-1` (the
build job derives the channel by comparing the new version against
`dist-tags.latest`). Review and approve as above; consumers opt in
explicitly:

```sh
npm install @acme/widget@legacy-1
```

Promoting a legacy release to `latest` by hand is possible
(`npm dist-tag add @acme/widget@1.4.1 latest`) but almost always a
mistake — `latest` moving backwards breaks consumers.

## 4. When a bad version ships

Published versions are immutable. Never republish, force-publish, or
unpublish-then-republish. Instead:

```sh
# 1. point `latest` back at the last good version (instant mitigation)
npm dist-tag add @acme/widget@1.4.0 latest

# 2. mark the bad version so installs warn
npm deprecate @acme/widget@1.5.0 "Broken build, use 1.5.1"

# 3. fix, then release the fix as a NEW version through the normal flow
npm version patch                           # 1.5.1
git push --follow-tags
```

If the bad version shipped malware or secrets (not just a bug), also:
rotate the exposed secrets, reject anything still staged
(`npm stage reject <stage-id>`), review the trusted-publisher and
maintainer list for tampering, and contact npm support — deletion beyond
the 72-hour unpublish policy is their call, not a CLI command.

## 5. Reference: stage lifecycle

| State | Command | Notes |
|---|---|---|
| Stage | `npm stage publish [--tag <t>]` | From CI via OIDC; no 2FA needed to stage |
| List | `npm stage list` | Also visible in the npmjs.com "Staged Packages" tab |
| Inspect | `npm stage view <id>` / `npm stage download <id>` | Audit the exact bytes |
| Approve | `npm stage approve <id>` | 2FA required; version goes live |
| Reject | `npm stage reject <id>` | 2FA prompt; discards the staged tarball |

Staged versions share the version index with published ones: you cannot
stage a version that already exists, and a staged version reserves its
number until approved or rejected. The dist-tag is fixed at staging time
too — to restage under a different channel, `npm stage reject` first.
