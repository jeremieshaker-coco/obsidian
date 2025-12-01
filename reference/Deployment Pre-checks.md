---
tags:
---
# Deployment Pre-checks

Before deploying any changes, it's important to perform the following pre-checks:

1.  **Analyze Impact**: Thoroughly analyze the potential impact of the changes.
    - Identify all services that might be affected.
    - Consider any potential side effects or edge cases.
    - Pay special attention to database migrations, as the order of operations (e.g., adding vs. dropping a column) is critical.
2.  **Deployment Sequencing**: If deploying multiple services, plan the order of deployments carefully. For example, when a client depends on a server, deploy the server first to ensure compatibility.
