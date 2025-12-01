---
tags:
---
# Fleet Service gRPC Endpoints

This document lists the gRPC endpoints for the **[[Fleet Service]]**.

### FleetLaborProviderScopeService

*   **`Get`**: Retrieves a list of available "labor" resources (e.g., [[Pilots]], [[FO (Field Operator)|Field Ops]]) based on certain criteria.
*   **`GetStream`**: Opens a stream to receive real-time updates on labor resources.

### FleetBeaconScopeService

*   **`Get`**: Retrieves a list of "beacons" (likely [[Robot]]s) within a certain geographic area.
*   **`GetStream`**: Opens a stream to receive real-time updates on beacons.
