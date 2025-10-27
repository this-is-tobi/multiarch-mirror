# Troubleshooting

This document covers common issues and solutions when building multi-architecture images.

## Build Issues

### Version Detection Fails

**Symptom:**
```
Error: tag is empty
Latest release: 
```

**Causes:**
1. GitHub API rate limiting
2. Repository doesn't have releases
3. Release tag format unexpected
4. Network connectivity issues

**Solutions:**

1. **Check repository has releases:**
   ```sh
   curl -s https://api.github.com/repos/owner/repo/releases/latest | jq -r '.tag_name'
   ```

2. **Add authentication to avoid rate limits:**
   ```yaml
   - name: Get latest release
     run: |
       TAG=$(curl -s \
         -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
         https://api.github.com/repos/owner/repo/releases/latest | jq -r '.tag_name')
   ```

3. **Alternative: Use git tags:**
   ```yaml
   - name: Get latest tag
     run: |
       TAG=$(git ls-remote --tags --refs --sort='version:refname' \
         https://github.com/owner/repo.git | tail -n1 | cut -d'/' -f3)
   ```

4. **Check tag format:**
   ```sh
   # Releases page shows: v1.2.3
   # But API returns: 1.2.3 (without 'v')
   # Adjust parsing accordingly
   TAG=$(curl ... | jq -r '.tag_name' | sed 's/^v//')
   ```

### Image Already Exists Check Fails

**Symptom:**
- Builds run even when image exists
- Or builds never run even for new versions

**Solutions:**

1. **Verify registry path:**
   ```sh
   # Check exact path
   docker manifest inspect ghcr.io/this-is-tobi/mirror/myapp:v1.0.0
   ```

2. **Check authentication:**
   ```yaml
   - name: Check image exists
     run: |
       TOKEN=$(echo ${{ secrets.GITHUB_TOKEN }} | base64)
       STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
         -H "Authorization: Bearer ${TOKEN}" \
         https://ghcr.io/v2/${{ inputs.NAMESPACE }}/myapp/manifests/${{ steps.release.outputs.tag }})
   ```

3. **Alternative method using docker:**
   ```yaml
   - name: Check image exists
     run: |
       if docker manifest inspect ghcr.io/.../myapp:${TAG} > /dev/null 2>&1; then
         echo "image-status=200" >> $GITHUB_OUTPUT
       else
         echo "image-status=404" >> $GITHUB_OUTPUT
       fi
   ```

### Docker Build Fails on ARM64

**Symptom:**
```
exec format error
```
or
```
Error: failed to solve: process "/bin/sh -c go build" did not complete successfully
```

**Causes:**
1. QEMU not set up correctly
2. Binary has wrong architecture
3. Using emulation instead of cross-compilation

**Solutions:**

1. **Enable QEMU:**
   ```yaml
   - name: Set up QEMU
     uses: docker/setup-qemu-action@v3
   ```

2. **Use cross-compilation:**
   ```dockerfile
   FROM --platform=$BUILDPLATFORM golang:1.24 AS builder
   ARG TARGETOS
   ARG TARGETARCH
   ENV GOOS=${TARGETOS}
   ENV GOARCH=${TARGETARCH}
   RUN go build -o /app/binary
   ```

3. **Verify binary architecture:**
   ```yaml
   - name: Check binary
     run: |
       file ./binary
       # Should show: ELF 64-bit LSB executable, ARM aarch64
       # Not: ELF 64-bit LSB executable, x86-64
   ```

4. **Use buildx properly:**
   ```yaml
   - uses: docker/build-push-action@v5
     with:
       platforms: linux/arm64
       # NOT: platform: linux/arm64 (singular, wrong!)
   ```

### Binary Compilation Fails

**Symptom:**
```
Error: GOARCH not set correctly
Error: undefined reference for ARM64
```

**Solutions:**

1. **Verify environment variables:**
   ```yaml
   - name: Debug
     run: |
       echo "GOOS=${GOOS}"
       echo "GOARCH=${GOARCH}"
       echo "ARCH=${{ matrix.images.arch }}"
   ```

2. **Check Dockerfile sets variables:**
   ```dockerfile
   ARG TARGETARCH
   ENV GOARCH=${TARGETARCH}
   
   # Or explicitly:
   RUN GOARCH=arm64 go build ...
   ```

3. **Use correct GOARCH values:**
   - amd64 → `GOARCH=amd64`
   - arm64 → `GOARCH=arm64`
   - arm/v7 → `GOARCH=arm GOARM=7`

4. **Disable CGO if needed:**
   ```dockerfile
   ENV CGO_ENABLED=0
   RUN go build -o /app/binary
   ```

### Build Cache Issues

**Symptom:**
- Cache not being used
- Cache growing indefinitely
- "no space left on device"

**Solutions:**

1. **Implement cache rotation:**
   ```yaml
   - uses: docker/build-push-action@v5
     with:
       cache-from: type=local,src=/tmp/.buildx-cache
       cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max

   - name: Move cache
     run: |
       rm -rf /tmp/.buildx-cache
       mv /tmp/.buildx-cache-new /tmp/.buildx-cache
   ```

2. **Clear cache manually:**
   ```sh
   # In GitHub Actions
   # Go to: Settings → Actions → Caches
   # Delete old caches
   ```

3. **Use registry cache (more robust):**
   ```yaml
   cache-from: type=registry,ref=ghcr.io/.../myapp:buildcache
   cache-to: type=registry,ref=ghcr.io/.../myapp:buildcache,mode=max
   ```

## Manifest Issues

### Manifest Creation Fails

**Symptom:**
```
Error: manifest unknown
Error: digest not found
```

**Solutions:**

1. **Verify all builds succeeded:**
   ```yaml
   # Check build job completed for all matrix entries
   needs: [infos, build]
   ```

2. **Check digest artifacts:**
   ```yaml
   - name: Download digests
     uses: actions/download-artifact@v4
     with:
       pattern: digests-myapp-*
       merge-multiple: true
   
   - name: Debug
     run: |
       ls -la /tmp/digests/myapp/
       cat /tmp/digests/myapp/*
   ```

3. **Verify digest format:**
   ```sh
   # Digests should be sha256:xxxxx (without sha256: in filename)
   # Filename: abc123def456
   # Full digest: sha256:abc123def456
   ```

4. **Check GHCR authentication:**
   ```yaml
   - name: Login to GHCR
     uses: docker/login-action@v3
     with:
       registry: ghcr.io
       username: ${{ github.actor }}
       password: ${{ secrets.GITHUB_TOKEN }}
   ```

### Multi-Arch Manifest Missing Platform

**Symptom:**
```sh
docker manifest inspect ghcr.io/.../myapp:latest
# Shows only amd64, missing arm64
```

**Solutions:**

1. **Check both builds completed:**
   ```sh
   # In GitHub Actions, verify:
   # - build (myapp, amd64) ✅
   # - build (myapp, arm64) ✅
   ```

2. **Verify digest upload:**
   ```yaml
   - name: Upload digest
     uses: actions/upload-artifact@v4
     with:
       name: digests-myapp-${{ matrix.images.arch }}  # Include arch!
       path: /tmp/digests/myapp/*
   ```

3. **Check merge download pattern:**
   ```yaml
   - name: Download digests
     uses: actions/download-artifact@v4
     with:
       pattern: digests-myapp-*  # Wildcard matches all arches
       merge-multiple: true
   ```

4. **Verify imagetools command:**
   ```yaml
   - name: Create manifest
     run: |
       docker buildx imagetools create \
         -t ghcr.io/.../myapp:latest \
         $(printf 'ghcr.io/.../myapp@sha256:%s ' *)
       # Should process 2+ digests
   ```

### Manifest Inspect Shows Wrong Architecture

**Symptom:**
```sh
# ARM64 image shows amd64 in manifest
```

**Solutions:**

1. **Check binary architecture:**
   ```yaml
   - name: Verify binary
     run: |
       file ./binary
       # ARM64 should show: ARM aarch64
   ```

2. **Verify platform flag:**
   ```yaml
   - uses: docker/build-push-action@v5
     with:
       platforms: ${{ matrix.images.platforms }}  # linux/arm64
   ```

3. **Check Dockerfile base image:**
   ```dockerfile
   # Wrong: Forces amd64
   FROM alpine:latest
   
   # Correct: Respects target platform
   FROM --platform=$TARGETPLATFORM alpine:latest
   ```

## Matrix Issues

### Matrix Empty or Not Loading

**Symptom:**
- Build job doesn't run
- "Matrix is empty" error

**Solutions:**

1. **Validate JSON:**
   ```sh
   cat apps/myapp/matrix.json | jq .
   # Should show prettified JSON
   # Errors indicate invalid JSON
   ```

2. **Check file path:**
   ```yaml
   - name: Read matrix
     run: |
       # Verify path matches directory structure
       cat apps/myapp/matrix.json | jq -c .
   ```

3. **Debug matrix output:**
   ```yaml
   - name: Read matrix
     id: matrix
     run: |
       MATRIX=$(cat apps/myapp/matrix.json | jq -c .)
       echo "matrix=${MATRIX}" >> $GITHUB_OUTPUT
       echo "Matrix content: ${MATRIX}"  # Debug
   ```

### Wrong Number of Build Jobs

**Symptom:**
- Expected 4 builds, only 2 run
- Or unexpected number of jobs

**Solutions:**

1. **Count matrix entries:**
   ```sh
   cat apps/myapp/matrix.json | jq 'length'
   ```

2. **Check for duplicates:**
   ```sh
   cat apps/myapp/matrix.json | jq 'group_by(.component, .arch) | map(select(length > 1))'
   ```

3. **Verify MULTI_ARCH filtering:**
   ```yaml
   if [ "${{ inputs.MULTI_ARCH }}" = "true" ]; then
     echo "matrix=$(cat apps/myapp/matrix.json | jq -c .)"
   else
     echo "matrix=$(cat apps/myapp/matrix.json | jq -c '[.[] | select(.arch == "amd64")]')"
   fi
   ```

### Matrix Values Not Accessible

**Symptom:**
```
Error: matrix.images.component is undefined
```

**Solutions:**

1. **Check matrix variable name:**
   ```yaml
   strategy:
     matrix:
       images: ${{ fromJSON(needs.infos.outputs.matrix) }}
   
   # Access with: matrix.images.component
   # NOT: matrix.component
   ```

2. **Verify fromJSON:**
   ```yaml
   # Correct:
   images: ${{ fromJSON(needs.infos.outputs.matrix) }}
   
   # Wrong:
   images: ${{ needs.infos.outputs.matrix }}  # Missing fromJSON!
   ```

## GitHub Actions Issues

### Workflow Not Triggering

**Symptom:**
- Workflow doesn't run on schedule
- Manual trigger doesn't work

**Solutions:**

1. **Check workflow syntax:**
   ```sh
   yamllint .github/workflows/myapp.yml
   # Or use actionlint
   actionlint .github/workflows/myapp.yml
   ```

2. **Verify cron syntax:**
   ```yaml
   on:
     schedule:
       - cron: '0 1 * * *'  # ✅ Correct
       # Not: cron: '0 1 * * * *'  # ❌ Wrong (6 fields)
   ```

3. **Check repository settings:**
   - Settings → Actions → General
   - Ensure Actions are enabled
   - Verify workflow permissions

4. **Default branch:**
   - Scheduled workflows only run on default branch
   - Merge to main/master first

### Permission Denied Pushing to GHCR

**Symptom:**
```
Error: denied: permission_denied
```

**Solutions:**

1. **Check workflow permissions:**
   ```yaml
   permissions:
     contents: read
     packages: write  # Required!
   ```

2. **Verify token scope:**
   ```yaml
   - name: Login to GHCR
     uses: docker/login-action@v3
     with:
       password: ${{ secrets.GITHUB_TOKEN }}  # Use this
       # Not: password: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
   ```

3. **Check package visibility:**
   - Package must be public or user must have write access
   - Settings → Packages → Change visibility

### Artifacts Not Found

**Symptom:**
```
Error: Artifact 'digests-myapp-arm64' not found
```

**Solutions:**

1. **Verify artifact upload:**
   ```yaml
   - name: Upload digest
     uses: actions/upload-artifact@v4
     with:
       name: digests-myapp-${{ matrix.images.arch }}
       path: /tmp/digests/myapp/*
       if-no-files-found: error  # Will fail if no files
   ```

2. **Check path exists:**
   ```yaml
   - name: Export digest
     run: |
       mkdir -p /tmp/digests/myapp
       echo "Created directory"
       ls -la /tmp/digests/myapp/
   ```

3. **Verify download pattern:**
   ```yaml
   - name: Download digests
     uses: actions/download-artifact@v4
     with:
       pattern: digests-myapp-*  # Must match upload name
       merge-multiple: true
   ```

## Testing & Debugging

### Test Build Locally

**Test Docker build:**
```sh
# Build for amd64
docker buildx build \
  --platform linux/amd64 \
  --tag myapp:test-amd64 \
  .

# Build for arm64
docker buildx build \
  --platform linux/arm64 \
  --tag myapp:test-arm64 \
  .
```

**Test cross-compilation:**
```sh
# Create test Dockerfile
cat > Dockerfile.test << 'EOF'
FROM --platform=$BUILDPLATFORM golang:1.24 AS builder
WORKDIR /build
ARG TARGETOS
ARG TARGETARCH
ENV GOOS=${TARGETOS} GOARCH=${TARGETARCH}
RUN echo "Building for ${GOOS}/${GOARCH}"
EOF

docker buildx build \
  --platform linux/arm64 \
  --file Dockerfile.test \
  .
```

### Debug Workflow Variables

**Add debug step:**
```yaml
- name: Debug variables
  run: |
    echo "Component: ${{ matrix.images.component }}"
    echo "Arch: ${{ matrix.images.arch }}"
    echo "Platform: ${{ matrix.images.platforms }}"
    echo "Tag: ${{ needs.infos.outputs.tag }}"
    echo "Matrix: ${{ needs.infos.outputs.matrix }}"
```

### Inspect Built Images

**Check image details:**
```sh
# Inspect manifest
docker manifest inspect ghcr.io/this-is-tobi/mirror/myapp:latest

# Pull specific architecture
docker pull --platform linux/arm64 ghcr.io/.../myapp:latest

# Check binary architecture
docker run --rm --platform linux/arm64 ghcr.io/.../myapp:latest sh -c 'file /app/binary'

# Test execution
docker run --rm --platform linux/arm64 ghcr.io/.../myapp:latest version
```

### Enable Debug Logging

**In GitHub Actions:**
1. Repository Settings → Secrets → Variables
2. Add variable: `ACTIONS_STEP_DEBUG` = `true`
3. Re-run workflow

**In Dockerfile:**
```dockerfile
# Add debug output
RUN echo "GOOS=${GOOS}" && \
    echo "GOARCH=${GOARCH}" && \
    ls -la /build/
```

## Performance Issues

### Build Takes Too Long

**Solutions:**

1. **Enable build cache:**
   ```yaml
   cache-from: type=local,src=/tmp/.buildx-cache
   cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max
   ```

2. **Use multi-stage builds:**
   ```dockerfile
   FROM golang:1.24 AS builder
   # Build steps
   
   FROM alpine:latest
   COPY --from=builder /app/binary /app/binary
   ```

3. **Reduce layer count:**
   ```dockerfile
   # Combine RUN commands
   RUN apt-get update && \
       apt-get install -y pkg1 pkg2 && \
       rm -rf /var/lib/apt/lists/*
   ```

4. **Use cross-compilation instead of QEMU:**
   - QEMU emulation is 5-10x slower
   - Cross-compilation is near-native speed

### Too Many Parallel Builds

**Issue:** GitHub Actions free tier limited to 20 concurrent jobs

**Solutions:**

1. **Reduce matrix size temporarily:**
   ```yaml
   # Test with amd64 only first
   echo "matrix=$(cat matrix.json | jq -c '[.[] | select(.arch == "amd64")]')"
   ```

2. **Add build dependencies:**
   ```yaml
   strategy:
     max-parallel: 5  # Limit concurrent jobs
   ```

3. **Upgrade to GitHub Pro:**
   - Increases concurrent job limit

## Common Error Messages

### "exec format error"

**Meaning:** Trying to run binary for wrong architecture

**Fix:** Verify binary architecture matches platform

### "no space left on device"

**Meaning:** Runner disk full (usually cache)

**Fix:** Implement cache rotation

### "manifest unknown"

**Meaning:** Image/tag doesn't exist in registry

**Fix:** Check image path, verify push succeeded

### "failed to solve: process did not complete successfully"

**Meaning:** Command failed during Docker build

**Fix:** Check specific command output, enable debug logging

### "resource not accessible by integration"

**Meaning:** Missing permissions

**Fix:** Add required permissions in workflow file

## Getting Help

### Check Workflow Logs

1. Go to GitHub Actions tab
2. Select failed workflow run
3. Expand failed job
4. Review step output
5. Look for error messages

### Useful Debug Commands

```yaml
# List files
- run: ls -laR /tmp/digests/

# Check environment
- run: env | sort

# Test registry access
- run: |
    docker login ghcr.io -u ${{ github.actor }} -p ${{ secrets.GITHUB_TOKEN }}
    docker images

# Verify buildx
- run: docker buildx ls
```

### Resources

- [Docker Buildx Issues](https://github.com/docker/buildx/issues)
- [GitHub Actions Forum](https://github.community/c/code-to-cloud/github-actions)
- [Multi-platform Guide](https://docs.docker.com/build/building/multi-platform/)

## Reporting Issues

When reporting an issue, include:

1. **Workflow file** (relevant sections)
2. **Matrix.json** configuration
3. **Error message** (exact text)
4. **Workflow run URL**
5. **Steps to reproduce**
6. **Expected vs actual behavior**
