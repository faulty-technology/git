# Signing and verification

Two independent signatures cover every release produced by this repo:

1. **Upstream source signature** — the `.tar.xz` downloaded from kernel.org is GPG-verified against Junio C Hamano's maintainer key before anything is compiled.
2. **Build-output attestation** — each `.tar.xz` asset we publish carries a SLSA v1 build-provenance statement, keyless-signed via GitHub OIDC + Sigstore Fulcio and logged to Rekor.

Either signature can be verified independently of this repo.

## Upstream source: Junio C Hamano's maintainer key

Pinned fingerprint:

```
96E0 7AF2 5771 9559  80DA D100 20D0 4E5A 7136 60A7
```

- Key type: RSA 4096.
- Primary UID: `Junio C Hamano <gitster@pobox.com>`.
- Created: 2011-10-01.
- This key signs the `git-X.Y.Z.tar.sign` files hosted at `mirrors.edge.kernel.org/pub/software/scm/git/`.

### How we obtain the key

The workflow fetches the key over HTTPS from the kernel.org docs PGP repo:

```
https://git.kernel.org/pub/scm/docs/kernel/pgpkeys.git/plain/keys/20D04E5A713660A7.asc
```

The fetched key's fingerprint is then compared against the pinned constant above. If they do not match, the build fails with `::error::Maintainer key fingerprint mismatch` and no compilation occurs.

### How the tarball itself is verified

```
xz -d git-${VERSION}.tar.xz
gpg --verify git-${VERSION}.tar.sign git-${VERSION}.tar
```

The `.tar.sign` file is an OpenPGP signature over the **uncompressed** tar (upstream convention). `gpg --verify` will exit non-zero on any modification, and the build fails loudly.

### Key rotation procedure

If Junio rotates his signing key upstream, our build will fail with a fingerprint-mismatch error on the next scheduled run. Do not update the pinned constant based on a single source. To rotate:

1. Read the rotation announcement on the `git@vger.kernel.org` mailing list (archived at [lore.kernel.org/git](https://lore.kernel.org/git/)).
2. Cross-check the new fingerprint from **at least two** independent sources:
   - the mailing-list announcement,
   - the `junio-gpg-pub` tag in [git/git](https://github.com/git/git),
   - the kernel.org pgpkeys repo,
   - keyserver lookups (`keys.openpgp.org`).
3. Open a PR updating `JUNIO_FINGERPRINT` in `.github/workflows/build-git.yml` with the new value. Include the announcement URL and all cross-check sources in the PR description.
4. Require human review before merge. This PR is the only place our build trust gets re-rooted; treat it like a key-ceremony change.

## Build outputs: SLSA v1 provenance attestations

Each `.tar.xz` asset we publish is attested via `actions/attest-build-provenance@v2`. The attestation records:

- the artifact digest,
- the exact workflow file and commit SHA that produced it,
- the runner image used,
- the inputs that triggered the run (scheduled vs manual, any `version`/`force` values),
- a transparency-log entry in Rekor for non-repudiation.

The attestation is signed with a short-lived certificate issued by Sigstore Fulcio, bound to the GitHub OIDC token of the workflow run. No long-lived private key exists, on our side or anyone else's.

### Verifying an attestation

```
gh attestation verify git-<version>-<platform>.tar.xz --repo faulty-technology/git
```

Under the hood `gh` pulls the attestation, verifies the signing cert chains to Sigstore's trusted root, confirms the Rekor inclusion proof, and validates that the build claim matches this repo.

A non-zero exit means do not trust the artifact.

## Why both?

The upstream GPG check ensures _"this is the source code Junio released"_. The build attestation ensures _"this binary was built from that source, by this repo's unmodified workflow, on a GitHub-hosted runner"_. Together they give a trust chain from Junio's key all the way to your `/opt/git/bin/git` binary, with no silent tampering possible in between.

## What's deliberately not included

- **macOS codesigning / notarization.** Requires an Apple Developer account and secrets. Not currently worth the operational cost. Workaround for downloaders: `xattr -dr com.apple.quarantine /opt/git`.
- **Our own GPG signing key for release assets.** Deliberately avoided: it would add a long-lived secret to manage and rotate without adding trust that isn't already established by the SLSA attestation.
- **Fully reproducible builds across arbitrary distros.** We pin to specific runner images and deterministic tar/xz flags, so builds are byte-identical on the same runner image. Cross-distro reproducibility is best-effort.
