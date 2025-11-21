# BuildKit Root-less Task

## Planton CLI Commands

```bash
# Register this task
planton tekton task register \
  --yaml-file=buildkit-root-less.yaml \
  --name="BuildKit Root-less" \
  --description="Build and push OCI images with BuildKit in daemon-less rootless mode" \
  --git-web-url="https://github.com/plantoncloud/tekton-hub/blob/main/tasks/buildkit-root-less.yaml" \
  --git-clone-url="https://github.com/plantoncloud/tekton-hub.git" \
  --git-file-path="tasks/buildkit-root-less.yaml" \
  --overview-markdown-file=buildkit-root-less.md \
  --tags="container-build,buildkit,dockerfile,rootless,security" \
  --platform

# Validate the manifest
planton tekton task validate --yaml-file=buildkit-root-less.yaml
```

## Purpose

Builds and pushes OCI-compatible container images using BuildKit in daemon-less rootless mode. This task provides enhanced security by running without privileged access, requiring only `/dev/fuse` device access.

## Key Difference from buildkit.yaml

The standard `buildkit.yaml` task requires `privileged: true` security context. This rootless variant runs with:
- `runAsUser: 1000`
- `runAsGroup: 1000`
- `allowPrivilegeEscalation: false`
- `capabilities: drop: ["ALL"]`
- `seccompProfile: type: Unconfined`

## Security Benefits

- **No privileged access required** - Runs as non-root user (UID/GID 1000)
- **Minimal capabilities** - Drops all Linux capabilities
- **Process isolation** - Uses `--oci-worker-no-process-sandbox` flag
- **Suitable for restricted environments** - Compatible with PodSecurityPolicies that disallow privileged containers

## Parameters

Same as buildkit.yaml task:

- **image** (required) - Full image reference including registry, repository, and tag
- **contextDir** (default: ".") - Build context directory path
- **dockerfilePath** (default: "Dockerfile") - Path to Dockerfile relative to context
- **platforms** (optional) - Comma-separated list of target platforms
- **cache** (default: "true") - Enable inline caching
- **push** (default: "true") - Push image to registry after build
- **buildArgs** (optional) - Multi-line string with KEY=value pairs
- **buildkitImage** (default: "docker.io/moby/buildkit:v0.23.2-rootless") - BuildKit image to use
- **dockerfile-config-map-name** (required) - Name of ConfigMap to store Dockerfile
- **dockerfile-config-map-namespace** (default: "planton-cloud-pipelines") - Namespace for ConfigMap
- **owner-identifier-label-key** (optional) - Label key for resource ownership tracking
- **owner-identifier-label-value** (optional) - Label value for resource ownership tracking

## Workspaces

- **source** (required) - Directory containing source code and Dockerfile
- **dockerconfig** (optional) - Docker config.json for registry authentication

## Steps

1. **export-dockerfile** - Stores Dockerfile content in a ConfigMap (runs as UID 1000)
2. **build-and-push** - Runs BuildKit in rootless daemon-less mode (runs as UID 1000)

## Environment Variables

- **BUILDKITD_FLAGS** - Set to `--oci-worker-no-process-sandbox` to disable process sandbox

## Frontend Configuration

Uses the dockerfile.v0 frontend. For projects requiring advanced ARG expansion, consider using the standard buildkit.yaml task with gateway.v0 frontend.

## When to Use

Choose this task when:
- Running in security-restricted Kubernetes clusters
- PodSecurityPolicies prohibit privileged containers
- Compliance requirements mandate non-root execution
- Building standard Dockerfiles without complex ARG expansion needs

