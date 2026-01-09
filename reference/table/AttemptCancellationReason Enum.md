---
tags:
  - enum
  - delivery
  - attempt
---
# AttemptCancellationReason Enum

**Used in**: [[Operations RDS Schema]], [[Deliveries V3 RDS Schema]], [[Dispatch Engine RDS Schema]]  
**Defined in**:
- [`service/operations/prisma/schema.prisma`](../../../delivery-platform/service/operations/prisma/schema.prisma)
- [`service/deliveries/prisma/schema.prisma`](../../../delivery-platform/service/deliveries/prisma/schema.prisma)
- [`service/dispatch-engine/prisma/schema.prisma`](../../../delivery-platform/service/dispatch-engine/prisma/schema.prisma)

Enum defining reasons why a delivery [[Attempt]] was cancelled. This is used across multiple services to maintain consistent cancellation reason tracking.

## Values

### Hardware & Software Issues
- `HardwareIssue` - Robot hardware failure
- `SoftwareIssue` - Software bug or issue
- `MerchantTabletIssue` - Merchant's tablet not working

### Merchant Issues
- `MerchantError` - General merchant error
- `MerchantRequested` - Merchant asked to cancel
- `MerchantUnresponsive` - Merchant not responding
- `LongLoad` - Taking too long to load food
- `LargeOrder` - Order too large for robot
- `MerchantTabletIssue` - Tablet issues

### Robot Issues
- `BotFlipped` - Robot tipped over
- `BotStuck` - Robot stuck and can't move
- `BotHit` - Robot was hit by something
- `BotRescue` - Robot needs field rescue
- `Abberations` / `Aberrations` - Unexpected robot behavior (both spellings exist)
- `PedestrianInterference` / `PedestrianFlip` - Pedestrian interference

### Environmental Issues
- `TerrainIssue` - Difficult terrain
- `Obstruction` - Path blocked
- `RouteBlocked` - Route unavailable
- `WeatherConditions` - Bad weather

### Customer Issues
- `UnclearDropOff` - Dropoff location unclear
- `CustomerUnresponsive` - Customer not responding
- `CustomerRequested` - Customer requested cancellation (deprecated)
- `CustomerRequested_Unable` - Customer unable to receive
- `CustomerRequested_Unwilling` - Customer refuses delivery
- `CustomerRequested_HighRise` - High-rise building (deprecated)
- `CustomerBlacklisted` - Customer on blacklist
- `CustomerIssues` - General customer issues (deprecated)
- `IdCheck` - Age verification failed

### Pilot Issues
- `PilotError` - Pilot made error
- `PilotSpeed` - Pilot too slow (deprecated)
- `PilotAvailability` - No pilots available

### System Issues
- `RobotAvailability` - No robots available
- `RobotDeliveriesDisabledByOperatingZone` - Zone has disabled robot deliveries
- `RobotDeliveriesDisabledByMerchant` - Merchant disabled robot deliveries
- `RoutingFailed` - Could not plan route
- `NotRobotAddressable` - Location not reachable by robot
- `OutsideAvailabilityWindow` - Outside operating hours
- `SMSFailure` - Could not send SMS to customer

### Courier Issues
- `CourierFailure` - Third-party courier failed

### Operational
- `OffboardingMx` - Merchant being offboarded
- `DeliveryWatchdog` - [[Watchdog System]] triggered cancellation
- `DeliveryCanceled` - Parent delivery was cancelled

### Deprecated Reasons
- `LoadError` - Loading error (deprecated)
- `Reorder` - Order being remade (deprecated)
- `Outages` - System outages (deprecated)
- `OverlappingETA` - ETA conflicts (deprecated)
- `RestrictedOrder` - Restricted items (deprecated)

### Other
- `Other` - Reason not in list

## Usage

The reason is recorded in:
- [[Attempt Table]] - `cancelReason` field
- [[Quote Table]] - `robotFailureReason` field (why robot couldn't be quoted)
- [[FoAssistanceRequestModel Table]] - `attemptCancellationReason` field

## Analytics Groupings

These reasons are often grouped for analytics:

1. **Robot Issues**: BotFlipped, BotStuck, BotHit, HardwareIssue, SoftwareIssue
2. **Merchant Issues**: MerchantError, MerchantRequested, LongLoad, LargeOrder
3. **Customer Issues**: CustomerUnresponsive, CustomerRequested variants, UnclearDropOff
4. **Availability**: PilotAvailability, RobotAvailability
5. **Routing**: RoutingFailed, NotRobotAddressable, RouteBlocked

## Related Concepts

- [[Attempt]] - Fulfillment attempts that can be cancelled
- [[Attempt Table]] - Database table storing attempts
- [[Delivery Status State Machine]] - How cancellations affect delivery status
- [[DeliveryIssue Table]] - Related issue tracking

