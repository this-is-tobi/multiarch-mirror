# Image Attestations and Signatures

This document explains the security features provided through image attestations and signatures.

## Overview

All images built by this project include comprehensive security attestations:
- **SBOM (Software Bill of Materials)** - Complete inventory of software components
- **Cryptographic Signatures** - Verification of image authenticity
- **SLSA Provenance** - Build process metadata and traceability

These attestations are automatically generated for every image build and stored as OCI artifacts in GitHub Container Registry (GHCR).

## Features

### SBOM (Software Bill of Materials)

Every image includes a Software Bill of Materials in SPDX format, generated using Trivy:
- Lists all packages and dependencies in the image
- Includes version information for each component
- Helps identify vulnerable dependencies
- Supports supply chain security requirements

### Cryptographic Signatures

All images are signed using Cosign with keyless signing:
- Uses GitHub OIDC for identity verification
- No secret key management required
- Signatures stored in GHCR alongside images
- Verifiable by anyone with Cosign installed

### SLSA Provenance

Build provenance metadata following SLSA v1.0 specification:
- Workflow and repository information
- Build parameters (registry, namespace, version)
- Build invocation ID and timestamps
- Upstream source material references
- Completeness indicators

## Verifying Images

### Prerequisites

Install the required tools:
```bash
# Install Cosign
brew install cosign

# Or download from https://github.com/sigstore/cosign/releases
```

### Verify Image Signature

Verify that an image was built by this repository:

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/this-is-tobi/multiarch-mirror" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/this-is-tobi/mirror/mattermost:10.3.1
```

Successful verification output:
```
Verification for ghcr.io/this-is-tobi/mirror/mattermost:10.3.1 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The code-signing certificate was verified using trusted certificate authority certificates
```

### View SBOM

View the Software Bill of Materials:

```bash
cosign verify-attestation \
  --type spdxjson \
  --certificate-identity-regexp "https://github.com/this-is-tobi/multiarch-mirror" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/this-is-tobi/mirror/mattermost:10.3.1 | jq -r .payload | base64 -d | jq .
```

To extract just the package list:

```bash
cosign verify-attestation \
  --type spdxjson \
  --certificate-identity-regexp "https://github.com/this-is-tobi/multiarch-mirror" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/this-is-tobi/mirror/mattermost:10.3.1 | \
  jq -r '.payload' | base64 -d | jq '.predicate.packages[] | {name: .name, version: .versionInfo}'
```

### View Provenance

View build provenance metadata:

```bash
cosign verify-attestation \
  --type slsaprovenance \
  --certificate-identity-regexp "https://github.com/this-is-tobi/multiarch-mirror" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/this-is-tobi/mirror/mattermost:10.3.1 | jq -r .payload | base64 -d | jq .
```

## Implementation Details

### Attestation Workflow

For each built image, the attestation process:

1. **Install Tools**
   - Cosign v3 (via sigstore/cosign-installer@v3)
   - Trivy (via aquasecurity/setup-trivy@v0.2.0)

2. **Generate SBOM**
   - Scan image with Trivy
   - Output SPDX JSON format
   - Includes all packages and dependencies

3. **Attest SBOM**
   - Attach SBOM as attestation to version tag
   - Attach SBOM to latest tag (if applicable)
   - Type: `spdxjson`

4. **Sign Images**
   - Sign version tag with Cosign
   - Sign latest tag (if applicable)
   - Uses keyless signing with GitHub OIDC

5. **Generate and Attest Provenance**
   - Create SLSA provenance metadata
   - Include build parameters and materials
   - Attach to version tag and latest tag (if applicable)
   - Type: `slsaprovenance`

### Storage

All attestations are stored in GHCR as OCI artifacts:
- No external storage required
- Attestations linked to specific image digests
- Accessible via standard OCI registry APIs
- Automatically replicated with images

### Permissions

The attestation job requires:
```yaml
permissions:
  contents: read      # Read repository content
  packages: write     # Write to GHCR
  id-token: write     # OIDC token for keyless signing
```

## Security Considerations

### Keyless Signing

This project uses Cosign's keyless signing mode:
- **No private keys to manage** - Eliminates key rotation and storage concerns
- **GitHub OIDC identity** - Signatures tied to GitHub Actions workflow identity
- **Public transparency log** - All signatures recorded in Rekor (public ledger)
- **Certificate-based verification** - Uses short-lived certificates from Fulcio CA

### Trust Model

When verifying images:
1. Trust GitHub's OIDC provider (`token.actions.githubusercontent.com`)
2. Trust the repository identity (`this-is-tobi/multiarch-mirror`)
3. Trust the public transparency log (Rekor)
4. Trust the certificate authority (Fulcio)

This eliminates the need to trust any long-lived secrets while maintaining strong cryptographic guarantees.

### Supply Chain Security

Attestations enable:
- **Dependency tracking** - Know exactly what's in your images
- **Vulnerability management** - Scan SBOMs for known CVEs
- **Build reproducibility** - Provenance links images to source
- **Compliance** - Meet regulatory requirements for software transparency

## Examples

### Verify All Tags for an Application

Verify multiple versions:

```bash
for tag in 10.3.1 10.3.0 10.2.1 latest; do
  echo "Verifying mattermost:$tag..."
  cosign verify \
    --certificate-identity-regexp "https://github.com/this-is-tobi/multiarch-mirror" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
    ghcr.io/this-is-tobi/mirror/mattermost:$tag
done
```

### Extract SBOM to File

Save SBOM for offline analysis:

```bash
cosign verify-attestation \
  --type spdxjson \
  --certificate-identity-regexp "https://github.com/this-is-tobi/multiarch-mirror" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/this-is-tobi/mirror/mattermost:10.3.1 | \
  jq -r '.payload' | base64 -d | jq '.predicate' > mattermost-10.3.1-sbom.json
```

### Scan SBOM for Vulnerabilities

Use Trivy to scan the extracted SBOM:

```bash
trivy sbom mattermost-10.3.1-sbom.json
```

## Troubleshooting

### Verification Fails

If signature verification fails:

1. **Check repository identity** - Ensure you're using the correct certificate identity regex
2. **Check image digest** - Attestations are linked to specific digests, not tags
3. **Check network** - Verification requires access to transparency log and certificate authority
4. **Check Cosign version** - Use Cosign v2.0 or later

### SBOM Not Found

If SBOM attestation is missing:

1. **Check image build date** - Attestations were added on [date], older images won't have them
2. **Check image source** - Only images built by this repository have attestations
3. **Use correct type** - Specify `--type spdxjson` not `--type spdx`

### Provenance Not Found

If provenance attestation is missing:

1. **Check attestation type** - Use `--type slsaprovenance`
2. **Check workflow status** - Attestation job may have failed during build
3. **Check permissions** - Ensure GHCR allows reading attestations

## References

- [Cosign Documentation](https://docs.sigstore.dev/cosign/overview/)
- [SLSA Provenance](https://slsa.dev/spec/v1.0/provenance)
- [SPDX Specification](https://spdx.dev/specifications/)
- [Trivy SBOM Generation](https://aquasecurity.github.io/trivy/latest/docs/supply-chain/sbom/)
- [Sigstore Project](https://www.sigstore.dev/)
