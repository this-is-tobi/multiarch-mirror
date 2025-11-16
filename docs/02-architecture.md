# Architecture & Workflows

This document explains how the multi-arch mirror build system works, including GitHub Actions workflows, Docker Buildx integration, and the automated multi-version build pipeline.

> **Note:** This document describes the current **multi-version build system** (default). For legacy single-version mode (deprecated), see Legacy Mode dedicated documentation.

## Overview

The build system consists of:
- **Orchestrator workflow** - Scheduler that triggers app-specific workflows every 6 hours
- **Multi-version workflows** - Build last N versions of each application (default: 10)
- **Flattened matrix strategy** - Parallel builds for versions × architectures
- **Docker Buildx** - Multi-platform image builder
- **Push-by-digest strategy** - Efficient multi-arch manifest creation
- **Semantic versioning** - Smart "latest" tag determination

## Build Pipeline Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  build.yml (Orchestrator - Every 6 hours at 00:00 / 06:00 / 12:00 / 18:00 UTC)      │
│  ├─ Triggers: schedule (cron) / manual (workflow_dispatch)                          │
│  ├─ Mode: Multi-version (default) or Legacy (deprecated)                            │
│  └─ Calls: *-multi-version.yml workflows (or legacy .yml if USE_MULTI_VERSION=false)│
└──────────────────────────────────────────┬──────────────────────────────────────────┘
                                           │
                   ┌───────────────────────┼────────────────────────┐
                   │                       │                        │
                   ▼                       ▼                        ▼
         ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
         │  Mattermost        │  │  Mostlymatter      │  │  Outline           │
         │  Multi-Version     │  │  Multi-Version     │  │  Multi-Version     │
         │  Workflow          │  │  Workflow          │  │  Workflow          │
         └─────────┬──────────┘  └─────────┬──────────┘  └──────────┬─────────┘
                   │                       │                        │
                   └───────────────────────┴────────────────────────┘
                                           │
                                           ▼
                      ┌──────────────────────────────────────────┐
                      │    Multi-Version 4-Job Flow              │
                      │  infos → build → prepare-merge → merge   │
                      │                                          │
                      │  (builds last N versions)                │
                      └──────────────────────────────────────────┘
```

## Multi-Version Build System

The current default build mode actively builds the last N versions (default: 10) of each application from upstream. This provides version history for rollbacks and reproducible deployments.

**Important:** All built versions remain in the registry forever (never deleted). The system fetches the last N versions from upstream and builds any missing ones, but does not prune or delete older versions. Storage accumulates over time.

### Key Concepts

1. **Active Version Window:** Fetch last N versions from upstream, build missing ones
2. **No Pruning:** All built image tags remain in registry indefinitely (storage accumulates)
3. **Flattened Matrix:** Pre-compute all `version × architecture` combinations
4. **Incremental Builds:** Only build versions that don't exist in registry
5. **Semantic Latest:** Use version comparison (not release date) to determine `latest`
6. **Per-Version Artifacts:** Each version gets its own digest artifacts

### Multi-Version Pipeline

```
infos job:
  ├─ Fetch last 10 releases from upstream
  ├─ Check GHCR for existing images (per version)
  ├─ Generate flattened matrix:
  │    [{v: "10.3.1", arch: "amd64", runner: "ubuntu-24.04"},
  │     {v: "10.3.1", arch: "arm64", runner: "ubuntu-24.04-arm"},
  │     {v: "10.3.0", arch: "amd64", runner: "ubuntu-24.04"},
  │     ...]
  ├─ Determine semantic latest: 10.3.1 (not 10.2.1)
  └─ Output: versions-matrix, has-builds

build job (if has-builds):
  ├─ Matrix expands to N × M parallel jobs
  │    (N versions × M architectures)
  ├─ Each job builds one version/arch combo
  └─ Upload digest: digests-app-10.3.1-amd64

prepare-merge job:
  ├─ Group versions from build matrix
  └─ Create merge matrix per version

merge job:
  └─ For each version:
      ├─ Download digests (all archs)
      ├─ Create multi-arch manifest
      ├─ Tag with version: 10.3.1
      └─ Tag with latest: (if is_latest)

attest job:
  └─ For each version:
      ├─ Generate SBOM with Trivy (SPDX format)
      ├─ Attest SBOM to version tag (and latest if applicable)
      ├─ Sign images with Cosign (keyless)
      ├─ Generate SLSA provenance metadata
      └─ Attest provenance to version tag (and latest if applicable)
```

### Detailed Job Flow (Legacy Mode)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Job 1: infos                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  Purpose: Version detection & build decision                              │  │
│  │  ─────────────────────────────────────────────────────────────────────────│  │
│  │  1. Checkout repository                                                   │  │
│  │  2. Read build matrix from matrix.json                                    │  │
│  │  3. Get latest version:                                                   │  │
│  │     • Mattermost:   GitHub Releases API                                   │  │
│  │     • Mostlymatter: Framagit Tags API (filtered & sorted)                 │  │
│  │     • Outline:      GitHub Releases API                                   │  │
│  │  4. Check if image exists in GHCR                                         │  │
│  │  5. Output: tag, build-matrix, image-status (200/404)                     │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │   if image-status == '404'        │
                    │   (image doesn't exist yet)       │
                    └─────────────────┬─────────────────┘
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Job 2: build (Matrix Strategy - Parallel Execution)                             │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  Purpose: Build multi-arch images in parallel                              │  │
│  │  ───────────────────────────────────────────────────────────────────────── │  │
│  │  Matrix: For each (arch × component) combination                           │  │
│  │  Example: [amd64, arm64] × [app] = 2 parallel jobs                         │  │
│  │                                                                            │  │
│  │  Steps per matrix job:                                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ 1. Clone source repository at detected version tag                   │  │  │
│  │  │                                                                      │  │  │
│  │  │ 2. Modify Dockerfile (if needed):                                    │  │  │
│  │  │    • Remove SHA pinning for ARM compatibility                        │  │  │
│  │  │    • Mostlymatter: Replace tarball extraction with binary download   │  │  │
│  │  │      - Old: curl -L $MM_PACKAGE | tar -xvz                           │  │  │
│  │  │      - New: mkdir /mattermost/bin && curl -L $MM_PACKAGE -o ... && \ │  │  │
│  │  │             chmod +x /mattermost/bin/mattermost                      │  │  │
│  │  │                                                                      │  │  │
│  │  │ 3. Setup Docker Buildx + QEMU (optional)                             │  │  │
│  │  │                                                                      │  │  │
│  │  │ 4. Login to GHCR                                                     │  │  │
│  │  │                                                                      │  │  │
│  │  │ 5. Build & Push Docker image:                                        │  │  │
│  │  │    • Platform: linux/amd64 OR linux/arm64                            │  │  │
│  │  │    • Push by digest (no tags yet)                                    │  │  │
│  │  │    • Output: sha256:abc123... (unique digest)                        │  │  │
│  │  │                                                                      │  │  │
│  │  │ 6. Export digest to file:                                            │  │  │
│  │  │    /tmp/digests/app/abc123...                                        │  │  │
│  │  │                                                                      │  │  │
│  │  │ 7. Upload digest as artifact:                                        │  │  │
│  │  │    digests-app-amd64, digests-app-arm64                              │  │  │
│  │  └──────────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                            │  │
│  │  Result: 2 separate images pushed to GHCR (one per architecture)           │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Job 3: merge                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  Purpose: Create multi-arch manifest from digests                         │  │
│  │  ─────────────────────────────────────────────────────────────────────────│  │
│  │  1. Download all digest artifacts:                                        │  │
│  │     • digests-app-amd64 → sha256:abc123...                                │  │
│  │     • digests-app-arm64 → sha256:def456...                                │  │
│  │                                                                           │  │
│  │  2. Generate tags using docker/metadata-action:                           │  │
│  │     • {version} (e.g., 11.0.4, 9.11.4)                                    │  │
│  │     • latest                                                              │  │
│  │                                                                           │  │
│  │  3. Create multi-arch manifest:                                           │  │
│  │     docker buildx imagetools create \                                     │  │
│  │       -t ghcr.io/namespace/app:{version} \                                │  │
│  │       -t ghcr.io/namespace/app:latest \                                   │  │
│  │       ghcr.io/namespace/app@sha256:abc123... \  (amd64)                   │  │
│  │       ghcr.io/namespace/app@sha256:def456...    (arm64)                   │  │
│  │                                                                           │  │
│  │  4. Inspect manifest to verify multi-arch support                         │  │
│  │                                                                           │  │
│  │  Result: Tagged multi-arch image available for both platforms             │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  Job 4: attest                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  Purpose: Generate and attach security attestations                       │  │
│  │  ─────────────────────────────────────────────────────────────────────────│  │
│  │  1. Install tools:                                                        │  │
│  │     • Cosign v3 (sigstore/cosign-installer@v3)                            │  │
│  │     • Trivy (aquasecurity/setup-trivy@v0.2.0)                             │  │
│  │                                                                           │  │
│  │  2. Generate SBOM (Software Bill of Materials):                           │  │
│  │     trivy image --format spdx-json --output sbom.spdx.json <image>        │  │
│  │                                                                           │  │
│  │  3. Attest SBOM to image tags:                                            │  │
│  │     cosign attest --type spdxjson --predicate sbom.spdx.json <image:tag>  │  │
│  │     (applied to version tag and latest if applicable)                     │  │
│  │                                                                           │  │
│  │  4. Sign images with keyless signing:                                     │  │
│  │     cosign sign --yes <image:tag>                                         │  │
│  │     (uses GitHub OIDC for identity)                                       │  │
│  │                                                                           │  │
│  │  5. Generate SLSA provenance:                                             │  │
│  │     • Build metadata (workflow, repository, commit)                       │  │
│  │     • Build parameters (registry, namespace, version)                     │  │
│  │     • Materials (upstream source reference)                               │  │
│  │                                                                           │  │
│  │  6. Attest provenance to image tags:                                      │  │
│  │     cosign attest --type slsaprovenance --predicate provenance.json       │  │
│  │                                                                           │  │
│  │  Result: Images with SBOM, signatures, and provenance in GHCR             │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘

Final Output:
  ghcr.io/namespace/app:{version} → Multi-arch manifest
  ├─ linux/amd64 → sha256:abc123...
  └─ linux/arm64 → sha256:def456...
  
  ghcr.io/namespace/app:latest → Same manifest
  
  Attestations (OCI artifacts in GHCR):
  ├─ SBOM (SPDX format)
  ├─ Cryptographic signature
  └─ SLSA provenance
```

> **Note:** The above describes **legacy single-version mode** (deprecated). This mode only builds the latest version and is kept for backward compatibility. For details on legacy mode, see the Legacy Mode dedicated documentation.

### Application-Specific Differences

#### Mattermost
```
Version Source: GitHub Releases API
Package Format: Pre-built tarball (.tar.gz)
Dockerfile:     Official from mattermost/mattermost
Modifications:  Remove SHA pinning only
Build Args:     MM_PACKAGE=https://releases.mattermost.com/.../mattermost-{version}-linux-{arch}.tar.gz
```

#### Mostlymatter
```
Version Source: Framagit Tags API (filtered with regex ^v[0-9]+\.[0-9]+\.[0-9]+-limitless$)
                Sorted semantically: v11.0.4 > v10.12.2
Package Format: Hybrid - Mattermost tarball + Mostlymatter binary
Dockerfile:     Official from mostlymatter repository
Modifications:  1. Remove SHA pinning
                2. Add MOSTLYMATTER_BINARY build arg
                3. Modify download to replace binary:
                   - Extract Mattermost tarball (provides all supporting files)
                   - Remove Mattermost binary
                   - Download Mostlymatter binary
                   - Set executable permissions
Build Args:     MM_PACKAGE=https://releases.mattermost.com/.../mattermost-{version}-linux-{arch}.tar.gz
                MOSTLYMATTER_BINARY=https://packages.framasoft.org/.../mostlymatter-{arch}-v{version}
Key Challenge:  Framasoft packages are binaries only, missing supporting files (i18n, templates, config)
Solution:       Use Mattermost's complete tarball, replace only the binary with Mostlymatter's version
```

#### Outline
```
Version Source: GitHub Releases API
Package Format: Node.js application
Dockerfile:     Official from outline/outline
Modifications:  Remove SHA pinning only
Build Args:     None (uses standard Node.js base images)
```

## Workflow Components

### 1. Orchestrator Workflow (`build.yml`)

**Purpose:** Schedule and coordinate all application builds

**Location:** `.github/workflows/build.yml`

**Triggers:**
- **Schedule:** Every 6 hours at 00:00 / 06:00 / 12:00 / 18:00 UTC (`cron: '0 */6 * * *'`)
- **Manual:** Via GitHub Actions UI (`workflow_dispatch`)

**Configuration:**
The orchestrator supports UI-based configuration for manual runs:

```yaml
workflow_dispatch:
  inputs:
    REGISTRY: string (default: ghcr.io)
    NAMESPACE: string (default: this-is-tobi/mirror)
    MULTI_ARCH: boolean (default: true)
    USE_QEMU: boolean (default: false)
    MAX_VERSIONS: number (default: 10)
    USE_MULTI_VERSION: boolean (default: true)
```

**Default Settings (Scheduled Runs):**
```yaml
env:
  USE_MULTI_VERSION: true   # Use multi-version workflows
  MAX_VERSIONS: 10          # Fetch and build last 10 versions from upstream
  MULTI_ARCH: true          # Build both amd64 and arm64
  USE_QEMU: false           # Native builds preferred
```

**Workflow Selection:**
The orchestrator conditionally calls either multi-version or legacy workflows:

```yaml
jobs:
  mattermost-multi-version:
    if: USE_MULTI_VERSION == 'true'
    uses: ./.github/workflows/mattermost-multi-version.yml
  
  mattermost:
    if: USE_MULTI_VERSION == 'false'
    uses: ./.github/workflows/mattermost.yml
  
  # Same pattern for mostlymatter and outline
```

**Parameters passed to workflows:**
- `REGISTRY`: Target registry (default: `ghcr.io`)
- `NAMESPACE`: Target namespace (default: `this-is-tobi/mirror`)
- `MULTI_ARCH`: Build for both architectures (default: `true`)
- `USE_QEMU`: Enable QEMU emulation (default: `false`)
- `MAX_VERSIONS`: Number of versions to fetch and build from upstream (default: `10`, multi-version only)

### 2. Multi-Version Application Workflows

Each application has a multi-version workflow (default) that fetches the last N versions from upstream and builds any missing ones.

**Workflows:**
- `.github/workflows/mattermost-multi-version.yml`
- `.github/workflows/mostlymatter-multi-version.yml`
- `.github/workflows/outline-multi-version.yml`

**Common Pattern:** 4-job pipeline (infos → build → prepare-merge → merge)

**Note:** These workflows do NOT delete old versions. All built versions accumulate in the registry over time.

#### Job 1: `infos` - Version Discovery & Matrix Generation

**Purpose:** Fetch multiple versions, check registry, generate flattened build matrix

**Steps:**

1. **Fetch last N versions from upstream**
   ```bash
   # Example for Mattermost
   VERSIONS=$(curl "https://api.github.com/repos/mattermost/mattermost/releases?per_page=${MAX_VERSIONS}" \
     | jq -r '[.[] | select(.prerelease == false) | .tag_name | ltrimstr("v")] | unique')
   ```

2. **Determine semantic "latest"**
   ```bash
   LATEST_VERSION=$(echo "$VERSIONS" | jq -r 'sort_by(split(".") | map(tonumber)) | reverse | .[0]')
   # Example: [10.3.1, 10.3.0, 10.2.1] → 10.3.1
   ```

3. **Check GHCR for each version**
   ```bash
   for VERSION in $(echo "$VERSIONS" | jq -r '.[]'); do
     IMAGE_STATUS=$(curl --head "https://ghcr.io/v2/.../manifests/${VERSION}")
     # If 404, add to build matrix
   done
   ```

4. **Generate flattened matrix (versions × architectures)**
   ```bash
   # Pre-compute all combinations with correct runner assignment
   BUILD_MATRIX='[
     {"version": "10.3.1", "arch": "amd64", "runner": "ubuntu-24.04", ...},
     {"version": "10.3.1", "arch": "arm64", "runner": "ubuntu-24.04-arm", ...},
     {"version": "10.3.0", "arch": "amd64", "runner": "ubuntu-24.04", ...},
     ...
   ]'
   ```

**Why Flattened Matrix?**
GitHub Actions doesn't support nested matrices. We can't do:
```yaml
matrix:
  version: [10.3.1, 10.3.0]
  arch: [amd64, arm64]
  runner: [ubuntu-24.04, ubuntu-24.04-arm]  # ❌ Wrong mapping!
```

Instead, we pre-compute all combinations with correct runner-per-arch mapping.

**Outputs:**
- `versions-matrix` - Flattened array of version/arch/runner combinations
- `has-builds` - Boolean flag indicating if any builds are needed

**Build Decision:**
- **Build if:** `has-builds == 'true'` (at least one version missing)
- **Skip if:** `has-builds == 'false'` (all versions exist)

#### Job 2: `build` - Parallel Multi-Arch Builds

**Purpose:** Build all missing version/architecture combinations in parallel

**Depends on:** `infos` job
**Condition:** `needs.infos.outputs.has-builds == 'true'`

**Matrix Strategy:**
```yaml
strategy:
  fail-fast: false
  matrix:
    config: ${{ fromJSON(needs.infos.outputs.versions-matrix) }}
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

#### Job 3: `prepare-merge` - Group Versions for Merging

**Purpose:** Create a merge matrix grouped by version

**Depends on:** `infos`, `build` jobs
**Condition:** `needs.infos.outputs.has-builds == 'true'`

**Key Steps:**

```bash
# Extract unique versions from build matrix and their is_latest flags
BUILD_MATRIX='${{ needs.infos.outputs.versions-matrix }}'
MERGE_MATRIX=$(echo "$BUILD_MATRIX" | jq -c '
  [group_by(.version) | .[] | {version: .[0].version, is_latest: .[0].is_latest}]
')
```

**Outputs:**
- `merge-matrix` - Array of unique versions with latest flags
  ```json
  [
    {"version": "10.3.1", "is_latest": true},
    {"version": "10.3.0", "is_latest": false},
    {"version": "10.2.1", "is_latest": false}
  ]
  ```

#### Job 4: `merge` - Create Multi-Arch Manifests

**Purpose:** Create multi-architecture manifests for each version

**Depends on:** `infos`, `build`, `prepare-merge` jobs
**Condition:** `needs.infos.outputs.has-builds == 'true'`

**Matrix Strategy:** Runs once per version

```yaml
strategy:
  matrix:
    config: ${{ fromJSON(needs.prepare-merge.outputs.merge-matrix) }}
```

**Key Steps:**

1. **Download all digests for this version**
   ```yaml
   - name: Download digests
     uses: actions/download-artifact@v4
     with:
       pattern: digests-mattermost-${{ matrix.config.version }}-*
       path: /tmp/digests/mattermost
       merge-multiple: true
   ```

2. **Generate tags (version + optionally latest)**
   ```yaml
   - name: Docker meta
     uses: docker/metadata-action@v5
     with:
       images: ghcr.io/.../mattermost
       tags: |
         type=raw,value=${{ matrix.config.version }}
         type=raw,value=latest,enable=${{ matrix.config.is_latest }}
   ```

3. **Create manifest and push**
   ```yaml
   - name: Create manifest list and push
     run: |
       docker buildx imagetools create $(jq -cr '.tags | map("-t " + .) | join(" ")' <<< "$DOCKER_METADATA_OUTPUT_JSON") \
         $(printf 'ghcr.io/.../mattermost@sha256:%s ' *)
   ```

**Result:**
- Version tag: `ghcr.io/.../mattermost:10.3.1` → multi-arch manifest
- Latest tag: `ghcr.io/.../mattermost:latest` → only for highest version

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
