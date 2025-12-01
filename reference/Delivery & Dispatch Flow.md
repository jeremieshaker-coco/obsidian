---
tags:
  - workflow
  - diagram
  - architecture
  - delivery
  - dispatch
---
# Delivery & Dispatch Flow

This diagram shows the complete lifecycle of a delivery, from the moment a request comes in from a partner or merchant to a robot being dispatched.

```mermaid
graph TD
    subgraph "External"
        Partner["Partner API (e.g., DoorDash)"]
    end

    subgraph "User Interfaces"
        merchant_ui["Merchant UI"]
    end

    subgraph "Backend Services"
        subgraph "TypeScript Services"
            integrations_ts["Integrations Service"]
            deliveries_ts["Deliveries Service"]
            dispatch_ts["Dispatch Engine"]
        end
        subgraph "Go Services"
            fleet_go["Fleet Service"]
            task_go["Task Service"]
            maps_go["Maps Service"]
            config_go["Config Service"]
        end
    end
    
    subgraph "AWS Infrastructure"
        rmq["RabbitMQ"]
    end

    subgraph "AWS Data Stores"
        rds_deliveries["Deliveries DB (RDS)"]
        rds_maps["Maps DB (RDS)"]
        dynamo_fleet_robots["fleet-robots Table (DynamoDB)"]
        dynamo_fleet_providers["fleet-provider-accounts Table (DynamoDB)"]
        dynamo_tasks["task-tasks Table (DynamoDB)"]
    end

    %% Data Flow
    Partner -->|REST| integrations_ts
    merchant_ui -->|REST| deliveries_ts
    integrations_ts -->|"Make Quote (REST)"| deliveries_ts
    deliveries_ts -->|REST| dispatch_ts
    
    %% Event Flow
    deliveries_ts -- "Publishes Updates" --> rmq
    rmq -- "Consumes Updates" --> integrations_ts
    
    dispatch_ts -->|gRPC| fleet_go
    dispatch_ts -->|gRPC| maps_go
    deliveries_ts -->|gRPC| task_go

    %% Config Service Usage
    integrations_ts -->|gRPC| config_go
    deliveries_ts -->|gRPC| config_go
    dispatch_ts -->|gRPC| config_go

    %% AWS Connections
    deliveries_ts -->|JDBC| rds_deliveries
    fleet_go -->|SDK| dynamo_fleet_robots
    fleet_go -->|SDK| dynamo_fleet_providers
    task_go -->|SDK| dynamo_tasks
    maps_go -->|JDBC| rds_maps
```

This flow involves the [[Integrations Service]], [[Deliveries Service]], [[Dispatch Engine]], [[Fleet Service]], [[Task Service]], [[Maps Service]], and [[Config Service]].
