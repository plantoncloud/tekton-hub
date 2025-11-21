# Kustomize Build Task

## Planton CLI Commands

```bash
# Register this task
planton tekton task register \
  --yaml-file=tasks/kustomize-build.yaml \
  --name="Kustomize Build" \
  --description="Build Kustomize overlays and store manifests in ConfigMap" \
  --git-web-url="https://github.com/plantoncloud/tekton-hub/blob/main/tasks/kustomize-build.yaml" \
  --git-clone-url="https://github.com/plantoncloud/tekton-hub.git" \
  --git-file-path="tasks/kustomize-build.yaml" \
  --overview-markdown-file=tasks/kustomize-build.md \
  --tags="kustomize,kubernetes,manifests" \
  --platform

# Validate the manifest
planton tekton task validate --yaml-file=tasks/kustomize-build.yaml
```

## Purpose

Builds Kubernetes manifests from Kustomize overlays and stores the generated YAML for each environment in a single ConfigMap. Each environment's manifests are base64-encoded and stored as a separate key in the ConfigMap.

## Key Features

- **Multi-environment support** - Processes all overlay directories under `overlays/`
- **Base64 encoding** - Encodes generated manifests for safe ConfigMap storage
- **Single ConfigMap output** - All environments stored in one ConfigMap with environment name as key
- **Owner labeling** - Supports custom labels for resource ownership tracking
- **Graceful fallback** - Creates empty ConfigMap if overlay directory doesn't exist

## Parameters

- **config-map-name** (required) - Name of ConfigMap to create or update
- **config-map-namespace** (required) - Namespace in which to create/update ConfigMap
- **project-root** (default: ".") - Relative path to project root inside cloned repo
- **kustomize-base-directory** (default: "") - Path to kustomize base directory (relative to project root)
- **owner-identifier-label-key** (default: "") - Optional owner identifier label key
- **owner-identifier-label-value** (default: "") - Optional owner identifier label value

## Workspaces

- **source** (required) - Directory where application source is located

## Expected Directory Structure

```
source/
└── <project-root>/
    └── <kustomize-base-directory>/
        └── overlays/
            ├── dev/
            │   └── kustomization.yaml
            ├── staging/
            │   └── kustomization.yaml
            └── prod/
                └── kustomization.yaml
```

## Behavior

### When Overlays Exist

For each environment directory under `overlays/`:
1. Runs `kubectl kustomize overlays/<env>`
2. Base64-encodes the generated YAML
3. Stores in ConfigMap with key `<env>` and value `<base64-encoded-yaml>`

### When Overlays Don't Exist

If the `overlays/` directory is not found:
1. Creates an empty ConfigMap with the specified name
2. Applies owner labels if provided
3. Exits successfully

## ConfigMap Structure

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
  namespace: <config-map-namespace>
  labels:
    <owner-identifier-label-key>: <owner-identifier-label-value>
data:
  dev: <base64-encoded-manifests>
  staging: <base64-encoded-manifests>
  prod: <base64-encoded-manifests>
```

## Step Details

The single step `kustomize-build`:
- Uses `bitnamilegacy/kubectl:latest` image (includes kubectl with kustomize)
- Changes to kustomize base directory
- Iterates over environment overlays
- Builds manifests for each environment
- Creates/updates ConfigMap with all environments
- Applies owner identifier label if provided

## Error Handling

- Fails if kustomize build fails for any environment
- Validates overlay directory exists before processing
- Uses `set -euxo pipefail` for strict error handling

## Planton Cloud Integration

This task is commonly used in Planton Cloud pipelines to:
- Generate deployment manifests for multiple environments in one build
- Store manifests in ConfigMaps for downstream deployment tasks
- Track which pipeline run generated which manifests via owner labels
- Enable manifest inspection in the web console

## Usage in Pipelines

Typically runs after image build steps to generate deployment manifests that reference the newly built image:

```yaml
- name: kustomize-build
  runAfter:
    - build-image
  taskRef:
    resolver: git
    params:
      - name: url
        value: "https://github.com/plantoncloud/tekton-hub.git"
      - name: pathInRepo
        value: "tasks/kustomize-build.yaml"
  params:
    - name: config-map-name
      value: "my-app-manifests"
    - name: project-root
      value: "."
    - name: kustomize-base-directory
      value: "_kustomize"
```

