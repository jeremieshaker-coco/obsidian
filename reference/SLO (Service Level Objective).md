---
tags:
  - slo
  - monitoring
  - reliability
---
# SLO (Service Level Objective)

A **Service Level Objective (SLO)** is a target value or range of values for a service level that is measured by a [[SLI (Service Level Indicator)]]. SLOs are a key component of Site Reliability Engineering (SRE) and form the foundation of a modern, user-centric [[Observability Strategy]].

### Core Concepts

-   **What it is:** An SLO is a formal promise to a user about a specific aspect of your service's reliability (e.g., availability, latency).
-   **Example:** "99.9% of quote requests over the last 28 days will return a successful response."
-   **Based on Metrics:** SLOs are always defined in terms of [[Datadog Observability for Dispatch|metrics]], not logs or events.

### Error Budgets

An SLO implies an **error budget**. If your availability SLO is 99.9%, your error budget is the remaining 0.1%. This budget is the acceptable level of unreliability that you are allowed to "spend" over the SLO's time window.

-   **Purpose:** The error budget is a data-driven tool for making decisions.
    -   If you have plenty of error budget left, you can confidently ship new features.
    -   If you have exhausted your error budget, all development focus must shift to reliability and stability.

### Alerting on SLOs

The most effective alerting strategies are based on the **burn rate** of the error budget.

-   **High-Severity Alert (Pages On-Call):** "Alert if we are burning through our 28-day error budget at a rate that will exhaust it in 24 hours." This indicates a severe, user-impacting outage.
-   **Low-Severity Alert (Tickets/Slack):** "Alert if we are burning through our 28-day error budget at a rate that will exhaust it in 7 days." This is a leading indicator of a problem that needs attention but is not yet a crisis.
