# Multiarch image mirror :twisted_rightwards_arrows:

The purpose of this repository is to create multi-arch images for official images not compatible with ARM64.

## Documentation

Comprehensive documentation is available:

1. [Introduction](./docs/01-readme.md) - Overview and key features
2. [Architecture & Workflows](./docs/02-architecture.md) - How the build system works (multi-version by default)
3. [Adding New Applications](./docs/03-adding-applications.md) - Step-by-step guide for contributors
4. [Build Configurations](./docs/04-build-configurations.md) - Matrix.json reference
5. [Troubleshooting](./docs/05-troubleshooting.md) - Common issues and solutions
6. [Legacy Mode](./docs/06-legacy-mode.md) - Deprecated single-version mode (will be removed)

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

The workflow runs every 6 hours (00:00 / 06:00 / 12:00 / 18:00 UTC) or can be triggered manually from the Github user interface.

### Multi-Version Build System

The CI actively builds the **last 10 versions** of each application (configurable via `MAX_VERSIONS`). Key features:
- Fetches the last N versions from upstream and builds any missing ones
- All built versions remain in registry (never deleted - storage accumulates over time)
- Version-specific tags maintained alongside `latest`
- Only missing images are built (efficient incremental updates)
- True "latest" is determined using semantic versioning (not release date)

> *__Notes:__ The CI checks which versions already exist in the registry and builds only the missing ones. The `latest` tag points to the highest semantic version. Old versions are never deleted but stop receiving updates once they fall outside the N most recent versions.*


## Images

| Application  | Pull command                                                  | Source                                                            | Available Tags                              |
| ------------ | ------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------- |
| Mattermost   | `docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest`   | [GitHub](https://github.com/mattermost/mattermost)                | `latest`, `10.3.1`, `10.3.0`, `10.2.1`, ... |
| Mostlymatter | `docker pull ghcr.io/this-is-tobi/mirror/mostlymatter:latest` | [Framagit](https://framagit.org/framasoft/framateam/mostlymatter) | `latest`, `11.0.4`, `11.0.3`, `11.0.2`, ... |
| Outline      | `docker pull ghcr.io/this-is-tobi/mirror/outline:latest`      | [GitHub](https://github.com/outline/outline)                      | `latest`, `0.82.2`, `0.82.1`, `0.82.0`, ... |

> *__Notes:__ All images are mirrored with official version tags. The CI actively builds the last 10 versions (older built versions remain available but stop receiving updates). Use `latest` for the most recent version or specify a version tag for reproducible deployments.*
