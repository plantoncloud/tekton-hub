# Buildpacks Custom Task

## Planton CLI Commands

```bash
# Register this task
planton tekton task register \
  --yaml-file=tasks/buildpacks-custom.yaml \
  --name="Buildpacks Custom" \
  --description="Customizable Buildpacks builds with extra bindings and certificate support" \
  --git-web-url="https://github.com/plantoncloud/tekton-hub/blob/main/tasks/buildpacks-custom.yaml" \
  --git-clone-url="https://github.com/plantoncloud/tekton-hub.git" \
  --git-file-path="tasks/buildpacks-custom.yaml" \
  --overview-markdown-file=tasks/buildpacks-custom.md \
  --tags="container-build,buildpacks,cnb,custom,certificates" \
  --platform

# Validate the manifest
planton tekton task validate --yaml-file=tasks/buildpacks-custom.yaml
```

## Purpose

Extended Cloud Native Buildpacks task that provides additional customization options including extra bindings, CA certificate bundle support, and verbose logging. Based on OpenShift Pipeline's buildpacks implementation.

## Key Features

- **Extra bindings support** - Mount additional files from bindings workspace
- **CA certificate bundles** - Copy certificate files to `SERVICE_BINDING_ROOT`
- **Glob pattern matching** - Select binding files using glob expressions
- **Verbose logging** - Enable detailed command execution output
- **Base64-encoded scripts** - Shell scripts embedded as base64 for reliability
- **Platform API 0.11** - Uses newer CNB Platform API version

## Lifecycle API

Uses Cloud Native Buildpacks Platform API version 0.11, newer than the standard buildpacks task (0.9).

## Parameters

- **IMAGE** (required) - Application's container image name and tag
- **BUILDER_IMAGE** (default: "mirror.gcr.io/paketobuildpacks/builder:base") - CNB builder image
- **CNB_PLATFORM_API** (default: "0.11") - Lifecycle platform API version
- **SUBDIRECTORY** (default: "") - Alternative CNB_APP_DIR relative to source workspace
- **ENV_VARS** (array, default: []) - Build-time environment variables
- **PROCESS_TYPE** (default: "web") - Application process type
- **BINDINGS_GLOB** (default: "*.pem") - Glob pattern for binding files in bindings workspace
- **RUN_IMAGE** (default: "") - Reference to a run image
- **CACHE_IMAGE** (default: "") - Persistent cache image name
- **SKIP_RESTORE** (default: "false") - Skip layer metadata or cached layer restoration
- **USER_ID** (default: "1000") - CNB container image user ID
- **GROUP_ID** (default: "1000") - CNB container image group ID
- **VERBOSE** (default: "false") - Enable verbose logging with command tracing

## Workspaces

- **source** (required) - Application source code
- **cache** (optional) - Cache directory (alternative to CACHE_IMAGE)
- **bindings** (optional) - Extra bindings and CA certificate bundle files

## Results

- **IMAGE_DIGEST** - Digest of the built image
- **IMAGE_URL** - Fully qualified container image name

## Steps

1. **prepare** - Loads embedded shell scripts from base64, sets workspace permissions, writes environment variables to `/platform/env`, and copies binding files matching BINDINGS_GLOB to SERVICE_BINDING_ROOT
2. **creator** - Runs the CNB lifecycle creator using embedded script
3. **report** - Extracts digest and URL from `/layers/report.toml`

## Embedded Scripts

The task includes four base64-encoded shell scripts:
- **common.sh** - Shared functions and environment variable setup
- **prepare.sh** - Workspace preparation and binding file copying
- **creator.sh** - CNB lifecycle creator invocation
- **report.sh** - Result extraction from build report

## Bindings Workspace

When the bindings workspace is mounted and BINDINGS_GLOB is set, files matching the glob pattern are copied from the bindings workspace to SERVICE_BINDING_ROOT. This allows buildpacks to access:
- CA certificate bundles
- Custom buildpack configuration files
- Service binding credentials

## Security Context

Steps run as `runAsNonRoot: false`, allowing full control for the CNB lifecycle.

## Verbose Mode

When VERBOSE is "true":
- Sets `CNB_LOG_LEVEL=debug`
- Enables shell `set -x` for command tracing
- Useful for troubleshooting buildpack behavior

## When to Use

Choose this task when:
- Custom CA certificates are required for private registries or dependencies
- Additional buildpack bindings need to be provided
- Detailed logging is needed for debugging builds
- Using newer Platform API features (0.11)
