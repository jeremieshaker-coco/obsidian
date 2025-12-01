---
tags:
  - service
  - typescript
  - backend
  - iot
---
# IotStreamerService

A service within the [[State Service]] responsible for processing real-time data streams from robots, which are fed through AWS IoT and SQS.

Its primary functions include:
- Processing robot heartbeats to monitor health and status.
- Updating the robot's state shadow.
- Handling [[Lid State Resynchronization]] by comparing heartbeat data with the active [[Lid Cycle]].

This service acts as a critical link between the physical fleet and the backend's state management system.
