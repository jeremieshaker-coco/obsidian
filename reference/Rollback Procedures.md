---
tags:
---
# Rollback Procedures

If a deployment goes wrong, here are the procedures for rolling back:

## Using `kubectl`

The preferred method for rollbacks is using `kubectl`.

- To see the history of deployments:
  ```bash
  kubectl rollout history deployment/<service-name> -n <namespace>
  ```
  Example: `kubectl rollout history deployment/trip-monitor -n karpenter`

- To undo a deployment and revert to the previous version:
  ```bash
  kubectl rollout undo deployment/<service-name> -n <namespace>
  ```

## Using `helm`

Helm can also be used to get better visibility into revisions, which is especially helpful if a deployment stalls.

- To see the history of a Helm release:
  ```bash
  helm history <release-name> -n <namespace>
  ```
  Example: `helm history trip-monitor -n karpenter`

## Using AWS CodeBuild

As an alternative, you can trigger a new deployment in AWS CodeBuild using the SHA of a previous, stable commit.

## Notion Documentation

For more detailed instructions, refer to the Notion document on rollbacks:
[Developer Workflow - Rollbacks](https://www.notion.so/Developer-Workflow-1e686fd0dcab8063b899d847a6169285?source=copy_link#1e686fd0dcab80cbb345e6342bb9ce7c)
