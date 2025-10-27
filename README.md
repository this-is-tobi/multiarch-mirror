# Multiarch image mirror :twisted_rightwards_arrows:

The purpose of this repository is to create multi-arch images for official images not compatible with ARM64.

## Documentation

Comprehensive documentation is available:

1. [Introduction](./docs/01-readme.md) - Overview and key features
2. [Architecture & Workflows](./docs/02-architecture.md) - How the build system works
3. [Adding New Applications](./docs/03-adding-applications.md) - Step-by-step guide for contributors
4. [Build Configurations](./docs/04-build-configurations.md) - Matrix.json reference
5. [Troubleshooting](./docs/05-troubleshooting.md) - Common issues and solutions

## Quick Start

### Using Images

```sh
# Docker automatically selects the correct architecture
docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest

# Explicitly specify architecture
docker pull --platform linux/arm64 ghcr.io/this-is-tobi/mirror/mattermost:latest
docker pull --platform linux/amd64 ghcr.io/this-is-tobi/mirror/mattermost:latest

# Verify multi-arch manifest
docker manifest inspect ghcr.io/this-is-tobi/mirror/mattermost:latest
```

### For Contributors

Each application receives one or more matrices to list the different architectures that need to be created and a workflow file to perform certain actions necessary to enable cross-platform compatibility and image construction.

See [Adding New Applications](./docs/03-adding-applications.md) for the complete guide.

## Automation

The workflow runs daily (01:00 am) or can be triggered manually from the Github user interface.

> *__Notes:__ Before each application build, the CI compares the last tag of the official version with the last tag of the mirror image to check whether a new build should be triggered.*


## Images

| Application | Pull command                                                |
| ----------- | ----------------------------------------------------------- |
| Mattermost  | `docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest` |
| Outline     | `docker pull ghcr.io/this-is-tobi/mirror/outline:latest`    |

> *__Notes:__ All images are mirrored with the official tag version.*
