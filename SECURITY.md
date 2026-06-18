# Security Policy

## Reporting a Vulnerability

Please do **not** open a public GitHub issue for security vulnerabilities.

Report security issues by emailing the maintainer directly or using the
[GitHub private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing/privately-reporting-a-security-vulnerability)
feature for this repository.

We will respond within 72 hours and coordinate a fix and disclosure timeline.

---

## Verifying Release Artifacts

Every release artifact is signed with GitHub's Sigstore-backed attestation
infrastructure. Before using any artifact, consumers should verify:

1. **SLSA build provenance** — proves the artifact was produced by this
   repository's CI workflow, not tampered with post-build.
2. **SBOM attestation** — proves the artifact is bound to a CycloneDX
   Software Bill of Materials.

### Prerequisites

- [GitHub CLI](https://cli.github.com/) `gh` ≥ 2.49.0
- Authenticated: `gh auth login`

### Verify Build Provenance

```bash
gh attestation verify <artifact-file> \
  --repo attested-delivery/attested-pipeline-template \
  --predicate-type "https://slsa.dev/provenance/v1"
```

Replace `<artifact-file>` with the downloaded release file, e.g.:

```bash
gh attestation verify attested-pipeline-template-1.0.0.tar.gz \
  --repo attested-delivery/attested-pipeline-template \
  --predicate-type "https://slsa.dev/provenance/v1"
```

### Verify SBOM Attestation

```bash
gh attestation verify <artifact-file> \
  --repo attested-delivery/attested-pipeline-template \
  --predicate-type "https://cyclonedx.org/bom"
```

### Verify Both in One Script

```bash
#!/usr/bin/env bash
set -euo pipefail

ARTIFACT="$1"
REPO="attested-delivery/attested-pipeline-template"

echo "Verifying build provenance..."
gh attestation verify "$ARTIFACT" \
  --repo "$REPO" \
  --predicate-type "https://slsa.dev/provenance/v1"
echo "PASS: build provenance"

echo "Verifying SBOM attestation..."
gh attestation verify "$ARTIFACT" \
  --repo "$REPO" \
  --predicate-type "https://cyclonedx.org/bom"
echo "PASS: SBOM attestation"

echo "All attestations verified for: $ARTIFACT"
```

Usage:

```bash
bash verify-artifact.sh attested-pipeline-template-1.0.0.tar.gz
```

### What a Passing Verification Looks Like

```
Loaded digest sha256:abc123... for file://attested-pipeline-template-1.0.0.tar.gz
Loaded 1 attestation from GitHub API
✓ Verification succeeded!

repo:          attested-delivery/attested-pipeline-template@refs/tags/v1.0.0
workflow:      .github/workflows/release.yml@refs/tags/v1.0.0
```

A failed verification exits non-zero and prints an error. **Treat any
verification failure as a supply-chain integrity breach — do not use the
artifact.**

---

## What the Attestations Prove

| Attestation | Predicate type | What it proves |
|---|---|---|
| SLSA build provenance | `https://slsa.dev/provenance/v1` | The artifact was built by this repo's workflow from a specific commit SHA, at a recorded time, with no tampering possible after signing |
| SBOM | `https://cyclonedx.org/bom` | The artifact is bound to a CycloneDX bill of materials listing all components and their digests |

Attestations are stored in the GitHub Attestations API and are cryptographically
signed via Sigstore's keyless signing infrastructure (Fulcio CA + Rekor
transparency log). They cannot be forged without control of the repository's
GitHub Actions OIDC token.

---

## Supply-Chain Security Posture

- All GitHub Actions used in this repository are pinned to immutable full
  40-character commit SHAs (never mutable tags or branch names).
- The pin-check CI job (`ci.yml`) enforces this on every PR.
- A CycloneDX SBOM is generated and attested on every release.
- The release pipeline is fail-closed: `verify` must pass before `publish`
  can execute. There is no path from build to release that bypasses
  attestation verification.
