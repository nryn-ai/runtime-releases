# Nryn Runtime releases

This repository holds **only signed release assets** of Nryn Runtime
(`nryn-runtimed`), the GGUF inference engine of Nryn OS. Its source is
private. Nothing here is edited after publication: each release is
immutable, and a bad release is superseded by a new version.

## What a release contains

For release `v<version>`:

| Asset | |
|---|---|
| `nryn-runtime_<version>_linux_amd64.tar.gz` | Linux x86-64: `nryn-runtimed` (static ELF) and `THIRD_PARTY_NOTICES.md`, at the top level |
| `nryn-runtime_<version>_linux_arm64.tar.gz` | Linux aarch64: likewise |
| `nryn-runtime_<version>_linux_<arch>.cdx.json` | CycloneDX 1.6 SBOM |
| `nryn-runtime_<version>_linux_<arch>.THIRD_PARTY_NOTICES.md` | third-party notices (also in the tarball) |
| `COPYLEFT-NONE.txt` | statement that no component requires its source to travel with the binary |
| `SHA256SUMS` | SHA-256 of every asset above |
| `SHA256SUMS.sigstore.json` | keyless cosign signature of `SHA256SUMS`, made by this repository's release workflow |

`nryn-runtimed` ships under the Nryn licence, which Nryn OS installs at
`/usr/share/licenses/nryn/LICENSE`.

## Verifying a release

```sh
v=0.5.0
cosign verify-blob SHA256SUMS --bundle SHA256SUMS.sigstore.json \
  --certificate-identity "https://github.com/nryn-ai/runtime-releases/.github/workflows/release.yml@refs/tags/v$v" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
sha256sum -c SHA256SUMS
```

A signature from any other identity is not a Nryn release.

## How a release is made

The source repository is tagged `v<version>` first (its `VERSION` file must
say `<version>`). Pushing the same tag here runs
`.github/workflows/release.yml`. Its jobs each hold only the permission they
need:

1. **preflight** (read): the tag is `vMAJOR.MINOR.PATCH` and no release of
   it exists.
2. **build** (read; in the protected `release` environment, which holds the
   secrets), natively on x86-64 and aarch64 runners, with the Go toolchain
   pinned by SHA-256:
   - checks out the private source at the same tag with a read-only deploy
     key, and runs its whole test suite;
   - builds both architectures (`CGO_ENABLED=0`, `-trimpath`, no VCS stamp,
     empty build id, times clamped to the source commit) and packages them;
   - runs the acceptance check of the supply spec on both binaries (strings
     gate; on the native one also `--version`, `--licence`, every help text);
   - unpacks the tarball the way the Nryn OS image build does, checks the
     binary is static, and runs the engine's invariant check (`verify`) on a
     generated tiny model.
3. **sign** (`id-token: write` only): checks that both runners built
   byte-identical binaries, writes `SHA256SUMS`, signs it keylessly, and
   verifies the signature.
4. **publish** (`contents: write` only): verifies the signature and every
   checksum again, then creates the release.

## Setting up this repository (once, before the first tag)

- **Environment `release`** (Settings → Environments): add the secrets
  `RUNTIME_SOURCE_DEPLOY_KEY` (the private half of a read-only deploy key
  added to `nryn-ai/runtime`) and `OLD_NAMES_PATTERN` (the regular
  expression of the runtime supply spec, §1.1). Limit its deployment
  branches and tags to `v*` tags; add required reviewers if a release should
  wait for approval. Keep no repository-level secrets.
- **Tag ruleset** (Settings → Rules → Rulesets): target tags `v*`; restrict
  creation, update and deletion; bypass list: repository admins (the owner)
  only.
- **Immutable releases** (Settings → General): on.
- **Actions** (Settings → Actions): default `GITHUB_TOKEN` permissions
  read-only; allow only the actions this workflow pins.
