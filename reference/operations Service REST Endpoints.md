---
tags:
---
*   `GET /operators/{id}`: Fetches an operator by ID.
*   `POST /operators`: Creates a new operator.
*   `POST /operators/{id}/action`: Sets the current action for an operator (e.g., `on-shift`, `off-shift`).
*   `GET /cities/{id}/hubs`: Fetches all hubs for a given city.
*   `GET /beta/tasks`: Fetches [[Task|tasks]] based on filters.
*   `GET /beta/tasks/{id}`: Fetches a single [[Task]] by ID.
*   `POST /beta/tasks/create`: Creates a new field ops [[Task]].
*   `POST /beta/tasks/cancel`: Cancels a field ops [[Task]].
*   `GET /beta/trips`: Fetches [[Trip|trips]].
*   `GET /beta/pilots/trip/{id}`: Fetches a single [[Trip]] by ID.
*   `GET /robots`: Fetches [[Robot|robots]] in the fleet, with optional filters.
*   `POST /robots/{serial}/deploy`: Deploys a [[Robot]].
*   `POST /robots/deploy`: Deploys one or more grounded [[Robot|robots]].
*   `POST /robots/{serial}/undeploy`: Undeploys a [[Robot]].
*   `GET /partners`: Fetches partners.
*   `GET /deliveries/{id}/geolocations`: Fetches the geo-locations associated with a [[Delivery]].
*   `GET /cities`: Fetches all cities.
*   `POST /request-fo`: Requests a [[FO (Field Operator)|Field Op]] for a [[Task]].
*   `GET /parking-lots`: Fetches parking lots.
*   `PUT /parking-lots/{id}`: Updates details for a parking lot.
*   `PUT /parking-lots/{id}/operatingHours`: Updates the operating hours for a parking lot.
*   `PUT /parking-lots/{id}/demandSchedule`: Updates the demand schedule for a parking lot.
*   `GET /pilots`: Fetches pilots.
*   `POST /pilots/{id}/{endpoint}`: Reserves a pilot for a [[Task]] (where `endpoint` is `reserve` or `release`).
*   `GET /deliveries/{id}/assignable-tasks`: Fetches [[Task|tasks]] that can be assigned to a [[Delivery]].
*   `GET /attempts/{id}/assignments`: Fetches assignments for a delivery [[Attempt]].
*   `GET /fieldOps/{cityId}`: Fetches [[FO (Field Operator)|Field Ops]] in a specific city.
