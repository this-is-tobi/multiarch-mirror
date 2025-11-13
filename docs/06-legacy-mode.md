# Legacy Single-Version Mode (Deprecated)

> **⚠️ DEPRECATION NOTICE**
> 
> Legacy single-version mode is **deprecated** and will be removed in a future release. 
> It is kept for backward compatibility during the migration period.
> 
> **Recommended:** Use the multi-version build system (current default). See the **Architecture & Workflows** documentation for details.

## Overview

Legacy mode builds **only the latest version** of each application. This was the original implementation before the multi-version system was introduced.

## Status

- **Current Default:** Multi-version mode (`USE_MULTI_VERSION: true`)
- **Legacy Workflows Available:** Yes, but not used by default
- **Removal Timeline:** To be determined (likely 6+ months after multi-version stabilization)

## How to Enable Legacy Mode

### Via Manual Trigger

1. Go to **Actions** → **Global build**
2. Click **Run workflow**
3. Set parameters:
   - `USE_MULTI_VERSION`: false
4. Click **Run workflow**

### Via Code (Not Recommended)

Edit `.github/workflows/build.yml`:

```yaml
env:
  USE_MULTI_VERSION: false  # Change from true to false
```

## Legacy Workflows

### Active Legacy Workflows

- `.github/workflows/mattermost.yml`
- `.github/workflows/mostlymatter.yml`
- `.github/workflows/outline.yml`

These workflows are only called when `USE_MULTI_VERSION: false`.

## How Legacy Mode Works

### Build Pipeline

```
┌────────────────────────────────────────────────────────────┐
│  1. INFOS JOB: Version Detection                           │
│  ────────────────────────────────────────────────────────  │
│  • Fetch latest version from upstream                      │
│  • Check if image exists in GHCR                           │
│  • Output: tag, image-status (200/404)                     │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼ (only if image-status == 404)
┌────────────────────────────────────────────────────────────┐
│  2. BUILD JOB: Build Latest Version Only                   │
│  ────────────────────────────────────────────────────────  │
│  Matrix: [amd64, arm64] (static from matrix.json)          │
│                                                            │
│  • Clone source at latest version tag                      │
│  • Build for each architecture                             │
│  • Push by digest                                          │
│  • Upload digest artifact                                  │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  3. MERGE JOB: Create Multi-Arch Manifest                  │
│  ────────────────────────────────────────────────────────  │
│  • Download digests for all architectures                  │
│  • Create multi-arch manifest                              │
│  • Tag with version + "latest"                             │
└────────────────────────────────────────────────────────────┘
```

### Key Characteristics

1. **Single Version Only**
   - Builds only the most recent release from upstream
   - Does not actively maintain version history (old tags persist but become stale)
   - Always tags as `latest` + version tag

2. **Simple Matrix**
   - Static matrix from `matrix.json`
   - No dynamic version expansion
   - Architecture-based parallelization only

3. **Simpler Logic**
   - No semantic versioning comparison
   - No multi-version coordination
   - Straightforward 3-job pipeline

## Limitations of Legacy Mode

### 1. No Active Version History Management
- Only builds the **latest version** from upstream
- Old version tags remain in registry but aren't maintained
- If upstream supports multiple major versions, only the newest major is mirrored
- No automated maintenance of historical versions

**Example:**
```bash
# Legacy mode behavior
docker pull ghcr.io/this-is-tobi/mirror/mattermost:10.3.0  # ✅ Available (built previously)
docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest  # → 10.3.0

# After new release (10.3.1)
docker pull ghcr.io/this-is-tobi/mirror/mattermost:10.3.0  # ✅ Still exists (not deleted)
docker pull ghcr.io/this-is-tobi/mirror/mattermost:10.3.1  # ✅ Newly built
docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest  # → 10.3.1

# However, 10.3.0 won't receive updates (security patches, rebuilds, etc.)
# And if 11.0.0 is released while 10.x is still maintained upstream:
# Legacy mode only mirrors 11.0.0, not both 10.x and 11.x branches
```

### 2. Limited Version Coverage
- Only mirrors the single "latest" release
- Doesn't track multiple maintained branches (e.g., 10.x LTS + 11.x stable)
- Historical versions not rebuilt if base images need updates
- Old tags exist but may become stale over time

### 3. Inefficient for Skipped Builds
- Checks if latest exists every time
- If exists, skips entirely (good)
- If missing, builds from scratch (no incremental)

### 4. No Semantic Versioning
- Uses "most recent release" as latest
- Doesn't compare version numbers
- Could tag older version as latest if release order is wrong

### 5. Rebuild Required for Rollback
- Want to rollback to previous version?
- Must manually rebuild old version
- No guarantee old version still builds (dependencies may have changed)

## Comparison: Legacy vs Multi-Version

| Feature                      | Legacy Mode                    | Multi-Version Mode             |
| ---------------------------- | ------------------------------ | ------------------------------ |
| **Actively Maintained**      | Latest version only            | Last N versions (default: 10)  |
| **Version Tags in Registry** | All built (accumulate forever) | All built (accumulate forever) |
| **Version Coverage**         | Single latest release          | Multiple concurrent branches   |
| **Rollback Support**         | Old tags available (stale)     | Recent versions maintained     |
| **Build Frequency**          | Every 6 hours                  | Every 6 hours                  |
| **Incremental Builds**       | Skip if exists, else rebuild   | Only missing versions built    |
| **Semantic Versioning**      | No - uses release order        | Yes - smart latest detection   |
| **CI Complexity**            | Simple (3-job pipeline)        | Moderate (4-job pipeline)      |
| **Storage Usage**            | Grows over time (all versions) | Grows over time (all versions) |
| **Build Time**               | Fast if exists, else full      | Optimized (incremental)        |

## When to Use Legacy Mode

### Valid Use Cases (Temporary)

1. **Testing Issues**
   - Multi-version mode has a bug
   - Need quick workaround
   - Use legacy until fix is deployed

2. **Reduced Build Activity**
   - Want to minimize build frequency for a single version
   - Testing environment with minimal requirements
   - Note: Both modes accumulate storage over time (neither deletes old versions)

3. **Debugging**
   - Comparing behavior between modes
   - Validating multi-version correctness

### Not Recommended

1. **Production Deployments**
   - Multi-version provides better stability
   - Rollback capability is valuable

2. **Long-Term Use**
   - Legacy mode will be removed
   - Migration will be required eventually

3. **New Projects**
   - Start with multi-version from beginning
   - No migration burden later

## Migration to Multi-Version Mode

### Step 1: Understand the Changes

**What Changes:**
- Registry will have multiple version tags
- `latest` tag behavior remains the same
- Old version tags persist (don't disappear)

**What Doesn't Change:**
- Build schedule (still every 6 hours)
- Image quality and architecture support
- Registry location and namespace

### Step 2: Test Multi-Version Mode

```bash
# Trigger manual build with multi-version
# Actions → Global build → Run workflow
#   USE_MULTI_VERSION: true
#   MAX_VERSIONS: 3  (start small)
```

**Verify:**
```bash
# Check multiple versions exist
docker pull ghcr.io/this-is-tobi/mirror/mattermost:10.3.1
docker pull ghcr.io/this-is-tobi/mirror/mattermost:10.3.0
docker pull ghcr.io/this-is-tobi/mirror/mattermost:10.2.1

# Verify latest points to highest version
docker manifest inspect ghcr.io/this-is-tobi/mirror/mattermost:latest
```

### Step 3: Update Deployments (Optional)

If you use `latest` tag, no changes needed. For version pinning:

**Before (Legacy):**
```yaml
image: ghcr.io/this-is-tobi/mirror/mattermost:latest
```

**After (Multi-Version):**
```yaml
# Option 1: Continue using latest (recommended for most)
image: ghcr.io/this-is-tobi/mirror/mattermost:latest

# Option 2: Pin to specific version (for stability)
image: ghcr.io/this-is-tobi/mirror/mattermost:10.3.1

# Option 3: Pin to major.minor (track patches)
image: ghcr.io/this-is-tobi/mirror/mattermost:10.3
```

### Step 4: Enable Multi-Version by Default

Once validated, multi-version mode is already the default:

```yaml
# .github/workflows/build.yml (already set)
env:
  USE_MULTI_VERSION: true  # Default
```

### Step 5: Monitor First Runs

- Check GitHub Actions logs
- Verify all versions build successfully
- Confirm `latest` tag points to correct version
- Test rollback by pulling older version

## Troubleshooting Legacy Mode

### Issue: Image Not Building

**Symptom:** Workflow completes but image doesn't update

**Cause:** Image already exists (status 200)

**Solution:**
```bash
# Check image status in workflow logs
# Look for: "Status code of mirror image: 200"

# If you need to force rebuild:
# 1. Delete image from GHCR (manual)
# 2. Re-run workflow
```

### Issue: Wrong Version Built

**Symptom:** Expected version X, got version Y

**Cause:** Upstream API returned different "latest"

**Solution:**
```bash
# Check what upstream reports as latest:
# For Mattermost:
curl -s https://api.github.com/repos/mattermost/mattermost/releases/latest \
  | jq -r '.tag_name'

# For Mostlymatter:
curl -s "https://framagit.org/api/v4/projects/12345/repository/tags" \
  | jq -r '[.[] | select(.name | test("^v[0-9]+\\.[0-9]+\\.[0-9]+-limitless$"))] | .[0].name'

# For Outline:
curl -s https://api.github.com/repos/outline/outline/releases/latest \
  | jq -r '.tag_name'
```

## Legacy Workflow Structure

### Example: mattermost.yml (Legacy)

```yaml
name: Mattermost

on:
  workflow_call:
    inputs:
      REGISTRY: ...
      NAMESPACE: ...

jobs:
  infos:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.infos.outputs.TAG }}
      build-matrix: ${{ steps.infos.outputs.BUILD_MATRIX }}
      image-status: ${{ steps.infos.outputs.IMAGE_STATUS }}
    steps:
      - name: Get latest version
        run: |
          # Fetch latest release
          TAG=$(curl ... | jq -r '.tag_name')
          
          # Check if exists
          IMAGE_STATUS=$(curl --head ... | grep HTTP)
          
          echo "TAG=$TAG" >> $GITHUB_OUTPUT
          echo "IMAGE_STATUS=$IMAGE_STATUS" >> $GITHUB_OUTPUT

  build:
    if: ${{ needs.infos.outputs.image-status == '404' }}
    needs: infos
    matrix:
      runners: [ubuntu-24.04, ubuntu-24.04-arm]
      images: ${{ fromJSON(needs.infos.outputs.build-matrix) }}
    steps:
      # Build for single version, multiple architectures
      
  merge:
    if: ${{ needs.infos.outputs.image-status == '404' }}
    needs: [infos, build]
    steps:
      # Create multi-arch manifest
      # Tag with version + latest
```

**Key Differences from Multi-Version:**
- Simple outputs: just tag, build-matrix, image-status
- Single version check
- Static matrix from JSON file
- Condition: `image-status == '404'` (not `has-builds == 'true'`)

## Timeline & Removal Plan

### Phase 1: Multi-Version Introduction (Current)
- [x] Multi-version workflows created
- [x] Set as default mode
- [x] Legacy workflows kept for compatibility
- [x] Documentation updated

### Phase 2: Stabilization
- [ ] Monitor multi-version mode in production
- [ ] Fix any issues that arise
- [ ] Gather user feedback
- [ ] Validate storage and performance

### Phase 3: Deprecation Notice
- [ ] Add deprecation warnings to legacy workflows
- [ ] Update documentation with removal timeline
- [ ] Notify users via README

### Phase 4: Removal
- [ ] Remove legacy workflow files
- [ ] Remove `USE_MULTI_VERSION` toggle from build.yml
- [ ] Update documentation (remove legacy docs)
- [ ] Clean up orchestrator workflow

## FAQ

### Q: Can I use both modes simultaneously?

**A:** No. The orchestrator (`build.yml`) uses `USE_MULTI_VERSION` toggle to select one mode:
- `true` → Calls multi-version workflows
- `false` → Calls legacy workflows

### Q: What happens to existing images when switching?

**A:** Nothing! Images in GHCR persist. Switching modes only affects future builds.

### Q: Can I switch back to legacy mode after testing multi-version?

**A:** Yes, simply set `USE_MULTI_VERSION: false`. However, multi-version images remain in registry.

### Q: Will legacy mode receive bug fixes?

**A:** Critical security fixes only. New features and improvements will be multi-version only.

### Q: How long will legacy mode be supported?

**A:** At least 6-12 months after multi-version stabilization. A deprecation notice will be issued with specific timeline.

### Q: What if I have custom workflows based on legacy mode?

**A:** You'll need to migrate to multi-version approach or maintain your custom workflows independently.

## Getting Help

If you need assistance with legacy mode:

1. **Check Documentation:** This guide and the **Troubleshooting** section
2. **Review Workflow Logs:** GitHub Actions → Workflow runs
3. **Compare with Multi-Version:** See the **Architecture & Workflows** documentation
4. **Open Issue:** If you find bugs or need help migrating
