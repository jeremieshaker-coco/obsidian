---
tags:
  - dispatch-engine
  - planning
  - workflow
  - sequence-diagram
---
# PlannerService - Replan Workflow

This document provides a detailed sequence diagram of the `replan` method in the [[PlannerService]].

```mermaid
sequenceDiagram
    participant CronJob as Cron Job
    participant PlannerService as PlannerService
    participant ResourceService as ResourceService
    participant DemandService as DemandService
    participant PlannerLib as Planner Library
    participant OperationsService as Operations Service
    participant AmqpPublisher as AMQP Publisher

    CronJob->>+PlannerService: Trigger replan()
    
    PlannerService->>+ResourceService: Get robots & pilots
    ResourceService-->>-PlannerService: Return latest state
    
    PlannerService->>+DemandService: Get active demands
    DemandService-->>-PlannerService: Return latest state

    PlannerService->>PlannerService: Identify orphaned demands
    loop For each orphaned demand
        PlannerService->>PlannerLib: findRobotEstimates()
        PlannerLib-->>PlannerService: Return best candidate robot
        alt Candidate Found
            PlannerService->>PlannerService: Update demand with new robot
        else No Candidate Found
            PlannerService->>PlannerService: Log failure to reassign
        end
    end

    PlannerService->>+PlannerLib: generateEstimates(all scheduled demands)
    PlannerLib-->>-PlannerService: Return new ETAs for all demands
    
    loop For each new estimate
        PlannerService->>+ResourceService: updateRobotDemand()
        ResourceService-->>-PlannerService: Acknowledge update
        
        PlannerService->>+DemandService: updatePlan()
        DemandService-->>-PlannerService: Acknowledge update
        
        opt If demand is a delivery
            PlannerService->>+AmqpPublisher: Publish EtaUpdate event
            AmqpPublisher-->>-PlannerService: Acknowledge publish
        end
    end

    loop For each idle robot
        PlannerService->>PlannerService: Check if next demand is ready to activate
        alt Demand is ready for activation
            PlannerService->>+OperationsService: activateTrip()
            OperationsService-->>-PlannerService: Acknowledge activation
            
            PlannerService->>+DemandService: active()
            DemandService-->>-PlannerService: Acknowledge activation
        end
        
        alt Robot is at delivery origin
            PlannerService->>+AmqpPublisher: Publish AtOrigin event
            AmqpPublisher-->>-PlannerService: Acknowledge publish
        end
    end
    
    PlannerService-->>-CronJob: End replan cycle
```
