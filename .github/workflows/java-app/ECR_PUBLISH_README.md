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

This workflow provides an end-to-end pipeline:

1. Checkout code
2. Build Java application (Gradle + Jib by default)
3. Compute Docker tag
4. Login to AWS ECR
5. Build & tag image
6. Run Trivy security scan
7. Push to ECR
8. Optionally push API spec to API Store

---

## Quick Start (Consumer Repo)

Create:

```
.github/workflows/publish.yml
```

```yaml
name: Publish Service Image

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main, hotfix-release** ]

jobs:
  push:
    uses: jupitermoney/github-workflows/.github/workflows/java-app/ecr-publish.yml@main

    with:
      ecr_repository: your-service-name

    secrets: inherit
```

That’s it 🚀

---

## Inputs

### Core

| Input          | Required | Default    | Description            |
| -------------- | -------- | ---------- | ---------------------- |
| ecr_repository | ✅        | –          | Name of ECR repository |
| aws_region     | ❌        | ap-south-1 | AWS region             |
| java_version   | ❌        | 11         | Java version           |

### Build

| Input         | Default                                        |
| ------------- | ---------------------------------------------- |
| build_command | `./gradlew clean build -x test jibDockerBuild` |
| enable_build  | true                                           |

### Tagging

| Input          | Default                            |
| -------------- | ---------------------------------- |
| tag_prefix     | rc                                 |
| tag_calculator | script generating DOCKER_IMAGE_TAG |

### Security

| Input                | Default                                                                |
| -------------------- | ---------------------------------------------------------------------- |
| approval_service_url | [https://trivy.audit.jupiter.money](https://trivy.audit.jupiter.money) |

### API Store

| Input            | Default        |
| ---------------- | -------------- |
| enable_api_store | true           |
| api_spec_path    | ./api/spec.yml |

### Hooks

| Input              | Description              |
| ------------------ | ------------------------ |
| pre_build_command  | Shell before build       |
| post_build_command | Shell after build        |
| extra_env          | Additional env variables |

---

## Secrets Contract(Optional but recommended)

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
* `ECR_REPOSITORY`
* `APPROVAL_SERVICE_URL`
* `APPROVAL_SERVICE_TOKEN`
* `CI_COMMIT_SHA`
* `GITHUB_USER`
* `GITHUB_TOKEN`
* `ARTIFACTORY_USER`
* `ARTIFACTORY_PASSWORD`

These can be consumed by:

* Gradle
* Jib authentication
* Trivy approval
* Custom scripts

---

## Common Use Cases

### 1. Custom Build Command

```yaml
with:
  ecr_repository: payments
  build_command: "./gradlew clean bootJar jibDockerBuild"
```

---

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

---

### 3. Pass Extra Env

```yaml
with:
  extra_env: |
    FEATURE_FLAG=true
    LOG_LEVEL=debug
```

---

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

## Local Compatibility

Your service should:

* Use Gradle + Jib
* Produce image:

```
$ECR_REGISTRY/$ECR_REPOSITORY:latest
```

before retag step.

---

## Troubleshooting

### ❌ Tag calculator must export DOCKER_IMAGE_TAG

Your custom script must end with:

```bash
echo "DOCKER_IMAGE_TAG=value" >> $GITHUB_ENV
```

---

### ❌ Artifactory Auth Failures

Ensure secrets exist in repo:

* ARTIFACTORY_USER
* ARTIFACTORY_PASSWORD

---

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
