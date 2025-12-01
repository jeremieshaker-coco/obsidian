---
tags:
---
# Main Branch Synchronization Policy

It's important to keep the `main` branch in sync with what is running in production.

## `main` as Source of Truth

The `main` branch is considered the source of truth for production. All deployments are made from `main`.

## Handling Rollbacks

If a deployment is rolled back in production, a corresponding revert commit should be made on the `main` branch. This ensures that `main` continues to accurately reflect the state of production. Merging to `main` and deploying are intentionally separate steps.

## Unfamiliar Changes on `main`

If you are about to deploy and notice unfamiliar changes on `main` that are not yet in production, you should not proceed with the deployment. Instead, reach out to the person who merged those changes to understand the context and why they haven't been deployed. Never deploy changes without a full understanding of what they do.
