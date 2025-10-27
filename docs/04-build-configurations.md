# Build Configurations

This document provides detailed reference for `matrix.json` files and build configuration patterns.

## Overview

Each application has a `matrix.json` file that defines:
- Which components to build
- Which architectures to support
- Platform mappings for Docker Buildx

The matrix is consumed by GitHub Actions to create parallel build jobs.

## Matrix.json Structure

### Basic Schema

```json
[
  {
    "component": "string",
    "arch": "string",
    "platforms": "string"
  }
]
```

### Field Definitions

#### `component` (required)

**Type:** String  
**Description:** Component name used in image tags and workflow logic

**Format:**
- Lowercase alphanumeric with hyphens
- Must match component name in source repository
- Used in final image name: `ghcr.io/namespace/COMPONENT:tag`

**Examples:**
```json
"component": "mattermost"
"component": "outline"
```

#### `arch` (required)

**Type:** String  
**Description:** Target CPU architecture

**Allowed values:**
- `amd64` - Intel/AMD 64-bit (x86_64)
- `arm64` - ARM 64-bit (aarch64)
- `arm/v7` - ARM 32-bit (for specialized cases)

**Examples:**
```json
"arch": "amd64"
"arch": "arm64"
```

**Usage in workflow:**
- Artifact naming: `digests-{component}-{arch}`
- Binary compilation: `GOARCH=${arch}`
- Platform selection: `linux/${arch}`

#### `platforms` (required)

**Type:** String  
**Description:** Docker platform identifier for Buildx

**Format:** `os/architecture[/variant]`

**Common values:**
```json
"platforms": "linux/amd64"
"platforms": "linux/arm64"
"platforms": "linux/arm/v7"
```

**Usage in workflow:**
```yaml
docker build --platform ${{ matrix.images.platforms }} ...
```

## Matrix Patterns

### Pattern 1: Single Component, Two Architectures

**Use case:** Simple applications (Mattermost, Outline)

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

**Result:**
- 2 parallel builds
- 1 multi-arch manifest
- Image: `ghcr.io/namespace/myapp:tag`

**Workflow matrix:**
```yaml
strategy:
  matrix:
    images: ${{ fromJSON(needs.infos.outputs.matrix) }}
    
# Creates 2 jobs:
# - build (myapp, amd64)
# - build (myapp, arm64)
```

### Pattern 2: Multiple Components, Two Architectures

**Use case:** Multi-component systems (microservices architectures)

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
  },
  {
    "component": "myapp-ui",
    "arch": "amd64",
    "platforms": "linux/amd64"
  },
  {
    "component": "myapp-ui",
    "arch": "arm64",
    "platforms": "linux/arm64"
  }
]
```

**Result:**
- 6 parallel builds (3 components × 2 arches)
- 3 multi-arch manifests (one per component)
- Images:
  - `ghcr.io/namespace/myapp-api:tag`
  - `ghcr.io/namespace/myapp-worker:tag`
  - `ghcr.io/namespace/myapp-ui:tag`

**Workflow matrix:**
```yaml
# Build job: 6 parallel jobs
strategy:
  matrix:
    images: ${{ fromJSON(needs.infos.outputs.matrix) }}

# Merge job: 3 parallel jobs (one per component)
strategy:
  matrix:
    component: ["myapp-api", "myapp-worker", "myapp-ui"]
```

### Pattern 3: Single Architecture Only

**Use case:** Testing or amd64-only builds

**File:** `apps/myapp/matrix.json`

```json
[
  {
    "component": "myapp",
    "arch": "amd64",
    "platforms": "linux/amd64"
  }
]
```

**Result:**
- 1 build job
- 1 single-arch manifest
- Faster build times for testing

**Workflow filtering:**
```yaml
- name: Read matrix
  run: |
    if [ "${{ inputs.MULTI_ARCH }}" = "true" ]; then
      echo "matrix=$(cat apps/myapp/matrix.json | jq -c .)"
    else
      echo "matrix=$(cat apps/myapp/matrix.json | jq -c '[.[] | select(.arch == "amd64")]')"
    fi
```

## Matrix Validation

### JSON Syntax Validation

```sh
# Validate JSON syntax
cat apps/myapp/matrix.json | jq .

# Should output prettified JSON if valid
# Should show error if invalid
```

### Schema Validation

**Required fields check:**
```sh
cat apps/myapp/matrix.json | jq '.[] | select(.component == null or .arch == null or .platforms == null)'

# Should return nothing if all entries valid
```

**Architecture values check:**
```sh
cat apps/myapp/matrix.json | jq '.[] | select(.arch != "amd64" and .arch != "arm64")'

# Should return nothing if all arches valid
```

**Platform format check:**
```sh
cat apps/myapp/matrix.json | jq '.[] | select(.platforms | test("^linux/(amd64|arm64|arm/v7)$") | not)'

# Should return nothing if all platforms valid
```

### Testing Matrix Filtering

**Get amd64 only:**
```sh
cat apps/myapp/matrix.json | jq '[.[] | select(.arch == "amd64")]'
```

**Get arm64 only:**
```sh
cat apps/myapp/matrix.json | jq '[.[] | select(.arch == "arm64")]'
```

**Get specific component:**
```sh
cat apps/myapp/matrix.json | jq '[.[] | select(.component == "myapp-api")]'
```

**Count total builds:**
```sh
cat apps/myapp/matrix.json | jq 'length'
```

**List all components:**
```sh
cat apps/myapp/matrix.json | jq '[.[].component] | unique'
```

**List all architectures:**
```sh
cat apps/myapp/matrix.json | jq '[.[].arch] | unique'
```

## Workflow Integration

### Reading Matrix in Workflow

```yaml
- name: Read matrix configuration
  id: matrix
  run: |
    MATRIX=$(cat apps/myapp/matrix.json | jq -c .)
    echo "matrix=${MATRIX}" >> $GITHUB_OUTPUT
```

### Conditional Matrix Loading

```yaml
- name: Read matrix with conditional architectures
  id: matrix
  run: |
    if [ "${{ inputs.MULTI_ARCH }}" = "true" ]; then
      # Load full matrix (all architectures)
      echo "matrix=$(cat apps/myapp/matrix.json | jq -c .)" >> $GITHUB_OUTPUT
    else
      # Load amd64 only
      echo "matrix=$(cat apps/myapp/matrix.json | jq -c '[.[] | select(.arch == "amd64")]')" >> $GITHUB_OUTPUT
    fi
```

### Using Matrix in Build Job

```yaml
build:
  strategy:
    matrix:
      images: ${{ fromJSON(needs.infos.outputs.matrix) }}
  steps:
    - name: Access matrix values
      run: |
        echo "Component: ${{ matrix.images.component }}"
        echo "Architecture: ${{ matrix.images.arch }}"
        echo "Platform: ${{ matrix.images.platforms }}"
```

### Using Matrix in Merge Job

**For single component:**
```yaml
merge:
  steps:
    - name: Download digests
      uses: actions/download-artifact@v4
      with:
        pattern: digests-myapp-*
        path: /tmp/digests/myapp
```

**For multiple components:**
```yaml
merge:
  strategy:
    matrix:
      component: ["myapp-api", "myapp-worker", "myapp-ui"]
  steps:
    - name: Download digests
      uses: actions/download-artifact@v4
      with:
        pattern: digests-${{ matrix.component }}-*
        path: /tmp/digests/${{ matrix.component }}
```

## Matrix Best Practices

### 1. Component Naming

**Good:**
```json
"component": "myapp-api"
"component": "postgres"
```

**Avoid:**
```json
"component": "MyApp"          // CamelCase
"component": "myapp_api"      // Underscores
"component": "my app"         // Spaces
"component": "app-v2"         // Version in name
```

### 2. Consistent Ordering

**Recommended order:**
1. Group by component
2. Order architectures: amd64, arm64, arm/v7

```json
[
  {"component": "api", "arch": "amd64", "platforms": "linux/amd64"},
  {"component": "api", "arch": "arm64", "platforms": "linux/arm64"},
  {"component": "worker", "arch": "amd64", "platforms": "linux/amd64"},
  {"component": "worker", "arch": "arm64", "platforms": "linux/arm64"}
]
```

### 3. Documentation

Add comments in README about matrix structure:

```markdown
## Build Matrix

- **Components:** 3 (api, worker, ui)
- **Architectures:** 2 (amd64, arm64)
- **Total builds:** 6 (3 × 2)

Matrix defined in `matrix.json`.
```

### 4. Testing Strategy

Test incrementally:
1. Start with 1 component, 1 architecture (1 build)
2. Add 1 more architecture (2 builds)
3. Add more components as needed

### 5. Performance Considerations

**Build parallelism:**
- GitHub Actions free tier: 20 concurrent jobs
- Each matrix entry = 1 job
- Keep total entries reasonable (<20 for free tier)

**Example:**
- 8 components × 2 arches = 16 jobs ✅ (under limit)
- 15 components × 2 arches = 30 jobs ⚠️ (may queue)

## Advanced Matrix Patterns

### Dynamic Component List

Instead of hardcoding component names in merge job:

```yaml
# In infos job - extract components from matrix
- name: Get component list
  id: components
  run: |
    COMPONENTS=$(cat apps/myapp/matrix.json | jq -c '[.[].component] | unique')
    echo "components=${COMPONENTS}" >> $GITHUB_OUTPUT

# In merge job - use dynamic list
merge:
  strategy:
    matrix:
      component: ${{ fromJSON(needs.infos.outputs.components) }}
```

### Conditional Builds Based on Component

```yaml
- name: Compile binary
  if: |
    matrix.images.component == 'api' ||
    matrix.images.component == 'worker'
  run: ./compile.sh
```

### Adding Metadata to Matrix

```json
[
  {
    "component": "api",
    "arch": "amd64",
    "platforms": "linux/amd64",
    "dockerfile": "Dockerfile.api",
    "requires_compilation": true
  }
]
```

Access in workflow:
```yaml
- name: Set Dockerfile path
  run: |
    echo "DOCKERFILE=${{ matrix.images.dockerfile || 'Dockerfile' }}" >> $GITHUB_ENV
```

## Matrix Examples

### Example 1: Mattermost

**File:** `apps/mattermost/matrix.json`

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

**Characteristics:**
- Single component
- 2 architectures
- Simple pattern
- 2 total builds

### Example 2: Outline

**File:** `apps/outline/matrix.json`

```json
[
  {
    "component": "outline",
    "arch": "amd64",
    "platforms": "linux/amd64"
  },
  {
    "component": "outline",
    "arch": "arm64",
    "platforms": "linux/arm64"
  }
]
```

**Characteristics:**
- Single component
- 2 architectures
- Node.js application
- No compilation needed
- 2 total builds

### Example 3: Generic Microservices

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
  },
  {
    "component": "myapp-scheduler",
    "arch": "amd64",
    "platforms": "linux/amd64"
  },
  {
    "component": "myapp-scheduler",
    "arch": "arm64",
    "platforms": "linux/arm64"
  },
  {
    "component": "myapp-ui",
    "arch": "amd64",
    "platforms": "linux/amd64"
  },
  {
    "component": "myapp-ui",
    "arch": "arm64",
    "platforms": "linux/arm64"
  }
]
```

**Characteristics:**
- 4 components (API, Worker, Scheduler, UI)
- 2 architectures per component
- 8 total builds
- 4 multi-arch manifests

## Troubleshooting Matrix Issues

### Issue: Matrix Empty in Workflow

**Symptom:** Build job doesn't run, matrix is empty

**Check:**
```sh
# Verify JSON is valid
cat apps/myapp/matrix.json | jq .

# Verify file path in workflow is correct
# Should be: cat apps/myapp/matrix.json
```

### Issue: Wrong Number of Build Jobs

**Symptom:** Expecting 4 jobs, only 2 run

**Check:**
```sh
# Count matrix entries
cat apps/myapp/matrix.json | jq 'length'

# Check for duplicates
cat apps/myapp/matrix.json | jq 'group_by(.component, .arch) | map(select(length > 1))'
```

### Issue: Build Job Uses Wrong Values

**Symptom:** amd64 job tries to build arm64

**Check workflow:**
```yaml
# Ensure correct platform is used
platforms: ${{ matrix.images.platforms }}  # ✅ Correct

# Not:
platforms: linux/${{ matrix.images.arch }}  # ❌ May not match platforms field
```

## Further Reading

- [GitHub Actions Matrix Strategy](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)
- [jq Manual](https://stedolan.github.io/jq/manual/) - JSON processing
