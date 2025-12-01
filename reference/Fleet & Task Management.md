---
tags:
  - architecture
  - diagram
  - fleet
  - task
  - operations
---
# Fleet & Task Management

This diagram shows the internal administrative and management side of the system, focusing on how the fleet of robots and operators is managed and how tasks are orchestrated.

```mermaid
graph TD
    subgraph "User Interfaces"
        mc_ui["Mission Control UI"]
    end

    subgraph "Backend Services"
        subgraph "TypeScript Services"
            operations_ts["Operations Service"]
        end
        subgraph "Go Services"
            fleet_go["Fleet Service"]
            task_go["Task Service"]
            task_assigner_go["Task Assigner"]
            task_executor_go["Task Executor"]
            anomaly_go["Anomaly Service"]
        end
    end

    subgraph "AWS Infrastructure"
        dynamo_fleet_robots["fleet-robots Table (DynamoDB)"]
        dynamo_fleet_providers["fleet-provider-accounts Table (DynamoDB)"]
        dynamo_tasks["task-tasks Table (DynamoDB)"]
        kafka["Kafka"]
    end

    %% UI to Backend
    mc_ui -->|REST| operations_ts

    %% Service to Service
    operations_ts -->|gRPC| task_go
    operations_ts -->|gRPC| fleet_go
    
    task_go -->|Event| kafka
    anomaly_go -->|Consumes| kafka
    
    %% Data Flow for Task Sub-System
    task_go -->|Writes to| dynamo_tasks
    task_assigner_go -->|Reads/Writes| dynamo_tasks
    task_executor_go -->|Reads/Writes| dynamo_tasks
    
    %% Fleet DB
    fleet_go -->|SDK| dynamo_fleet_robots
    fleet_go -->|SDK| dynamo_fleet_providers
```

This flow involves the [[Operations Service]], [[Task Service]], [[Fleet Service]], [[Task Assigner]], [[Task Executor]], and [[Anomaly Service]].
