---
tags:
  - service
  - go
  - backend
---
# Integrations Service

*   **Responsibilities:** Serves as a gateway for external partners, mapping their external APIs to Coco's internal APIs (e.g., creating a delivery quote). It receives asynchronous updates on [[Delivery]] status by consuming events from RabbitMQ.
*   **AWS Integration:**
    *   **SQS:** Uses queues like `coco-dev-order-integrations-webhook` to process incoming webhook events from partners.
