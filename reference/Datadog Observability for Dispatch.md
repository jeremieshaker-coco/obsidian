---
tags:
  - observability
  - monitoring
  - datadog
  - dispatch-engine
---
# Datadog Observability for Dispatch

A key goal for improving the [[Dispatch Engine]] is to enhance its observability. This allows for better debugging, monitoring of system health, and analysis of dispatching decisions. This note provides a practical guide for implementing our [[Observability Strategy]] in Datadog to monitor the [[Dispatch Engine]], specifically focusing on [[Dispatch Robot Reassignment]].

### Key Features Enabled by Observability

-   **Granular Rejection Reasons:** The system can now provide specific reasons for failed estimates, such as robot health, battery status, or distance, moving beyond a generic `RobotAvailability` reason.
-   **Clear Assignment Audit Log:** We now have a clear audit log for [[Robot]] assignments, detailing *why* and *when* an assignment changes. This is the foundation for investigating reassignments.

### Key Metrics for Dashboards

-   **Metric Name:** `dispatch-engine.planner.replan.demand_orphaned`
-   **Description:** A `count` metric that increments every time a [[Demand]] is orphaned, triggering a reassignment.
-   **Key Tag:** `reason` (e.g., `reason:Offline`, `reason:Unhealthy`). This is crucial for building powerful widgets.
-   **Usage:**
    -   Create a timeseries graph grouped by the `reason` tag to see *why* reassignments are happening over time.
    -   Create a pie chart grouped by `reason` for a high-level summary.

### Key Logs for Investigation

To trace the lifecycle of a [[Demand]], we have three key structured log messages:

1.  `[replan] orphaning demand from robot` - The root cause of a reassignment. Contains the `extra.issues` array.
2.  `[replan] robot reassignment` - Confirms the reassignment, showing `oldDeviceId` and `newDeviceId`.
3.  `[estimate] estimate assignment` - Shows the initial [[Robot]] assignment.

### Datadog Implementation Guide

#### 1. Create a `demandId` Facet
To enable easy click-to-filter functionality, create a **Facet** on the `@demandId` attribute in the Datadog Log Explorer. This is a one-time setup that indexes the field for searching and filtering.

#### 2. Build a Log Query with Template Variables
For a reusable investigation tool on a dashboard, use a **Template Variable** named `$demand_id`.

-   **Set the Default Value:** Set the default value of `$demand_id` to `*` to show all demands when the variable is empty.
-   **Widget Query:** Use the following query in your log stream widget. It will automatically filter when an ID is entered into the variable text box.
    ```
    @demandId:$demand_id (message:"[estimate] estimate assignment" OR message:"[replan] orphaning demand from robot" OR message:"[replan] robot reassignment")
    ```

This setup allows any engineer to quickly go from a high-level metric on a dashboard to the specific, detailed logs for a single problematic delivery with a simple copy-paste of the `demandId`.
