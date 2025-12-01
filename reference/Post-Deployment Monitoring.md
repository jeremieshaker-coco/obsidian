---
tags:
---
# Post-Deployment Monitoring

After a deployment, it's crucial to monitor the service to ensure everything is working as expected.

## Monitoring Plan

Before deploying, have a plan for what to monitor. This includes identifying key metrics and logs to check.

## Using DataDog

DataDog is the primary tool for post-deployment monitoring.

1.  **Check Logs**:
    - Navigate to the **Logs > Explorer** section in DataDog.
    - Filter logs by the service you deployed (e.g., `service:trip-monitor`).
    - Verify that new logs are appearing as expected and check for any unexpected errors or spikes in error rates. If you see new errors, verify if they existed before the deployment.

2.  **Check Dashboards**:
    - Review relevant DataDog dashboards. The names of the dashboards can often guide you to the right ones.
    - For large changes, consider creating a new dashboard to monitor the specific features or changes being deployed.

Staging environments also have logs available in DataDog for pre-production verification.
