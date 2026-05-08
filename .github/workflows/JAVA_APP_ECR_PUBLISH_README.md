# Universal ECR Publisher – GitHub Workflows

This repository provides **reusable GitHub Actions workflows** for building, scanning, and publishing Docker images to **AWS ECR** in a standardized and secure way.

## Available Templates

| Template                                | Primary Use Case                                       |
| --------------------------------------- | ------------------------------------------------------ |
| **java-app-ecr-publish.yml**            | Build Java/Gradle services with Jib and publish to ECR |
| (future) node-app-ecr-publish.yml       | Node services                                          |
| (future) generic-docker-ecr-publish.yml | Raw Dockerfile projects                                |

---

# java-app-ecr-publish.yml

### What This Template Does

This workflow is **zero-config for services using the `kotlin-spring-java21` plugin**. It reads project configuration directly from Gradle via `./gradlew generateServiceMetadata`, so there is no need to repeat ECR repository names, Java versions, or API spec paths in the workflow file.

End-to-end pipeline:

1. Checkout code
2. Setup Java 21
3. Compute Docker tag
4. Login to AWS ECR
5. Run `./gradlew generateServiceMetadata` → `build/metadata/service-metadata.json`
6. Build all Jib modules (`./gradlew clean build -x test jibDockerBuild`)
7. Tag each module's image with the computed tag
8. Run Trivy security scan (primary module)
9. Push all module images to ECR
10. Push publishable API specs to API Store (parallel, one job per spec)

---

## Quick Start (Consumer Repo)

Create `.github/workflows/publish.yml`:

```yaml
name: Publish Service Image

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main, hotfix-release** ]

jobs:
  push:
    uses: jupitermoney/github-workflows/.github/workflows/java-app-ecr-publish.yml@main
    secrets: inherit
```

That's it. ECR repo names, Java version, and API spec paths all come from `build.gradle.kts` via the plugin — no `with:` block needed.

---

## How Metadata Drives CI

The workflow runs `./gradlew generateServiceMetadata` before building. This task (provided by `kotlin-spring-java21`) writes `build/metadata/service-metadata.json` containing:

| Metadata field | Used by CI for |
|---|---|
| `jibModules[].toImage` | ECR repository name per module |
| `jibModules[].javaVersion` | Base image version (informational) |
| `openApiSpecs[].specFile` where `publishClients == true` | API Store push targets |

Multi-module projects (e.g., a monorepo with 4 Spring Boot services) are handled automatically — each module gets tagged and pushed in a sequential loop within a single job.

---

## Inputs

### Build

| Input         | Default                                        | Description |
| ------------- | ---------------------------------------------- | ----------- |
| build_command | `./gradlew clean build -x test jibDockerBuild` | Full build command |
| enable_build  | true                                           | Skip build step if false |

### Tagging

| Input          | Default                            | Description |
| -------------- | ---------------------------------- | ----------- |
| tag_prefix     | rc                                 | Prefix for computed tag |
| tag_calculator | script generating DOCKER_IMAGE_TAG | Custom tag strategy (see below) |

### Security

| Input                | Default                          | Description |
| -------------------- | -------------------------------- | ----------- |
| approval_service_url | https://trivy.audit.jupiter.money | Trivy approval endpoint |

### Docker / Push

| Input       | Default | Description |
| ----------- | ------- | ----------- |
| enable_push | true    | Skip ECR push if false |

### API Store

| Input            | Default | Description |
| ---------------- | ------- | ----------- |
| enable_api_store | true    | Set false to skip API Store entirely |

### Hooks

| Input              | Description              |
| ------------------ | ------------------------ |
| pre_build_command  | Shell before build       |
| post_build_command | Shell after build        |
| extra_env          | Additional env variables |

### Infrastructure

| Input      | Default    | Description |
| ---------- | ---------- | ----------- |
| aws_region | ap-south-1 | AWS region  |

---

## Secrets Contract (Optional but recommended)

* `AWS_ACCESS_KEY_ID`
* `AWS_ACCESS_KEY_SECRET`
* `APPROVAL_SERVICE_TOKEN`
* `CI_GITHUB_USER`
* `CI_GITHUB_TOKEN`
* `ARTIFACTORY_USER`
* `ARTIFACTORY_PASSWORD`
* `API_STORE_CLIENT_ID`
* `API_STORE_CLIENT_SECRET`

---

## Environment Variables Available by Default

Every run automatically exposes:

* `ECR_REGISTRY`
* `APPROVAL_SERVICE_URL`
* `APPROVAL_SERVICE_TOKEN`
* `CI_COMMIT_SHA`
* `GITHUB_USER`
* `GITHUB_TOKEN`
* `ARTIFACTORY_USER`
* `ARTIFACTORY_PASSWORD`

---

## Common Use Cases

### 1. Push to Multiple ECR Repositories (Multi-Tenant)

Services that need to push the same image to multiple ECR repositories (e.g., one per tenant or team) can use the `ecr_targets` input instead of `ecr_registry`.

```yaml
jobs:
  publish:
    uses: jupitermoney/github-workflows/.github/workflows/jvm-post-merge.yml@main
    with:
      java_version: "21"
      ecr_targets: |
        [
          {"registry": "454518750364.dkr.ecr.ap-south-1.amazonaws.com", "repository": "nexus",                "tag_prefix": "rc"},
          {"registry": "454518750364.dkr.ecr.ap-south-1.amazonaws.com", "repository": "ds-jm-nexus-service", "tag_prefix": "master"},
          {"registry": "454518750364.dkr.ecr.ap-south-1.amazonaws.com", "repository": "cstech-nexus",        "tag_prefix": "rc"},
          {"registry": "454518750364.dkr.ecr.ap-south-1.amazonaws.com", "repository": "investment-nexus",    "tag_prefix": "rc"}
        ]
    secrets: inherit
```

Each entry in `ecr_targets` produces one build-push job. The image is built once per module, then each target gets its own tag (using the entry's `tag_prefix`) and push. Targets can span different AWS accounts by specifying different `registry` values.

When `ecr_targets` is omitted, behaviour is unchanged — the single `ecr_registry` input is used with a `rc` tag prefix.

### 2. Custom Build Command

```yaml
with:
  build_command: "./gradlew clean bootJar jibDockerBuild"
```

### 2. Custom Tag Strategy

```yaml
with:
  tag_calculator: |
    if [[ "$GITHUB_EVENT_NAME" == "pull_request" ]]; then
      TAG="pr-${GITHUB_SHA:0:8}"
    else
      TAG="main-${GITHUB_SHA:0:8}"
    fi
    echo "DOCKER_IMAGE_TAG=$TAG" >> $GITHUB_ENV
```

### 3. Pass Extra Env

```yaml
with:
  extra_env: |
    FEATURE_FLAG=true
    LOG_LEVEL=debug
```

### 4. Skip API Store

```yaml
with:
  enable_api_store: false
```

---

## Expected Image Tags

Default format:

```
rc-<short-sha>
```

Example:

```
rc-a1b2c3d4
```

---

## Plugin Requirement

Your service must use the `kotlin-spring-java21` Gradle plugin (v0.3.4+). The plugin provides the `generateServiceMetadata` task that produces `build/metadata/service-metadata.json`.

Your `build.gradle.kts` doesn't change — the metadata is collected automatically from config you've already written.

---

## Migrating Existing Services

1. Bump plugin to `0.3.4+` in `settings.gradle.kts`
2. Remove the `with:` block from your workflow call (or strip `ecr_repository`, `java_version`, `api_spec_path` from it)
3. Done — CI reads everything from Gradle config on next push

---

## Known Limitations (POC)

**Trivy multi-image scanning**: Trivy currently runs only on the first jib module (primary image). For monorepos with multiple modules, secondary images are pushed without individual Trivy scans. Full multi-image support requires updating `jupitermoney/security-automations` to accept an image array, or restructuring into a matrix job.

---

## Troubleshooting

### ❌ build/metadata/service-metadata.json not found

The `generateServiceMetadata` Gradle task failed. Check that:
* The `kotlin-spring-java21` plugin is applied
* `./gradlew generateServiceMetadata` succeeds locally

### ❌ jibModules array is empty

No Jib modules are configured in `build.gradle.kts`. Add at least one module using `kotlinSpringJib { }`.

### ❌ Tag calculator must export DOCKER_IMAGE_TAG

Your custom script must end with:

```bash
echo "DOCKER_IMAGE_TAG=value" >> $GITHUB_ENV
```

### ❌ Artifactory Auth Failures

Ensure secrets exist in repo:

* ARTIFACTORY_USER
* ARTIFACTORY_PASSWORD

### ❌ Trivy Blocked

Add approval token:

* APPROVAL_SERVICE_TOKEN

---

## Ownership

Platform Engineering – CI/CD

For changes contact:
#team-platform
[ci-cd@jupiter.money](mailto:ci-cd@jupiter.money)

---
