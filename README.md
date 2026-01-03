# Tekton Hub

Reusable Tekton pipelines and tasks for building and deploying applications on Planton Cloud.

## Purpose

This repository provides production-ready Tekton resources that handle common CI/CD workflows:

- **Container image builds** using Dockerfile, Buildpacks, or BuildKit
- **Kustomize manifest generation** for Kubernetes deployments
- **Cloudflare Worker bundling** and artifact storage

These resources are registered in Planton Cloud and made available through the web console's source code template library.

## Repository Structure

```
tekton-hub/
├── pipelines/       # Complete CI/CD workflows
└── tasks/           # Reusable task components
```

## Pipelines

Pipelines orchestrate multiple tasks to complete full workflows:

- **buildpacks.yaml** - Build images using Cloud Native Buildpacks
- **dockerfile.yaml** - Build images using Dockerfile with BuildKit
- **kustomize.yaml** - Generate Kubernetes manifests from Kustomize directories
- **cloudflare-worker.yaml** - Bundle and upload Cloudflare Workers to R2 storage

## Tasks

Tasks perform specific operations and can be composed into pipelines:

- **buildkit.yaml** - BuildKit daemon-less image builds (privileged)
- **buildkit-root-less.yaml** - BuildKit daemon-less image builds (rootless/unprivileged)
- **buildpacks.yaml** - Standard Cloud Native Buildpacks builds
- **buildpacks-custom.yaml** - Customizable Buildpacks builds with extra bindings
- **buildpacks-phases.yaml** - Buildpacks builds with individual lifecycle phases
- **kustomize-build.yaml** - Build Kustomize overlays and store in ConfigMaps

## Planton Cloud Integration

These resources are registered in Planton Cloud as platform-provided templates. Users can browse and use them through the web console's template library.

All tasks and pipelines follow Planton Cloud conventions:
- Store outputs in ConfigMaps for downstream consumption
- Support owner identifier labels for resource tracking
- Use the `planton-cloud-pipelines` namespace for execution

## Usage

Reference tasks and pipelines using Tekton's git resolver:

```yaml
taskRef:
  resolver: git
  params:
    - name: url
      value: "https://github.com/plantonhq/tekton-hub.git"
    - name: revision
      value: "main"
    - name: pathInRepo
      value: "tasks/buildkit.yaml"
```

Or register them in Planton Cloud for web console access:

```bash
planton tekton task register \
  --yaml-file=tasks/buildkit.yaml \
  --name="BuildKit Daemon-less" \
  --description="Build container images using BuildKit in daemon-less mode" \
  --git-web-url="https://github.com/plantonhq/tekton-hub/blob/main/tasks/buildkit.yaml" \
  --git-clone-url="https://github.com/plantonhq/tekton-hub.git" \
  --git-file-path="tasks/buildkit.yaml" \
  --tags="container-build,buildkit" \
  --platform
```
