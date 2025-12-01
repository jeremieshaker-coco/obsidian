---
tags:
  - service
  - go
  - backend
---
# Device Service

*   **Responsibilities:** Acts as a bridge to the physical [[Robot|robots]]. It consumes telemetry and heartbeats from SQS (fed by AWS IoT) and sends commands back to [[Robot|robots]] by publishing messages to AWS IoT MQTT topics.
*   **AWS Integration:**
    *   **SQS:** Consumes from queues like `device-mqtt-events-dev` which receive data from AWS IoT.
    *   **S3:** Leverages buckets such as `coco-front-camera-images-dev` for storing images and logs from devices.
