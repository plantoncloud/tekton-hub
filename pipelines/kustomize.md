# Kustomize Pipeline

## Planton CLI Commands

```bash
# Register this pipeline
planton tekton pipeline register \
  --yaml-file=kustomize.yaml \
  --name="Build Kustomize Directory" \
  --description="Generate Kubernetes manifests from Kustomize directory for all environments" \
  --git-web-url="https://github.com/plantonhq/tekton-hub/blob/main/pipelines/kustomize.yaml" \
  --git-clone-url="https://github.com/plantonhq/tekton-hub.git" \
  --git-file-path="pipelines/kustomize.yaml" \
  --overview-markdown-file=kustomize.md \
  --tags="pipeline,kustomize,kubernetes,manifests" \
  --platform

# Validate the manifest
planton tekton pipeline validate --yaml-file=kustomize.yaml
```

## Purpose

Focused pipeline that clones a Git repository and generates Kubernetes manifests from Kustomize overlays for all environments. This pipeline is useful for manifest-only operations without container image builds.

## Workflow

1. **git-checkout** - Clone source code from Git repository
2. **kustomize-build** - Generate Kubernetes manifests for all environments

## Parameters

### Git Parameters
- **git-url** (required) - Git repository URL to clone
- **git-revision** (default: "main") - Git revision to checkout
- **git-branch** (default: "") - Git branch name

### Kustomize Parameters
- **kustomize-manifests-config-map-name** (required) - Name of ConfigMap to store service manifests
- **project-root** (default: ".") - Relative path to project root inside cloned repo
- **kustomize-base-directory** (default: "") - Relative path to kustomize base directory inside project root

### Ownership Parameters
- **owner-identifier-label-key** (default: "") - Optional owner identifier label key for ConfigMap
- **owner-identifier-label-value** (default: "") - Optional owner identifier label value

## Workspaces

- **source** (required) - Workspace where source code is cloned

## Task Details

### 1. git-checkout

Uses Tekton Hub's `git-clone` task (version 0.9) to clone the repository.

**Runs**: Always  
**Image**: `ghcr.io/tektoncd/github.com/tektoncd/pipeline/cmd/git-init:v0.45.0`

### 2. kustomize-build

Generates Kubernetes manifests for all environments and stores in a ConfigMap.

**Runs after**: git-checkout  
**Task reference**: git resolver pointing to `tasks/kustomize-build.yaml`  
**Namespace**: Creates ConfigMap in `planton-cloud-pipelines` namespace

## Output

Creates a single ConfigMap containing:
- One key per environment (dev, staging, prod, etc.)
- Base64-encoded Kubernetes manifests as values
- Owner identifier labels if provided

## When to Use

Choose this pipeline for:
- Manifest-only repositories (no application code)
- Validating Kustomize configurations
- Updating deployment manifests without rebuilding images
- Infrastructure-as-code repositories
- GitOps workflows where manifests are versioned separately
- Previewing manifest changes before deployment

## Typical Workflow

1. Developer updates Kustomize overlays in Git
2. Pipeline runs on push/PR
3. Generated manifests stored in ConfigMap
4. Downstream deployment process reads ConfigMap
5. Applies manifests to target clusters

## Comparison with Other Pipelines

Unlike `buildpacks.yaml` or `dockerfile.yaml` pipelines:
- No container image build
- No package credentials workspace
- Simpler and faster execution
- Focused only on manifest generation

## Planton Cloud Usage

This pipeline is commonly used for:
- Standalone Kustomize repositories
- Configuration-only updates
- Testing Kustomize changes
- Generating manifests for review before deployment
- Updating environment-specific configurations
