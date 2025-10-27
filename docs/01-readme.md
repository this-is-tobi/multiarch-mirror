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

### Automated Daily Builds
- Scheduled workflows run daily at 01:00 AM UTC
- Automatically detect new upstream versions
- Build and push only when new versions are available
- Can be triggered manually via GitHub Actions UI

### Multi-Architecture Support
- **amd64** (x86_64) - Standard Intel/AMD processors
- **arm64** (aarch64) - ARM-based processors

### Version Synchronization
- Mirrors official release tags (e.g., `v2.14.0`)
- Includes `latest` tag for convenience
- Pre-build checks prevent unnecessary builds
- Smart version comparison logic

### Production Ready
- Published to GitHub Container Registry (GHCR)
- Multi-arch manifests for automatic platform selection
- Docker Buildx for true cross-compilation
- Comprehensive testing and verification

## Currently Supported Applications

### Mattermost
Open-source team collaboration platform (Slack alternative).

**Pull command:**
```sh
docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest
```

**Architecture:** Single binary application with multi-arch support

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
│       ├── build.yml   # Daily orchestrator
│       ├── mattermost.yml
│       └── outline.yml
├── apps/
│   ├── mattermost/     # Mattermost build configuration
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

## Version Policy

- **Official version tags**: Mirror exact upstream versions (e.g., `v9.11.0`, `v2.14.0`)
- **Latest tag**: Always points to most recent version
- **No pre-releases**: Only stable releases are mirrored
- **Automatic updates**: New versions detected and built daily

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
