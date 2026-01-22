diff --git a/lib/common/src/exchanges/payload.types.ts b/lib/common/src/exchanges/payload.types.ts
index 8ed3c12a2d..2d5605c07d 100644
--- a/lib/common/src/exchanges/payload.types.ts
+++ b/lib/common/src/exchanges/payload.types.ts
@@ -401,6 +401,7 @@ export interface DeviceGpsLocation {
   horizontalAccuracy: number;
   altitude: number;
   speedMph: number;
+  updatedAt: number;
 }
 
 export interface DeviceBatterySnapshot {
@@ -548,6 +549,13 @@ export interface DeviceCargoChangedEvent {
   changedAt: number;
 }
 
+// These TripEvents.* are intentionally separate from Operations.TripTransitioned:
+// - Design goal: events should be specific, action-oriented, and carry context (intent + timing).
+// - TripEvents.* are serial-routed (routing key = serial) and include timing/cargo fields:
+//   startedAt/resumedAt/cancelledAt/completedAt, hasCargo, and cancel reason.
+// - Operations.TripTransitioned is generic and repository-layered (DB state change only), with
+//   payload = Shared.TripWithTask + inRescueMode and routing key = tripType.tripStatus.stopType.
+// TODO: Consider unifying once TripTransitioned can carry action context (intent + timestamps).
 export interface TripStartedEvent {
   serial: string;
   tripId: string;
@@ -578,6 +586,10 @@ export interface TripCompletedEvent {
   completedAt: number;
 }
 
+// Added to publish deployment state at the service layer with location context.
+// Overlaps: `Names.Robots.Deployment` (`lib/common/src/exchanges/names.ts`),
+// `Names.State.TransitionCompleted` (`lib/common/src/exchanges/names.ts`).
+// TODO: consolidate overlapping robot-state/deployment signals into a single canonical event.
 export interface DeploymentDeployedEvent {
   serial: string;
   locationId: string;
@@ -591,6 +603,12 @@ export interface DeploymentUndeployedEvent {
   endedAt: number;
 }
 
+// Added to publish FO task effects on robot state (maintenance/pickup flags).
+// Overlaps: `Names.Operations.FoTaskCreated` (`lib/common/src/exchanges/names.ts`),
+// `Names.Operations.FoTaskStarted` (`lib/common/src/exchanges/names.ts`),
+// `Names.Operations.FoTaskCompleted` (`lib/common/src/exchanges/names.ts`),
+// `Names.Operations.FoTaskCanceled` (`lib/common/src/exchanges/names.ts`).
+// TODO: unify with other robot-state streams to reduce duplicate event types.
 export interface FoTaskCreatedEvent {
   serial: string;
   taskId: string;
@@ -613,6 +631,10 @@ export interface FoTaskUpdatedEvent {
   undergoingMaintenance?: boolean;
 }
 
+// Added to publish delivery load/unload as explicit robot state signals.
+// Overlaps: `Names.Deliveries.AttemptTransitioned` (`lib/common/src/exchanges/names.ts`),
+// `Names.Deliveries.ProviderUpdated` (`lib/common/src/exchanges/names.ts`).
+// TODO: converge delivery/robot-state events so this can be removed.
 export interface DeliveryLoadedEvent {
   serial: string;
   deliveryId: string;
diff --git a/service/deliveries/src/modules/delivery/service/delivery.service.ts b/service/deliveries/src/modules/delivery/service/delivery.service.ts
index 20bbcec87f..9d6c61dc16 100644
--- a/service/deliveries/src/modules/delivery/service/delivery.service.ts
+++ b/service/deliveries/src/modules/delivery/service/delivery.service.ts
@@ -34,7 +34,6 @@ import {
 } from '@coco/types/deliveries';
 import { QuoteLimitedError } from '@coco/types/errors';
 import { IntegrationRequestedHandoffPayload } from '@coco/types/integrations';
-import { TripType } from '@coco/types/operations';
 import { LidEventReason, LidEventSource } from '@coco/types/state';
 
 import { DropoffService } from './dropoff.service';
@@ -1383,12 +1382,6 @@ export class DeliveryService {
             return new Error('robot provider missing bot serial, should never happen');
           }
 
-          await this.publisher.publishRobotStateUpdated({
-            serial: delivery.rescueAttempt.driverName,
-            hasFood: false,
-            tripType: TripType.DELIVERY,
-          });
-
           break;
         case AttemptStatus.Canceled:
           return await this.handleRescueAttemptCourierCanceledUpdate(update, delivery, attempt);
diff --git a/service/deliveries/src/modules/providers/robot/robot.service.ts b/service/deliveries/src/modules/providers/robot/robot.service.ts
index 987bd3729a..4a32937054 100644
--- a/service/deliveries/src/modules/providers/robot/robot.service.ts
+++ b/service/deliveries/src/modules/providers/robot/robot.service.ts
@@ -22,7 +22,6 @@ import {
   LimitingFactor,
   LocationType,
 } from '@coco/types/dispatch-engine';
-import { RobotStateEventState, TripType } from '@coco/types/operations';
 import { LidEventReason, LidEventSource, LidState } from '@coco/types/state';
 
 import { OperationsService } from './operations.service';
@@ -562,15 +561,6 @@ export class RobotProviderService implements IDeliveryProvider {
       driverName: deviceId,
     });
 
-    await this.publisher.publishRobotStateUpdated({
-      serial: deviceId,
-      hasFood: true,
-      needsMovement: true,
-      tripType: TripType.DELIVERY,
-      operationState: RobotStateEventState.ON_TRIP,
-      attemptCancellationReason: null,
-    });
-
     await this.publisher.publishDeliveryLoaded({
       serial: deviceId,
       deliveryId: params.deliveryId,
diff --git a/service/deliveries/src/shared/amqp-publisher.service.ts b/service/deliveries/src/shared/amqp-publisher.service.ts
index 715a23e3b6..116f300971 100644
--- a/service/deliveries/src/shared/amqp-publisher.service.ts
+++ b/service/deliveries/src/shared/amqp-publisher.service.ts
@@ -78,14 +78,6 @@ export class AmqpPublisher {
     return this.amqp.publish(Exchanges.prefixWithEnv(exchange), routingKey, payload);
   }
 
-  public async publishRobotStateUpdated(
-    payload: Exchanges.PayloadOf<Exchanges.Names.Robots.StateChange>,
-  ): Promise<void> {
-    this.logger.log({ message: 'publishRobotStateUpdated called, publishing payload', payload });
-    await this.amqp.publish(Exchanges.prefixWithEnv(Exchanges.Names.Robots.StateChange), '', payload);
-    this.logger.log({ message: 'publishRobotStateUpdated called, payload published', payload });
-  }
-
   public async publishDeliveryLoaded(
     payload: Exchanges.PayloadOf<Exchanges.Names.DeliveryEvents.Loaded>,
   ): Promise<void> {
diff --git a/service/device/src/events/device-heartbeat.event.ts b/service/device/src/events/device-heartbeat.event.ts
index 590d7d7f65..39b0cc98ad 100644
--- a/service/device/src/events/device-heartbeat.event.ts
+++ b/service/device/src/events/device-heartbeat.event.ts
@@ -22,6 +22,7 @@ export interface DeviceGpsLocation {
   horizontalAccuracy: number;
   altitude: number;
   speedMph: number;
+  updatedAt: number;
 }
 
 export interface DeviceBatterySnapshot {
diff --git a/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts b/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts
index 068ee66f0c..a9a041f506 100644
--- a/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts
+++ b/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts
@@ -11,6 +11,7 @@ import {
   ResourceType,
   RobotSupplyEventPayload,
 } from '@coco/types/dispatch-engine';
+import { RobotStateEventState, TripType } from '@coco/types/operations';
 
 import { AmqpPublisher } from '../../../shared/amqp-publisher.service';
 import { SemaphoreService } from '../../semaphore/service/semaphore.service';
@@ -95,21 +96,56 @@ export class RobotEventsHandler {
     tracer.dogstatsd?.increment('dispatch.engine.handler.robot.deployment.success', 1);
   }
 
-  // Operational State (will be rerouted to device/fleet later)
   @UseCls()
-  @RabbitSubscribeWithDlqAndRetry(Exchanges.Names.Robots.StateChange, '#', 'dispatch-engine/stage-change')
-  async handleRobotStateChange(payload: Exchanges.PayloadOf<Exchanges.Names.Robots.StateChange>): Promise<void> {
-    tracer.dogstatsd?.increment('dispatch.engine.handler.robot.state.change.invoked', 1);
-    this.logger.debug('received robot state change data', { extra: payload });
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Started,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Started]({ serial: '#' }),
+    'dispatch-engine/trip-started',
+  )
+  async handleTripStarted(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Started>): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.startedAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: null,
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.TripEvents.Started,
+        note: 'Trip started',
+        state: RobotStateEventState.ON_TRIP,
+      },
+      needsMovement: true,
+      hasFood: payload.hasCargo ?? null,
+      tripType: payload.tripType,
+      needsMaintenance: null,
+      needsPickup: null,
+      driveable: null,
+      operationState: RobotStateEventState.ON_TRIP,
+      attemptCancellationReason: null,
+      undergoingMaintenance: null,
+    };
 
-    if (!(await this.semaphore.gotLockWithRetry('robot', payload.serial))) {
-      this.logger.error('could not acquire lock', { extra: payload });
-      tracer.dogstatsd?.increment('dispatch.engine.handler.robot.state.change.failure', 1);
-      return;
-    }
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
 
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Resumed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Resumed]({ serial: '#' }),
+    'dispatch-engine/trip-resumed',
+  )
+  async handleTripResumed(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Resumed>): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
     const robotSupplyEvent: RobotSupplyEventPayload = {
-      timestamp: new Date().getTime(),
+      timestamp: payload.resumedAt,
       battery: null,
       healthy: null,
       type: ResourceType.Robot,
@@ -117,17 +153,221 @@ export class RobotEventsHandler {
       location: null,
       features: {
         robotSerial: payload.serial,
-        event: Exchanges.Names.Robots.StateChange,
-        note: 'Robot state change',
+        event: Exchanges.Names.TripEvents.Resumed,
+        note: 'Trip resumed',
+        state: RobotStateEventState.ON_TRIP,
+      },
+      needsMovement: true,
+      hasFood: null,
+      tripType: payload.tripType,
+      needsMaintenance: null,
+      needsPickup: null,
+      driveable: null,
+      operationState: RobotStateEventState.ON_TRIP,
+      attemptCancellationReason: null,
+      undergoingMaintenance: null,
+    };
+
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
+
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Cancelled,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Cancelled]({ serial: '#' }),
+    'dispatch-engine/trip-cancelled',
+  )
+  async handleTripCancelled(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Cancelled>): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.cancelledAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: null,
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.TripEvents.Cancelled,
+        note: 'Trip cancelled',
+        state: RobotStateEventState.PARKED,
+      },
+      needsMovement: null,
+      hasFood: null,
+      tripType: TripType.NONE,
+      needsMaintenance: null,
+      needsPickup: null,
+      driveable: null,
+      operationState: RobotStateEventState.PARKED,
+      attemptCancellationReason: null,
+      undergoingMaintenance: null,
+    };
+
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
+
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Completed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Completed]({ serial: '#' }),
+    'dispatch-engine/trip-completed',
+  )
+  async handleTripCompleted(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Completed>): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.completedAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: null,
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.TripEvents.Completed,
+        note: 'Trip completed',
+        state: RobotStateEventState.PARKED,
+      },
+      needsMovement: null,
+      hasFood: false,
+      tripType: TripType.NONE,
+      needsMaintenance: null,
+      needsPickup: null,
+      driveable: null,
+      operationState: RobotStateEventState.PARKED,
+      attemptCancellationReason: null,
+      undergoingMaintenance: null,
+    };
+
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
+
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeploymentEvents.Deployed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeploymentEvents.Deployed]({ serial: '#' }),
+    'dispatch-engine/deployment-started',
+  )
+  async handleDeploymentStarted(
+    payload: Exchanges.PayloadOf<Exchanges.Names.DeploymentEvents.Deployed>,
+  ): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.startedAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: this.mapDeploymentLocation(payload.location),
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.DeploymentEvents.Deployed,
+        note: 'Robot deployed',
+        state: RobotStateEventState.DEPLOYED,
       },
-      needsMovement: payload.needsMovement ?? null,
-      hasFood: payload.hasFood ?? null,
-      tripType: payload.tripType ?? null,
+      needsMovement: null,
+      hasFood: null,
+      tripType: null,
+      needsMaintenance: null,
+      needsPickup: null,
+      driveable: null,
+      operationState: RobotStateEventState.DEPLOYED,
+      attemptCancellationReason: null,
+      undergoingMaintenance: null,
+    };
+
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
+
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeploymentEvents.Undeployed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeploymentEvents.Undeployed]({ serial: '#' }),
+    'dispatch-engine/deployment-ended',
+  )
+  async handleDeploymentEnded(
+    payload: Exchanges.PayloadOf<Exchanges.Names.DeploymentEvents.Undeployed>,
+  ): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.endedAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: null,
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.DeploymentEvents.Undeployed,
+        note: 'Robot undeployed',
+        state: RobotStateEventState.OFF_DUTY,
+      },
+      needsMovement: null,
+      hasFood: null,
+      tripType: null,
+      needsMaintenance: null,
+      needsPickup: null,
+      driveable: null,
+      operationState: RobotStateEventState.OFF_DUTY,
+      attemptCancellationReason: null,
+      undergoingMaintenance: null,
+    };
+
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
+
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.FoTaskEvents.Created,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.FoTaskEvents.Created]({ serial: '#' }),
+    'dispatch-engine/fo-task-created',
+  )
+  async handleFoTaskCreated(payload: Exchanges.PayloadOf<Exchanges.Names.FoTaskEvents.Created>): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.createdAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: null,
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.FoTaskEvents.Created,
+        note: 'FO task created',
+      },
+      needsMovement: null,
+      hasFood: null,
+      tripType: null,
       needsMaintenance: payload.needsMaintenance ?? null,
       needsPickup: payload.needsPickup ?? null,
-      driveable: payload.driveable ?? null,
-      operationState: payload.operationState ?? null,
-      attemptCancellationReason: payload.attemptCancellationReason ?? null,
+      driveable: null,
+      operationState: null,
+      attemptCancellationReason: null,
       undergoingMaintenance: payload.undergoingMaintenance ?? null,
     };
 
@@ -137,8 +377,148 @@ export class RobotEventsHandler {
       scope: EventScope.EXTERNAL,
       type: EventType.Supply,
     });
+  }
 
-    await this.semaphore.clearLock('robot', payload.serial);
-    tracer.dogstatsd?.increment('dispatch.engine.handler.robot.state.change.success', 1);
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.FoTaskEvents.Updated,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.FoTaskEvents.Updated]({ serial: '#' }),
+    'dispatch-engine/fo-task-updated',
+  )
+  async handleFoTaskUpdated(payload: Exchanges.PayloadOf<Exchanges.Names.FoTaskEvents.Updated>): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.updatedAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: null,
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.FoTaskEvents.Updated,
+        note: 'FO task updated',
+      },
+      needsMovement: null,
+      hasFood: null,
+      tripType: null,
+      needsMaintenance: payload.needsMaintenance ?? null,
+      needsPickup: payload.needsPickup ?? null,
+      driveable: null,
+      operationState: null,
+      attemptCancellationReason: null,
+      undergoingMaintenance: payload.undergoingMaintenance ?? null,
+    };
+
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
+
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeliveryEvents.Loaded,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeliveryEvents.Loaded]({ serial: '#' }),
+    'dispatch-engine/delivery-loaded',
+  )
+  async handleDeliveryLoaded(payload: Exchanges.PayloadOf<Exchanges.Names.DeliveryEvents.Loaded>): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.loadedAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: null,
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.DeliveryEvents.Loaded,
+        note: 'Delivery loaded',
+        state: RobotStateEventState.ON_TRIP,
+      },
+      needsMovement: true,
+      hasFood: true,
+      tripType: TripType.DELIVERY,
+      needsMaintenance: null,
+      needsPickup: null,
+      driveable: null,
+      operationState: RobotStateEventState.ON_TRIP,
+      attemptCancellationReason: null,
+      undergoingMaintenance: null,
+    };
+
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
+
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeliveryEvents.Unloaded,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeliveryEvents.Unloaded]({ serial: '#' }),
+    'dispatch-engine/delivery-unloaded',
+  )
+  async handleDeliveryUnloaded(payload: Exchanges.PayloadOf<Exchanges.Names.DeliveryEvents.Unloaded>): Promise<void> {
+    this.logger.withClsFields({ deviceId: payload.serial });
+    const robotSupplyEvent: RobotSupplyEventPayload = {
+      timestamp: payload.unloadedAt,
+      battery: null,
+      healthy: null,
+      type: ResourceType.Robot,
+      robotSerial: payload.serial,
+      location: null,
+      features: {
+        robotSerial: payload.serial,
+        event: Exchanges.Names.DeliveryEvents.Unloaded,
+        note: 'Delivery unloaded',
+      },
+      needsMovement: null,
+      hasFood: false,
+      tripType: null,
+      needsMaintenance: null,
+      needsPickup: null,
+      driveable: null,
+      operationState: null,
+      attemptCancellationReason: null,
+      undergoingMaintenance: null,
+    };
+
+    await this.publisher.publish(Exchanges.Names.DispatchEngine.SupplyEvent, {
+      supply: robotSupplyEvent,
+      source: EventSinkSource.OPERATIONS_ROBOTS,
+      scope: EventScope.EXTERNAL,
+      type: EventType.Supply,
+    });
+  }
+
+  private mapDeploymentLocation(
+    location: Exchanges.PayloadOf<Exchanges.Names.DeploymentEvents.Deployed>['location'],
+  ): RobotSupplyEventPayload['location'] | null {
+    if (!location) return null;
+    let locationType = LocationType.ParkingLot;
+    if (location.type) {
+      switch (location.type) {
+        case 'STORAGE':
+          locationType = LocationType.Pod;
+          break;
+        case 'PARKING_LOT':
+          locationType = LocationType.ParkingLot;
+          break;
+      }
+    }
+    return {
+      type: locationType,
+      robotLocationId: location.id,
+      geo: {
+        latitude: location.latitude,
+        longitude: location.longitude,
+      },
+    };
   }
 }
diff --git a/service/operations/src/modules/fleet-management/services/robot-fleet-management.service.ts b/service/operations/src/modules/fleet-management/services/robot-fleet-management.service.ts
index 3cca59eda4..e9b23f9c89 100644
--- a/service/operations/src/modules/fleet-management/services/robot-fleet-management.service.ts
+++ b/service/operations/src/modules/fleet-management/services/robot-fleet-management.service.ts
@@ -6,7 +6,7 @@ import { ILogger, RobotLocation, RobotState, RobotStateEvent } from '@coco/commo
 import { DispatchEngineClientService } from '@coco/dispatch-engine-client';
 import { FleetClientService } from '@coco/fleet-client';
 import { DeploymentBody } from '@coco/types/dispatch-engine';
-import { RobotStateEventState, TripType } from '@coco/types/operations';
+import { RobotStateEventState } from '@coco/types/operations';
 
 import { PartnersService } from 'src/modules/partners/services/partners.service';
 import { PublisherService } from 'src/modules/publisher/services/publisher.service';
@@ -118,13 +118,6 @@ export class RobotFleetManagementService {
         });
       });
 
-    await this.operationsPublisher.publishRobotStateUpdated({
-      serial: botSerial,
-      tripType: TripType.NONE,
-      operationState: RobotStateEventState.OFF_DUTY,
-      attemptCancellationReason: null,
-    });
-
     await this.operationsPublisher.publishDeploymentUndeployed({
       serial: botSerial,
       locationId: storageLocation.id,
@@ -340,12 +333,6 @@ export class RobotFleetManagementService {
                 serial,
                 error,
               });
-            }),
-            await this.operationsPublisher.publishRobotStateUpdated({
-              serial: serial,
-              tripType: TripType.NONE,
-              operationState: RobotStateEventState.PARKED,
-              attemptCancellationReason: null,
             });
         }),
       );
diff --git a/service/operations/src/modules/fo-requests/fo-requests.service.ts b/service/operations/src/modules/fo-requests/fo-requests.service.ts
index cc6879e16e..52ce89e33b 100644
--- a/service/operations/src/modules/fo-requests/fo-requests.service.ts
+++ b/service/operations/src/modules/fo-requests/fo-requests.service.ts
@@ -149,12 +149,6 @@ export class FoRequestsService {
       notes,
     });
 
-    await this.operationsPublisher.publishRobotStateUpdated({
-      serial: trip.botSerial,
-      tripType: trip.type,
-      attemptCancellationReason: this.mapPAttemptCancellationReason(attemptCancellationReason),
-    });
-
     const deliveryRecoverable = await this.tripRecoverable(attemptCancellationReason);
     if (type == FoAssistanceRequestType.BOT_RESCUE && trip.type == TripType.DELIVERY) {
       if (trip.attemptId == null) {
diff --git a/service/operations/src/modules/fo-tasks/fo-tasks.service.ts b/service/operations/src/modules/fo-tasks/fo-tasks.service.ts
index 1c4f9295df..7b51b9a75e 100644
--- a/service/operations/src/modules/fo-tasks/fo-tasks.service.ts
+++ b/service/operations/src/modules/fo-tasks/fo-tasks.service.ts
@@ -173,12 +173,6 @@ export class FoTasksService {
           break;
       }
 
-      await this.operationsPublisher.publishRobotStateUpdated({
-        serial: options.botSerial,
-        needsMaintenance,
-        needsPickup,
-      });
-
       await this.operationsPublisher.publishFoTaskCreated({
         serial: options.botSerial,
         taskId: task.id,
@@ -320,11 +314,6 @@ export class FoTasksService {
       if (currentSubtask.type === FoSubtaskType.TRAVEL && currentSubtask.destinationType == FoSubtaskLocationType.BOT) {
         const botSerial = FoTasksService.getBotSerialForTask(task);
         if (botSerial) {
-          await this.operationsPublisher.publishRobotStateUpdated({
-            serial: botSerial,
-            undergoingMaintenance: true,
-          });
-
           await this.operationsPublisher.publishFoTaskUpdated({
             serial: botSerial,
             taskId: task.id,
@@ -1151,12 +1140,6 @@ export class FoTasksService {
     const botSerial = FoTasksService.getBotSerialForTask(task);
     if (botSerial) {
       const needsMaintenance = await this.shouldTasksSetNeedsMaintainenceOnBot(botSerial);
-      await this.operationsPublisher.publishRobotStateUpdated({
-        serial: botSerial,
-        needsMaintenance: needsMaintenance,
-        undergoingMaintenance: false,
-      });
-
       await this.operationsPublisher.publishFoTaskUpdated({
         serial: botSerial,
         taskId: task.id,
diff --git a/service/operations/src/modules/fo-tasks/handlers/delivery.handlers.ts b/service/operations/src/modules/fo-tasks/handlers/delivery.handlers.ts
index f0763fc54e..0a5308ae40 100644
--- a/service/operations/src/modules/fo-tasks/handlers/delivery.handlers.ts
+++ b/service/operations/src/modules/fo-tasks/handlers/delivery.handlers.ts
@@ -5,7 +5,6 @@ import { Exchanges, ILogger, RabbitSubscribeWithDlqAndRetry } from '@coco/common
 import { AttemptStatus, DeliveryEventType, DeliveryStatus, Provider } from '@coco/types/deliveries';
 import { FoTaskType } from '@coco/types/operations';
 
-import { PublisherService } from 'src/modules/publisher/services/publisher.service';
 import { FoTasksService } from '../fo-tasks.service';
 
 @Injectable()
@@ -13,7 +12,6 @@ export class FoTaskDeliveryHandlers {
   constructor(
     private readonly logger: ILogger,
     private readonly service: FoTasksService,
-    private readonly publisher: PublisherService,
   ) {
     this.logger.setContext(this.constructor.name);
   }
@@ -60,13 +58,6 @@ export class FoTaskDeliveryHandlers {
       return;
     }
 
-    const needsMaintenance = await this.service.shouldTasksSetNeedsMaintainenceOnBot(robotSerial);
-    await this.publisher.publishRobotStateUpdated({ serial: robotSerial, needsMaintenance: needsMaintenance });
-    this.logger.debug({
-      message: '[handleDeliveryTermination] published robot state updated',
-      robotSerial,
-      needsMaintenance,
-    });
   }
 
   // we are not subscribing to attempt transitions because (1) it's a legacy event (2) it doesn't contain bot serial
@@ -100,7 +91,6 @@ export class FoTaskDeliveryHandlers {
     // attempt was rescued, empty bot task is safe to cancel
     await this.service.cancelTasksForDelivery(payload.delivery.id, FoTaskType.EMPTY_BOT);
 
-    // update robot state accordingly
     const robotSerial = robotAttempt.driverName;
     if (!robotSerial) {
       this.logger.warn({
@@ -109,7 +99,5 @@ export class FoTaskDeliveryHandlers {
       });
       return;
     }
-    const needsMaintenance = await this.service.shouldTasksSetNeedsMaintainenceOnBot(robotSerial);
-    await this.publisher.publishRobotStateUpdated({ serial: robotSerial, needsMaintenance: needsMaintenance });
   }
 }
diff --git a/service/operations/src/modules/publisher/services/publisher.service.ts b/service/operations/src/modules/publisher/services/publisher.service.ts
index 8b1faf2158..5a17d04a4f 100644
--- a/service/operations/src/modules/publisher/services/publisher.service.ts
+++ b/service/operations/src/modules/publisher/services/publisher.service.ts
@@ -12,14 +12,6 @@ export class PublisherService {
     this.logger.setContext(PublisherService.name);
   }
 
-  public async publishRobotStateUpdated(
-    payload: Exchanges.PayloadOf<Exchanges.Names.Robots.StateChange>,
-  ): Promise<void> {
-    this.logger.debug({ message: 'publishRobotStateUpdated called, publishing payload', payload });
-    await this.amqp.publish(Exchanges.prefixWithEnv(Exchanges.Names.Robots.StateChange), '', payload);
-    this.logger.debug({ message: 'publishRobotStateUpdated called, payload published', payload });
-  }
-
   public async publishTripStarted(
     payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Started>,
   ): Promise<void> {
diff --git a/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts b/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts
index 90176e1454..10c9ebc241 100644
--- a/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts
+++ b/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts
@@ -1,6 +1,7 @@
 import { Injectable } from '@nestjs/common';
 
 import { Exchanges, ILogger, RabbitSubscribeWithDlqAndRetry, RobotConnectivity } from '@coco/common';
+import { RobotStateEventState, TripType } from '@coco/types/operations';
 
 import { RobotEphemeralDataService } from '../services/robot-ephemeral-data.service';
 import { RobotsService } from '../services/robots.service';
@@ -48,22 +49,151 @@ export class RobotStateChangeHandler {
     await this.robotEphemeralDataService.setConnectivity(data.serial, data.overall.overallStatus === Status.ONLINE);
   }
 
-  // Operational State (will be rerouted to device/fleet later)
-  @RabbitSubscribeWithDlqAndRetry(Exchanges.Names.Robots.StateChange, '#', 'operations/robot-stage-change')
-  async handleRobotState(payload: Exchanges.PayloadOf<Exchanges.Names.Robots.StateChange>) {
-    this.logger.debug({ message: 'operations: received robot state data', data: payload });
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Started,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Started]({ serial: '#' }),
+    'operations/trip-started',
+  )
+  async handleTripStarted(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Started>) {
+    this.logger.debug({ message: 'operations: received trip started', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
+      needsMovement: true,
+      requiresPilot: true,
+      hasFood: payload.hasCargo,
+      tripType: payload.tripType,
+      operationState: RobotStateEventState.ON_TRIP,
+    });
+  }
 
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Resumed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Resumed]({ serial: '#' }),
+    'operations/trip-resumed',
+  )
+  async handleTripResumed(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Resumed>) {
+    this.logger.debug({ message: 'operations: received trip resumed', data: payload });
     await this.robotService.updateRobotState(payload.serial, {
       serial: payload.serial,
-      needsMovement: payload.needsMovement,
-      requiresPilot: payload.needsMovement,
-      hasFood: payload.hasFood,
+      needsMovement: true,
+      requiresPilot: true,
       tripType: payload.tripType,
-      operationState: payload.operationState,
+      operationState: RobotStateEventState.ON_TRIP,
+    });
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Cancelled,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Cancelled]({ serial: '#' }),
+    'operations/trip-cancelled',
+  )
+  async handleTripCancelled(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Cancelled>) {
+    this.logger.debug({ message: 'operations: received trip cancelled', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
+      tripType: TripType.NONE,
+      operationState: RobotStateEventState.PARKED,
+    });
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Completed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Completed]({ serial: '#' }),
+    'operations/trip-completed',
+  )
+  async handleTripCompleted(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Completed>) {
+    this.logger.debug({ message: 'operations: received trip completed', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
+      hasFood: false,
+      tripType: TripType.NONE,
+      operationState: RobotStateEventState.PARKED,
+    });
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeploymentEvents.Deployed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeploymentEvents.Deployed]({ serial: '#' }),
+    'operations/deployment-started',
+  )
+  async handleDeploymentStarted(payload: Exchanges.PayloadOf<Exchanges.Names.DeploymentEvents.Deployed>) {
+    this.logger.debug({ message: 'operations: received deployment started', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
+      operationState: RobotStateEventState.DEPLOYED,
+    });
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeploymentEvents.Undeployed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeploymentEvents.Undeployed]({ serial: '#' }),
+    'operations/deployment-ended',
+  )
+  async handleDeploymentEnded(payload: Exchanges.PayloadOf<Exchanges.Names.DeploymentEvents.Undeployed>) {
+    this.logger.debug({ message: 'operations: received deployment ended', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
+      operationState: RobotStateEventState.OFF_DUTY,
+    });
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.FoTaskEvents.Created,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.FoTaskEvents.Created]({ serial: '#' }),
+    'operations/fo-task-created',
+  )
+  async handleFoTaskCreated(payload: Exchanges.PayloadOf<Exchanges.Names.FoTaskEvents.Created>) {
+    this.logger.debug({ message: 'operations: received fo task created', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
+      needsMaintenance: payload.needsMaintenance,
+      needsPickup: payload.needsPickup,
+      undergoingMaintenance: payload.undergoingMaintenance,
+    });
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.FoTaskEvents.Updated,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.FoTaskEvents.Updated]({ serial: '#' }),
+    'operations/fo-task-updated',
+  )
+  async handleFoTaskUpdated(payload: Exchanges.PayloadOf<Exchanges.Names.FoTaskEvents.Updated>) {
+    this.logger.debug({ message: 'operations: received fo task updated', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
       needsMaintenance: payload.needsMaintenance,
       needsPickup: payload.needsPickup,
       undergoingMaintenance: payload.undergoingMaintenance,
-      driveable: payload.driveable,
+    });
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeliveryEvents.Loaded,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeliveryEvents.Loaded]({ serial: '#' }),
+    'operations/delivery-loaded',
+  )
+  async handleDeliveryLoaded(payload: Exchanges.PayloadOf<Exchanges.Names.DeliveryEvents.Loaded>) {
+    this.logger.debug({ message: 'operations: received delivery loaded', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
+      hasFood: true,
+      needsMovement: true,
+      requiresPilot: true,
+      tripType: TripType.DELIVERY,
+      operationState: RobotStateEventState.ON_TRIP,
+    });
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeliveryEvents.Unloaded,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeliveryEvents.Unloaded]({ serial: '#' }),
+    'operations/delivery-unloaded',
+  )
+  async handleDeliveryUnloaded(payload: Exchanges.PayloadOf<Exchanges.Names.DeliveryEvents.Unloaded>) {
+    this.logger.debug({ message: 'operations: received delivery unloaded', data: payload });
+    await this.robotService.updateRobotState(payload.serial, {
+      serial: payload.serial,
+      hasFood: false,
     });
   }
 }
diff --git a/service/operations/src/modules/robots/services/robots.service.ts b/service/operations/src/modules/robots/services/robots.service.ts
index 9141f25947..c1c67e17ea 100644
--- a/service/operations/src/modules/robots/services/robots.service.ts
+++ b/service/operations/src/modules/robots/services/robots.service.ts
@@ -819,15 +819,8 @@ export class RobotsService {
     if (botSerial == null)
       throw new NotFoundException(`Could not find robot serial for assistance request ${requestId}`);
 
-    const err = await this.groundBot(botSerial, requestId, assistanceRequest.notes);
+    await this.groundBot(botSerial, requestId, assistanceRequest.notes);
 
-    if (err === null) {
-      await this.operationsPublisher.publishRobotStateUpdated({
-        serial: botSerial,
-        operationState: RobotStateEventState.GROUNDED,
-        tripType: TripType.NONE,
-      });
-    }
   }
 
   private async groundBot(serial: string, requestId?: string, reason = ''): Promise<Error | null> {
@@ -846,12 +839,6 @@ export class RobotsService {
         return new Error('error grounding bot');
       });
 
-    await this.operationsPublisher.publishRobotStateUpdated({
-      serial: serial,
-      operationState: RobotStateEventState.GROUNDED,
-      tripType: TripType.NONE,
-    });
-
     return null;
   }
 
diff --git a/service/operations/src/modules/trips/publishers/pilot-trips.publisher.ts b/service/operations/src/modules/trips/publishers/pilot-trips.publisher.ts
index 1621279b0d..d5104ac07c 100644
--- a/service/operations/src/modules/trips/publishers/pilot-trips.publisher.ts
+++ b/service/operations/src/modules/trips/publishers/pilot-trips.publisher.ts
@@ -3,7 +3,7 @@ import { Injectable } from '@nestjs/common';
 
 import { Exchanges, ILogger } from '@coco/common';
 import { StopType } from '@coco/exchange-defs/dist/exchanges/legacy-trip/shared/legacy-shared';
-import { FoAssistanceRequestType, RobotStateEventState, TripType } from '@coco/types/operations';
+import { FoAssistanceRequestType } from '@coco/types/operations';
 
 import { PublisherService } from 'src/modules/publisher/services/publisher.service';
 import { PrismaService } from '../../../core/prisma/prisma.service';
@@ -134,14 +134,6 @@ export class PilotTripsPublisher {
     await this.publishTaskTransitioned(mapTripToTaskPayload(data));
     await this.publishTripTransitioned(mapTripToTripPayload(data));
     if (data.robotSerial) {
-      await this.operationsPublisher.publishRobotStateUpdated({
-        serial: data.robotSerial,
-        operationState: RobotStateEventState.PARKED,
-        tripType: TripType.NONE,
-        attemptCancellationReason: null,
-        hasFood: false,
-      });
-
       await this.operationsPublisher.publishTripCompleted({
         serial: data.robotSerial,
         tripId: data.id,
@@ -155,13 +147,6 @@ export class PilotTripsPublisher {
     await this.publishTaskTransitioned(mapTripToTaskPayload(data));
     await this.publishTripTransitioned(mapTripToTripPayload(data));
     if (data.robotSerial) {
-      await this.operationsPublisher.publishRobotStateUpdated({
-        serial: data.robotSerial,
-        operationState: RobotStateEventState.PARKED,
-        tripType: TripType.NONE,
-        attemptCancellationReason: null,
-      });
-
       await this.operationsPublisher.publishTripCancelled({
         serial: data.robotSerial,
         tripId: data.id,
diff --git a/service/operations/src/modules/trips/repositories/pilot-trips.repository.ts b/service/operations/src/modules/trips/repositories/pilot-trips.repository.ts
index 5377488674..a9d71c3e08 100644
--- a/service/operations/src/modules/trips/repositories/pilot-trips.repository.ts
+++ b/service/operations/src/modules/trips/repositories/pilot-trips.repository.ts
@@ -11,7 +11,7 @@ import {
 
 import { ILogger } from '@coco/common';
 import { RouteSource } from '@coco/types/maps';
-import { AttemptCancellationReason, FoAssistanceRequestType, RobotStateEventState } from '@coco/types/operations';
+import { AttemptCancellationReason, FoAssistanceRequestType } from '@coco/types/operations';
 
 import { FoTaskTerminalStates } from 'src/modules/fo-tasks/definitions/constants';
 import { PublisherService } from 'src/modules/publisher/services/publisher.service';
@@ -668,13 +668,6 @@ export class PilotTripsRepository {
       });
       return;
     }
-    await this.operationsPublisher.publishRobotStateUpdated({
-      serial: data.botSerial,
-      tripType: this.mapRawTypeToType(trip.trip.type),
-      needsMovement: true,
-      operationState: RobotStateEventState.ON_TRIP,
-      attemptCancellationReason: null,
-    });
   }
 
   mapRawTypeToType(rawType: RawTripType): TripType | undefined {
diff --git a/service/operations/src/modules/trips/services/pilot-trips.service.ts b/service/operations/src/modules/trips/services/pilot-trips.service.ts
index d6cafe4504..465ef7c481 100644
--- a/service/operations/src/modules/trips/services/pilot-trips.service.ts
+++ b/service/operations/src/modules/trips/services/pilot-trips.service.ts
@@ -12,7 +12,6 @@ import {
   CreateTripInput,
   FoAssistanceRequestType,
   ReportRouteIssueResponse,
-  RobotStateEventState,
   RouteIssue,
   RouteIssueType,
   TripState,
@@ -166,7 +165,6 @@ export class PilotTripsService {
       (r) => r.attemptCancellationReason === AttemptCancellationReason.Abberations,
     );
     if (cancelRequested && (parkRequested || hasAbberations)) {
-      await this.operationsPublisher.publishRobotStateUpdated({ serial, needsMovement: false });
       await this.cancelPilotTrip(
         trip.id,
         hasAbberations
@@ -237,14 +235,6 @@ export class PilotTripsService {
       throw error;
     }
 
-    await this.operationsPublisher.publishRobotStateUpdated({
-      serial: botSerial,
-      needsMovement: true,
-      operationState: RobotStateEventState.ON_TRIP,
-      attemptCancellationReason: null,
-      hasFood: true,
-    });
-
     await this.operationsPublisher.publishTripStarted({
       serial: botSerial,
       tripId: trip.id,
@@ -280,13 +270,6 @@ export class PilotTripsService {
         return;
       }
 
-      await this.operationsPublisher.publishRobotStateUpdated({
-        serial: trip.robotSerial,
-        needsMovement: true,
-        operationState: RobotStateEventState.ON_TRIP,
-        attemptCancellationReason: null,
-      });
-
       await this.operationsPublisher.publishTripStarted({
         serial: trip.robotSerial,
         tripId: trip.id,
diff --git a/service/state/src/iot-streamer/iot-streamer.service.ts b/service/state/src/iot-streamer/iot-streamer.service.ts
index ce9b1e6e24..58e590bc5d 100644
--- a/service/state/src/iot-streamer/iot-streamer.service.ts
+++ b/service/state/src/iot-streamer/iot-streamer.service.ts
@@ -856,6 +856,7 @@ export class IotStreamerService implements OnModuleInit, OnModuleDestroy {
           horizontalAccuracy: location.errHorz,
           altitude: 0,
           speedMph: 0,
+          updatedAt: location.updatedAt,
         }
       : undefined;
 
diff --git a/service/state/src/publisher/services/publisher.service.ts b/service/state/src/publisher/services/publisher.service.ts
index 27f7d88188..b9fe907581 100644
--- a/service/state/src/publisher/services/publisher.service.ts
+++ b/service/state/src/publisher/services/publisher.service.ts
@@ -1,22 +1,13 @@
-import { AmqpConnection } from '@golevelup/nestjs-rabbitmq';
 import { Injectable } from '@nestjs/common';
 
-import { Exchanges, IStrictLogger } from '@coco/common';
+import { IStrictLogger } from '@coco/common';
 
 @Injectable()
 export class PublisherService {
   constructor(
     protected readonly logger: IStrictLogger,
-    private readonly amqp: AmqpConnection,
   ) {
     this.logger.setContext(PublisherService.name);
   }
 
-  public async publishRobotStateUpdated(
-    payload: Exchanges.PayloadOf<Exchanges.Names.Robots.StateChange>,
-  ): Promise<void> {
-    this.logger.info('publishRobotStateUpdated called, publishing payload', { extra: payload });
-    await this.amqp.publish(Exchanges.prefixWithEnv(Exchanges.Names.Robots.StateChange), '', payload);
-    this.logger.info('publishRobotStateUpdated called, payload published', { extra: payload });
-  }
 }
diff --git a/service/state/src/state/robot-state.handler.ts b/service/state/src/state/robot-state.handler.ts
index bf5109eb5a..435ab695f5 100644
--- a/service/state/src/state/robot-state.handler.ts
+++ b/service/state/src/state/robot-state.handler.ts
@@ -15,16 +15,38 @@ export class RobotStateChangeHandler {
   }
 
   @UseCls()
-  @RabbitSubscribeWithDlqAndRetry(Exchanges.Names.Robots.StateChange, '#', 'state/robot-state-change')
-  async handleRobotStateChange(payload: Exchanges.PayloadOf<Exchanges.Names.Robots.StateChange>): Promise<void> {
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.FoTaskEvents.Created,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.FoTaskEvents.Created]({ serial: '#' }),
+    'state/fo-task-created',
+  )
+  async handleFoTaskCreated(payload: Exchanges.PayloadOf<Exchanges.Names.FoTaskEvents.Created>): Promise<void> {
     try {
       this.logger.withClsFields({ deviceId: payload.serial });
-      this.logger.debug('state handleRobotStateChange: received message', { extra: payload });
+      this.logger.debug('state handleFoTaskCreated: received message', { extra: payload });
       if (payload.needsMaintenance !== undefined) {
         await this.state.updateNeedsMaintenance(payload.serial, payload.needsMaintenance);
       }
     } catch (e) {
-      this.logger.error('handleRobotStateChange: could not update state attribute', { extra: { payload, error: e } });
+      this.logger.error('handleFoTaskCreated: could not update state attribute', { extra: { payload, error: e } });
+    }
+  }
+
+  @UseCls()
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.FoTaskEvents.Updated,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.FoTaskEvents.Updated]({ serial: '#' }),
+    'state/fo-task-updated',
+  )
+  async handleFoTaskUpdated(payload: Exchanges.PayloadOf<Exchanges.Names.FoTaskEvents.Updated>): Promise<void> {
+    try {
+      this.logger.withClsFields({ deviceId: payload.serial });
+      this.logger.debug('state handleFoTaskUpdated: received message', { extra: payload });
+      if (payload.needsMaintenance !== undefined) {
+        await this.state.updateNeedsMaintenance(payload.serial, payload.needsMaintenance);
+      }
+    } catch (e) {
+      this.logger.error('handleFoTaskUpdated: could not update state attribute', { extra: { payload, error: e } });
     }
   }
 }
diff --git a/service/state/src/state/state-machine/listeners/state-transition.subscriber.ts b/service/state/src/state/state-machine/listeners/state-transition.subscriber.ts
index 43ff4b3aa1..10fb2070a2 100644
--- a/service/state/src/state/state-machine/listeners/state-transition.subscriber.ts
+++ b/service/state/src/state/state-machine/listeners/state-transition.subscriber.ts
@@ -2,7 +2,6 @@ import { AmqpConnection } from '@golevelup/nestjs-rabbitmq';
 import { Injectable } from '@nestjs/common';
 
 import { Exchanges, IStrictLogger } from '@coco/common';
-import { RobotStateEventState } from '@coco/types/operations';
 
 import { EnteredStateEvent, FailedTransitionEvent, OnEnteredState, OnFailedTransition } from '../../../state-machine';
 import { StateMachineException } from '../../../state-machine/exceptions/state-machine.exception';
@@ -21,19 +20,6 @@ export class StateTransitionSubscriber {
     this.logger.setContext(StateTransitionSubscriber.name);
   }
 
-  mapRobotState(st: RobotState): RobotStateEventState {
-    switch (st) {
-      case RobotState.GROUNDED:
-        return RobotStateEventState.GROUNDED;
-      case RobotState.OFF_DUTY:
-        return RobotStateEventState.OFF_DUTY;
-      case RobotState.ON_TRIP:
-        return RobotStateEventState.ON_TRIP;
-      case RobotState.PARKED:
-        return RobotStateEventState.PARKED;
-    }
-  }
-
   @OnEnteredState(robotGraphName)
   private async onEnteredState(event: EnteredStateEvent<RobotState, RobotStateTransition, RobotStateSubject>) {
     await this.stateService.addNewState(event.subject);
@@ -54,29 +40,10 @@ export class StateTransitionSubscriber {
       transition: event.transition.name,
       fromState: event.fromState,
       toState: event.transition.to,
-      // @ts-expect-error (Brad) I don't even care
       request: request,
     };
     this.publishSilently(exchange, rmqEvent);
 
-    const serial = event.subject.job.serial;
-    const toState = event.transition.to;
-    const currentState = event.fromState;
-
-    const updatePayload: Exchanges.PayloadOf<Exchanges.Names.Robots.StateChange> = {
-      serial,
-      operationState: this.mapRobotState(toState),
-    };
-
-    // TODO: delete this hack once we're comfortable with task state
-    if (toState === RobotState.GROUNDED) {
-      updatePayload.needsMaintenance = true;
-    } else if (currentState === RobotState.GROUNDED) {
-      updatePayload.needsMaintenance = false;
-      updatePayload.undergoingMaintenance = false;
-    }
-
-    this.publishLoudly(updatePayload);
   }
 
   @OnFailedTransition(robotGraphName)
@@ -104,7 +71,6 @@ export class StateTransitionSubscriber {
       transition: event.transitionName,
       fromState: event.subject.getState(),
       toState: event.transition?.to,
-      // @ts-expect-error (Brad) I don't even care and i guess it works?
       request,
     };
     this.publishSilently(exchange, rmqEvent);
@@ -125,11 +91,6 @@ export class StateTransitionSubscriber {
     }
   }
 
-  private async publishLoudly(payload: Exchanges.PayloadOf<Exchanges.Names.Robots.StateChange>): Promise<void> {
-    this.logger.debug('publishLoudly called, publishing payload', { deviceId: payload.serial, extra: { payload } });
-    await this.amqp.publish(Exchanges.prefixWithEnv(Exchanges.Names.Robots.StateChange), '', payload);
-  }
-
   private publishSilently<P extends { serial: string; toState: RobotState }>(
     exchange: Exchanges.Names.State,
     payload: P,
diff --git a/service/state/src/state/state-transition.service.ts b/service/state/src/state/state-transition.service.ts
index 1969a6b36f..c405eaff41 100644
--- a/service/state/src/state/state-transition.service.ts
+++ b/service/state/src/state/state-transition.service.ts
@@ -19,7 +19,6 @@ import { robotGraph } from './state-machine/robot-graph';
 import { GenericTransitionData, StateTransitionJob } from './state-transition-job';
 import { RequestTransitionResponse } from './typings';
 import { AppConfigService } from '../config/config.service';
-import { PublisherService } from '../publisher/services/publisher.service';
 
 const TRANSITION_TIMEOUT = 15000;
 
@@ -55,7 +54,6 @@ export class StateTransitionService {
     private readonly sqs: SQS,
     private readonly eventEmitter: EventEmitter2,
     private readonly stateService: StateService,
-    protected readonly operationsPublisher: PublisherService,
   ) {
     this.logger.setContext(StateTransitionService.name);
     this.queueUrl = config.get('stateTransitionQueueUrl');
diff --git a/service/trip-monitor/src/modules/trip-monitor/trip-monitor.handler.ts b/service/trip-monitor/src/modules/trip-monitor/trip-monitor.handler.ts
index 167d5a616b..e052dfd38b 100644
--- a/service/trip-monitor/src/modules/trip-monitor/trip-monitor.handler.ts
+++ b/service/trip-monitor/src/modules/trip-monitor/trip-monitor.handler.ts
@@ -5,6 +5,7 @@ import { Exchanges, ILogger, RabbitSubscribeWithDlqAndRetry } from '@coco/common
 import { TripMonitorService } from './trip-monitor.service';
 
 type HeartbeatPayload = Exchanges.PayloadTypes.IoT[Exchanges.Names.IoT.Heartbeat];
+type DeviceHeartbeatPayload = Exchanges.PayloadTypes.DeviceEvents[Exchanges.Names.DeviceEvents.Heartbeat];
 
 @Injectable()
 export class TripMonitorHandler {
@@ -22,6 +23,71 @@ export class TripMonitorHandler {
     await this.tripMonitorService.updateFromHeartbeat(data);
   }
 
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeviceEvents.Heartbeat,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeviceEvents.Heartbeat]({ serial: '#' }),
+    'trip-monitoring/device-heartbeat',
+  )
+  async handleDeviceHeartbeat(data: DeviceHeartbeatPayload) {
+    await this.tripMonitorService.updateFromDeviceHeartbeat(data);
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Started,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Started]({ serial: '#' }),
+    'trip-monitoring/trip-started',
+  )
+  async handleTripStarted(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Started>): Promise<void> {
+    this.tripMonitorService.setTripActive(payload.serial, true);
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Resumed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Resumed]({ serial: '#' }),
+    'trip-monitoring/trip-resumed',
+  )
+  async handleTripResumed(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Resumed>): Promise<void> {
+    this.tripMonitorService.setTripActive(payload.serial, true);
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Cancelled,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Cancelled]({ serial: '#' }),
+    'trip-monitoring/trip-cancelled',
+  )
+  async handleTripCancelled(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Cancelled>): Promise<void> {
+    this.tripMonitorService.setTripActive(payload.serial, false);
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.TripEvents.Completed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.TripEvents.Completed]({ serial: '#' }),
+    'trip-monitoring/trip-completed',
+  )
+  async handleTripCompleted(payload: Exchanges.PayloadOf<Exchanges.Names.TripEvents.Completed>): Promise<void> {
+    this.tripMonitorService.setTripActive(payload.serial, false);
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeploymentEvents.Deployed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeploymentEvents.Deployed]({ serial: '#' }),
+    'trip-monitoring/deployment-started',
+  )
+  async handleDeploymentStarted(payload: Exchanges.PayloadOf<Exchanges.Names.DeploymentEvents.Deployed>): Promise<void> {
+    this.tripMonitorService.setDeploymentActive(payload.serial, true);
+  }
+
+  @RabbitSubscribeWithDlqAndRetry(
+    Exchanges.Names.DeploymentEvents.Undeployed,
+    Exchanges.RoutingKeys.PublisherFactory[Exchanges.Names.DeploymentEvents.Undeployed]({ serial: '#' }),
+    'trip-monitoring/deployment-ended',
+  )
+  async handleDeploymentEnded(
+    payload: Exchanges.PayloadOf<Exchanges.Names.DeploymentEvents.Undeployed>,
+  ): Promise<void> {
+    this.tripMonitorService.setDeploymentActive(payload.serial, false);
+  }
+
   @RabbitSubscribeWithDlqAndRetry(
     Exchanges.Names.Operations.TripDestinationUpdated,
     '#',
diff --git a/service/trip-monitor/src/modules/trip-monitor/trip-monitor.service.ts b/service/trip-monitor/src/modules/trip-monitor/trip-monitor.service.ts
index 6fc934ae8a..f9731c8f10 100644
--- a/service/trip-monitor/src/modules/trip-monitor/trip-monitor.service.ts
+++ b/service/trip-monitor/src/modules/trip-monitor/trip-monitor.service.ts
@@ -6,6 +6,7 @@ import { basicMatch, calculateRouteRemaining, LngLat } from '@coco/routing-engin
 import { RecordNotFoundError } from '@coco/types/errors';
 import { TemporalGeo, Trip } from '@coco/types/trips';
 
+import { deriveOperationState } from '../../shared/derive-operation-state';
 import { routeToLegacy } from '../../shared/transformers/route-to-legacy-route';
 import {
   TripDeletionReason,
@@ -21,6 +22,8 @@ const MAX_LOG_FREQUENCY_HEARTBEAT_MILLIS = 60_000;
 @Injectable()
 export class TripMonitorService {
   lastLogged: Map<string, number> = new Map();
+  // Dummy map to store the operation state by serial, this would instead be stored in a persistent store like Redis or a database.
+  private readonly operationStateBySerial = new Map<string, { hasActiveTrip: boolean; isDeployed: boolean }>();
 
   constructor(
     private readonly logger: ILogger,
@@ -43,6 +46,24 @@ export class TripMonitorService {
     return canLog;
   }
 
+  setTripActive(serial: string, hasActiveTrip: boolean): void {
+    const entry = this.operationStateBySerial.get(serial) ?? { hasActiveTrip: false, isDeployed: false };
+    entry.hasActiveTrip = hasActiveTrip;
+    this.operationStateBySerial.set(serial, entry);
+  }
+
+  setDeploymentActive(serial: string, isDeployed: boolean): void {
+    const entry = this.operationStateBySerial.get(serial) ?? { hasActiveTrip: false, isDeployed: false };
+    entry.isDeployed = isDeployed;
+    this.operationStateBySerial.set(serial, entry);
+  }
+
+  private getDerivedOperationState(serial: string): Heartbeat.OperationState | undefined {
+    const entry = this.operationStateBySerial.get(serial);
+    if (!entry) return undefined;
+    return deriveOperationState(entry);
+  }
+
   async updateFromHeartbeat(data: Heartbeat.HeartbeatV1): Promise<void | Error> {
     const trip = await this.tripService.getTripBySerial(data.serial);
     if (trip instanceof Error) {
@@ -131,6 +152,35 @@ export class TripMonitorService {
     }
   }
 
+  async updateFromDeviceHeartbeat(
+    data: Exchanges.PayloadTypes.DeviceEvents[Exchanges.Names.DeviceEvents.Heartbeat],
+  ): Promise<void | Error> {
+    const location = data.device.location;
+    if (!location) {
+      return Error('no location update');
+    }
+
+    const mappedHeartbeat: Heartbeat.HeartbeatV1 = {
+      version: 1,
+      serial: data.deviceName,
+      createdAt: data.device.createdAt,
+      receivedAt: data.receivedAt,
+      processedAt: data.receivedAt,
+      healthy: true,
+      components: {},
+      operationState: this.getDerivedOperationState(data.deviceName),
+      location: {
+        latitude: location.latitude,
+        longitude: location.longitude,
+        heading: location.heading,
+        errHorz: location.horizontalAccuracy,
+        updatedAt: location.updatedAt,
+      },
+    };
+
+    return this.updateFromHeartbeat(mappedHeartbeat);
+  }
+
   // TODO (christie): Potential race condidtion if this triggers at the same
   // time as the heartbeat handler
   async updateFromTripDestinationUpdated(tripId: string, destination: CompleteLocation): Promise<void | Error> {
diff --git a/service/trip-monitor/src/shared/derive-operation-state.ts b/service/trip-monitor/src/shared/derive-operation-state.ts
new file mode 100644
index 0000000000..937861db61
--- /dev/null
+++ b/service/trip-monitor/src/shared/derive-operation-state.ts
@@ -0,0 +1,24 @@
+import { Heartbeat } from '@coco/common';
+
+type OperationStateInputs = {
+  hasActiveTrip: boolean;
+  isDeployed: boolean;
+  isGrounded?: boolean;
+};
+
+export const deriveOperationState = ({
+  hasActiveTrip,
+  isDeployed,
+  isGrounded,
+}: OperationStateInputs): Heartbeat.OperationState => {
+  if (isGrounded) {
+    return 'GROUNDED';
+  }
+  if (hasActiveTrip) {
+    return 'ON_TRIP';
+  }
+  if (isDeployed) {
+    return 'PARKED';
+  }
+  return 'OFF_DUTY';
+};
