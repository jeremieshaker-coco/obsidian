---
tags:
  - heartbeat
  - robot
  - operations
---
# Active Robot Heartbeat

The heartbeat for active robots is sent every **20 seconds**.

This is configured in `delivery-platform/service/operations/src/modules/robots/services/robots-jobs.service.ts`. A repeating job is scheduled with a 20,000ms interval.

This job is processed by `ActiveRobotHeartbeatWorker`, which then publishes the location of robots currently on a trip. This serves as a [[Heartbeat]] for active robots.

There is also a consumer that considers heartbeats older than 3 minutes to be stale, which is consistent with a 20-second heartbeat interval. This is located in `coco-services/fleet/internal/consumer/robot_consumer.go`.

See also: [[Heartbeat Frequencies]]
