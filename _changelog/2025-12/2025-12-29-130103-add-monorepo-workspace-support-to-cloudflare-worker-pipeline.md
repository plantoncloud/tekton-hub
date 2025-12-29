# Add Monorepo Workspace Support to Cloudflare Worker Pipeline

**Date**: December 29, 2025
**Type**: Enhancement
**Components**: Pipeline Management, Configuration

## Summary

Added flexible yarn workspace support to the Cloudflare Worker pipeline, enabling monorepo setups where `yarn install` needs to run from a different directory than the worker build. This maintains backward compatibility for standalone workers while supporting workspace protocols like `workspace:*`.

## Problem Statement / Motivation

The Cloudflare Worker pipeline's `install-dependencies` task always ran `yarn install` in the `project-root` directory. This broke when building workers from monorepos using yarn workspaces with `workspace:*` protocol dependencies.

### Pain Points

- **Monorepo builds failed**: Workers with `workspace:*` dependencies couldn't resolve them when installed in isolation
- **No workspace support**: Pipeline assumed all dependencies were local to the worker directory
- **Limited flexibility**: No way to specify a different installation directory

### Error Example

```
npm error code EUNSUPPORTEDPROTOCOL
npm error Unsupported URL Type "workspace:": workspace:*
```

## Solution / What's New

Added a new `package-manager-directory` parameter that controls where the package manager runs, separate from where wrangler builds the worker.

### Directory Resolution Logic

```
package-manager-directory set?
    ├── Yes → Install dependencies there
    └── No  → Fall back to project-root (backward compatible)
```

### New Parameter

```yaml
- name: package-manager-directory
  type: string
  description: "Directory to run package manager install from (defaults to project-root, set to '.' for monorepo workspaces)"
  default: ""
```

## Implementation Details

### Pipeline Changes

**File**: `pipelines/cloudflare-worker.yaml`

1. Added `package-manager-directory` parameter to pipeline spec
2. Updated `install-dependencies` task to accept both `project-root` and `package-manager-directory`
3. Modified install script to resolve directory with fallback logic:

```sh
# Determine install directory (defaults to project-root if not specified)
INSTALL_DIR="$(params.package-manager-directory)"
if [ -z "$INSTALL_DIR" ]; then
  INSTALL_DIR="$(params.project-root)"
fi

cd "$INSTALL_DIR"
echo "Installing dependencies in: $INSTALL_DIR"
```

4. Changed `workingDir` from `/workspace/source/$(params.project-root)` to `/workspace/source` to enable dynamic directory resolution

### Documentation Updates

**File**: `pipelines/cloudflare-worker.md`

- Added `package-manager-directory` to Git Parameters section
- Updated `install-dependencies` task description with directory resolution logic
- Added new "Monorepo Support" section with usage examples

## Benefits

### For Standalone Workers
- **No changes needed**: Empty default preserves existing behavior
- **Backward compatible**: All existing pipeline configurations continue to work

### For Monorepo Users
- **Workspace support**: Can now use `workspace:*` protocol dependencies
- **Flexible installation**: Run `yarn install` at workspace root, build at worker directory
- **Sparse checkout friendly**: Works with selective directory cloning

## Impact

### Files Changed
- `pipelines/cloudflare-worker.yaml` - Added parameter and updated task logic
- `pipelines/cloudflare-worker.md` - Documentation updates

### Usage Examples

**Standalone worker** (unchanged):
```yaml
params:
  - name: project-root
    value: "my-worker"
```

**Monorepo with yarn workspaces**:
```yaml
params:
  - name: project-root
    value: "backend/services/my-worker"
  - name: package-manager-directory
    value: "."
  - name: sparse-checkout-directories
    value: "package.json,.yarnrc.yml,yarn.lock,shared/packages,backend/services/my-worker"
```

## Related Work

- **Planton Cloud TypeScript Stubs Migration**: This enhancement was motivated by Planton Cloud's migration to yarn workspaces for sharing TypeScript protobuf stubs across their web console and Cloudflare workers

---

**Status**: ✅ Production Ready
**Timeline**: ~30 minutes implementation

