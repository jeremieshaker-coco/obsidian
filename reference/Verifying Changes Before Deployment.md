---
tags:
---
# Verifying Changes Before Deployment

To ensure that no unintended changes are introduced, follow this verification process:

1.  **Get Last Deployed Commit**: Find the commit SHA of the most recent deployment from AWS CodeBuild.
2.  **Compare with `main`**: Use GitHub's compare feature to see the difference between the last deployed commit and the current tip of the `main` branch.
3.  **Review the Diff**: Carefully review the diff to confirm that only the intended changes are included.

This process helps prevent accidental deployments of unrelated or unverified changes that may be on the `main` branch.
