# Dockerfile Pipeline

## Planton CLI Commands

```bash
# Register this pipeline
planton tekton pipeline register \
  --yaml-file=dockerfile.yaml \
  --name="Build and Push Image with Kaniko" \
  --description="Build container images using Dockerfile with BuildKit and generate Kustomize manifests" \
  --git-web-url="https://github.com/plantoncloud/tekton-hub/blob/main/pipelines/dockerfile.yaml" \
  --git-clone-url="https://github.com/plantoncloud/tekton-hub.git" \
  --git-file-path="pipelines/dockerfile.yaml" \
  --overview-markdown-file=dockerfile.md \
  --tags="pipeline,dockerfile,buildkit,container-build,kustomize" \
  --platform

# Validate the manifest
planton tekton pipeline validate --yaml-file=dockerfile.yaml
```

## Purpose

Complete CI/CD pipeline that builds container images from Dockerfiles using BuildKit in daemon-less mode and generates Kubernetes deployment manifests using Kustomize.

## Workflow

1. **git-checkout** - Clone source code from Git repository
2. **setup-package-credentials** (conditional) - Copy package credentials if needed
3. **build-image** - Build and push container image using BuildKit
4. **kustomize-build** - Generate Kubernetes manifests for all environments

## Parameters

### Git Parameters
- **git-url** (required) - Git repository URL to clone
- **git-revision** (default: "main") - Git revision to checkout
- **git-branch** (default: "") - Git branch name
- **sparse-checkout-directories** (default: "") - Comma-separated directory patterns for sparse checkout

### Build Parameters
- **setup-package-credentials** (default: "false") - Set to "true" to copy credentials from package-credentials workspace
- **image-name** (required) - Full destination image (registry/repo:tag)
- **dockerfile-path** (default: "Dockerfile") - Relative path to Dockerfile
- **project-root** (default: ".") - Relative path to project root inside cloned repo

### ConfigMap Parameters
- **kustomize-manifests-config-map-name** (required) - Name of ConfigMap to store service manifests
- **dockerfile-config-map-name** (required) - Name of ConfigMap to store Dockerfile content
- **kustomize-base-directory** (default: "_kustomize") - Relative path to kustomize base directory inside project root

### Ownership Parameters
- **owner-identifier-label-key** (default: "") - Optional owner identifier label key for ConfigMaps
- **owner-identifier-label-value** (default: "") - Optional owner identifier label value

## Workspaces

- **source** (required) - Workspace where source code is cloned
- **package-credentials** (required) - Workspace storing package credentials required for build

## Task Details

### 1. git-checkout

Uses Tekton Hub's `git-clone` task (version 0.9) to clone the repository.

**Runs**: Always  
**Image**: `ghcr.io/tektoncd/github.com/tektoncd/pipeline/cmd/git-init:v0.45.0`

### 2. setup-package-credentials

Conditionally runs when `setup-package-credentials` parameter is "true".

**When**: `setup-package-credentials == "true"`  
**Runs after**: git-checkout  
**Purpose**: Copies files from package-credentials workspace to `.package-credentials/` in source workspace  
**Image**: alpine:3

### 3. build-image

Uses the `buildkit.yaml` task from this repository to build the container image.

**Runs after**: git-checkout  
**Task reference**: git resolver pointing to `tasks/buildkit.yaml`

**Key parameters**:
- `image`: Target image name
- `contextDir`: Project root directory
- `dockerfilePath`: Path to Dockerfile
- `cache`: Enabled by default for faster rebuilds
- `dockerfile-config-map-namespace`: `planton-cloud-pipelines`
- `dockerfile-config-map-name`: Stores Dockerfile for audit

### 4. kustomize-build

Generates Kubernetes manifests for all environments and stores in a ConfigMap.

**Runs after**: git-checkout (parallel with build-image)  
**Task reference**: git resolver pointing to `tasks/kustomize-build.yaml`  
**Namespace**: Creates ConfigMap in `planton-cloud-pipelines` namespace

## Parallelization

The `kustomize-build` task runs in parallel with `build-image` since they don't depend on each other. Both run after `git-checkout` completes.

## BuildKit Features

This pipeline uses BuildKit daemon-less mode which provides:
- **Inline caching** - Reuses layers from previous builds stored in registry
- **Multi-platform support** - Can build for multiple architectures
- **Privileged mode** - Runs with elevated permissions for BuildKit operations
- **Modern Dockerfile frontend** - Uses gateway.v0 with dockerfile:1 for proper ARG expansion

## Dockerfile Export

The build-image task stores the Dockerfile content in a ConfigMap, enabling:
- Audit trail of which Dockerfile was used
- Web console display of Dockerfile contents
- Correlation between builds and source configuration

## Package Credentials

When `setup-package-credentials` is enabled:
1. Copies files from package-credentials workspace
2. Makes them available at `.package-credentials/`
3. Can be referenced in Dockerfile `COPY` instructions or build scripts

Typical use cases:
- Private npm registries (.npmrc)
- Maven/Gradle credentials
- Google Cloud service accounts
- Custom CA certificates

## Planton Cloud Usage

This pipeline is commonly used for:
- Projects with existing Dockerfiles
- Custom build processes requiring fine control
- Multi-stage Docker builds
- Applications with complex dependencies

## When to Use

Choose this pipeline for:
- Projects with existing Dockerfile
- Need full control over build process
- Multi-stage builds
- Custom base images
- Complex dependency management
- Language/framework combinations not covered by buildpacks
