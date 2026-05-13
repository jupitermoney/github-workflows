# github-workflows

Reusable GitHub Actions workflows and shared CI/CD templates.

This repo is meant to be consumed from other repositories using GitHub reusable workflows. It is a central place to store shared workflow templates, composite actions, and future CI/CD building blocks across stacks.

Today, the main implemented templates are for JVM services:

| Workflow | Purpose |
| --- | --- |
| [`.github/workflows/jvm-pre-merge.yml`](./.github/workflows/jvm-pre-merge.yml) | PR validation |
| [`.github/workflows/jvm-post-merge.yml`](./.github/workflows/jvm-post-merge.yml) | Post-merge build, scan, and ECR publish |

## Repo Scope

It is intended to host reusable templates for other platforms and use cases as well, for example:

- JVM service validation and publish workflows
- language-specific CI templates for other stacks
- deployment or release automation templates
- shared composite actions used by reusable workflows

As new templates are added, they should be documented here and in the folder-level READMEs under [`.github/actions`](./.github/actions/README.md) and [`.github/workflows`](./.github/workflows/README.md).

## Available Guides

- [Reusable workflow catalog](./.github/workflows/README.md)
- [Composite actions guide](./.github/actions/README.md)
- [JVM workflow integration guide](./.github/workflows/JAVA_APP_ECR_PUBLISH_README.md)
- [Pre-merge workflow file](./.github/workflows/jvm-pre-merge.yml)
- [Post-merge workflow file](./.github/workflows/jvm-post-merge.yml)

## Current JVM Templates

The current JVM setup is metadata-driven and built around reusable workflows plus shared composite actions.

For integration details, required inputs, secrets, supported services, and expected project contract, use:

- [JVM workflow integration guide](./.github/workflows/JAVA_APP_ECR_PUBLISH_README.md)
- [Reusable workflow catalog](./.github/workflows/README.md)
- [Composite actions guide](./.github/actions/README.md)
