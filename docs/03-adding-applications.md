# Adding New Applications

This document provides a step-by-step guide for adding new applications to the multi-arch mirror repository.

## Overview

Adding a new application involves:
1. Understanding the application's architecture
2. Creating the matrix configuration
3. Writing the workflow file
4. Testing the implementation
5. Updating documentation

## Prerequisites

Before adding a new application, ensure you have:
- Access to the application's GitHub repository (or source)
- Understanding of the application's build process
- Knowledge of any special requirements (Go compilation, Node.js, etc.)
- Familiarity with Docker and Dockerfiles

## Step-by-Step Guide

### Step 1: Analyze the Application

#### Questions to Answer:

1. **What is the upstream repository?**
   - Example: `https://github.com/mattermost/mattermost`

2. **How are releases published?**
   - GitHub Releases with tags?
   - Container registry tags?
   - Version file in repository?

3. **What architectures are officially supported?**
   - Check existing Dockerfiles for `ARG TARGETARCH`
   - Look for hardcoded architecture references

4. **Does it require compilation?**
   - **Go application:** Requires binary cross-compilation
   - **Node.js application:** May work with multi-arch base images
   - **Python application:** Usually architecture-agnostic
   - **Static assets:** Always architecture-agnostic

5. **Are there multiple components?**
   - Single monolithic application (Mattermost, Outline)
   - Multiple microservices (requires separate matrix entries per component)

6. **What base images does it use?**
   - Official images (usually multi-arch)
   - Custom images (may need investigation)

#### Example Analysis - Mattermost:

```
Repository: github.com/mattermost/mattermost
Releases: GitHub Releases with tags (v9.11.0, v9.10.1, etc.)
Architecture: amd64 only (official)
Compilation: Go application - requires binary cross-compilation
Components: Single application
Base: Alpine Linux (multi-arch)
Special notes: Dockerfile expects pre-compiled binary
```

### Step 2: Create Directory Structure

Create the application directory under `apps/`:

```sh
mkdir -p apps/myapp
cd apps/myapp
```

### Step 3: Create Matrix Configuration

Create `matrix.json` to define build configurations.

#### Pattern 1: Single Component, Multi-Arch

**File:** `apps/myapp/matrix.json`

```json
[
  {
    "component": "myapp",
    "arch": "amd64",
    "platforms": "linux/amd64"
  },
  {
    "component": "myapp",
    "arch": "arm64",
    "platforms": "linux/arm64"
  }
]
```

#### Pattern 2: Multiple Components, Multi-Arch

**File:** `apps/myapp/matrix.json`

```json
[
  {
    "component": "myapp-api",
    "arch": "amd64",
    "platforms": "linux/amd64"
  },
  {
    "component": "myapp-api",
    "arch": "arm64",
    "platforms": "linux/arm64"
  },
  {
    "component": "myapp-worker",
    "arch": "amd64",
    "platforms": "linux/amd64"
  },
  {
    "component": "myapp-worker",
    "arch": "arm64",
    "platforms": "linux/arm64"
  }
]
```

**Matrix Fields:**
- `component` - Component name (used in image tags)
- `arch` - Target architecture (amd64, arm64)
- `platforms` - Docker platform string (linux/amd64, linux/arm64)

### Step 4: Create Workflow File

Create `.github/workflows/myapp.yml`.

#### Template Workflow

```yaml
name: MyApp

on:
  workflow_call:
    inputs:
      REGISTRY:
        required: true
        type: string
      NAMESPACE:
        required: true
        type: string
      MULTI_ARCH:
        required: true
        type: boolean
      USE_QEMU:
        required: true
        type: boolean
  workflow_dispatch:
    inputs:
      REGISTRY:
        description: Target registry to push images
        required: true
        type: string
        default: ghcr.io
      NAMESPACE:
        description: Target namespace to the given registry
        required: true
        type: string
        default: this-is-tobi/mirror
      MULTI_ARCH:
        description: Build for both amd64 and arm64
        required: true
        type: boolean
        default: true
      USE_QEMU:
        description: Use QEMU emulator for arm64
        required: true
        type: boolean
        default: true

permissions:
  contents: read
  packages: write

jobs:
  infos:
    name: Get MyApp and mirror infos
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.release.outputs.tag }}
      matrix: ${{ steps.matrix.outputs.matrix }}
      image-status: ${{ steps.check.outputs.image-status }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Read matrix
        id: matrix
        run: |
          if [ "${{ inputs.MULTI_ARCH }}" = "true" ]; then
            echo "matrix=$(cat apps/myapp/matrix.json | jq -c .)" >> $GITHUB_OUTPUT
          else
            echo "matrix=$(cat apps/myapp/matrix.json | jq -c '[.[] | select(.arch == "amd64")]')" >> $GITHUB_OUTPUT
          fi

      - name: Get latest release
        id: release
        run: |
          TAG=$(curl -s https://api.github.com/repos/owner/myapp/releases/latest | jq -r '.tag_name')
          echo "tag=${TAG}" >> $GITHUB_OUTPUT
          echo "Latest release: ${TAG}"

      - name: Check if image already exists
        id: check
        run: |
          TOKEN=$(echo ${{ secrets.GITHUB_TOKEN }} | base64)
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            -H "Authorization: Bearer ${TOKEN}" \
            https://ghcr.io/v2/${{ inputs.NAMESPACE }}/myapp/manifests/${{ steps.release.outputs.tag }})
          echo "image-status=${STATUS}" >> $GITHUB_OUTPUT
          echo "Image status: ${STATUS}"

  build:
    name: Build ${{ matrix.images.component }} for ${{ matrix.images.arch }}
    runs-on: ubuntu-latest
    needs: infos
    if: needs.infos.outputs.image-status == '404'
    strategy:
      matrix:
        images: ${{ fromJSON(needs.infos.outputs.matrix) }}
    steps:
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Set up QEMU
        if: inputs.USE_QEMU
        uses: docker/setup-qemu-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ inputs.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Checkout source repository
        run: |
          git clone --depth 1 --branch ${{ needs.infos.outputs.tag }} \
            https://github.com/owner/myapp.git .

      # Add compilation steps here if needed
      # See "Compilation Strategies" section below

      - name: Build and push by digest
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          provenance: false
          platforms: ${{ matrix.images.platforms }}
          outputs: type=image,name=${{ inputs.REGISTRY }}/${{ inputs.NAMESPACE }}/${{ matrix.images.component }},push-by-digest=true,name-canonical=true,push=true

      - name: Export digest
        run: |
          mkdir -p /tmp/digests/${{ matrix.images.component }}
          digest="${{ steps.build.outputs.digest }}"
          touch "/tmp/digests/${{ matrix.images.component }}/${digest#sha256:}"
          echo "Exported digest: $digest"

      - name: Upload digest
        uses: actions/upload-artifact@v4
        with:
          name: digests-${{ matrix.images.component }}-${{ matrix.images.arch }}
          path: /tmp/digests/${{ matrix.images.component }}/*
          if-no-files-found: error
          retention-days: 1

  merge:
    name: Create multi-arch manifest for myapp
    runs-on: ubuntu-latest
    needs: [infos, build]
    if: needs.infos.outputs.image-status == '404'
    steps:
      - name: Download digests
        uses: actions/download-artifact@v4
        with:
          pattern: digests-myapp-*
          path: /tmp/digests/myapp
          merge-multiple: true

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ inputs.REGISTRY }}/${{ inputs.NAMESPACE }}/myapp
          tags: |
            type=raw,value=${{ needs.infos.outputs.tag }}
            type=raw,value=latest

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ inputs.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Create manifest list and push
        working-directory: /tmp/digests/myapp
        run: |
          docker buildx imagetools create $(jq -cr '.tags | map("-t " + .) | join(" ")' <<< "$DOCKER_METADATA_OUTPUT_JSON") \
            $(printf '${{ inputs.REGISTRY }}/${{ inputs.NAMESPACE }}/myapp@sha256:%s ' *)

      - name: Inspect image
        run: |
          docker buildx imagetools inspect ${{ inputs.REGISTRY }}/${{ inputs.NAMESPACE }}/myapp:${{ needs.infos.outputs.tag }}
```

### Step 5: Handle Compilation (If Needed)

#### Strategy A: Go Application

If the application requires Go compilation, add this step before the build:

```yaml
- name: Cross-compile Go binary
  run: |
    ARCH=${{ matrix.images.arch }}
    
    # Create temporary cross-compile Dockerfile
    cat > Dockerfile.cross-compile << 'EOF'
    FROM --platform=$BUILDPLATFORM golang:1.24 AS builder
    WORKDIR /build
    COPY . .
    ARG TARGETOS
    ARG TARGETARCH
    ENV GOOS=${TARGETOS}
    ENV GOARCH=${TARGETARCH}
    RUN go build -o /app/myapp ./cmd/myapp
    EOF
    
    # Build and extract binary
    docker buildx build \
      --platform linux/${ARCH} \
      --file Dockerfile.cross-compile \
      --output type=local,dest=/tmp/binaries \
      .
    
    # Copy to expected location
    cp /tmp/binaries/myapp ./myapp
    chmod +x ./myapp
    
    echo "✅ Binary compiled for ${ARCH}"
    file ./myapp
```

#### Strategy B: Node.js Application

Node.js applications typically don't require special compilation:

```yaml
# Usually no special steps needed
# Just ensure base image is multi-arch (node:20-alpine)
```

#### Strategy C: Python Application

Python applications are usually architecture-agnostic:

```yaml
# Usually no special steps needed
# Some binary dependencies may require attention
```

### Step 6: Update Main Build Workflow

Add your application to `.github/workflows/build.yml`:

```yaml
jobs:
  # ... existing jobs ...

  myapp:
    uses: ./.github/workflows/myapp.yml
    with:
      REGISTRY: ghcr.io
      NAMESPACE: this-is-tobi/mirror
      MULTI_ARCH: true
      USE_QEMU: true
```

### Step 7: Update Documentation

#### Update README.md

Add your application to the images table:

```markdown
| MyApp       | `docker pull ghcr.io/this-is-tobi/mirror/myapp:latest`       |
```

#### Create App-Specific Documentation

Create `apps/myapp/README.md`:

```markdown
# MyApp Multi-Architecture Build

This directory contains the build configuration for creating multi-architecture Docker images for MyApp.

## Components

- **myapp**: Main application

## Build Matrix

The build matrix is defined in `matrix.json`:
- 2 architectures: amd64, arm64
- 1 component: myapp
- Total builds: 2 (1 component × 2 arches)

## Compilation

[Describe any special compilation requirements]

## Workflow

See `.github/workflows/myapp.yml` for the complete build workflow.

## Testing

```sh
# Pull multi-arch image
docker pull ghcr.io/this-is-tobi/mirror/myapp:latest

# Test on ARM64
docker run --rm --platform linux/arm64 ghcr.io/this-is-tobi/mirror/myapp:latest version

# Test on amd64
docker run --rm --platform linux/amd64 ghcr.io/this-is-tobi/mirror/myapp:latest version
```

## Upstream

- Repository: https://github.com/owner/myapp
- Official images: [Not available for ARM64]
```

### Step 8: Testing

#### Test Workflow Syntax

```sh
# Validate YAML syntax
yamllint .github/workflows/myapp.yml

# Or use actionlint
actionlint .github/workflows/myapp.yml
```

#### Test Matrix JSON

```sh
# Validate JSON
cat apps/myapp/matrix.json | jq .

# Test filtering for single arch
cat apps/myapp/matrix.json | jq '[.[] | select(.arch == "amd64")]'
```

#### Run Test Build

1. **Push to GitHub**
   ```sh
   git add .
   git commit -m "feat: add myapp multi-arch support"
   git push
   ```

2. **Trigger Manually**
   - Go to GitHub Actions
   - Select "MyApp" workflow
   - Click "Run workflow"
   - Set parameters:
     - `MULTI_ARCH`: false (test amd64 only first)
     - `USE_QEMU`: false

3. **Monitor Build**
   - Check workflow logs
   - Verify version detection
   - Check build output
   - Verify digest export

4. **Test Image**
   ```sh
   # Pull the image
   docker pull ghcr.io/this-is-tobi/mirror/myapp:latest
   
   # Inspect manifest
   docker manifest inspect ghcr.io/this-is-tobi/mirror/myapp:latest
   
   # Test run
   docker run --rm ghcr.io/this-is-tobi/mirror/myapp:latest version
   ```

5. **Test Multi-Arch**
   - Re-run workflow with `MULTI_ARCH`: true
   - Verify both architectures built
   - Check manifest has both platforms

## Common Patterns

### Pattern: Single Binary Application

**Examples:** Mattermost, most Go applications

**Key Points:**
- One Dockerfile
- Binary cross-compilation required
- Simple matrix (2 entries)

**Matrix:**
```json
[
  {"component": "app", "arch": "amd64", "platforms": "linux/amd64"},
  {"component": "app", "arch": "arm64", "platforms": "linux/arm64"}
]
```

### Pattern 2: Multi-Component System

**Examples:** Microservices architectures with separate components

**Key Points:**
- Multiple Dockerfiles (one per component)
- Some components need compilation, others don't
- Complex matrix (components × architectures)

**Matrix:**
```json
[
  {"component": "api", "arch": "amd64", "platforms": "linux/amd64"},
  {"component": "api", "arch": "arm64", "platforms": "linux/arm64"},
  {"component": "worker", "arch": "amd64", "platforms": "linux/amd64"},
  {"component": "worker", "arch": "arm64", "platforms": "linux/arm64"},
  {"component": "ui", "arch": "amd64", "platforms": "linux/amd64"},
  {"component": "ui", "arch": "arm64", "platforms": "linux/arm64"}
]
```

**Workflow additions:**
- Component classification logic
- Per-component Dockerfile paths
- Per-component merge jobs

### Pattern: Node.js Application

**Examples:** Outline

**Key Points:**
- No compilation needed (usually)
- Multi-arch base images (node:20-alpine)
- May need native module handling

**Special considerations:**
```yaml
# Some native modules may need compilation
- name: Rebuild native modules
  if: matrix.images.arch == 'arm64'
  run: npm rebuild --arch=arm64
```

## Troubleshooting

### Issue: Version Detection Fails

**Problem:** `curl` to GitHub API returns empty or error

**Solutions:**
1. Check repository name is correct
2. Verify release exists on GitHub
3. Try alternative: `git ls-remote --tags`
4. Add authentication if rate-limited

### Issue: Docker Build Fails on ARM64

**Problem:** "exec format error" or build hangs

**Solutions:**
1. Ensure QEMU is enabled (`USE_QEMU: true`)
2. Check Dockerfile uses `--platform=$BUILDPLATFORM`
3. Verify binary has correct architecture (`file binary`)
4. Use cross-compilation instead of emulation

### Issue: Manifest Creation Fails

**Problem:** "manifest unknown" or digest not found

**Solutions:**
1. Verify all matrix builds succeeded
2. Check digest artifacts uploaded correctly
3. Ensure GHCR authentication works
4. Check digest format (should be sha256:xxx)

### Issue: Image Exists Check Always Fails

**Problem:** Builds trigger even when image exists

**Solutions:**
1. Verify GHCR registry path is correct
2. Check authentication token
3. Try alternative check method:
   ```sh
   docker manifest inspect ghcr.io/.../app:tag > /dev/null 2>&1
   echo $?  # 0 = exists, non-zero = doesn't exist
   ```

## Best Practices

1. **Start Simple:** Begin with amd64-only, then add arm64
2. **Test Incrementally:** Test each step before moving forward
3. **Use Cache:** Implement build cache for faster iterations
4. **Document Special Cases:** Note any unusual requirements
5. **Monitor Build Times:** Optimize slow steps
6. **Version Pin:** Pin action versions for stability
7. **Clean Up:** Remove temporary files after use

## Checklist

Before submitting:

- [ ] `matrix.json` created and validated
- [ ] Workflow file created and syntax-checked
- [ ] Compilation steps added (if needed)
- [ ] Main build workflow updated
- [ ] README.md updated with new app
- [ ] App-specific README created
- [ ] Test build completed successfully (amd64)
- [ ] Multi-arch build tested (amd64 + arm64)
- [ ] Images verified on both architectures
- [ ] Manifest contains both platforms
- [ ] Documentation complete

## Examples

Refer to existing implementations:
- **Simple:** `apps/mattermost/` - Single Go binary, GitHub releases
- **Node.js:** `apps/outline/` - Node.js application
- **Advanced:** `apps/mostlymatter/` - Hybrid approach (Mattermost tarball + custom binary)

### Mostlymatter: Advanced Hybrid Build

Mostlymatter demonstrates a unique approach where we combine packages from different sources.

**Challenge:**
- Mostlymatter (Mattermost fork by Framasoft) distributes only pre-compiled binaries
- Mattermost distributes complete tarballs with supporting files (i18n, templates, config, fonts)
- Need all supporting files + the Mostlymatter binary

**Solution:** Hybrid approach
1. Download Mattermost's complete tarball (has all supporting files)
2. Replace only the binary with Mostlymatter's version
3. Keep all paths as `/mattermost` (matches original Dockerfile)

**Key Implementation Details:**

```yaml
# Version detection from Framagit API
FULL_TAG=$(curl -s https://framagit.org/api/v4/projects/framasoft%2Fframateam%2Fmostlymatter/repository/tags | jq -r '
  [.[] | select(.name | test("^v[0-9]+\\.[0-9]+\\.[0-9]+-limitless$"))] 
  | map(.name) 
  | sort_by(. | ltrimstr("v") | rtrimstr("-limitless") | split(".") | map(tonumber)) 
  | reverse 
  | .[0]')

# Extract clean version (v11.0.4-limitless → 11.0.4)
TAG=$(echo "$FULL_TAG" | sed 's/^v//' | sed 's/-limitless$//')

# Dockerfile modifications (minimal approach)
sed -i -E 's/(FROM [^@]+)@sha256:[a-f0-9]{64}(.*)/\1\2/' ./server/build/Dockerfile
sed -i '/^ARG MM_PACKAGE=/a ARG MOSTLYMATTER_BINARY' ./server/build/Dockerfile
sed -i 's#curl -L \$MM_PACKAGE | tar -xvz#curl -L \$MM_PACKAGE | tar -xvz \&\& rm /mattermost/bin/mattermost \&\& curl -L \$MOSTLYMATTER_BINARY -o /mattermost/bin/mattermost \&\& chmod +x /mattermost/bin/mattermost#' ./server/build/Dockerfile
```

**Matrix Configuration:**

```json
[
  {
    "arch": "amd64",
    "platforms": "linux/amd64",
    "build": {
      "context": "./server/build",
      "dockerfile": "./server/build/Dockerfile",
      "args": {
        "MM_PACKAGE": "https://releases.mattermost.com/${TAG}/mattermost-${TAG}-linux-amd64.tar.gz",
        "MOSTLYMATTER_BINARY": "https://packages.framasoft.org/projects/mostlymatter/mostlymatter-amd64-v${TAG}"
      }
    }
  },
  {
    "arch": "arm64",
    "platforms": "linux/arm64",
    "build": {
      "context": "./server/build",
      "dockerfile": "./server/build/Dockerfile",
      "args": {
        "MM_PACKAGE": "https://releases.mattermost.com/${TAG}/mattermost-${TAG}-linux-arm64.tar.gz",
        "MOSTLYMATTER_BINARY": "https://packages.framasoft.org/projects/mostlymatter/mostlymatter-arm64-v${TAG}"
      }
    }
  }
]
```

**Benefits:**
- ✅ Minimal Dockerfile modifications (only 3 sed commands)
- ✅ All supporting files guaranteed present and version-matched
- ✅ No directory renaming complexity
- ✅ Uses official Dockerfile structure from Mostlymatter repository
- ✅ Simple and maintainable

**Source:** Full implementation in `.github/workflows/mostlymatter.yml`

