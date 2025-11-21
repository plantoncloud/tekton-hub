# Buildpacks Pipeline

## Planton CLI Commands

```bash
# Register this pipeline
planton tekton pipeline register \
  --yaml-file=pipelines/buildpacks.yaml \
  --name="Build and Push Image with Buildpacks" \
  --description="Build container images using Cloud Native Buildpacks and generate Kustomize manifests" \
  --git-web-url="https://github.com/plantoncloud/tekton-hub/blob/main/pipelines/buildpacks.yaml" \
  --git-clone-url="https://github.com/plantoncloud/tekton-hub.git" \
  --git-file-path="pipelines/buildpacks.yaml" \
  --overview-markdown-file=pipelines/buildpacks.md \
  --tags="pipeline,buildpacks,container-build,kustomize" \
  --platform

# Validate the manifest
planton tekton pipeline validate --yaml-file=pipelines/buildpacks.yaml
```

## Purpose

Complete CI/CD pipeline that builds container images from source code using Cloud Native Buildpacks and generates Kubernetes deployment manifests using Kustomize.

## Workflow

1. **git-checkout** - Clone source code from Git repository
2. **setup-package-credentials** (conditional) - Copy package credentials if needed
3. **build-image** - Build and push container image using Buildpacks
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
- **buildpacks-builder-image** (default: "paketobuildpacks/builder-jammy-base") - Buildpacks builder image to use
- **project-root** (default: ".") - Relative path to project root inside cloned repo

### Kustomize Parameters
- **kustomize-manifests-config-map-name** (required) - Name of ConfigMap to store service manifests
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

Uses the `buildpacks.yaml` task from this repository to build the container image.

**Runs after**: setup-package-credentials  
**Task reference**: git resolver pointing to `tasks/buildpacks.yaml`  
**Environment**: Sets `GOOGLE_APPLICATION_CREDENTIALS` to the service account JSON path

**Key parameters**:
- `BUILDER_IMAGE`: Uses the specified buildpacks builder
- `APP_IMAGE`: Target image name
- `SOURCE_SUBPATH`: Project root directory
- `ENV_VARS`: Includes Google credentials path

### 4. kustomize-build

Generates Kubernetes manifests for all environments and stores in a ConfigMap.

**Runs after**: git-checkout (parallel with build-image)  
**Task reference**: git resolver pointing to `tasks/kustomize-build.yaml`  
**Namespace**: Creates ConfigMap in `planton-cloud-pipelines` namespace

## Parallelization

The `kustomize-build` task runs in parallel with `build-image` since they don't depend on each other. Both run after `git-checkout` completes.

## Google Cloud Integration

When `setup-package-credentials` is enabled, the pipeline:
1. Copies Google service account JSON from package-credentials workspace
2. Makes it available at `.package-credentials/google-service-account.json`
3. Passes credential path to buildpacks via `GOOGLE_APPLICATION_CREDENTIALS`

This enables access to:
- Google Artifact Registry for dependencies
- Google Cloud Storage for build artifacts
- Other GCP services during build

## Planton Cloud Usage

Typical usage in Planton Cloud:
- Builder: `paketobuildpacks/builder-jammy-base`
- Namespace: `planton-cloud-pipelines`
- Package credentials: Mounted as workspace from secret
- Owner labels: Track pipeline runs to specific resources

## When to Use

Choose this pipeline for:
- Applications without Dockerfiles
- Automatic language/framework detection
- Standardized build processes
- Projects using Java, Node.js, Python, Go, etc.
- Builds requiring private package repository access
