# Railway Deployment Issues & Fixes - Novu Project

## Overview
Attempting to deploy Novu (a Node.js monorepo) to Railway with 4 services: API, Worker, WebSocket, and Dashboard.

## Project Structure
- **Repository**: AddexSolutions/novu (fork of novuhq/novu)
- **Framework**: Nx monorepo with pnpm
- **Services to deploy**:
  - novu-api (apps/api/Dockerfile)
  - novu-worker (apps/worker/Dockerfile)
  - novu-ws (apps/ws/Dockerfile)
  - novu-dashboard (apps/dashboard/dockerfile)

## Issues Encountered & Fixes Attempted

### Issue 1: Docker Cache Mount Syntax Errors
**Error**: `Cache mounts MUST be in the format --mount=type=cache,id=<cache-id>`

**Root Cause**: Railway's Railpack/Docker builder doesn't support Docker buildkit mount syntax (`--mount=type=cache` and `--mount=type=secret`).

**What I Tried**:
1. Simplified cache mounts to just `--mount=type=cache,id=<cache-id>` (removed `target=` parameter)
2. Removed all secret mounts (`--mount=type=secret,id=...`)
3. Eventually removed ALL `--mount` flags since Railway doesn't support any mount syntax

**Files Modified**:
- `apps/api/Dockerfile`
- `apps/worker/Dockerfile`
- `apps/ws/Dockerfile`
- `apps/dashboard/dockerfile`

**Result**: ✅ Fixed - Builds no longer fail on mount syntax

### Issue 2: Missing Directory Copy Errors
**Error**: `failed to calculate checksum of ref ... "/meta": not found`

**Root Cause**: Dockerfiles try to `COPY ./meta ./deps ./pkg .` but these directories don't exist in the repository root.

**What I Tried**:
- Commented out the COPY commands for these non-existent directories
- Assumption: These directories are created by the build process itself

**Files Modified**:
- `apps/api/Dockerfile`
- `apps/ws/Dockerfile`
- `apps/worker/Dockerfile`

**Result**: ❌ Still failing - API and WS still show this error

### Issue 3: Nx Cloud Authentication Failures
**Error**: `NX Cloud: Workspace is unable to be authorized. Exiting run.`

**Root Cause**: Railway builds don't have access to Nx Cloud credentials, and the prebuild step tries to connect to Nx Cloud.

**What I Tried**:
1. Set `NX_CLOUD_ACCESS_TOKEN=""`
2. Set `NX_DAEMON=false`
3. Set `NX_CLOUD=false`
4. Set `CI=true`
5. Set `NX_SKIP_NX_CACHE=true`
6. Set `NX_CLOUD_DISABLE=true`
7. Changed worker build command from `pnpm run build` to `pnpm run build:worker` (skips prebuild)

**Files Modified**:
- `apps/worker/Dockerfile`
- `apps/api/Dockerfile`
- `apps/ws/Dockerfile`
- `apps/dashboard/dockerfile`

**Result**: ❌ Worker still fails on Nx Cloud auth

### Issue 4: Build Context Problems
**Error**: `No projects found in "/usr/src/app"` and `No package.json found`

**Root Cause**: When building with Docker (API/WS), the workspace context seems incorrect.

**What I Tried**:
- Removed directory copies that were failing
- Kept the build process as-is

**Result**: ❌ API and WS fail during pnpm install/build phase

## Current Status by Service

### ✅ Dashboard (Working)
- Uses Railpack (not Docker)
- Successfully builds and deploys
- No cache mount issues

### ❌ API (Still Failing)
- Uses Docker builder
- Error: `"/meta": not found` during COPY step
- Build fails before pnpm install

### ❌ Worker (Still Failing)
- Uses Railpack
- Error: Nx Cloud authentication during prebuild
- Despite all environment variable attempts

### ❌ WS (Still Failing)
- Uses Docker builder
- Error: `No projects found in "/usr/src/app"` during pnpm install
- Gets past directory copy but fails on workspace detection

## Railway-Specific Challenges Discovered

1. **Railpack vs Docker**: Different services use different builders
   - Some services use Railpack (Node.js detection)
   - Some services use Docker (explicit dockerfilePath)
   - They behave differently with the same monorepo

2. **Mount Syntax**: Railway doesn't support ANY Docker buildkit mount syntax
   - No `--mount=type=cache`
   - No `--mount=type=secret`
   - No buildkit features at all

3. **Build Context**: The build context differs between Railpack and Docker builds
   - Railpack copies the entire repo
   - Docker builds may have different working directories

4. **Nx Cloud**: CI environments need special handling for Nx workspaces

5. **Monorepo Structure**: The Dockerfiles assume certain directories exist that may be generated

## Questions for Railway Experts

1. **Why do some services use Railpack vs Docker?** Can we force all to use the same builder?

2. **How to handle monorepos with Dockerfiles that expect generated directories?** Should these directories be created before Docker build?

3. **What's the correct way to disable Nx Cloud in Railway CI?** Environment variables don't seem sufficient.

4. **Why does Railpack succeed where Docker fails?** Can we use Railpack for all services?

5. **How to handle workspace detection in Docker builds?** The "No projects found" suggests workspace configuration issues.

## Configuration Files

### railway.toml
```toml
[[services]]
name = "novu-api"
dockerfile = "apps/api/Dockerfile"

[[services]]
name = "novu-worker"
dockerfile = "apps/worker/Dockerfile"

[[services]]
name = "novu-ws"
dockerfile = "apps/ws/Dockerfile"

[[services]]
name = "novu-dashboard"
dockerfile = "apps/dashboard/dockerfile"
```

### Current Environment Variables Tried
```
NX_CLOUD_ACCESS_TOKEN=""
NX_DAEMON=false
NX_CLOUD=false
CI=true
NX_SKIP_NX_CACHE=true
NX_CLOUD_DISABLE=true
```

## Next Steps Needed

1. **Determine correct build strategy**: Should all services use Railpack or Docker?
2. **Fix workspace context**: Ensure monorepo structure works in both build environments
3. **Resolve Nx Cloud issues**: Find Railway-compatible way to disable Nx Cloud
4. **Handle missing directories**: Either create them or modify Dockerfiles to not need them

## Files Modified
- `apps/api/Dockerfile` - Removed mounts and directory copies
- `apps/worker/Dockerfile` - Removed mounts, added Nx Cloud env vars, changed build command
- `apps/ws/Dockerfile` - Removed mounts and directory copies
- `apps/dashboard/dockerfile` - Removed mounts, added Nx Cloud env vars

---

**Created**: November 23, 2025
**Status**: Multiple services still failing, seeking Railway expert input</contents>
</xai:function_call">Create RAILWAY_DEPLOYMENT_ISSUES.md documenting all issues and fixes attempted
