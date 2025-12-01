---
tags:
---
# Deployment Process

This note outlines the general process for deploying services.

While this document refers to AWS CodeDeploy, the current automated deployment system for services in [[coco-services]] and [[delivery-platform]] uses [[AWS CodeBuild]] to orchestrate deployments via Helm to a Kubernetes cluster. Manual deployments via the AWS CodeDeploy UI may be a legacy process or used for different types of services (e.g., [[coco-acu]]).

## Process

1.  **Assess Impact**: Before anything else, analyze the impact of the changes. Consider side effects, impacted services, and any database changes. For database migrations, the order of operations is critical.
2.  **Merge to Main**: Once the impact is understood and the change is ready, merge it to the `main` branch. All deployments should be from `main` to ensure it reflects what's in production.
3.  **Deploy**: Deployments can be done via the CLI (e.g., using Warp) or the AWS CodeDeploy UI. It's recommended to use the specific commit SHA for the deployment to be certain about what is being deployed.
    - In CodeDeploy, select the project (e.g., `trip-monitor`).
    - If deploying multiple services, consider the correct sequence of deployments. Generally, deploy the server-side changes before the client-side changes.

See also:
- [[Deployment Pre-checks]]
- [[Verifying Changes Before Deployment]]
- [[Post-Deployment Monitoring]]
- [[Rollback Procedures]]
- [[Main Branch Synchronization Policy]]
- [[Deployment Communication Policy]]
- [[Software Automation and CI-CD]]
