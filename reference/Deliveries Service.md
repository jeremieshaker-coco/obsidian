---
tags:
  - service
  - go
  - backend
---
# Deliveries Service

*   **Responsibilities:** Manages the state of a [[Delivery]], which is a request from an external partner. It acts as a wrapper around different delivery providers (e.g., DoorDash, Uber, Coco). It distinguishes between a [[Delivery]] and an [[Attempt]], where a single delivery may have multiple attempts (e.g., a primary [[Robot]] attempt and a human rescue). Communication with other services is heavily event-driven via RabbitMQ.
*   **AWS Integration:**
    *   **RDS:** Stores core delivery and order data in the `deliveryplatform` PostgreSQL database.
    *   **S3:** Uses buckets like `delivery-verification-images-dev` for storing delivery-related images.
*   **Data Warehouse:**
    *   Operational data synced to [[Redshift Data Warehouse]] via Fivetran
    *   Current schema: [[Deliveries V3 RDS Schema]]
    *   Legacy schema: [[Deliveries RDS Schema]]
    *   Integration tracking: [[Integration Provider Tables]] (OLO, DoorDash)
    *   CRM integration: [[Salesforce Data Sync]]
