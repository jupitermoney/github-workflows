# JVM Reusable Workflow Integration Guide

This guide explains how to integrate the reusable JVM workflows from `jupitermoney/github-workflows` into a consumer repository.

## Workflows to Use

| Workflow | When to use it |
| --- | --- |
| `jvm-pre-merge.yml` | Validate pull requests, and optionally publish PR images |
| `jvm-post-merge.yml` | Build and publish images after merge to `master` or `hotfix-*` |

## Recommended Consumer Setup

### PR workflow

Create `.github/workflows/pr-check.yml` in your repository:

```yaml
name: PR Check

on:
  pull_request:
    branches:
      - master
      - hotfix-**

jobs:
  check:
    uses: jupitermoney/github-workflows/.github/workflows/jvm-pre-merge.yml@main
    with:
      java_version: "21"
      publish: true
      services: "postgres"
    secrets: inherit
```

### Post-merge workflow

Create `.github/workflows/publish-image.yml` in your repository:

```yaml
name: Publish image

on:
  push:
    branches:
      - master
      - hotfix-**

jobs:
  publish:
    uses: jupitermoney/github-workflows/.github/workflows/jvm-post-merge.yml@main
    with:
      java_version: "21"
    secrets: inherit
```

## How Integration Works

Both reusable workflows expect the consumer repository to generate project metadata by running:

```bash
./gradlew generateServiceMetadata
```

That task must produce:

```text
build/metadata/service-metadata.json
```

The workflows read that file to determine:

- the Jib modules to build
- the ECR image names for those modules
- whether Sonar is enabled

## Expected Metadata Shape

At minimum, the metadata should include:

```json
{
  "jibModules": [
    {
      "module": ":services:sample:server",
      "toImage": "sample-service"
    }
  ],
  "sonar": {
    "enabled": true
  }
}
```

If `jibModules` is empty, publish jobs will fail because the build matrix cannot be created.

## Workflow Behavior

### `jvm-pre-merge.yml`

This workflow:

1. optionally starts local dependencies from the `services` input
2. sets up checkout, Java, and Gradle cache
3. runs `generateServiceMetadata`
4. runs `./gradlew build`
5. optionally builds and pushes PR-tagged images when `publish: true`

Use `publish: true` only if you want PR builds to publish to the staging ECR registry.

### `jvm-post-merge.yml`

This workflow:

1. optionally starts local dependencies from the `services` input
2. sets up checkout, Java, and Gradle cache
3. runs `generateServiceMetadata`
4. runs `./gradlew build` on push events
5. runs module-wise image build and push using a matrix
6. runs Trivy scan before each push

## Inputs

### `jvm-pre-merge.yml`

| Input | Default | Required | Notes |
| --- | --- | --- | --- |
| `java_version` | `"21"` | Yes | Java runtime used by CI |
| `publish` | `false` | No | Enables PR image publish |
| `services` | `""` | No | Comma-separated services, for example `postgres,minio` |
| `ecr_registry` | `454518750364.dkr.ecr.ap-south-1.amazonaws.com` | No | Used only when `publish: true` |

### `jvm-post-merge.yml`

| Input | Default | Required | Notes |
| --- | --- | --- | --- |
| `java_version` | `"21"` | Yes | Java runtime used by CI |
| `services` | `""` | No | Comma-separated services, for example `postgres,minio` |
| `ecr_registry` | `454518750364.dkr.ecr.ap-south-1.amazonaws.com` | No | Target ECR registry |

## Supported Service Containers

The optional `services` input currently supports:

- `postgres`
- `mysql`
- `minio`

Examples:

```yaml
with:
  services: "postgres"
```

```yaml
with:
  services: "postgres,minio"
```

## Secrets Contract

Use `secrets: inherit` in the calling workflow and make sure the consumer repository or org provides:

- `CI_GITHUB_USER`
- `CI_GITHUB_TOKEN`
- `CI_GITHUB_TOKEN_METADATA_REPO`
- `ARTIFACTORY_USER`
- `ARTIFACTORY_PASSWORD`
- `SONARQUBE_TOKEN`

For publish flows, also provide:

- `AWS_ACCESS_KEY_ID`
- `AWS_ACCESS_KEY_SECRET`
- `APPROVAL_SERVICE_TOKEN`

## Image Tag Format

The ECR build/push action computes tags automatically:

- pull request builds use `pr-<short-sha>`
- pushes to `hotfix-*` use `hotfix-<short-sha>`
- other pushes use `rc-<short-sha>`

Examples:

- `pr-a1b2c3d4`
- `hotfix-a1b2c3d4`
- `rc-a1b2c3d4`

## Troubleshooting

### `build/metadata/service-metadata.json not found`

`generateServiceMetadata` did not produce the expected file. Verify that:

- `./gradlew generateServiceMetadata` succeeds locally
- the metadata file is written to `build/metadata/service-metadata.json`

### `jibModules array is empty`

The workflow could not find publishable modules. Verify that your Gradle metadata includes at least one `jibModules` entry.

### PR publish job fails on AWS or Trivy steps

Check that these secrets are available to the consumer repo:

- `AWS_ACCESS_KEY_ID`
- `AWS_ACCESS_KEY_SECRET`
- `APPROVAL_SERVICE_TOKEN`

## Summary

To integrate this template in your repository:

1. add a PR workflow that calls `jvm-pre-merge.yml`
2. add a push workflow that calls `jvm-post-merge.yml`
3. ensure `generateServiceMetadata` produces valid metadata
4. inherit the required secrets

Once those are in place, the shared workflows will handle testing, matrix generation, image tagging, scanning, and ECR publish centrally.
