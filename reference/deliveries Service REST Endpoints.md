---
tags:
---
*   `GET /api/v2/delivery/query-v2`: Queries for a list of [[Delivery|deliveries]].
*   `GET /api/v2/delivery/{id}/attempt-history`: Fetches the [[Attempt]] history for a [[Delivery]].
*   `GET /api/v2/delivery/{id}/info`: Fetches detailed information for a [[Delivery]].
*   `PATCH /api/v2/robot-loaded-hook/{attemptId}`: A webhook that is called when a [[Robot]] is loaded for a delivery [[Attempt]].
*   `POST /api/v2/deliveries/{id}/delivery-issues/collection`: Reports issues for a [[Delivery]].
