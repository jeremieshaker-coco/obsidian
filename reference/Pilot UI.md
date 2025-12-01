---
tags:
  - frontend
  - ui
  - pilot
  - operations
---
# Pilot UI

An interface for remote [[FO (Field Operator)|pilots]]. It connects to the [[Operations Service]] for task information and the [[Device Service]] for video streams and [[Robot]] control.

## Features

### Trip Management
- View assigned trips and deliveries
- Accept/cancel trips
- Monitor robot status and location
- Track delivery progress

### Robot Control
- View live camera feeds
- Issue navigation commands
- Monitor telemetry (battery, connectivity, GPS)
- Control lid mechanisms

### Troubleshooting
- Guided troubleshooting flows
- Hard/soft reset capabilities
- Modem and GPS resets
- SIM card switching
- Report route/map issues

### Incident Management
- File incident reports
- Request field operator assistance
- Document delivery issues
- Route reassignment

### Breaks & Scheduling
- Start/end scheduled breaks
- View shift schedules
- Track active time

## Analytics

Pilot operations tracked via [[Segment Analytics Integration]]:
- Trip lifecycle events (accept, cancel, complete)
- Troubleshooting steps and outcomes
- Robot reset operations
- Incident reports
- Route issue reporting
- Field operator requests

Events stored in [[Delivery Platform Events Schema]]:
- `pilot_cancel_trip` - Trip cancellation
- `pilot_troubleshooting_step` - Navigation through troubleshooting
- `pilot_hard_reset` / `pilot_soft_reset` - Reset operations
- `pilot_incident_report` - Incident filed
- `pilot_route_issue` - Map/routing problem
- `pilot_requested_field_op` - Field assistance needed
- `pilot_start_break` - Break started

## Related Concepts

- [[Operations Service]] - Trip and task management
- [[Device Service]] - Robot control and telemetry
- [[FO (Field Operator)]] - Pilot role
- [[Trip]] - Delivery trips
- [[Task]] - High-level delivery tasks
- [[Robot]] - Autonomous delivery robots
- [[Delivery Platform Events Schema]] - Analytics data
- [[Pilot Conducts a Delivery Flow]] - Standard delivery workflow
