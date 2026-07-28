# 📦 Package Publishing Guide

> The complete guide to publishing a CLI tool to seven channels at once - GitHub Releases, npm (trusted publishing), Homebrew, Scoop, winget, AUR, and Chocolatey - with release automation, credential tracking, and recovery procedures.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)

## ✨ Features

- ✅ One release pipeline that fans out to seven package channels
- ✅ npm trusted publishing (OIDC) with provenance, no long-lived tokens
- ✅ The one-time bootstrap dance npm requires before OIDC works
- ✅ A credential tracking table so nothing expires silently
- ✅ Privacy notes: what npm publishes about you and how to control it
- ✅ Recovery procedures for partially failed releases
- ✅ Real failure modes from each channel and how to handle them

## 🚀 Quick Start

1. Tag releases with [Release Please](https://github.com/googleapis/release-please) from Conventional Commits
2. Build once with GoReleaser (or your packager), publish everywhere from that one bundle
3. Set up npm trusted publishing → [npm](#npm-trusted-publishing)
4. Track every credential in one table → [Credential tracking](#credential-tracking)

## 🎯 Who Is This For?

- Developers shipping a CLI tool who want it installable everywhere
- Maintainers moving from npm tokens to trusted publishing (OIDC)
- Anyone who has had a release half-fail and wants a recovery path
- People who keep losing track of which token expires when

## 📋 Table of Contents

- [The Architecture](#the-architecture)
- [GitHub Releases](#github-releases)
- [npm (Trusted Publishing)](#npm-trusted-publishing)
- [Homebrew and Scoop](#homebrew-and-scoop)
- [winget](#winget)
- [AUR](#aur)
- [Chocolatey](#chocolatey)
- [Credential Tracking](#credential-tracking)
- [Release Recovery](#release-recovery)
- [Monitoring Checklist](#monitoring-checklist)
- [Security Notes](#security-notes)
- [FAQ](#faq)

---

## The Architecture

One workflow owns the whole release. Release Please watches `main` for Conventional Commits and maintains a release pull request; merging that PR creates the version tag. The same workflow then runs a publish job per channel, all consuming one build.

One repository setting trips everyone on the first run: Release Please opens PRs from a workflow, and GitHub blocks that by default with `GitHub Actions is not permitted to create or approve pull requests`. Enable it under Settings → Actions → General → **Allow GitHub Actions to create and approve pull requests**, or via the CLI:

```sh
gh api --method PUT repos/OWNER/REPO/actions/permissions/workflow \
  -f default_workflow_permissions=read \
  -F can_approve_pull_request_reviews=true
```

```
main (Conventional Commits)
  └─ Release Please → release PR → merge → tag vX.Y.Z
       └─ build (GoReleaser: archives + checksums + release bundle)
            ├─ github     (archives + deb/rpm/apk/pkg.tar.zst + checksums.txt)
            ├─ npm        (7 packages via OIDC trusted publishing)
            ├─ homebrew   (cask in a tap repository)
            ├─ scoop      (manifest in a bucket repository)
            ├─ winget     (PR to microsoft/winget-pkgs)
            ├─ aur        (PKGBUILD push over SSH)
            └─ chocolatey (nupkg push, then human moderation)
```

Principles that make this recoverable:

- Build once. Every channel publishes from the same checksummed bundle, so a retry never rebuilds different bytes.
- Idempotent publishers. Each channel job checks whether its version already exists and skips the write if so. Re-running the workflow with an existing tag is a safe way to resume a partial failure.
- Fail loudly. No fallback credentials. If OIDC breaks, the npm job fails visibly instead of silently publishing through a token with weaker guarantees.
- Immutable tags. Protect `v*` tags with a ruleset so nothing can move or delete what a checksum points at.

The npm side uses a launcher-plus-platform-packages layout: one launcher package with `optionalDependencies` on six platform packages (`linux`/`darwin`/`windows` × `x64`/`arm64`), each carrying a single binary. That is seven packages to publish and seven trusted publishers to configure.

---

## GitHub Releases

The simplest channel: the workflow's own `GITHUB_TOKEN` uploads archives and a `checksums.txt`. Nothing to configure and nothing to expire.

Two lessons that cost me a release:

- GoReleaser reads `GITHUB_TOKEN`, the `gh` CLI reads `GH_TOKEN`. If your build step only exports one of them, the other tool sends an empty authorization header and fails with `401 Bad credentials`. Export both.
- The first release with a predecessor behaves differently from the very first release. GoReleaser's changelog calls the compare API (`compare/v0.1.0...v0.2.0`) only when there is something to compare against, so a missing token can hide until release two.

Publish draft-first: create the release as a draft, upload and verify the checksummed assets, then flip it to published. A half-uploaded draft is invisible; a half-uploaded published release is not.

### Linux distro packages

GoReleaser's [nFPM integration](https://goreleaser.com/customization/nfpm/) builds native Linux packages from the same binaries at no extra cost: `.deb` (Debian/Ubuntu), `.rpm` (Fedora/RHEL/openSUSE), `.apk` (Alpine), and `.pkg.tar.zst` (Arch), per architecture. They ship as GitHub release assets alongside the archives, so users can `dpkg -i` or `rpm -i` a download directly without any repository setup on my side.

This is distinct from the AUR channel below: the `.pkg.tar.zst` asset is a prebuilt package someone can `pacman -U`, while the AUR entry is a PKGBUILD that fetches and verifies the release archives. Hosting actual apt/rpm repositories (with signing and index metadata) is a separate undertaking this guide does not cover; the release-asset route gets native packages into users' hands without it.

---

## npm (Trusted Publishing)

Trusted publishing exchanges the GitHub Actions OIDC identity for a single-use publish credential at publish time. There is no stored secret, nothing to rotate, and public packages published from public repositories get [provenance](https://docs.npmjs.com/generating-provenance-statements) automatically: a "Built and signed on GitHub Actions" badge that links each version to the exact workflow run that built it.

Requirements: a GitHub-hosted runner, Node 22.14+, npm 11.5.1+, and `id-token: write` in the workflow permissions.

### Configuring a trusted publisher

Each package needs its own trusted publisher. On npmjs.com go to Package → Settings → Trusted Publisher → GitHub Actions and enter:

| Field | Value |
|---|---|
| Organization or user | your GitHub username |
| Repository | the repository name |
| Workflow filename | the **top-level** workflow file |
| Environment | leave empty unless you use one |
| Allowed actions | `npm publish` |

The workflow filename is the one gotcha. If your publish job lives in a reusable workflow called from another workflow, npm validates against the **calling** workflow, not the reusable one. Point the trusted publisher at the entry-point file (for me that is `release-please.yml`, which calls `publish.yml`).

Newer npm versions can do this from the CLI in a logged-in session:

```sh
npm trust github PACKAGE_NAME \
  --file release-please.yml \
  --repo YOUR_USER/YOUR_REPO \
  --allow-publish
```

npm does not validate the configuration when you save it. A wrong workflow filename or repository surfaces only on the next real publish, which is exactly when you want it to fail loudly rather than fall back to a token.

### The bootstrap problem

npm will not let you configure a trusted publisher for a package that does not exist yet. Every new package name needs one direct publish first. The clean pattern is a throwaway `0.0.0` stub:

```sh
# minimal package.json + README, then from a logged-in npm session:
npm publish --access public --tag bootstrap --provenance=false
```

- `--tag bootstrap` keeps the stub off the `latest` dist-tag going forward, though npm assigns `latest` to a package's first publish regardless, so the stub holds `latest` until your first real release replaces it.
- `--provenance=false` because a local direct publish has no CI identity to attest.
- Publish from an interactive npm session (browser 2FA) rather than minting a token for it. If you must use a token for bulk bootstraps, make it a granular token with the shortest practical expiry and revoke it the moment the trusted publishers are configured.

Then configure the trusted publisher for each name, and lock the package down:

```sh
npm access set mfa=publish PACKAGE_NAME
```

`mfa=publish` requires 2FA for publishes and disallows tokens. OIDC trusted publishing is unaffected, so after this the only two ways to publish are your browser session and your CI workflow. npm has announced that bypass-2FA tokens stop working for package-management changes in early August 2026 and for direct publishing around January 2027, so the token-free setup is also the future-proof one.

Order matters: publish stub → configure trusted publisher → set `mfa=publish` → revoke any bootstrap token. [`scripts/npm-oidc-bootstrap.sh`](scripts/npm-oidc-bootstrap.sh) runs the whole sequence for any set of package names:

```sh
scripts/npm-oidc-bootstrap.sh all --repo OWNER/REPO PACKAGE [PACKAGE...]
```

The phases (`publish`, `trust`, `lock`, `verify`) are separate and resumable, and the stub publish has a retry loop because recently recycled names can return `409 Conflict` while the registry settles. Run one phase at a time if you prefer, and use `--packages-file names.txt` for longer lists.

### ⚠️ What npm publishes about you

npm puts account identity into public registry metadata, and most people find out too late:

- `maintainers` in the packument shows the **current** email on your npm account.
- Every published version freezes `_npmUser` with the name and email in effect **at publish time**. Changing your account email later does not touch already-published versions, because version metadata is immutable.
- Anyone can read this with `npm view PACKAGE --json`.

So before your first publish, set the npm account email to a **publishing alias** on your own domain rather than a personal address (see my [custom domain email guide](https://github.com/jishnuteegala/custom-domain-email-guide) for a free way to run one). OIDC-published versions sidestep the issue entirely: their `_npmUser` is `GitHub Actions <npm-oidc-no-reply@github.com>`.

Two related things worth knowing:

- Your npm avatar is your account email's hash looked up on Gravatar. If you want an avatar on an alias, add the alias as a verified additional email in Gravatar and assign an image to it.
- If a personal email does end up frozen into published versions, act quickly. npm's [unpublish policy](https://docs.npmjs.com/policies/unpublish) allows a full unpublish within 72 hours of publish, or later only while the package has under 300 weekly downloads, a single maintainer, and no external dependents. Wait too long and you can exceed those criteria, at which point the metadata is effectively permanent (and third-party mirrors replicate it regardless, so speed matters).

### The cost of unpublishing

A full unpublish is not a clean slate, so treat it as a last resort:

- The registry keeps a permanent tombstone: the package name, every version number ever published, and timestamps. `npm view` returns "Unpublished on DATE" forever rather than a clean 404.
- Unpublished version numbers can **never be reused**. If you unpublish 0.1.0 and 0.2.0, your next release must be 0.3.0 (with Release Please, put `Release-As: 0.3.0` in a commit body to force the jump).
- The name is locked for 24 hours before anyone can publish to it again.
- Unpublishing deletes the package settings, including trusted publisher configuration, so you are back to the bootstrap problem: stub publish, re-trust, re-lock.

### Verifying provenance after a release

```sh
version=1.2.3
for p in your-package your-package-{linux,darwin,windows}-{x64,arm64}; do
  npm view "$p@$version" --json dist.attestations \
    | node -e 'const a=JSON.parse(require("fs").readFileSync(0,"utf8"));process.exit(a&&a.provenance?0:1)' \
    && echo "$p: provenance OK" || echo "$p: NO PROVENANCE"
done
```

Also spot-check `_npmUser` on a version: OIDC publishes show `GitHub Actions <npm-oidc-no-reply@github.com>`. If a version shows your account instead, that publish went through a token and you should find out why.

---

## Homebrew and Scoop

Both channels are just commits to repositories you own: a Homebrew tap (`homebrew-tap`) and a Scoop bucket (`scoop-bucket`). The release workflow pushes an updated cask/manifest pinned to the new version's download URLs and checksums.

Credential: one fine-grained GitHub PAT (`PACKAGES_GITHUB_TOKEN`) with access to only those two repositories and a single permission, **Contents: Read and write**. No account permissions. Fine-grained PATs have a mandatory expiry, so this one goes in the tracking table.

---

## winget

The workflow pushes a version branch to your fork of [microsoft/winget-pkgs](https://github.com/microsoft/winget-pkgs) and opens a PR upstream. GoReleaser generates the manifests.

The credential (`WINGET_GITHUB_TOKEN`) is the awkward one. A fine-grained PAT scoped to your fork can push the branch but **cannot open the upstream PR**, because fine-grained PATs cannot act on repositories in organizations (like `microsoft`) that have not opted in. Use a **classic** PAT with the single `public_repo` scope. Set an expiry and track it.

Failure modes I have met:

- The PR sits in a validation pipeline and can come back with labels like `Validation-Defender-Error` (Microsoft Defender flagged the installed binary during dynamic testing). Download your own release archives, scan them locally (`MpCmdRun -Scan -ScanType 3 -File path\to\binary.exe`), and if clean, comment on the PR saying you cannot reproduce it and ask for investigation. That comment also clears the `Needs-Author-Feedback` label. There is a [validation failure guide](https://github.com/microsoft/winget-pkgs/blob/master/doc/ValidationFailureGuide.md) covering every label.
- Merges are gated on human moderators, so budget days, not minutes.
- Make the publisher script surface `gh pr create` errors rather than suppressing them, or a failed PR creation looks like success.

---

## AUR

The workflow pushes an updated PKGBUILD to the AUR over SSH. The credential (`AUR_KEY`) is a dedicated unencrypted Ed25519 private key whose public half is registered on the AUR account:

```sh
ssh-keygen -t ed25519 -f ~/.ssh/aur_yourpackage_ed25519 -N ""
gh secret set AUR_KEY --repo YOUR_USER/YOUR_REPO < ~/.ssh/aur_yourpackage_ed25519
```

Keep the keypair outside any repository. SSH keys do not expire, so the maintenance task is a quarterly check that the registered public keys on the AUR account are exactly the ones you expect.

---

## Chocolatey

The workflow packs a checksum-pinned `nupkg` and pushes it with `CHOCOLATEY_API_KEY` (from your Chocolatey account page; these keys do not expire).

The catch is moderation. A first-time package goes through automated validation, virus scanning, and then a **human moderator**, which can take days to weeks. Two consequences:

- A successful push means **submitted**, not installable. Verify approval on the package page, not in CI.
- While version N sits in the moderation queue, pushing version N+1 is rejected with a `403`. Do not fight it: wait for approval, then re-run the release workflow for the newer tag and let the idempotent publisher resubmit.

---

## Credential Tracking

Every channel credential in one table, reviewed quarterly and after any maintainer, repository, or account change. Copy this into your project docs and keep it current; the values below are the shape mine takes:

| Credential | Type | Where it lives | Expires | Quarterly check | Rotation |
|---|---|---|---|---|---|
| npm trusted publisher | OIDC config (no secret) | npm package settings | Never | Audit repo/workflow values on all packages | Update immediately if the repo or workflow renames |
| `PACKAGES_GITHUB_TOKEN` | Fine-grained PAT | GitHub Actions secret | **Yes** - note the date | Provider-side expiry, repo list, Contents permission | New PAT, update secret, verify on next release, revoke old |
| `WINGET_GITHUB_TOKEN` | Classic PAT, `public_repo` | GitHub Actions secret | **Yes** - note the date | Provider-side expiry and scope | Same as above |
| `AUR_KEY` | Ed25519 private key | GitHub Actions secret | Never | Registered public keys on AUR match expectations | New pair, add public key on AUR, update secret, verify, remove old key |
| `CHOCOLATEY_API_KEY` | API key | GitHub Actions secret | Never | Account and moderation notifications | Regenerate on Chocolatey, update secret, verify on next submission |
| `GITHUB_TOKEN` | Workflow token | Managed by Actions | Per-run | Nothing | Nothing |

Rules that keep this honest:

- Record the **expiry date** next to every PAT when you create it, because GitHub's expiry emails arrive close to the deadline and CI is where you find out otherwise.
- `gh secret list --repo YOUR_USER/YOUR_REPO` shows names and update times, never values. A recent update time proves a value was stored, not that it works.
- An existing-tag re-run is a non-destructive state check, but publishers skip writes for versions that already exist, so it cannot prove a **replacement** credential has publish authority. Final validation of a rotated credential is the next new version.
- Set secrets by letting `gh` prompt (`gh secret set NAME --repo ...` with no value argument) so tokens never enter shell history.
- If a credential may be exposed: revoke at the provider **first**, then rotate, then inspect audit logs. Deleting the GitHub secret alone revokes nothing.

---

## Release Recovery

When a release half-fails (some channels published, some did not):

1. Do not create a new version. The tag exists and some channels have the bytes; a new version turns one problem into two.
2. Fix the root cause (usually a credential or a workflow bug) on `main`.
3. Re-run the release workflow with the **existing tag** (a `workflow_dispatch` input on the entry workflow). The idempotent publishers verify completed channels and resume the missing ones from the original build bundle.
4. Verify each channel against the [monitoring checklist](#monitoring-checklist).

Design your dispatch path so manual retries enter through the same top-level workflow as normal releases; with npm trusted publishing, the calling workflow filename is part of the OIDC identity, so a different entry point fails npm's check.

---

## Monitoring Checklist

After every release:

1. The release workflow completed, every publish job green
2. GitHub release assets match `checksums.txt`
3. All npm packages exist at the new version, provenance verified, `_npmUser` shows the OIDC identity
4. Homebrew cask and Scoop manifest reference the new version
5. winget PR open or merged, validation labels clean
6. AUR package version and checksums updated
7. Chocolatey shows **approved** on the package page, not just pushed

---

## Security Notes

- No fallback credentials in the publish path. A fallback turns a loud failure into a silent downgrade; my npm launcher once published through a leftover token fallback without provenance while the six platform packages went through OIDC, and nothing failed to tell me.
- Pin third-party actions to full commit SHAs. The tj-actions/changed-files compromise worked by moving version tags to malicious code, and publish workflows hold credentials worth stealing. Let Dependabot bump the pins.
- Keep default workflow permissions read-only and grant `id-token: write` only to the job that publishes.
- Require maintainer approval before workflows run on PRs from forks, and never check out fork code in a privileged context (`pull_request_target`, `workflow_run`).
- Protect `v*` tags from deletion and force-push with a repository ruleset.
- Keep credentials in GitHub Actions secrets, never in files, and never echo them in scripts (`read -s` for interactive entry).

---

## FAQ

### Q: Why several npm packages instead of one with postinstall downloads?
**A:** Platform packages under `optionalDependencies` install only the binary for the current platform, work offline from the npm cache, and avoid postinstall scripts, which many organisations block.

### Q: Can I use one trusted publisher for all my packages?
**A:** No, each package carries its own trusted publisher configuration, and each package supports only one. Scripting the CLI (`npm trust github ...`) makes bulk setup bearable.

### Q: What if OIDC publishing breaks entirely?
**A:** The recovery path is deliberately manual: temporarily allow tokens on each affected package, mint a short-lived granular token, recover the release, then restore `mfa=publish` and revoke the token. Painful by design, because the painless version is a standing token.

### Q: Do I need to bootstrap again for every new version?
**A:** No. The bootstrap is once per package **name**. Only unpublishing a package (which deletes its settings) puts you back there.

### Q: My npm publish returns 409 Conflict, what gives?
**A:** Usually a recently unpublished or recycled name where the registry has not settled its writes. Retry with a delay; a 30-second loop with a handful of attempts has always been enough for me.

### Q: Which channels need a human in the loop?
**A:** winget (Microsoft moderators merge the PR) and Chocolatey (moderation queue for each new package, plus validation for each version). Everything else is fully automated.

---

## Changelog

Releases are cut by [Release Please](https://github.com/googleapis/release-please) from Conventional Commits, using the GitHub changelog renderer. It maintains [CHANGELOG.md](CHANGELOG.md) automatically; no manual edits needed. This guide practises what it preaches: see [The Architecture](#the-architecture).

---

## 🤝 Contributing

Found an issue or have a suggestion? Contributions are welcome!

1. Fork this repository
2. Create a feature branch: `git checkout -b feature/improvement`
3. Make your changes and commit
4. Push to your fork: `git push origin feature/improvement`
5. Open a Pull Request

For commit message conventions, see [CONTRIBUTING.md](CONTRIBUTING.md).

---

## 📝 License

This guide is released under the **MIT License** - see the [LICENSE](LICENSE) file for details.

---

## Additional Resources

Official documentation for each channel in scope:

- [GitHub: About releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)
- [npm: Trusted publishing for npm packages](https://docs.npmjs.com/trusted-publishers)
- [npm: Generating provenance statements](https://docs.npmjs.com/generating-provenance-statements)
- [npm: Unpublish policy](https://docs.npmjs.com/policies/unpublish)
- [Homebrew: Taps (third-party repositories)](https://docs.brew.sh/Taps)
- [Scoop: Buckets](https://scoop.sh/#/buckets)
- [winget: Submit packages to Windows Package Manager](https://learn.microsoft.com/en-us/windows/package-manager/package/repository)
- [Arch Wiki: AUR submission guidelines](https://wiki.archlinux.org/title/AUR_submission_guidelines)
- [Chocolatey: Create Chocolatey packages](https://docs.chocolatey.org/en-us/create/create-chocolatey-packages/)
- [Chocolatey: Moderation process](https://docs.chocolatey.org/en-us/community-repository/moderation/)

And the automation this guide builds on:

- [GoReleaser documentation](https://goreleaser.com/)
- [Release Please](https://github.com/googleapis/release-please)
