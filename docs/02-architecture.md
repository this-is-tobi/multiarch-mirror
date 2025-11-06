# Architecture & Workflows

This document explains how the multi-arch mirror build system works, including GitHub Actions workflows, Docker Buildx integration, and the automated build pipeline.

## Overview

The build system consists of:
- **Orchestrator workflow** - Daily scheduler that triggers app-specific workflows
- **App-specific workflows** - Individual build pipelines for each application
- **Matrix strategy** - Parallel builds for different components and architectures
- **Docker Buildx** - Multi-platform image builder
- **Push-by-digest strategy** - Efficient multi-arch manifest creation

## Build Pipeline Architecture

```
┌─────────────────────────────────────────────────────────┐
│  build.yml (Daily Scheduler - 01:00 AM UTC)             │
│  - Triggers: schedule / manual                          │
│  - Calls: mattermost.yml, mostlymatter.yml, outline.yml │
└──────────────────┬──────────────────────────────────────┘
                   │
       ┌───────────┴───────────┬─────────────┐
       │                       │             │
       ▼                       ▼             ▼
  ┌─────────┐             ┌─────────┐  ┌─────────┐
  │ Matter- │             │ Mostly- │  │ Outline │
  │ most    │             │ matter  │  │ Workflow│
  └────┬────┘             └────┬────┘  └────┬────┘
       │           │           │             │
       ▼           ▼           ▼             ▼
  ┌─────────────────────────────────┐
  │  Job 1: infos                   │
  │  - Read matrix.json             │
  │  - Check GitHub releases        │
  │  - Compare versions             │
  │  - Check GHCR for existing tags │
  └────────────┬────────────────────┘
               │
               ▼ (if new version)
  ┌─────────────────────────────────┐
  │  Job 2: build (Matrix Strategy) │
  │  - Clone source repo            │
  │  - Cross-compile binaries       │
  │  - Build Docker image           │
  │  - Push by digest to GHCR       │
  │  - Upload digest artifact       │
  └────────────┬────────────────────┘
               │
               ▼
  ┌─────────────────────────────────┐
  │  Job 3: merge                   │
  │  - Download all digests         │
  │  - Create multi-arch manifest   │
  │  - Tag with version + latest    │
  │  - Inspect final manifest       │
  └─────────────────────────────────┘
```

## Workflow Components

### 1. Orchestrator Workflow (`build.yml`)

**Purpose:** Schedule and coordinate all application builds

**Location:** `.github/workflows/build.yml`

**Triggers:**
- **Schedule:** Daily at 01:00 AM UTC (`cron: '0 1 * * *'`)
- **Manual:** Via GitHub Actions UI (`workflow_dispatch`)

**Structure:**
```yaml
on:
  schedule:
    - cron: '0 1 * * *'
  workflow_dispatch:

jobs:
  mattermost:
    uses: ./.github/workflows/mattermost.yml
  
  mostlymatter:
    uses: ./.github/workflows/mostlymatter.yml
    
  outline:
    uses: ./.github/workflows/outline.yml
```

**Parameters passed to apps:**
- `REGISTRY`: Target registry (default: `ghcr.io`)
- `NAMESPACE`: Target namespace (default: `this-is-tobi/mirror`)
- `MULTI_ARCH`: Build for both architectures (default: `true`)
- `USE_QEMU`: Enable QEMU emulation (default: `true`)

### 2. Application Workflows

Each application has a dedicated workflow file following the same pattern.

#### Job 1: `infos`

**Purpose:** Gather information and decide whether to build

**Steps:**
1. **Read matrix configuration**
   ```yaml
   - name: Read matrix
     run: |
       echo "matrix=$(cat apps/mattermost/matrix.json | jq -c .)" >> $GITHUB_OUTPUT
   ```

2. **Get latest official release**
   ```yaml
   - name: Get latest release
     run: |
       TAG=$(curl -s https://api.github.com/repos/mattermost/mattermost/releases/latest | jq -r '.tag_name')
       echo "tag=${TAG}" >> $GITHUB_OUTPUT
   ```

3. **Check GHCR for existing images**
   ```yaml
   - name: Check image exists
     run: |
       STATUS=$(docker manifest inspect ghcr.io/.../mattermost:${TAG} > /dev/null 2>&1 && echo "200" || echo "404")
       echo "image-status=${STATUS}" >> $GITHUB_OUTPUT
   ```

**Outputs:**
- `tag` - Latest version to build
- `matrix` - Build matrix from JSON
- `image-status` - Whether image already exists (200/404)

**Build Decision:**
- **Build if:** `image-status == '404'` (image doesn't exist)
- **Skip if:** `image-status == '200'` (image already exists)

#### Job 2: `build`

**Purpose:** Build multi-architecture images in parallel

**Depends on:** `infos` job
**Condition:** `needs.infos.outputs.image-status == '404'`

**Matrix Strategy:**
```yaml
strategy:
  matrix:
    images: ${{ fromJSON(needs.infos.outputs.matrix) }}
```

**Example matrix:**
```json
[
  {
    "component": "mattermost",
    "arch": "amd64",
    "platforms": "linux/amd64"
  },
  {
    "component": "mattermost",
    "arch": "arm64",
    "platforms": "linux/arm64"
  }
]
```

**Key Steps:**

1. **Setup Docker Buildx**
   ```yaml
   - name: Set up Docker Buildx
     uses: docker/setup-buildx-action@v3
   ```

2. **Setup QEMU (optional)**
   ```yaml
   - name: Set up QEMU
     if: inputs.USE_QEMU
     uses: docker/setup-qemu-action@v3
   ```

3. **Clone source repository**
   ```yaml
   - name: Checkout source
     run: |
       git clone --depth 1 --branch ${{ needs.infos.outputs.tag }} \
         https://github.com/mattermost/mattermost.git .
   ```

4. **Cross-compile binaries** (for applications requiring it)
   - Uses temporary Dockerfiles with `FROM --platform=$BUILDPLATFORM`
   - Sets `GOOS=linux GOARCH=${ARCH}`
   - Extracts binaries with `--output type=local`
   - Places in expected locations

5. **Build Docker image**
   ```yaml
   - name: Build and push
     uses: docker/build-push-action@v5
     with:
       platforms: ${{ matrix.images.platforms }}
       outputs: type=image,name=...,push-by-digest=true,name-canonical=true,push=true
   ```

6. **Export digest**
   ```yaml
   - name: Export digest
     run: |
       mkdir -p /tmp/digests/${{ matrix.images.component }}
       digest="${{ steps.build.outputs.digest }}"
       touch "/tmp/digests/${{ matrix.images.component }}/${digest#sha256:}"
   ```

7. **Upload digest artifact**
   ```yaml
   - name: Upload digest
     uses: actions/upload-artifact@v4
     with:
       name: digests-${{ matrix.images.component }}-${{ matrix.images.arch }}
       path: /tmp/digests/${{ matrix.images.component }}/*
   ```

**Build Cache:**
- Uses local cache (`/tmp/.buildx-cache`)
- Cache rotation to prevent unlimited growth
- Significantly speeds up subsequent builds

#### Job 3: `merge`

**Purpose:** Create multi-architecture manifests

**Depends on:** `infos`, `build` jobs
**Condition:** `needs.infos.outputs.image-status == '404'`

**Matrix Strategy:** Runs once per component

**Key Steps:**

1. **Download all digests**
   ```yaml
   - name: Download digests
     uses: actions/download-artifact@v4
     with:
       pattern: digests-${{ matrix.component }}-*
       path: /tmp/digests/${{ matrix.component }}
       merge-multiple: true
   ```

2. **Generate tags**
   ```yaml
   - name: Docker meta
     uses: docker/metadata-action@v5
     with:
       images: ghcr.io/.../mattermost
       tags: |
         type=raw,value=${{ needs.infos.outputs.tag }}
         type=raw,value=latest
   ```

3. **Create manifest and push**
   ```yaml
   - name: Create manifest list and push
     run: |
       docker buildx imagetools create \
         -t ghcr.io/.../mattermost:${{ needs.infos.outputs.tag }} \
         -t ghcr.io/.../mattermost:latest \
         $(printf 'ghcr.io/.../mattermost@sha256:%s ' *)
   ```

4. **Inspect final manifest**
   ```yaml
   - name: Inspect image
     run: |
       docker buildx imagetools inspect ghcr.io/.../mattermost:${{ needs.infos.outputs.tag }}
   ```

**Output:** Multi-arch manifest containing both amd64 and arm64 digests

## Docker Buildx Integration

### What is Docker Buildx?

Docker Buildx is the next-generation build system that supports:
- Multi-platform image building
- Cross-compilation without emulation
- Build caching and optimization
- Push-by-digest for efficient manifest creation

### Cross-Compilation Strategy

**Traditional approach (slow):**
```
QEMU emulation → Build in emulated environment → Slow!
```

**Buildx approach (fast):**
```
Build on native platform → Cross-compile for target → Fast!
```

### Platform Variables

Docker Buildx provides automatic platform variables:

| Variable         | Description              | Example     |
| ---------------- | ------------------------ | ----------- |
| `BUILDPLATFORM`  | Platform of build host   | linux/amd64 |
| `TARGETPLATFORM` | Platform of target image | linux/arm64 |
| `TARGETOS`       | OS of target             | linux       |
| `TARGETARCH`     | Architecture of target   | arm64       |
| `TARGETVARIANT`  | Variant of target        | v7          |

**Usage in Dockerfile:**
```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.24 AS builder

ARG TARGETOS
ARG TARGETARCH

ENV GOOS=${TARGETOS}
ENV GOARCH=${TARGETARCH}

RUN go build -o /app/binary
```

### Push-by-Digest Strategy

**Why push by digest?**
- Avoids tag conflicts during parallel builds
- Enables atomic multi-arch manifest creation
- More efficient than building complete multi-arch in single job

**How it works:**
```
Build amd64 → Push with digest → sha256:abc123
Build arm64 → Push with digest → sha256:def456

Create manifest → Points to both digests → Tag as :latest
```

## Build Optimizations

### Caching Strategy

**Layer Caching:**
- Uses Docker BuildKit cache
- Caches intermediate layers locally
- Significantly speeds up repeated builds

**Cache Management:**
```yaml
cache-from: type=local,src=/tmp/.buildx-cache
cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max

# Rotate cache to prevent unlimited growth
- name: Move cache
  run: |
    rm -rf /tmp/.buildx-cache
    mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

### Parallel Matrix Builds

**Benefits:**
- Multiple architectures built simultaneously
- Multiple components built simultaneously
- Reduces total build time from hours to minutes

**Example parallelization:**
- Multi-component app: Multiple builds in parallel (N components × 2 arches)
- Single app: 2 builds in parallel (1 component × 2 arches)

### Conditional Execution

**Skip existing images:**
```yaml
if: needs.infos.outputs.image-status == '404'
```

**Skip unnecessary steps:**
```yaml
- name: Compile binary
  if: matrix.images.component == 'my-app'
```

## Security Considerations

### Registry Authentication

**GitHub Container Registry (GHCR):**
```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

**Automatic token:** GitHub provides `GITHUB_TOKEN` automatically
**Permissions:** Set in workflow file:
```yaml
permissions:
  contents: read
  packages: write
```

### Provenance

**Disabled for compatibility:**
```yaml
provenance: false
```

Multi-platform manifests can conflict with attestation manifests. Disabling provenance ensures compatibility.

## Monitoring & Debugging

### Workflow Logs

Each workflow run provides detailed logs:
- Version detection results
- Build progress for each matrix job
- Digest exports
- Manifest creation output

### Inspecting Manifests

```sh
# View multi-arch manifest
docker manifest inspect ghcr.io/this-is-tobi/mirror/mattermost:latest

# Expected output:
{
  "manifests": [
    {
      "digest": "sha256:abc...",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    },
    {
      "digest": "sha256:def...",
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      }
    }
  ]
}
```

### Testing Images

```sh
# Test on ARM64
docker pull --platform linux/arm64 ghcr.io/this-is-tobi/mirror/mattermost:latest
docker run --rm --platform linux/arm64 ghcr.io/.../mattermost:latest version

# Should NOT show "exec format error"

# Test on amd64
docker pull --platform linux/amd64 ghcr.io/this-is-tobi/mirror/mattermost:latest
docker run --rm --platform linux/amd64 ghcr.io/.../mattermost:latest version
```

## Workflow Files Reference

### Minimal Workflow Structure

```yaml
name: App Name

on:
  workflow_call:
    inputs:
      REGISTRY: {required: true, type: string}
      NAMESPACE: {required: true, type: string}
      MULTI_ARCH: {required: true, type: boolean}
      USE_QEMU: {required: true, type: boolean}
  workflow_dispatch:
    inputs: # Same as above with defaults

permissions:
  contents: read
  packages: write

jobs:
  infos:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.release.outputs.tag }}
      matrix: ${{ steps.matrix.outputs.matrix }}
      image-status: ${{ steps.check.outputs.status }}
    steps:
      # Read matrix, check version, check GHCR

  build:
    runs-on: ubuntu-latest
    needs: infos
    if: needs.infos.outputs.image-status == '404'
    strategy:
      matrix:
        images: ${{ fromJSON(needs.infos.outputs.matrix) }}
    steps:
      # Setup, clone, compile, build, push digest

  merge:
    runs-on: ubuntu-latest
    needs: [infos, build]
    if: needs.infos.outputs.image-status == '404'
    steps:
      # Download digests, create manifest, tag
```

## Performance Metrics

**Typical build times:**
- Mattermost: ~15-20 minutes (2 arch builds)
- Mostlymatter: ~15-20 minutes (2 arch builds, same as Mattermost)
- Outline: ~10-15 minutes (2 arch builds)

**Cache benefits:**
- First build: 100% time
- Subsequent builds with cache: 30-50% time reduction

## Application-Specific Implementations

### Mattermost & Mostlymatter

**Source:**
- Mattermost: GitHub - `github.com/mattermost/mattermost`
- Mostlymatter: Framagit - `framagit.org/framasoft/framateam/mostlymatter`

**Build Approach:**
- Uses pre-built binaries from respective package registries
- Same Dockerfile structure (`server/build/Dockerfile`)
- Single binary Go application

**Key Differences:**

| Aspect            | Mattermost                | Mostlymatter             |
| ----------------- | ------------------------- | ------------------------ |
| Source            | GitHub                    | Framagit                 |
| API Endpoint      | GitHub Releases API       | Framagit Tags API        |
| Version Detection | Latest release            | Filtered & sorted tags   |
| Tag Format        | `vX.Y.Z`                  | `vX.Y.Z-limitless`       |
| Package URL       | `releases.mattermost.com` | `packages.framasoft.org` |
| Clone Method      | `actions/checkout@v4`     | Direct `git clone`       |

**Mostlymatter Version Detection:**
```bash
# Framagit returns tags unsorted and with mixed formats
# Filter: Only stable releases (vX.Y.Z-limitless)
# Sort: Semantic version comparison ([11,0,4] > [10,12,2])
curl framagit.org/api/v4/.../tags | jq -r '
  [.[] | select(.name | test("^v[0-9]+\\.[0-9]+\\.[0-9]+-limitless$"))] 
  | sort_by(. | ltrimstr("v") | rtrimstr("-limitless") | split(".") | map(tonumber)) 
  | reverse | .[0]'
```

### Outline

**Source:** GitHub - `github.com/outline/outline`

**Build Approach:**
- Node.js application with multi-arch base images
- Uses official Dockerfile from repository
- No special compilation required

## Further Reading

- [Docker Buildx Documentation](https://docs.docker.com/buildx/)
- [GitHub Actions Matrix Strategy](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)
- [Multi-platform Images](https://docs.docker.com/build/building/multi-platform/)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
