# Cloudflare Worker Pipeline

## Planton CLI Commands

```bash
# Register this pipeline
planton tekton pipeline register \
  --yaml-file=cloudflare-worker.yaml \
  --name="Build and Upload Cloudflare Worker" \
  --description="Build Cloudflare Worker bundle and upload to R2 storage with Kustomize manifest generation" \
  --git-web-url="https://github.com/plantoncloud/tekton-hub/blob/main/pipelines/cloudflare-worker.yaml" \
  --git-clone-url="https://github.com/plantoncloud/tekton-hub.git" \
  --git-file-path="pipelines/cloudflare-worker.yaml" \
  --overview-markdown-file=cloudflare-worker.md \
  --tags="pipeline,cloudflare,workers,r2,serverless" \
  --platform

# Validate the manifest
planton tekton pipeline validate --yaml-file=cloudflare-worker.yaml
```

## Purpose

Specialized pipeline for Cloudflare Workers that bundles JavaScript/TypeScript code using Wrangler, uploads the bundle to Cloudflare R2 storage, stores artifact metadata in a ConfigMap, and generates Kustomize manifests for deployment configuration.

## Workflow

1. **git-checkout** - Clone source code from Git repository
2. **install-dependencies** - Install npm/yarn/pnpm dependencies (auto-detected)
3. **bundle-worker** - Bundle Worker code using Wrangler CLI
4. **upload-to-r2** - Upload bundle to Cloudflare R2 storage
5. **store-artifact-metadata** - Store R2 location and bundle metadata in ConfigMap
6. **kustomize-build** - Generate Kubernetes manifests for deployment configuration

## Parameters

### Git Parameters
- **git-url** (required) - Git repository URL to clone
- **git-revision** (default: "main") - Git revision to checkout
- **git-branch** (default: "") - Git branch name
- **project-root** (default: ".") - Relative path to project root inside cloned repo
- **package-manager-directory** (default: "") - Directory to run package manager install from (defaults to project-root if empty, set to "." for monorepo workspaces)
- **sparse-checkout-directories** (default: "") - Comma-separated directory patterns for sparse checkout

### R2 Storage Parameters
- **r2-bucket-name** (required) - Name of R2 bucket to upload worker bundle
- **r2-endpoint-url** (required) - R2 S3-compatible endpoint URL
- **r2-object-key** (required) - Object key/path in R2 bucket (e.g., service-name/commit-sha.js)

### ConfigMap Parameters
- **worker-script-artifact-config-map-name** (required) - Name of ConfigMap to store worker script artifact metadata
- **kustomize-manifests-config-map-name** (required) - Name of ConfigMap to store service manifests
- **kustomize-base-directory** (default: "_kustomize") - Relative path to kustomize base directory inside project root

### Ownership Parameters
- **owner-identifier-label-key** (default: "") - Optional owner identifier label key for ConfigMaps
- **owner-identifier-label-value** (default: "") - Optional owner identifier label value

## Workspaces

- **source** (required) - Workspace where source code is cloned
- **r2-credentials** (required) - Workspace containing R2 access credentials (access-key-id and secret-access-key files)

## Task Details

### 1. git-checkout

Uses Tekton Hub's `git-clone` task (version 0.9) to clone the repository.

**Runs**: Always  
**Image**: `ghcr.io/tektoncd/github.com/tektoncd/pipeline/cmd/git-init:v0.45.0`

### 2. install-dependencies

Auto-detects package manager and installs dependencies.

**Runs after**: git-checkout  
**Image**: node:20-alpine  

**Directory resolution**:
- If `package-manager-directory` is set, installs from that directory
- Otherwise, installs from `project-root`
- This enables monorepo support where `yarn install` runs at workspace root while wrangler builds happen in the worker subdirectory

**Detection logic**:
- Checks for yarn.lock → uses Yarn
- Checks for pnpm-lock.yaml → uses pnpm
- Checks for package-lock.json → uses npm ci
- Checks for package.json → uses npm install
- No package.json → skips installation

### 3. bundle-worker

Bundles the Cloudflare Worker using Wrangler CLI.

**Runs after**: install-dependencies  
**Image**: node:20-alpine

**Operations**:
1. Verifies `wrangler.toml` exists in project root
2. Runs `npx wrangler deploy --dry-run --outdir=dist`
3. Validates `dist/index.js` was created
4. Calculates SHA256 hash of bundle
5. Measures bundle size in bytes

**Results**:
- `bundle-hash`: SHA256 hash of bundled script
- `bundle-size`: Bundle size in bytes

### 4. upload-to-r2

Uploads the bundle to Cloudflare R2 using S3-compatible API.

**Runs after**: bundle-worker  
**Image**: amazon/aws-cli:latest

**Operations**:
1. Reads R2 credentials from r2-credentials workspace
2. Sets AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
3. Uploads `dist/index.js` to R2 bucket at specified object key
4. Constructs artifact URL

**Result**:
- `artifact-url`: Full R2 URL to uploaded bundle

### 5. store-artifact-metadata

Stores artifact location and metadata in a ConfigMap.

**Runs after**: upload-to-r2  
**Image**: bitnami/kubectl:latest

**ConfigMap contents**:
- `bucket`: R2 bucket name
- `path`: Object key in bucket
- `content-hash`: SHA256 hash from bundle step
- `size-bytes`: Bundle size from bundle step

**Labels**:
- `app.kubernetes.io/managed-by=planton-cloud`
- Owner identifier label if provided

### 6. kustomize-build

Generates Kubernetes manifests for deployment configuration.

**Runs after**: git-checkout (parallel with other tasks)  
**Task reference**: git resolver pointing to `tasks/kustomize-build.yaml`  
**Namespace**: Creates ConfigMap in `planton-cloud-pipelines` namespace

## R2 Credentials

The r2-credentials workspace must contain two files:
- `access-key-id` - R2 access key ID
- `secret-access-key` - R2 secret access key

These are read by the upload-to-r2 step and used for S3-compatible authentication.

## Bundle Metadata

The artifact metadata ConfigMap enables:
- Tracking which bundle is deployed
- Content integrity verification via hash
- Bundle size monitoring
- Linking deployments to specific commits

## Parallel Execution

The `kustomize-build` task runs in parallel with the worker build/upload pipeline since they don't depend on each other. This reduces total pipeline execution time.

## Planton Cloud Usage

This pipeline is used for:
- Building Cloudflare Workers from source
- Storing worker bundles in R2 for deployment
- Tracking artifact metadata for deployments
- Managing worker deployment configurations

## When to Use

Choose this pipeline for:
- Cloudflare Workers projects
- Serverless edge compute applications
- Projects with wrangler.toml configuration
- Workers requiring build/bundle step
- TypeScript or modern JavaScript workers
- Multi-environment worker deployments

## Monorepo Support

For monorepos using yarn/npm/pnpm workspaces with `workspace:*` protocol dependencies, set `package-manager-directory` to the workspace root:

**Standalone worker** (default behavior):
```yaml
params:
  - name: project-root
    value: "my-worker"
  # package-manager-directory defaults to project-root
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

This runs `yarn install` at the monorepo root (resolving `workspace:*` dependencies) while wrangler builds from the worker's `project-root`.

## Output Artifacts

After pipeline completion:
1. **R2 Bucket**: Contains worker bundle at specified object key
2. **Artifact ConfigMap**: Contains bundle location and metadata
3. **Manifests ConfigMap**: Contains Kustomize-generated deployment manifests
