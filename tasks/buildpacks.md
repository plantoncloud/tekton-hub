# Buildpacks Task

## Planton CLI Commands

```bash
# Register this task
planton tekton task register \
  --yaml-file=buildpacks.yaml \
  --name="Buildpacks" \
  --description="Build source into container images using Cloud Native Buildpacks" \
  --git-web-url="https://github.com/plantonhq/tekton-hub/blob/main/tasks/buildpacks.yaml" \
  --git-clone-url="https://github.com/plantonhq/tekton-hub.git" \
  --git-file-path="tasks/buildpacks.yaml" \
  --overview-markdown-file=buildpacks.md \
  --tags="container-build,buildpacks,cnb" \
  --platform

# Validate the manifest
planton tekton task validate --yaml-file=buildpacks.yaml
```

## Purpose

Builds application source code into container images using Cloud Native Buildpacks. This task automatically detects the application type and applies appropriate buildpacks to create production-ready container images without requiring a Dockerfile.

## Key Features

- **Automatic language detection** - Detects application type and selects appropriate buildpacks
- **No Dockerfile required** - Builds images directly from source code
- **Optimized caching** - Supports persistent cache workspace or cache image for faster rebuilds
- **Build-time environment variables** - Pass configuration to buildpacks during image creation
- **Layer reuse** - Efficiently caches and reuses layers across builds
- **Process type configuration** - Define default process to run in the container

## Lifecycle API

Uses Cloud Native Buildpacks Platform API version 0.9.

## Parameters

- **APP_IMAGE** (required) - Name and tag for the built container image
- **BUILDER_IMAGE** (required) - CNB builder image containing lifecycle and buildpacks
- **SOURCE_SUBPATH** (default: "") - Subdirectory within source workspace containing application code
- **ENV_VARS** (array, default: []) - Environment variables to set during build-time
- **PROCESS_TYPE** (default: "web") - Default process type to set on the image
- **RUN_IMAGE** (default: "") - Reference to a specific run image to use
- **CACHE_IMAGE** (default: "") - Persistent cache image name (when cache workspace not provided)
- **SKIP_RESTORE** (default: "false") - Skip layer metadata writing and cache restoration
- **USER_ID** (default: "0") - User ID of the builder image user
- **GROUP_ID** (default: "0") - Group ID of the builder image user
- **PLATFORM_DIR** (default: "empty-dir") - Name of the platform directory volume

## Workspaces

- **source** (required) - Directory where application source code is located
- **cache** (optional) - Directory where cache is stored (alternative to CACHE_IMAGE)
- **dockerconfig** (optional) - Docker config.json file for registry authentication

## Results

- **APP_IMAGE_DIGEST** - The digest of the built image
- **APP_IMAGE_URL** - The full URL of the built image

## Steps

1. **prepare** - Sets permissions on workspaces and creates environment variable files in `/platform/env`
2. **create** - Runs the CNB lifecycle creator to build the image
3. **results** - Extracts image digest and URL from build report

## Environment Variable Configuration

Build-time environment variables are written as individual files in `/platform/env` directory. Each KEY=value pair becomes a file where the filename is the key and contents are the value.

## Google Cloud Integration

The task includes support for Google Cloud credentials via the `GOOGLE_APPLICATION_CREDENTIALS` environment variable, pointing to `/workspace/source/.package-credentials/google-service-account.json`.

## Security Context

The `create` step runs as root (UID 0, GID 0) to allow the CNB lifecycle full access to layer manipulation.

## Planton Cloud Usage

Commonly used in Planton Cloud pipelines with:
- `BUILDER_IMAGE`: `paketobuildpacks/builder-jammy-base`
- Package credentials mounted at `.package-credentials/` for private dependency access

