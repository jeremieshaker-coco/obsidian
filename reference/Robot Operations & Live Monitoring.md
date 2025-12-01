---
tags:
  - workflow
  - diagram
  - architecture
  - robot
  - monitoring
  - operations
---
# Robot Operations & Live Monitoring

This diagram illustrates the real-time command, control, and monitoring of a robot during an active trip. It focuses on the tools used by internal operators (Pilots, MRO, Mission Control).

```mermaid
graph TD
    subgraph "Physical World"
        robot["Robot (Physical Device)"]
    end

    subgraph "User Interfaces"
        pilot_ui["Pilot UI"]
        mro_ui["MRO UI"]
        mc_ui["Mission Control UI"]
    end

    subgraph "Backend Services"
        subgraph "TypeScript Services"
            operations_ts["Operations Service"]
            device_ts["Device Service"]
            state_ts["State Service"]
            trip_monitor_ts["Trip Monitor Service"]
        end
        subgraph "Go Services"
            robot_go["Robot Service"]
            maps_go["Maps Service"]
        end
    end
    
    subgraph "AWS Infrastructure"
        aws_iot["AWS IoT (MQTT)"]
        sqs["SQS"]
        dynamo_state["device-state-tracking Table (DynamoDB)"]
        elasticache["robot-redis Cache (ElastiCache)"]
        s3["S3"]
    end

    %% Robot to Cloud Data Flow
    robot -- "Heartbeats, Events (MQTT)" --> aws_iot
    aws_iot -- "Forwards Data" --> sqs
    sqs -- "Consumes Events" --> device_ts
    sqs -- "Consumes Events" --> state_ts

    %% UI to Backend for Commands
    pilot_ui -->|REST| operations_ts
    pilot_ui -->|REST| device_ts
    mro_ui -->|REST| device_ts
    mro_ui -->|REST| state_ts
    mc_ui -->|REST| device_ts
    mc_ui -->|REST| state_ts
    
    %% Backend Command Flow to Robot
    device_ts -- "Publishes Commands (MQTT)" --> aws_iot
    aws_iot -- "Sends Commands" --> robot

    %% Other Service Interactions
    trip_monitor_ts -->|gRPC / REST| maps_go
    state_ts -->|gRPC| robot_go
    
    %% AWS Connections
    device_ts -->|SDK| s3
    state_ts -->|SDK| dynamo_state
    robot_go -->|SDK| elasticache
```

This flow involves the [[Operations Service]], [[Device Service]], [[State Service]], [[Trip Monitor Service]], [[Robot Service]], and [[Maps Service]].
