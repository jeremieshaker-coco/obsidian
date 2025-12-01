---
tags:
  - pattern
  - dispatch-engine
---
# Continuous Replanning

**Continuous Replanning** is the core process within the [[Dispatch Engine]] that allows it to adapt to the constantly changing state of the world.

Instead of creating a static plan, a cron job (`ReplanWorker`) periodically triggers the `planner.service` to re-evaluate the entire plan for all outstanding [[Demand|Demands]].

This process takes into account:
- The latest [[Robot]] locations and statuses.
- Current pilot availability.
- Any new or canceled [[Demand|Demands]].

This ensures that the dispatching decisions are always based on the most up-to-date information, leading to higher efficiency and resilience. This is a key part of the [[Dispatch Engine Workflow]].
