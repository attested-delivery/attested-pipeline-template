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

## Verifying gate-verdict attestations

Beyond provenance + SBOM (verified with `--repo`, above), this template
seam-signs a verdict for every artifact-characterizing gate, bound to the subject
digest: **SAST** (CodeQL), **SCA** (OSV), and **IaC/license** (Trivy) in
`release.yml`, and **DAST** (ZAP) in `dast.yml`. Each is signed by the central
attestation seam `reusable-attest-scan.yml`; under SLSA Build L3 the signer
identity is that central workflow, so verification **requires**
`--signer-workflow` (one signer/predicate per command) — `--owner`/`--repo`
alone is insufficient:

```bash
# Seam-signed gate verdicts over the subject they were bound to.
for PT in sast sca iac-license dast; do
  gh attestation verify "$SUBJECT" --owner attested-delivery \
    --signer-workflow attested-delivery/.github/.github/workflows/reusable-attest-scan.yml \
    --predicate-type "https://attested-delivery.github.io/attestations/${PT}/v1"
done
```

The **VEX** disposition is self-signed by a different workflow
(`reusable-vex.yml`), so it pins its own signer:

```bash
gh attestation verify "$SUBJECT" --owner attested-delivery \
  --signer-workflow attested-delivery/.github/.github/workflows/reusable-vex.yml \
  --predicate-type https://openvex.dev/ns/v0.2.0
```

Supply-chain posture (Scorecard) is a repo-level signal surfaced in
`quality-gates.yml`, not an artifact verdict. The Rust reference
`attested-delivery/rust-template` wires the same seam plus container-scan (Trivy
image). A passing verification proves the gate **ran and recorded a verdict**
bound to the subject digest; read the predicate body for the verdict itself
(signed ≠ passed).

---

## Supply-Chain Security Posture

- All GitHub Actions used in this repository are pinned to immutable full
  40-character commit SHAs (never mutable tags or branch names).
- The pin-check CI job (`ci.yml`) enforces this on every PR.
- A CycloneDX SBOM is generated and attested on every release.
- The release pipeline is fail-closed: `verify` must pass before `publish`
  can execute. There is no path from build to release that bypasses
  attestation verification.

---

## Trusted tool acquisition (verified fetches) — recommended for every workflow

Every step that pulls an external package, binary, or tool into a job is a
supply-chain entry point. **Adopters should apply this to *all* workflows**, not
just the release pipeline. Handle each fetch in this order of preference:

1. **Prefer the runner's pre-installed tools.** GitHub-hosted `ubuntu-latest`
   already ships `gh`, `jq`, `git`, `curl`, `wget`, `tar`, `node`, `go`,
   `python`, `docker`, and more — the runner image is your trust root. If a tool
   is already present, **use it directly and add no install step**.
2. **Otherwise, use a SHA-pinned action.** Pin every `uses:` to a full
   40-character commit SHA (the `pin-check` gate enforces this); the action's
   pin is the integrity boundary.
3. **Otherwise, download → verify → fail closed**, using the *strongest*
   integrity mechanism available, on this descending ladder — and never below a
   checksum:

   `gh attestation verify`  ›  `cosign verify`/`verify-blob` (or `gpg --verify` /
   `minisign`)  ›  `sha256sum -c` against a **pinned digest** (the minimum floor).

   Run under `set -euo pipefail`, pin the exact version, and verify **before**
   executing the artifact. Where only a checksum is available, leave a `# TODO`
   to upgrade when the publisher ships signed provenance. See `release.yml`
   (Syft) and `ci.yml` (actionlint) for the pattern:

   ```bash
   set -euo pipefail
   VERSION="1.45.1"
   SHA256="<sha256 of the pinned release artifact, from its published checksums>"
   curl -sSfL -o tool.tar.gz "https://…/v${VERSION}/tool_${VERSION}_linux_amd64.tar.gz"
   echo "${SHA256}  tool.tar.gz" | sha256sum -c -   # aborts the job on mismatch
   tar xzf tool.tar.gz -C "${RUNNER_TEMP}/bin" tool
   ```

4. **Never pipe-to-shell.** `curl … | sh`, `curl … | bash`, and `curl … | tar`
   execute unverified bytes — there is no integrity check before code runs.
   Always download to a file, verify, then run.
5. **Package installs use lockfile / registry integrity, pinned and fail-closed:**
   `npm ci` and `pnpm install --frozen-lockfile` / `yarn --immutable` (lockfile
   hashes), `corepack` (signed package-manager keys), `go install pkg@version`
   (Go checksum database), `cargo install --locked --version X` (crates.io index
   checksums). Never unpinned (`cargo install foo`) and never failure-swallowing
   (`… || true`).
