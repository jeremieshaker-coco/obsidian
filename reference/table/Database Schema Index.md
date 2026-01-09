---
tags:
  - index
  - database
  - reference
---
# Database Schema Index

This is a comprehensive index of all database schemas, tables, and enums in the delivery platform. See [[Database Architecture Overview]] for architectural patterns.

## Service Schemas

### [[Operations RDS Schema]]
**Service**: [[Operations Service]]  
**Purpose**: Operational workflow management

Key Tables:
- [[Task Table]] - Operational tasks
- [[Trip Table]] - Physical robot trips  
- [[Robot Table]] - Robot device registry
- [[PilotModel Table]] - Pilot operator data
- [[FoTaskModel Table]] - Field operations tasks

### [[Deliveries V3 RDS Schema]]
**Service**: [[Deliveries Service]]  
**Purpose**: Delivery lifecycle management

Key Tables:
- [[Delivery V3 Table]] - Main delivery records
- [[Attempt Table]] - Fulfillment attempts
- Quote Table - Delivery quotes
- Customer Deliveries Table - Customer records

### [[Dispatch Engine RDS Schema]]
**Service**: [[Dispatch Engine]]  
**Purpose**: Resource planning and scheduling

Key Tables:
- [[Demand Table]] - Delivery/movement demands
- [[Robot Planning Table]] - Robot scheduling state
- Plan Table - Execution plans

### [[State RDS Schema]]
**Service**: [[State Service]]  
**Purpose**: Robot hardware state tracking

Key Tables:
- LidCycleEventHistory - Lid operation events
- PinEntryEvent - PIN entry tracking
- Connectivity - Robot connectivity status

### [[Integrations RDS Schema]]
**Service**: [[Integrations Service]]  
**Purpose**: Partner integration management

Key Tables:
- Delivery Integration Table - Processed deliveries
- [[ExternalState Table]] - Integration status tracking
- Pii Table - Customer PII data

### [[Trip Monitor RDS Schema]]
**Service**: [[Trip Monitor Service]]  
**Purpose**: Real-time trip monitoring

Key Tables:
- Trip Monitoring Table - GPS traces and progress

### [[Device RDS Schema]]
**Service**: [[Device Service]]  
**Purpose**: Device deployment tracking

Key Tables:
- Deployment Table - Hardware deployments

## Key Enums

### Task & Trip Enums
- [[TaskStatus Enum]] - Task lifecycle states
- TaskType Enum - Types of tasks
- [[TripStatus Enum]] - Trip lifecycle states
- TripType Enum - Types of trips

### Delivery & Attempt Enums
- DeliveryStatus V3 Enum - Delivery lifecycle states
- [[AttemptStatus Enum]] - Attempt lifecycle states (30+ values)
- [[AttemptProvider Enum]] - Robot/DoorDash/Uber
- [[AttemptCancellationReason Enum]] - 40+ cancellation reasons

### Dispatch Enums
- [[DemandStatus Enum]] - Demand lifecycle states
- DemandType Enum - Types of demands
- LimitingFactor Enum - Resource constraints

### Operator Enums
- OperatorStatus Enum - Operator availability
- OperatorRole Enum - Pilot/Dispatch/Support/Field
- [[ShiftStatus Enum]] - Shift states
- PilotAssignmentStatus Enum - Assignment states

### State Enums
- RobotStateEventState Enum - Operational states
- LidEvent Enum - Lid operation events
- ConnectivityStatus Enum - Online/Offline

### Integration Enums
- IntegrationType Enum - Partner types
- Source Enum - Integration sources
- Region Enum - US/EU

## Architecture Patterns

- [[Database Architecture Overview]] - Overall architecture
- [[Microservices Database Pattern]] - Database-per-service pattern
- [[Entity Relationship Diagram]] - Cross-service relationships

## Schema Documentation by Service

### Operations Service Tables
Task Management:
- [[Task Table]], TaskHistory Table
- [[Trip Table]], TripHistory Table

Robot Management:
- [[Robot Table]], RobotStateHistory Table
- RobotDeployment Table, RobotCheckInHistory Table

Pilot Management:
- [[PilotModel Table]], PilotShiftModel Table
- PilotAssignmentModel Table, PilotReservationHistoryModel Table

Field Operations:
- [[FoTaskModel Table]], FoSubtaskModel Table
- FoAssignmentModel Table, FoAssistanceRequestModel Table

Geography:
- Hub Table, City Table, Location Table

### Deliveries Service Tables
Core:
- [[Delivery V3 Table]], [[Attempt Table]]
- Quote Table, DeliveryAttempts Table

Tracking:
- AttemptHistory Table, NotificationHistory Table

Monitoring:
- WatchdogSchema Table, WatchdogAction Table
- DeliveryIssue Table

Customer:
- Customer Deliveries Table, Dropoff Table

Geography:
- Location Deliveries Table

### Dispatch Engine Tables
Planning:
- [[Demand Table]], Plan Table
- [[Robot Planning Table]]
- RobotPlan Table, PilotPlan Table

Analytics:
- Analytics_EstimateRequest Table
- Analytics_EstimateResponse Table
- Analytics_Estimate Table

Resources:
- Resource Table
- Location Dispatch Table

### State Service Tables
Events:
- LidCycleEventHistory Table
- PinEntryEvent Table
- StateHistory Table

Status:
- LidCycle Table
- Connectivity Table
- ConnectivityHistory Table
- Alerts Table

Context:
- LidEventContext Table
- TransitionRequest Table

### Integrations Service Tables
Processing:
- Delivery Integration Table
- DeliveryError Table
- Item Table

State Management:
- [[ExternalState Table]]
- DoorDashUpdate Table
- DoorDashUpdateV2 Table
- UberTransportPlan Table

Customer:
- Customer Integration Table
- Pii Table
- PiiAccessAudit Table

Geography:
- Location Integration Table
- Address Table

API Tracking:
- IncomingRequest Table
- ClientRequests Table
- ClientResponses Table

Legacy (Deprecated):
- Merchant Integration Table
- Integration Config Table
- Partner Table
- PartnerConfig Table

## Data Flow Diagrams

See these documents for data flow across services:
- [[Delivery & Dispatch Flow]]
- [[Incoming Delivery Request and Robot Assignment Flow]]
- [[Trip Creation and Pilot Assignment Flow]]
- [[Customer Unlocks Robot for Pickup Flow]]

## Analytics

For analytics and reporting across all databases:
- [[Redshift Data Warehouse]] - Centralized analytics
- [[State RDS Schema]] - How to join state to deliveries
- [[Redshift Query Patterns]] - Common query patterns

## Related Concepts

- [[Prisma Client Generation in Monorepo]] - How schemas are managed
- [[Operations Service]] - Core operational service
- [[Deliveries Service]] - Core delivery service
- [[Dispatch Engine]] - Planning service

