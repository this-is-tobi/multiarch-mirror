# Multiarch Image Mirror :twisted_rightwards_arrows:

This project provides multi-architecture (amd64 + arm64) Docker images for applications that don't officially support ARM64.

## Overview

This project aims to:
- **Enable ARM64 compatibility** for applications without official ARM64 images
- **Automate multi-arch builds** using GitHub Actions and Docker Buildx
- **Mirror official versions** with corresponding tags
- **Provide production-ready images** via GitHub Container Registry (GHCR)

## Why Multi-Arch Mirrors?

Many popular enterprise applications don't provide official ARM64 images, limiting deployment options on:
- Apple Silicon Macs (M1/M2/M3)
- ARM-based servers (AWS Graviton, Azure Ampere)
- Raspberry Pi and edge devices
- ARM Kubernetes clusters

This repository bridges that gap by creating multi-architecture images that work seamlessly across platforms.

## Key Features

### Automated Multi-Version Builds
- Scheduled workflows run every 6 hours (00:00 / 06:00 / 12:00 / 18:00 UTC)
- Actively builds the **last 10 versions** of each application (configurable)
- All built versions remain in registry (never deleted - storage accumulates over time)
- Mirrors official release tags (e.g., `10.3.1`, `0.82.2`)
- Automatically detect new upstream versions
- Build only missing images (efficient incremental updates)
- Smart semantic versioning to determine true "latest"
- Version-specific tags for reproducible deployments
- Can be triggered manually via GitHub Actions UI

### Multi-Architecture Support
- **amd64** (x86_64) - Standard Intel/AMD processors
- **arm64** (aarch64) - ARM-based processors
- Multi-arch manifests for automatic platform selection
- Docker Buildx for true cross-compilation

### Security and Attestations
- **SBOM (Software Bill of Materials)** - Complete dependency inventory in SPDX format
- **Cryptographic signatures** - Cosign keyless signing with GitHub OIDC
- **SLSA Provenance** - Build metadata and traceability
- All attestations stored in GHCR as OCI artifacts
- Verifiable with standard Cosign tools

### Production Ready
- Published to GitHub Container Registry (GHCR)
- Pre-build checks prevent unnecessary rebuilds
- Comprehensive testing and verification
- No authentication required for pulling images

## Currently Supported Applications

### Mattermost
Open-source team collaboration platform (Slack alternative).

**Pull command:**
```sh
docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest
```

**Architecture:** Single binary application with multi-arch support

### Mostlymatter
Fork of Mattermost by Framasoft without user limitations (Slack alternative without restrictions).

**Pull command:**
```sh
docker pull ghcr.io/this-is-tobi/mirror/mostlymatter:latest
```

**Architecture:** Single binary application with multi-arch support (same as Mattermost)  
**Source:** Framagit - [framasoft/framateam/mostlymatter](https://framagit.org/framasoft/framateam/mostlymatter)

### Outline
Modern wiki and knowledge base (Notion alternative).

**Pull command:**
```sh
docker pull ghcr.io/this-is-tobi/mirror/outline:latest
```

**Architecture:** Node.js application with ARM64 compatibility

## Supported Platforms

- **GitHub Actions** - Build infrastructure
- **Docker Buildx** - Multi-platform image builder
- **GHCR** - Image registry
- **QEMU** (optional) - Emulation for cross-platform builds

## Project Structure

```
multi-arch-mirror/
├── .github/
│   └── workflows/      # GitHub Actions workflows
│       ├── build.yml   # Orchestrator (runs every 6 hours)
│       ├── mostlymatter.yml
│       ├── mattermost.yml
│       └── outline.yml
├── apps/
│   ├── mattermost/     # Mattermost build configuration
│   ├── mostlymatter/   # Mostlymatter build configuration
│   └── outline/        # Outline build configuration
├── docs/               # Documentation (you are here!)
└── README.md           # Project overview
```

## Quick Start

### Using the Images

Simply use the pull command with your desired architecture:

```sh
# Docker will automatically select the correct architecture
docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest

# Explicitly specify architecture
docker pull --platform linux/arm64 ghcr.io/this-is-tobi/mirror/mattermost:latest
docker pull --platform linux/amd64 ghcr.io/this-is-tobi/mirror/mattermost:latest

# Verify multi-arch manifest
docker manifest inspect ghcr.io/this-is-tobi/mirror/mattermost:latest
```

## How It Works

1. **Version Detection**: Workflow checks latest official release
2. **Build Decision**: Compares with existing mirror tag
3. **Multi-Arch Build**: Docker Buildx creates images for both architectures
4. **Push by Digest**: Individual architecture images pushed separately
5. **Manifest Creation**: Multi-arch manifest created pointing to both digests
6. **Tagging**: Images tagged with version and `latest`
7. **Attestations**: SBOM, signatures, and provenance generated and attached

## Security and Verification

All images include comprehensive security attestations. To verify an image:

```bash
# Install Cosign
brew install cosign

# Verify image signature
cosign verify \
  --certificate-identity-regexp "https://github.com/this-is-tobi/multiarch-mirror" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/this-is-tobi/mirror/mattermost:10.3.1
```

For complete documentation on attestations and verification, see [Attestations Documentation](07-attestations.md).

## Version Policy

- **Multi-version building**: Actively builds the last 10 versions (configurable) - older built versions remain available but stop receiving updates
- **Storage policy**: All built versions remain in registry forever (never deleted - storage accumulates over time)
- **Official version tags**: Mirror exact upstream versions (e.g., `10.3.1`, `0.82.2`)
- **Latest tag**: Points to highest semantic version (not just most recent release)
- **No pre-releases**: Only stable releases are mirrored
- **Automatic updates**: New versions detected and built every 6 hours
- **Incremental builds**: Only missing versions are built (efficient)

## Registry Information

**Registry:** GitHub Container Registry (ghcr.io)  
**Namespace:** `this-is-tobi/mirror`  
**Visibility:** Public  
**Authentication:** Not required for pulling

## Related Resources

- [Docker Buildx Documentation](https://docs.docker.com/buildx/working-with-buildx/)
- [Multi-platform Images Guide](https://docs.docker.com/build/building/multi-platform/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GHCR Documentation](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

## Contributing

This is a personal project, but contributions are welcome! Feel free to:
- Open issues for new application requests
- Submit pull requests with improvements
- Share feedback and suggestions

## License

This project is open source and available for personal and commercial use.
