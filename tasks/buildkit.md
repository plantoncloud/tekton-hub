# BuildKit Daemon-less Task

## Planton CLI Commands

```bash
# Register this task
planton tekton task register \
  --yaml-file=buildkit.yaml \
  --name="BuildKit Daemon-less" \
  --description="Build and push OCI images using BuildKit in daemon-less mode" \
  --git-web-url="https://github.com/plantonhq/tekton-hub/blob/main/tasks/buildkit.yaml" \
  --git-clone-url="https://github.com/plantonhq/tekton-hub.git" \
  --git-file-path="tasks/buildkit.yaml" \
  --overview-markdown-file=buildkit.md \
  --tags="container-build,buildkit,dockerfile" \
  --platform

# Validate the manifest
planton tekton task validate --yaml-file=buildkit.yaml
```

## Purpose

Builds and pushes OCI-compatible container images using BuildKit in daemon-less mode. This task runs BuildKit without requiring a separate daemon process, making it suitable for containerized CI/CD environments.

## Key Features

- **Daemon-less operation** - Runs BuildKit without external daemon dependencies
- **Multi-platform builds** - Supports building for multiple architectures (e.g., linux/amd64, linux/arm64)
- **Inline caching** - Leverages registry-based layer caching to speed up subsequent builds
- **Dockerfile export** - Stores the Dockerfile content in a ConfigMap for audit and reference
- **Build argument support** - Accepts custom build arguments for parameterized Dockerfiles
- **Registry authentication** - Supports Docker config for private registry access

## Security Context

Requires `privileged: true` security context even when using the rootless BuildKit image. This is a BuildKit requirement for daemon-less operation.

## Parameters

- **image** (required) - Full image reference including registry, repository, and tag
- **contextDir** (default: ".") - Build context directory path
- **dockerfilePath** (default: "Dockerfile") - Path to Dockerfile relative to context
- **platforms** (optional) - Comma-separated list of target platforms
- **cache** (default: "true") - Enable inline caching
- **push** (default: "true") - Push image to registry after build
- **buildArgs** (optional) - Multi-line string with KEY=value pairs
- **buildkitImage** (default: "moby/buildkit:v0.23.2-rootless") - BuildKit image to use
- **dockerfile-config-map-name** (required) - Name of ConfigMap to store Dockerfile
- **dockerfile-config-map-namespace** (default: "planton-cloud-pipelines") - Namespace for ConfigMap
- **owner-identifier-label-key** (optional) - Label key for resource ownership tracking
- **owner-identifier-label-value** (optional) - Label value for resource ownership tracking

## Workspaces

- **source** (required) - Directory containing source code and Dockerfile
- **dockerconfig** (optional) - Docker config.json for registry authentication

## Steps

1. **export-dockerfile** - Reads Dockerfile, base64-encodes it, and stores in a ConfigMap with optional owner labels
2. **build-and-push** - Runs BuildKit in daemon-less mode to build and push the image

## Frontend Configuration

Uses the modern gateway.v0 frontend with dockerfile:1 for proper ARG variable expansion. This avoids known bugs in the legacy dockerfile.v0 frontend with ARG expansion in COPY instructions.

## Planton Cloud Integration

The Dockerfile export step enables Planton Cloud to:
- Display Dockerfile contents in the web console
- Track which Dockerfile was used for each build
- Associate builds with source code configuration

