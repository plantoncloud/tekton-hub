# Cloudflare Worker Pipeline

**Pipeline**: `build-and-upload-cloudflare-worker`  
**Namespace**: `planton-cloud-pipelines`

## Overview

This Tekton pipeline builds Cloudflare Worker bundles and uploads them to R2 storage. It supports automatic package manager detection (npm/yarn/pnpm), bundles workers using Wrangler, and stores artifact metadata in Kubernetes ConfigMaps for downstream deployment processes.

## Pipeline Stages

### 1. Git Checkout
- Clones source repository
- Supports sparse checkout for monorepos
- Uses Tekton Hub's `git-clone` task (v0.9)

### 2. Install Dependencies
- Auto-detects package manager:
  - **Yarn**: If `yarn.lock` exists
  - **pnpm**: If `pnpm-lock.yaml` exists
  - **npm**: If `package-lock.json` or `package.json` exists
- Runs frozen lockfile install for reproducible builds
- Uses Node.js 20 Alpine image

### 3. Bundle Worker
- Validates `wrangler.toml` exists
- Bundles worker using `npx wrangler deploy --dry-run --outdir=dist`
- Calculates bundle SHA256 hash
- Records bundle size in bytes
- Outputs `dist/index.js` bundle

### 4. Upload to R2
- Uploads bundle to Cloudflare R2 using AWS S3-compatible API
- Uses credentials from `r2-credentials` secret
- Supports custom R2 endpoint URLs
- Returns full artifact URL

### 5. Store Artifact Metadata
- Creates Kubernetes ConfigMap with:
  - `bucket`: R2 bucket name
  - `path`: R2 object key
  - `content-hash`: SHA256 hash
  - `size-bytes`: Bundle size
- Labels with owner identifier
- Used by pipeline customizers to inject bundle reference

### 6. Kustomize Build
- Processes `_kustomize/` directory
- Generates deployment manifests
- Stores in ConfigMap for deployment stage
- Runs in parallel with bundle/upload

## Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `git-url` | string | Git repository URL | *required* |
| `git-revision` | string | Git branch/tag/commit | `main` |
| `project-root` | string | Project root path in repo | `.` |
| `sparse-checkout-directories` | string | Comma-separated paths for sparse checkout | `""` |
| `r2-bucket-name` | string | R2 bucket name | *required* |
| `r2-endpoint-url` | string | R2 S3-compatible endpoint | *required* |
| `r2-object-key` | string | Object path in R2 (e.g., `service/commit.js`) | *required* |
| `worker-script-artifact-config-map-name` | string | ConfigMap name for artifact metadata | *required* |
| `kustomize-manifests-config-map-name` | string | ConfigMap name for kustomize output | *required* |
| `kustomize-base-directory` | string | Kustomize base directory | `_kustomize` |
| `owner-identifier-label-key` | string | Optional label key for ownership | `""` |
| `owner-identifier-label-value` | string | Optional label value for ownership | `""` |

## Workspaces

| Workspace | Description |
|-----------|-------------|
| `source` | Cloned source code |
| `r2-credentials` | R2 access credentials (must contain `r2-credentials` secret) |

## Required Secret

**Name**: `r2-credentials`  
**Keys**:
- `access-key-id`: R2 S3-compatible access key ID
- `secret-access-key`: R2 S3-compatible secret access key

**Example**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: r2-credentials
  namespace: planton-cloud-pipelines
type: Opaque
stringData:
  access-key-id: "<R2_ACCESS_KEY_ID>"
  secret-access-key: "<R2_SECRET_ACCESS_KEY>"
```

## Usage Example

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: build-my-worker
  namespace: planton-cloud-pipelines
spec:
  pipelineRef:
    name: build-and-upload-cloudflare-worker
  params:
    - name: git-url
      value: "https://github.com/org/repo.git"
    - name: git-revision
      value: "main"
    - name: project-root
      value: "backend/services/my-worker"
    - name: r2-bucket-name
      value: "planton-cloudflare-worker-scripts"
    - name: r2-endpoint-url
      value: "https://074755a78d8e8f77c119a90a125e8a06.r2.cloudflarestorage.com"
    - name: r2-object-key
      value: "my-worker/abc123def456.js"
    - name: worker-script-artifact-config-map-name
      value: "my-worker-artifact-metadata"
    - name: kustomize-manifests-config-map-name
      value: "my-worker-manifests"
    - name: owner-identifier-label-key
      value: "planton.cloud/service-id"
    - name: owner-identifier-label-value
      value: "svc-abc123"
  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
    - name: r2-credentials
      secret:
        secretName: r2-credentials
```

## Output Artifacts

### 1. Worker Bundle
- **Location**: R2 bucket at specified path
- **Format**: Single JavaScript file (ES module)
- **Naming**: Typically `{service-name}/{commit-sha}.js`

### 2. Artifact Metadata ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-worker-artifact-metadata
  namespace: planton-cloud-pipelines
  labels:
    app.kubernetes.io/managed-by: planton-cloud
    planton.cloud/service-id: svc-abc123
data:
  bucket: planton-cloudflare-worker-scripts
  path: my-worker/abc123def456.js
  content-hash: 9a3b2c1d...
  size-bytes: "245678"
```

### 3. Kustomize Manifests ConfigMap
Contains deployment manifests from `_kustomize/` directory.

## Project Structure Requirements

```
project-root/
├── wrangler.toml          # Required: Wrangler configuration
├── package.json           # Required: Dependencies
├── package-lock.json      # Optional: npm lockfile
├── yarn.lock              # Optional: yarn lockfile
├── pnpm-lock.yaml         # Optional: pnpm lockfile
├── src/
│   └── index.ts          # Worker source code
└── _kustomize/
    ├── base/
    │   ├── kustomization.yaml
    │   └── service.yaml  # CloudflareWorker manifest
    └── overlays/
        ├── dev/
        ├── staging/
        └── prod/
```

## Wrangler Configuration

**Minimum `wrangler.toml`**:
```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2024-09-23"

[build]
command = ""
```

The pipeline uses `wrangler deploy --dry-run` which bundles the worker without deploying it.

## Error Handling

### Missing wrangler.toml
```
ERROR: wrangler.toml not found in project root
```
**Solution**: Add wrangler.toml to project root

### Bundle not created
```
ERROR: Bundle not created at dist/index.js
```
**Solution**: Check wrangler configuration and build errors

### R2 upload failed
```
upload: failed: s3://bucket/key Connection refused
```
**Solution**: Verify R2 endpoint URL and credentials

## Performance

**Typical execution times**:
- Git checkout: 5-10 seconds
- Install dependencies: 20-60 seconds (cached: 5-10 seconds)
- Bundle worker: 5-15 seconds
- Upload to R2: 1-5 seconds
- Store metadata: 1-2 seconds
- **Total**: ~1-2 minutes

## Integration with Planton Cloud

This pipeline is invoked by the ServiceHub build workflow when:
1. `Service.spec.package_type = CLOUDFLARE_WORKER_SCRIPT`
2. Git commit is pushed to service repository
3. Pipeline workflow creates PipelineRun with this pipeline

**Downstream integration**:
1. CloudflareWorkerPipelineCustomizer reads artifact metadata ConfigMap
2. Injects bundle reference into CloudflareWorker manifest
3. Pulumi downloads bundle from R2 and deploys to Cloudflare

## Comparison with Container Pipelines

| Aspect | Container (Docker/Buildpacks) | Cloudflare Worker |
|--------|------------------------------|-------------------|
| **Build Output** | Container image | JavaScript bundle |
| **Artifact Storage** | Container registry | R2 bucket |
| **Build Tool** | Kaniko/Buildpacks | Wrangler (esbuild) |
| **Upload Tool** | docker push | aws-cli (S3 API) |
| **Bundle Size** | 50MB - 2GB | 100KB - 10MB |
| **Build Time** | 2-10 minutes | 30-90 seconds |

## Troubleshooting

### Dependencies not installing
**Check**: Lockfile exists and matches package.json

### Bundle too large
**Check**: Run `npx wrangler deploy --dry-run` locally to see bundle size

### R2 credentials invalid
**Check**: Verify secret exists and has correct keys

### ConfigMap not created
**Check**: ServiceAccount has permission to create ConfigMaps in namespace

## Future Enhancements

- [ ] Add source map generation
- [ ] Support multiple worker scripts per service
- [ ] Add bundle size validation (fail if >10MB)
- [ ] Cache node_modules between builds
- [ ] Add TypeScript type checking step
- [ ] Support Durable Objects bundling
- [ ] Add bundle compression metrics
- [ ] Support custom wrangler commands

## References

- [Tekton Pipelines Documentation](https://tekton.dev/docs/pipelines/)
- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Wrangler CLI Documentation](https://developers.cloudflare.com/workers/wrangler/)
- [R2 S3 API Documentation](https://developers.cloudflare.com/r2/api/s3/)

