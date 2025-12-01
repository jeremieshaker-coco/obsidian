---
tags:
  - observability
  - metrics
  - datadog
---
# Datadog Metrics

This document provides a central repository of all custom Datadog metrics published across the `delivery-platform` codebase.

## [[Dispatch Engine]]

Metrics related to the core dispatching and planning logic.

### [[PlannerService]]

-   `dispatch-engine.planner.replan.duration`: The time it takes for a single `replan` cycle to complete.
-   `dispatch-engine.planner.replan`: A counter for each time the `replan` method is invoked.
-   `dispatch-engine.planner.failures`: The number of planning failures, tagged by `type` (the `LimitingFactor`).
-   `dispatch-engine.planner.failures.total`: The total number of planning failures.
-   `dispatch-engine.planner.delay`: The delay in minutes for a scheduled demand, tagged by `type` (the `DemandType`).
-   `dispatch-engine.planner.replan.orphans.total`: The total number of orphaned demands detected.
-   `dispatch-engine.planner.replan.orphans.failed_reassignment`: The number of orphaned demands that could not be reassigned to a new robot.
-   `dispatch-engine.planner.replan.eta_updates.total`: The number of ETA updates published.
-   `dispatch-engine.planner.replan.at_origin_events.total`: The number of `AtOrigin` events published.
-   `dispatch-engine.planner.replan.activations.total`: The number of demands that are successfully activated.
-   `dispatch-engine.planner.replan.activations.failures`: The number of demands that failed to activate.

### ResourceService (Supply)

-   `dispatch.engine.service.resource.pilot.hydration.error`: Incremented when there is an error hydrating pilot data.
-   `dispatch.engine.service.resource.robot.hydration.error`: Incremented when there is an error hydrating robot data.

### Event Handlers

-   `dispatch.engine.handler.demand.update.invoked`: Incremented when the external demand update handler is invoked.
-   `dispatch.engine.handler.demand.update.success`: Incremented on successful demand updates.
-   `dispatch.engine.handler.demand.update.failure`: Incremented on failed demand updates.
-   `dispatch.engine.handler.robot.deployment.invoked`: Incremented when the robot stage change handler for deployments is invoked.
-   `dispatch.engine.handler.robot.deployment.success`: Incremented on successful deployment updates.
-   `dispatch.engine.handler.robot.deployment.failure`: Incremented on failed deployment updates.
-   `dispatch.engine.handler.robot.state.change.invoked`: Incremented when the robot stage change handler for general state changes is invoked.
-   `dispatch.engine.handler.robot.state.change.success`: Incremented on successful state change updates.
-   `dispatch.engine.handler.robot.state.change.failure`: Incremented on failed state change updates.

## [[State Service]]

Metrics related to the `state` service and its interaction with IoT devices.

-   `state.pin.lid_open.success`: Successful lid open attempts using a PIN.
-   `state.pin.lid_open.failure.incorrect_pin`: Failed lid open attempts due to an incorrect PIN.
-   `state.pin.lid_open.failure.other`: Other failures during lid open attempts.
-   `state.pin.lid_close.abort`: Lid close process was aborted.
-   `state.pin.lid_close.confirm`: Lid close process was confirmed.
-   `state.pin.lid_close.other`: Other events during the lid close process.

## [[Operations Service]]

Metrics related to the `operations` service, which handles robot commands and fleet management.

-   `operations.robots.controller.openlid.attempt`: An attempt was made to open a robot's lid.
-   `operations.robots.controller.openlid.success`: A robot's lid was successfully opened.
-   `operations.robots.controller.openlid.failure.robot.does.not.exist`: Lid open failed because the robot does not exist.
-   `operations.robots.controller.openlid.failure.robot.grounded`: Lid open failed because the robot is grounded.
-   `operations.robots.controller.openlid.failure.robot.offline`: Lid open failed because the robot is offline.
-   `operations.robots.controller.openlid.failure.robot.on.trip`: Lid open failed because the robot is on a trip.
-   `operations.robots.controller.openlid.failure.robot.unhealthy`: Lid open failed because the robot is unhealthy.
-   `operations.robots.controller.openlid.failure.robot.undergoing.maintenance`: Lid open failed because the robot is undergoing maintenance.
-   `operations.robots.controller.openlid.failure.robot.needs.maintenance`: Lid open failed because the robot needs maintenance.
-   `operations.robots.controller.openlid.failure.robot.not.in.parking.lot`: Lid open failed because the robot is not in a parking lot.
-   `operations.fleet.deploy.blocked.for.maintenance`: A deployment was blocked because the robot requires maintenance.

## [[Integrations Service]]

Metrics related to third-party integrations like DoorDash and Uber.

### Uber

-   `integrations.uber.offer.unsupported_job_count`: An offer was received with an unsupported number of jobs.
-   `integrations.uber.integration.not.found`: The Uber integration for a given store was not found.
-   `integrations.uber.integration.not.active`: The Uber integration for a given store is not active.
-   `integrations.uber.robot.serial.not.found`: The robot serial number associated with an Uber order was not found.
-   `integrations.uber.offer.unsupported_waypoint_count`: An offer was received with an unsupported number of waypoints.
-   `integrations.uber.self.limited`: The integration is self-throttling due to capacity or other limits.
-   `integrations.uber.cancel.dropoff.too.far`: A cancellation was triggered because the dropoff location is too far.
-   `integrations.uber.update.failed`: A status update failed to send to Uber.
-   `integrations.uber.cancel.failed`: A cancellation failed to send to Uber.

### DoorDash (v1, v2, v3)

-   `integrations.doordash.v3.simulator.events`: An event was processed by the DoorDash v3 simulator.
-   `integrations.doordash.v3.service.events`: An event was processed by the DoorDash v3 service, tagged by `event`.
-   `integrations.doordash.v3.service.errors`: An error occurred in the DoorDash v3 service, tagged by `err`.
-   `integrations.doordash.quote.rejection.ddreason`: A quote was rejected, tagged by DoorDash's reason.
-   `integrations.doordash.no.estimate`: An estimate could not be provided, tagged by `reason`.
-   `integrations.doordashv3.accept.quote.failure`: The `acceptQuote` call failed in the v3 integration.
-   `integrations.doordash.v3.updateService.events`: An event was processed by the DoorDash v3 update service, tagged by `event`.
-   `integrations.doordash.v3.updateService.errors`: An error occurred in the DoorDash v3 update service, tagged by `err`.
-   `integrations.doordash.v3.client.events`: An event was processed by the DoorDash v3 client, tagged by `event`.
-   `integrations.doordash.v3.client.errors`: An error occurred in the DoorDash v3 client, tagged by `err`.
-   `integrations.doordash.estimate.self.limited`: A DoorDash estimate was self-throttled.
-   `integrations.doordash.estimate.self.limited.pickup`: A DoorDash estimate was self-throttled at the pickup stage.
-   `integrations.doordash.request.quote.failure`: A `requestQuote` call failed.
-   `integrations.doordash.accept.quote.failure`: An `acceptQuote` call failed.
-   `integrations.doordash.status.update.404.delivery.canceled`: A status update failed with a 404 because the delivery was canceled.
-   `integrations.doordash.transition.failure`: A state transition failed to update to DoorDash.
-   `integrations.doordash.opt.out`: A customer opted out of robot delivery, tagged as `permanent`.
-   `integrations.doordash.order.already.finalized`: A DoorDash order was already in a final state (delivered or canceled) when an update was attempted.
-   `integrations.doordash.delivery.not.found`: A delivery associated with a DoorDash order was not found in the system.

### Delivery Updates

-   `integrations.update.status.failure`: A generic failure to update a delivery status to an integration partner.
-   `integrations.update.transition.failure`: A generic failure to transition a state for an integration partner.
-   `integrations.update.cancel.failure`: A generic failure to cancel a delivery with an integration partner.
-   `integrations.update.opt.out.failure`: A generic failure to process an opt-out request.

## [[Deliveries Service]]

-   `deliveries.watchdog.rate.limit.exceeded`: The watchdog service exceeded a rate limit, tagged by `key`.
-   `deliveries.watchdog.report.issue`: An issue was reported by the watchdog.
-   `deliveries.watchdog.cancel.delivery`: The watchdog canceled a delivery.
-   `deliveries.watchdog.change.provider`: The watchdog changed the delivery provider.
-   `deliveries.provider.uber.request.error`: An API request to Uber from the deliveries service failed.
-   `deliveries.provider.doordash.request.error`: An API request to DoorDash from the deliveries service failed.
-   `deliveries.repo.quote.find.failure`: The quote repository failed to find a quote.
-   `deliveries.service.delivery.created`: A new delivery was successfully created.
-   `deliveries.service.delivery.updated`: A delivery was successfully updated.
-   `deliveries.service.delivery.canceled`: A delivery was successfully canceled.
-   `deliveries.service.estimate.success`: A delivery estimate was successfully generated.
-   `deliveries.service.estimate.failure`: A delivery estimate failed to be generated, tagged by `reason`.

## Product Health Dashboard Metrics

This section outlines the "golden" metrics that should be included in a high-level product health dashboard to provide an at-a-glance view of the entire platform's performance.

### Funnel Health

-   `demand.creations` (Counter): The number of new demands created, tagged by `demand_type`. Represents the top-of-funnel request volume.
-   `demand.status_transitions` (Counter): Tracks the lifecycle of a demand as it moves through the system, tagged by `from_status` and `to_status`. Used to build a real-time fulfillment funnel.

### Customer Experience

-   `delivery.completion_rate` (Rate): The percentage of deliveries that are successfully completed versus canceled. This is a critical business KPI.
-   `delivery.total_duration` (Distribution): The end-to-end time of a delivery from creation to completion. Measures the customer's wait time.

### System Capacity

-   `supply.availability` (Gauge): The number of available resources (pilots, robots), tagged by `resource_type` and `zone`.
-   `demand.cancellations` (Counter): The number of canceled demands, tagged by `reason`. Helps diagnose why deliveries are not being fulfilled.
