# git-builds

Automated, signed, reproducible builds of the latest stable upstream `git` for:

| Platform     | Runner              |
|--------------|---------------------|
| linux-x86_64 | `ubuntu-24.04`      |
| linux-arm64  | `ubuntu-24.04-arm`  |
| macos-arm64  | `macos-latest`      |

A GitHub Actions workflow checks upstream tags every Monday for the newest non-rc `vX.Y.Z`, downloads the matching tarball from `mirrors.edge.kernel.org`, GPG-verifies it against Junio C Hamano's pinned maintainer key, builds it, and publishes a GitHub Release with per-platform `.tar.xz` assets plus a SLSA v1 build-provenance attestation for each asset.

## Install a release

```
curl -LO https://github.com/faulty-technology/git/releases/latest/download/git-<version>-<platform>.tar.xz
mkdir -p ~/.local
tar -C ~/.local -xJf git-<version>-<platform>.tar.xz
export PATH="$HOME/.local/bin:$PATH"
git --version
```

Builds are compiled with `RUNTIME_PREFIX=YesPlease`, so `git` locates its libexec helpers, templates, and system config relative to the binary at runtime. The tarball is a flat `bin/`, `libexec/`, `lib/`, `share/`, `etc/` tree that merges into whichever directory you extract into — pick anywhere. Third-party shared libraries (libcurl, libpcre2, and their transitive deps) are bundled in `lib/` and loaded via rpath, so the binary runs without any additional system packages. Other common targets:

- System-wide: `sudo tar -C /usr/local -xJf git-<version>-<platform>.tar.xz`
- Isolated: `sudo mkdir -p /opt/git && sudo tar -C /opt/git -xJf git-<version>-<platform>.tar.xz`

On macOS, if Gatekeeper balks because the binary is unsigned:

```
sudo xattr -dr com.apple.quarantine <install-root>
```

Notarization would require an Apple Developer account; it is not currently in scope.

## Verify a release

Every release asset is covered by a GitHub Actions build-provenance attestation (SLSA v1, keyless-signed via GitHub OIDC + Sigstore Fulcio, recorded in the Rekor transparency log). To verify:

```
gh attestation verify git-<version>-<platform>.tar.xz --repo faulty-technology/git
```

`gh attestation verify` will confirm:

- the artifact was produced by **this repo's** workflow,
- by the `build` job at the commit pinned in the attestation,
- from a GitHub-hosted runner, with no post-build tampering.

The `SHA256SUMS` file on the release lists the digest of every asset. `runner-*.meta` files record the exact runner image version used for each leg, so you can reproduce bit-for-bit on the same image.

For the upstream-source verification story (GPG, pinned fingerprint, key rotation), see [`SIGNING.md`](SIGNING.md).

## Triggering a build manually

**Actions → Build git → Run workflow**. Optional inputs:

- `version`: override the auto-detected version, e.g. `2.54.0`.
- `force`: `true` to delete and rebuild an existing release for the same version.

The scheduled run skips automatically if a release for the detected version already exists.

## What this repo contains

- `.github/workflows/build-git.yml` — the build + release workflow.
- `SIGNING.md` — how source and output integrity is established.
- No source code of our own — upstream is pulled fresh on each run.
