# Reusable Workflows

This folder contains the reusable GitHub Actions workflows that consumer repositories import from `jupitermoney/github-workflows`.

## Public Entry Points

| Workflow | What it does |
| --- | --- |
| `jvm-pre-merge.yml` | Pull request validation workflow for JVM services, with optional PR image publish |
| `jvm-post-merge.yml` | Post-merge workflow that builds, scans, and publishes Jib images to ECR |

## Other Files

| File | Purpose |
| --- | --- |
| `semgrep.yml` | Repository-local workflow for Semgrep |
| `JAVA_APP_ECR_PUBLISH_README.md` | Integration guide for consumers of the reusable JVM workflows |

## How Consumer Repos Use These Workflows

A consumer repository should call these workflows using `uses:`.

Example PR workflow:

```yaml
jobs:
  check:
    uses: jupitermoney/github-workflows/.github/workflows/jvm-pre-merge.yml@main
    with:
      java_version: "21"
      publish: true
      services: "postgres"
    secrets: inherit
```

Example post-merge workflow:

```yaml
jobs:
  publish:
    uses: jupitermoney/github-workflows/.github/workflows/jvm-post-merge.yml@main
    with:
      java_version: "21"
    secrets: inherit
```

## How To Update

When updating a reusable workflow in this folder:

1. treat it as a shared platform contract for all consumer repositories
2. prefer additive changes over breaking input or secret contract changes
3. keep the workflow aligned with the composite actions in [`.github/actions`](../actions/README.md)
4. update consumer-facing docs if usage changes

## Update Checklist

- Confirm the workflow filename stays stable if consumers already reference it.
- Confirm declared `workflow_call.inputs` still match documented usage.
- Confirm required secrets are still available through `secrets: inherit`.
- Confirm generated metadata outputs still match what matrix jobs expect.
- Confirm branch-specific behavior like `pr`, `rc`, and `hotfix` tagging still works.
- Update examples in [README.md](../../README.md) if the recommended integration pattern changes.
- Update [JAVA_APP_ECR_PUBLISH_README.md](./JAVA_APP_ECR_PUBLISH_README.md) if setup steps or expectations change.

## Breaking Change Guidance

These workflows are imported remotely by other repositories. A breaking change here can fail CI across multiple services.

Be careful when changing:

- workflow filenames
- `workflow_call` input names or defaults
- required secrets
- metadata expectations from `generateServiceMetadata`
- image tagging or publish behavior

If a breaking change is necessary, document it clearly in the root README and the integration guide.

## Adding A New Reusable Workflow

If you add a new shared workflow:

1. define it under this folder with `on: workflow_call`
2. keep the interface small and documented
3. reuse composite actions from `.github/actions` where possible
4. add it to this README and the root [README.md](../../README.md)
