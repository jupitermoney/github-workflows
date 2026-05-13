# GitHub Composite Actions

This folder contains shared composite GitHub Actions used by the reusable workflows in `.github/workflows`.

## What Lives Here

| Action | What it does |
| --- | --- |
| `gradle-setup` | Checks out code, sets up Java, and restores Gradle cache |
| `generate-metadata` | Runs `./gradlew generateServiceMetadata` and extracts workflow outputs from `build/metadata/service-metadata.json` |
| `ecr-build-push` | Builds one Jib module, computes image tags, runs Trivy, and pushes to ECR |
| `setup-services` | Starts optional local dependencies like `postgres`, `mysql`, and `minio` |

## How These Actions Are Used

These actions are internal building blocks for the reusable workflows:

- jvm-pre-merge.yml
- jvm-post-merge.yml

Consumer repositories should usually call the reusable workflows, not these actions directly.

## How To Update

When updating an action in this folder:

1. identify which reusable workflow depends on it
2. keep inputs and outputs backward compatible unless you are intentionally making a breaking change
3. update the workflow docs if the behavior, inputs, outputs, or required secrets change
4. verify references in `.github/workflows` still match the action contract

## Update Checklist

- If you change `generate-metadata`, confirm the `matrix` and `sonar_enabled` outputs still match what the workflows expect.
- If you change `ecr-build-push`, confirm image tagging, AWS auth, Trivy scan, and push behavior still work for both PR and post-merge flows.
- If you change `setup-services`, update supported service names in the docs.
- If you change `gradle-setup`, confirm checkout depth and Gradle cache behavior still fit Sonar and build requirements.

## Breaking Change Guidance

Changes here can affect many consumer repositories at once because workflows reference these actions from `@main`.

Before merging behavior changes:

- review impact on PR flow and post-merge flow
- update README.md if consumer usage changes
- update JAVA_APP_ECR_PUBLISH_README.md if integration expectations change

## Adding A New Action

If you add a new action:

1. create a new folder with an `action.yml`
2. keep the action focused on one concern
3. wire it into a reusable workflow only if the behavior is shared across repositories
4. document the new action in this file
