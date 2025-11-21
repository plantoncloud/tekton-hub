# Buildpacks Phases Task

## Planton CLI Commands

```bash
# Register this task
planton tekton task register \
  --yaml-file=buildpacks-phases.yaml \
  --name="Buildpacks Phases" \
  --description="Build images using individual CNB lifecycle phases for debugging and customization" \
  --git-web-url="https://github.com/plantoncloud/tekton-hub/blob/main/tasks/buildpacks-phases.yaml" \
  --git-clone-url="https://github.com/plantoncloud/tekton-hub.git" \
  --git-file-path="tasks/buildpacks-phases.yaml" \
  --overview-markdown-file=buildpacks-phases.md \
  --tags="container-build,buildpacks,cnb,lifecycle,debugging" \
  --platform

# Validate the manifest
planton tekton task validate --yaml-file=buildpacks-phases.yaml
```

## Purpose

Executes Cloud Native Buildpacks lifecycle as individual phases rather than using the all-in-one creator. This approach provides greater visibility into each lifecycle step and enables debugging, customization, and intermediate testing.

## Key Features

- **Phase-by-phase execution** - Runs detect, analyze, restore, build, and export as separate steps
- **Intermediate debugging** - Includes steps to dump environment variables and test builds
- **Gradle integration testing** - Contains a test-gradle step to verify credentials before buildpack execution
- **Lifecycle 0.20.x compatibility** - Uses buildpacksio/lifecycle:0.20.6 for analyze/restore/export phases
- **Java version control** - Configurable JVM version via BP_JVM_VERSION parameter

## Lifecycle API

Uses Cloud Native Buildpacks Platform API version 0.7, compatible with Lifecycle v0.20.x.

## Individual Phases

1. **detect** - Examines source code and selects applicable buildpacks
2. **analyze** - Analyzes the previous image for layer reuse
3. **restore** - Restores cached layers from previous builds
4. **build** - Executes buildpack build logic to create application layers
5. **export** - Assembles layers into final container image and pushes to registry

## Parameters

- **BUILDER_IMAGE** (required) - Builder image with lifecycle and buildpacks
- **IMAGE** (required) - Final image URL where build will be pushed
- **CACHE** (default: "empty-dir") - Name of persistent app cache volume
- **PLATFORM_DIR** (default: "empty-dir") - Name of platform directory volume
- **USER_ID** (default: "1000") - User ID of builder image user
- **GROUP_ID** (default: "1000") - Group ID of builder image user
- **PROCESS_TYPE** (default: "web") - Default process type for final image
- **SOURCE_SUBPATH** (default: "") - Subdirectory within source workspace
- **USER_HOME** (default: "/tekton/home") - Absolute path to user home directory
- **BP_JVM_VERSION** (default: "17") - Java version to install (8, 11, 17, 21)

## Workspaces

- **source** (required) - Source code for the application

## Steps

1. **prepare** - Sets ownership and permissions on workspaces (runs as privileged)
2. **copy-stack-toml** - Copies stack configuration from builder image to layers directory
3. **detect** - Runs lifecycle detector to identify buildpacks
4. **analyze** - Runs lifecycle analyzer to determine layer reuse
5. **restore** - Runs lifecycle restorer to retrieve cached layers
6. **dump-env** - Prints all environment variables for debugging
7. **test-gradle** - Verifies Gradle build with credentials (uses gradle:8.10.2-jdk21 image)
8. **build** - Runs lifecycle builder to execute buildpack logic
9. **export** - Runs lifecycle exporter to create and push final image

## Debugging Features

### Environment Dump

The `dump-env` step outputs all environment variables and verifies Google Cloud credentials:

```bash
echo "Dumping environment variables..."
env
echo "checking gac"
cat $GOOGLE_APPLICATION_CREDENTIALS
```

### Gradle Test

The `test-gradle` step uses the official Gradle Docker image to:
- List source workspace contents
- Verify credential file locations
- Attempt a Gradle assemble build before buildpacks run

This validates that dependencies and credentials are properly configured.

## Google Cloud Integration

Configured to use Google service account credentials at `/workspace/source/.package-credentials/google-service-account.json` for:
- Private artifact repositories (Artifact Registry, Cloud Storage)
- Build-time dependency resolution

## When to Use

Choose this task when:
- Debugging buildpack detection or build failures
- Customizing intermediate phases of the lifecycle
- Testing credential configuration before full builds
- Understanding how buildpacks process your source code
- Developing custom buildpack configurations

## Differences from Standard Buildpacks Task

- Runs phases individually instead of using the creator
- Includes debugging and testing steps
- Uses specific lifecycle image versions for reproducibility
- Provides visibility into intermediate layer and metadata files
- Enables inspection between phases
