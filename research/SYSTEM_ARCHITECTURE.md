# Coco System Architecture

This document describes the overall architecture of our system, covering the `delivery-platform` (TypeScript/NestJS) and `coco-services` (Go) repositories, including how services are organized, how they communicate, and how data is shared.

---

## Repository Overview

### Two Main Backend Repositories

| Repository | Language | Framework | Primary Use |
|------------|----------|-----------|-------------|
| `delivery-platform` | TypeScript | NestJS | Core delivery operations, web applications, business logic |
| `coco-services` | Go | Custom (Cobra/Viper) | High-performance microservices, data processing, fleet management |

### Supporting Repositories

| Repository | Purpose |
|------------|---------|
| `coco-protos` | Protobuf definitions for gRPC contracts |
| `coco-cjs-gen` | Generated TypeScript/JavaScript clients from protos |
| `coco-infra` | Terraform infrastructure definitions |
| `data-pipeline` | DBT models for analytics/data warehouse |

---

## delivery-platform Architecture

### Monorepo Structure

```
delivery-platform/
├── lib/                  # Shared libraries (@coco/*)
├── service/              # Backend microservices
├── web/                  # Frontend applications
├── packages/             # Additional shared packages
├── infra/                # Helm charts and deployment configs
└── client/               # Mobile/native clients
```

### Shared Libraries (`lib/`)

The `lib/` directory contains reusable NestJS modules published as internal packages under the `@coco` scope:

| Library | Purpose |
|---------|---------|
| `@coco/common` | Core utilities, logging, authentication, exchanges, error handling |
| `@coco/state-client` | HTTP client for the State service |
| `@coco/fleet-client` | Client for Fleet/Config service (robot locations) |
| `@coco/deliveries-client` | Client for Deliveries service |
| `@coco/dispatch-engine-client` | Client for Dispatch Engine |
| `@coco/maps-client` | Client for Maps service |
| `@coco/merchant-client` | Client for Merchant configuration |
| `@coco/pilots-client` | Client for Pilots service |
| `@coco/routing-engine` | Routing algorithms and geometry utilities |
| `@coco/exchange-defs` | RabbitMQ exchange and routing key definitions |
| `@coco/types` | Shared TypeScript type definitions |
| `@coco/eslint-config` | Shared ESLint configurations |
| `@coco/shared-config` | Shared prettier and tsconfig |

### Backend Services (`service/`)

Each service is a standalone NestJS application with its own database schema:

| Service | Description | Database |
|---------|-------------|----------|
| `deliveries` | Core delivery lifecycle management | PostgreSQL (Prisma) |
| `operations` | Trips, tasks, hubs, pilots, robots | PostgreSQL (Prisma) |
| `state` | Robot state machine, connectivity, lid cycles | PostgreSQL (Prisma) |
| `device` | Device provisioning, deployments, groups | PostgreSQL (Prisma) |
| `integrations` | External partner integrations (Uber, DoorDash, Olo) | PostgreSQL (Prisma) |
| `dispatch-engine` | Supply/demand matching, route planning | PostgreSQL (Prisma) |
| `trip-monitor` | Real-time trip monitoring and route updates | PostgreSQL (Prisma) |
| `data-pipeline` | Analytics data processing | - |

### Frontend Applications (`web/`)

| Application | Description | Tech Stack |
|-------------|-------------|------------|
| `mission-control` | Internal operations dashboard | Next.js, React |
| `mro` | MRO (Maintenance, Repair, Operations) portal | Next.js, React |
| `merchant` | Merchant-facing delivery tracking | Next.js, React |
| `merchant-fleet` | Merchant fleet management | Next.js, React |
| `pilot` | Pilot mobile web app | Next.js, React |
| `tracker` | Customer delivery tracking | Next.js, React |
| `qr` | QR code scanning interface | Next.js, React |
| `maps` | Internal mapping tools | Next.js, React |
| `tasks` | Task management interface | Next.js, React |
| `mx-manager` | Merchant payment/financial portal | Next.js, React |

---

## coco-services Architecture (Go)

### Microservice Structure

Each Go service follows a consistent pattern:

```
service-name/
├── cmd/                  # CLI commands (Cobra)
├── internal/
│   ├── config/          # Service configuration
│   ├── consumer/        # Message queue consumers
│   ├── entities/        # Domain entities
│   ├── handlers/        # HTTP/gRPC handlers
│   ├── managers/        # Business logic orchestration
│   ├── orchestrators/   # Complex workflow coordination
│   └── repos/           # Data access layer
├── main.go              # Entry point
└── go.mod               # Go module definition
```

### Go Services

| Service | Description | Data Store |
|---------|-------------|------------|
| `fleet` | Fleet management, Uber integration, robot state | DynamoDB |
| `task` | External task management (field ops) | DynamoDB |
| `task-assigner` | Task assignment logic | - |
| `task-executor` | Task execution workflows | - |
| `config` | Configuration management (merchants, hubs, etc.) | PostgreSQL |
| `maps` | Mapping and geospatial data | PostgreSQL |
| `robot` | Robot heartbeat processing | DynamoDB |
| `mds` | Mobility Data Specification compliance | Redshift, Athena |
| `anomaly` | Anomaly detection | - |
| `sandbox` | Webhook testing/sandbox environment | - |
| `simulators` | Robot simulation for testing | - |

### Core Library (`core/`)

Shared Go packages used across all services:

| Package | Purpose |
|---------|---------|
| `auth` | JWT validation, role-based access |
| `cocodb` | PostgreSQL database utilities |
| `config` | Configuration management (Viper) |
| `dynamo` | DynamoDB utilities |
| `kafka` | Kafka producer/consumer |
| `kinesis` | Kinesis stream handling |
| `rabbitmq` | RabbitMQ producer/consumer |
| `redis` | Redis client utilities |
| `sqs` | SQS consumer utilities |
| `log` | Structured logging |
| `http` | HTTP server/client utilities |
| `profiler` | Metrics and profiling |
| `flags` | Feature flag integration (LaunchDarkly) |

---

## Service Communication Patterns

### 1. RabbitMQ (Primary Async Communication)

RabbitMQ is the primary message broker for event-driven communication between services.

#### Exchange Organization

Exchanges are organized by domain and defined in `@coco/common/exchanges/names.ts`:

```typescript
// Example exchange domains
enum Deliveries {
  DeliveryEvent = 'deliveries/delivery-event',
  AttemptUpdated = 'deliveries/attempt-updated',
  ProviderUpdated = 'deliveries/provider-updated',
  // ...
}

enum Operations {
  TaskCreated = 'operations/task-created',
  TripTransitioned = 'operations/trip-transitioned',
  // ...
}

enum State {
  RequestTransition = 'state/request-transition',
  TransitionCompleted = 'state/transition-completed',
  // ...
}

enum Devices {
  LidCommandSent = 'devices/lid-command-sent',
  DeviceProvisioned = 'devices/provisioned',
  // ...
}
```

#### Exchange Types

All exchanges use **topic** type for flexible routing:

```typescript
const exchanges: RabbitMQExchangeConfig[] = Object.values({
  ...Exchanges.Names.State,
  ...Exchanges.Names.IoT,
  ...Exchanges.Names.Devices,
  // ...
}).map((exchangeName) => ({
  type: 'topic',
  name: Exchanges.prefixWithEnv(exchangeName), // e.g., 'staging.deliveries/delivery-event'
}));
```

#### Message Flow Example

```
[Integrations Service] 
    → publishes to 'deliveries/delivery-event' exchange
    → [Deliveries Service] consumes via routing key pattern
    → processes delivery
    → publishes to 'operations/trip-created' exchange
    → [Operations Service] consumes
    → publishes to 'state/request-transition' exchange
    → [State Service] consumes and manages robot state
```

### 2. gRPC (Service-to-Service Sync)

Used primarily for synchronous calls to Go services:

```typescript
// TypeScript client using generated protos
import { ListRobotLocationsResponse } from '@coco/coco-cjs-gen/coco/config/fleet/service/v1/fleet_pb';
```

```go
// Go gRPC server implementation
func (t taskServer) Create(ctx context.Context, req *task.CreateRequest) (*task.CreateResponse, error) {
    newTask, err := t.taskOrchestrator.CreateAndEmit(ctx, req.NewTask)
    // ...
}
```

Proto definitions live in `coco-protos/coco/` organized by domain:
- `coco/task/v1/task.proto`
- `coco/fleet/v1/fleet.proto`
- `coco/robot/v1/robot.proto`
- `coco/notification/v1/*.proto`

### 3. HTTP REST APIs

#### Frontend → Backend

Web applications communicate with backend services via REST:

```typescript
// Frontend API client
const apiClient: ApiClient = {
  deliveries: deliveryApi,    // → deliveries service
  devices: deviceApi,         // → device service
  orders: ordersApi,          // → integrations/orders
  operations: operationsApi,  // → operations service
  robotState: robotStateApi,  // → state service
  maps: mapsApi,             // → maps service
};
```

#### Backend → Backend (HTTP Clients)

Services use typed HTTP clients for REST communication:

```typescript
// StateClientService example
@Injectable()
export class StateClientService {
  constructor(@Inject(STATE_CLIENT_OPTIONS) options: StateClientOptions) {
    this.client = got.extend({
      prefixUrl: options.baseUrl + '/state',
      headers: { 'x-api-key': options.apiKey },
    });
  }

  async lidCommandSync(deviceId: string, state: LidState): Promise<void> {
    await this.client.post(PATHS.LID_COMMAND_SYNC, { json: req });
  }
}
```

### 4. Kafka (High-Volume Streaming)

Used for high-throughput data streaming:
- Robot heartbeats
- Telemetry data
- Integration events from external systems

```go
// Go Kafka consumer
type Consumer interface {
    ReadMessage(timeout time.Duration) (*confluentkafka.Message, error)
    Shutdown(ctx context.Context) error
}

type Producer interface {
    Publish(ctx context.Context, topic string, body []byte) error
    Shutdown(ctx context.Context)
}
```

### 5. Real-Time Updates (Pusher)

For real-time frontend updates:

```typescript
// Service pushing updates to frontends
PusherModule.forRootAsync({
  inject: [AppConfigService],
  useFactory: (config: AppConfigService) => config.get('pusher'),
}),
```

---

## Data Architecture

### Database Strategy

Each service owns its data with **database-per-service** pattern:

| Service | Database | ORM/Client |
|---------|----------|------------|
| deliveries | PostgreSQL | Prisma |
| operations | PostgreSQL | Prisma |
| state | PostgreSQL | Prisma |
| device | PostgreSQL | Prisma |
| integrations | PostgreSQL | Prisma |
| dispatch-engine | PostgreSQL | Prisma |
| trip-monitor | PostgreSQL | Prisma |
| config (Go) | PostgreSQL | sqlx |
| maps (Go) | PostgreSQL | sqlx |
| fleet (Go) | DynamoDB | aws-sdk |
| task (Go) | DynamoDB | aws-sdk |
| robot (Go) | DynamoDB | aws-sdk |

### Prisma Schema Examples

Each NestJS service has its own Prisma schema defining its domain models:

**State Service** (`service/state/prisma/schema.prisma`):
- StateHistory, TransitionRequest, Connectivity, LidCycle

**Deliveries Service** (`service/deliveries/prisma/schema.prisma`):
- Delivery, Attempt, Quote, Customer, Dropoff, WatchdogSchema

**Operations Service** (`service/operations/prisma/schema.prisma`):
- Task, Trip, Hub, Robot, Pilot, Location

### Data Sharing Patterns

Services share data through:

1. **Events**: Primary method - services publish domain events
2. **API Calls**: For synchronous data needs
3. **Shared Caches**: Redis for frequently accessed data (robot locations, feature flags)
4. **Analytics Layer**: Redshift/DBT for cross-service analytics

```
[Service A] → publishes event → RabbitMQ → [Service B] → stores in own DB
                                         → [Service C] → stores in own DB
```

### Redis Usage

Shared Redis for:
- Feature flags (LaunchDarkly cache)
- Robot location caching (fleet-client)
- Session data
- Rate limiting
- Distributed locks (semaphores)

```typescript
BullModule.forRootAsync({
  useFactory: (config: AppConfigService) => ({
    connection: {
      port: redis.port,
      host: redis.host,
      // ...
    },
  }),
}),
```

---

## Deployment Architecture

### Kubernetes/Helm

Services are deployed via Helm charts:

```
infra/
├── configs/
│   ├── deliveries/values.yaml.tpl
│   ├── operations/values.yaml.tpl
│   ├── state/values.yaml.tpl
│   └── ...
└── helm/
    └── service/
        ├── Chart.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── virtualservice.yaml
```

### Environment Prefixing

All exchanges and queues are prefixed with environment:

```typescript
export function prefixWithEnv(exchangeName: string): string {
  return `${process.env.ENV}.${exchangeName}`;
}
// Result: 'staging.deliveries/delivery-event'
```

### Service Configuration

Go services use Viper for configuration:

```go
type Config struct {
    config.App
    config.AWS
    config.Log
    config.CocoDB
    config.Kafka
    config.Kinesis
    config.LaunchDarkly
    config.RabbitMQ
    config.Redis
    config.Slack
    config.Uber
    // Service-specific config
    DynamoDB DynamoDB `mapstructure:"dynamodb"`
}
```

---

## Module Organization (NestJS)

### Service Module Structure

Each NestJS service follows a modular architecture:

```
service/deliveries/src/
├── app.module.ts           # Root module
├── config/                 # Configuration module
├── prisma/                 # Database module
├── rabbitmq.module.ts      # Message broker module
├── shared/                 # Shared utilities
└── modules/
    ├── delivery/           # Delivery domain module
    │   ├── delivery.module.ts
    │   ├── delivery.handler.ts    # RabbitMQ handlers
    │   ├── controllers/           # HTTP endpoints
    │   ├── service/               # Business logic
    │   ├── repo/                  # Data access
    │   └── dto/                   # Data transfer objects
    ├── attempt/            # Attempt domain module
    ├── customer/           # Customer domain module
    ├── notifications/      # Notification handlers
    ├── providers/          # External provider integrations
    └── watchdog/           # Health monitoring
```

### Module Dependencies

```typescript
@Module({
  imports: [
    AppConfigModule,              // Configuration
    LoggerV2Module.registerAsync(...), // Logging
    BullModule.forRootAsync(...), // Job queues (Redis)
    FeatureFlagsModule.forRootAsync(...), // LaunchDarkly
    AuthModule,                   // Authentication
    PrismaModule.register(),      // Database
    AppRabbitMQModule,           // Message broker
    PusherModule.forRootAsync(...), // Real-time updates
    // Domain modules
    StateModule,
    ConnectivityModule,
    // ...
  ],
})
export class AppModule {}
```

---

## Key Architectural Patterns

### 1. Event-Driven Architecture

Services communicate primarily through events:

```
Trigger → Service A → Event → RabbitMQ → Service B → Event → RabbitMQ → Service C
```

### 2. Database Per Service

Each service owns and manages its own database schema. No direct database sharing between services.

### 3. API Gateway Pattern

Frontend applications access backend services through dedicated API endpoints, not direct database access.

### 4. Shared Libraries

Common functionality is extracted to `lib/` packages and shared across services via npm workspaces.

### 5. Proto-First for Go Services

Go services define contracts in protobuf, generating clients for both Go and TypeScript.

### 6. Feature Flags

LaunchDarkly integration across all services for gradual rollouts and A/B testing.

---

## Technology Stack Summary

| Layer | Technology |
|-------|------------|
| Frontend | Next.js, React, TypeScript |
| Backend (TS) | NestJS, TypeScript |
| Backend (Go) | Go, Cobra, Viper |
| Primary DB | PostgreSQL |
| NoSQL DB | DynamoDB |
| Message Broker | RabbitMQ |
| Streaming | Kafka, Kinesis |
| Cache | Redis |
| Real-time | Pusher |
| ORM | Prisma (TS), sqlx (Go) |
| API | REST, gRPC, WebSockets |
| Infrastructure | Kubernetes, Helm, Terraform |
| Monitoring | Datadog, Prometheus |
| Feature Flags | LaunchDarkly |
| Analytics | Redshift, DBT |
