# attested-pipeline-template

A language-agnostic GitHub repository template demonstrating the **attested release architecture**: every artifact that reaches consumers is byte-identical to what was validated, carries SLSA provenance and SBOM attestations that re-verify at every hop, and publication is fail-closed on verification.

A tag publishes nothing unattested.

## The Invariant

```
build → attest-provenance → generate+attest-SBOM → fail-closed-verify → [tag-gated] publish
```

No step is skippable. If `verify` fails, `publish` does not run — GitHub Actions
enforces this via the `needs:` dependency graph.

## What's in This Template

```
.github/
  workflows/
    release.yml   # The attested release pipeline (see below)
    ci.yml        # CI + mandatory action-pin check
SECURITY.md       # Verification instructions for consumers
README.md         # This file
LICENSE           # Apache-2.0
```

### `release.yml` — The Attested Release Pipeline

| Job | What it does | Key permissions |
|---|---|---|
| `meta` | Resolves artifact name + version from the git ref | `contents: read` |
| `build` | Builds the artifact, attests SLSA build provenance | `id-token: write`, `attestations: write` |
| `sbom` | Generates a CycloneDX SBOM via Syft, attests it | `id-token: write`, `attestations: write` |
| `verify` | **Fail-closed**: verifies provenance + SBOM on every artifact | `attestations: read` |
| `publish` | Creates the GitHub Release with checksums | `contents: write` — only on tag push, only if `verify` passes |

The pipeline supports `workflow_dispatch` as a dry-run: the full
build → attest → verify chain runs, but `publish` is skipped (tag-gated).

### `ci.yml` — CI + Pin Check

Includes a mandatory `pin-check` job that inspects every workflow file and
exits 1 if any `uses:` line references an action by tag or branch name instead
of a full 40-character commit SHA.

## Adapting This Template

### 1. Use this as a template repository

Click **"Use this template"** on GitHub to create a new repository.

### 2. Replace the build step

In `release.yml`, find the `# ADAPT THIS STEP` block in the `build` job:

```yaml
- name: Build artifact
  run: |
    # Replace this with your toolchain's build command.
    # Output MUST land under dist/
    git archive --format=tar.gz ...
```

Examples by language:

**Go:**
```yaml
- name: Build artifact
  run: |
    mkdir -p dist
    GOOS=linux GOARCH=amd64 go build -o dist/${NAME}-${VERSION}-linux-amd64 ./cmd/...
```

**Rust:**
```yaml
- name: Build artifact
  run: |
    cargo build --release --locked
    mkdir -p dist
    cp target/release/mybinary dist/${NAME}-${VERSION}-linux-amd64
```

**Node.js:**
```yaml
- name: Build artifact
  run: |
    npm ci
    npm run build
    mkdir -p dist
    tar -czf dist/${NAME}-${VERSION}.tar.gz --exclude=node_modules .
```

**Docker:**
Use `actions/attest-build-provenance` with `subject-digest` pointing at the
image digest from `docker buildx build --iidfile`.

### 3. Update action SHAs

Action SHAs in this template were resolved on **2026-06-18**. Rotate them
periodically:

```bash
gh api repos/actions/checkout/commits/v4 --jq .sha
gh api repos/actions/attest-build-provenance/commits/v2 --jq .sha
gh api repos/actions/attest-sbom/commits/v2 --jq .sha
gh api repos/anchore/sbom-action/commits/v0 --jq .sha
gh api repos/softprops/action-gh-release/commits/v2 --jq .sha
gh api repos/actions/upload-artifact/commits/v4 --jq .sha
gh api repos/actions/download-artifact/commits/v4 --jq .sha
```

Update the SHA and the trailing `# vX.Y.Z` comment in the workflow file.

### 4. Update the repo reference in SECURITY.md

Replace `attested-delivery/attested-pipeline-template` with your org/repo.

### 5. Release

```bash
git tag v1.0.0
git push origin v1.0.0
```

The release pipeline fires automatically. Watch it at:
`https://github.com/<org>/<repo>/actions`

### 6. Verify a release artifact

```bash
gh attestation verify <artifact-file> \
  --repo <org>/<repo> \
  --predicate-type "https://slsa.dev/provenance/v1"
```

See [SECURITY.md](SECURITY.md) for the full verification guide.

## Pinned Actions Reference

All actions are pinned to full 40-char SHAs (resolved 2026-06-18):

| Action | Tag | SHA |
|---|---|---|
| `actions/checkout` | v4.2.2 | `34e114876b0b11c390a56381ad16ebd13914f8d5` |
| `actions/attest-build-provenance` | v2.4.0 | `e8998f949152b193b063cb0ec769d69d929409be` |
| `actions/attest-sbom` | v2.3.0 | `bd218ad0dbcb3e146bd073d1d9c6d78e08aa8a0b` |
| `anchore/sbom-action` | v0.24.0 | `e22c389904149dbc22b58101806040fa8d37a610` |
| `softprops/action-gh-release` | v2.2.2 | `3bb12739c298aeb8a4eeaf626c5b8d85266b0e65` |
| `actions/upload-artifact` | v4.6.2 | `ea165f8d65b6e75b540449e92b4886f43607fa02` |
| `actions/download-artifact` | v4.3.0 | `d3f86a106a0bac45b974a628896c90dbdf5c8093` |

## Architecture Notes

### Why fail-closed verification?

A build pipeline that attests but doesn't verify provides a false sense of
security. An attacker who can tamper with the artifact post-build (e.g., a
compromised artifact store) would produce an artifact whose attestation
verification fails. The fail-closed `verify` job catches this before publish.

### Why SHA-pinned actions?

A tag like `actions/checkout@v4` is mutable — the repo owner can move the tag
to a different commit at any time. A SHA like
`actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5` is immutable.
Supply-chain attacks against CI/CD tooling (e.g., tj-actions/changed-files,
reviewdog) exploit mutable action references.

### Why SLSA provenance?

SLSA (Supply-chain Levels for Software Artifacts) provenance attestations
cryptographically bind an artifact to the workflow run that produced it,
including the source commit SHA, the workflow file path, and the runner
environment. This prevents substitution attacks where a different build
produces a different artifact with the same name.

### Dry-run without a tag

```bash
gh workflow run release.yml
```

This exercises the full build → attest → verify chain. The `publish` job
is skipped because `startsWith(github.ref, 'refs/tags/')` is false.

## License

Apache-2.0. See [LICENSE](LICENSE).
