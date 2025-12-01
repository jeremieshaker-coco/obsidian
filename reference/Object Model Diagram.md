---
tags:
  - diagram
  - uml
  - architecture
---
# Object Model Diagram

This diagram shows the high-level relationships between the core data objects in the system.

```mermaid
classDiagram
    direction RL
    class Delivery {
        +String id
        +DeliveryStatus status
        +List~Attempt~ attempts
    }
    class Attempt {
        +String id
        +AttemptStatus status
        +ProviderType provider
    }
    class Task {
        +String id
        +TaskStatus status
        +TaskType type
    }
    class Trip {
        +String id
        +TripStatus status
    }
    class Robot {
        +String serial
        +RobotStatus status
    }
    class Operator {
        +String id
        +OperatorRole role
        +OperatorStatus status
    }
    class Quote {
        +String id
        +Location pickup
        +Location dropoff
    }

    Delivery "1" --o "1..*" Attempt : "consists of"
    Delivery "1" -- "1" Quote : "is based on"
    Attempt "1" -- "1" Task : "is fulfilled by"
    Task "1" --o "1..*" Trip : "is composed of"
    Task "1" -- "1" Robot : "is assigned to"
    Task "1" -- "1" Operator : "is assigned to"
```

This diagram shows the relationships between [[Delivery]], [[Attempt]], [[Task]], [[Trip]], [[Robot]], and [[Operator]].
