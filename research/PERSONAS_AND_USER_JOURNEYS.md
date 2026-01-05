# Personas & User Journeys

This document catalogs all personas who interact with the Coco autonomous delivery platform and their key user journeys.

---

## Table of Contents
1. [Persona Overview](#1-persona-overview)
2. [Customer](#2-customer)
3. [Merchant](#3-merchant)
4. [Pilot](#4-pilot)
5. [Field Operator (FO)](#5-field-operator-fo)
6. [Dispatch Operator](#6-dispatch-operator)
7. [Pilot Supervisor](#7-pilot-supervisor)
8. [General Manager (GM)](#8-general-manager-gm)
9. [Head of Operations](#9-head-of-operations)
10. [Operations Support](#10-operations-support) *(Fleet Ops Oncall + Support Agent)*
11. [Engineering](#11-engineering) *(includes Software Oncall)*
12. [Operations Analyst](#12-operations-analyst) *(fleet effectiveness & improvement)*
13. [MRO Technician](#13-mro-technician)
14. [System Administrator](#14-system-administrator)
15. [Partner Platform](#15-partner-platform)

---

## 1. Persona Overview

| Persona | Primary Goal | Key Applications |
|---------|--------------|------------------|
| **Customer** | Receive delivery | Uber Eats / DoorDash app |
| **Merchant** | Load orders into robots | Merchant Portal, Tablet, Robot keypad |
| **Pilot** | Remotely operate robots | Pilot UI, DriveU |
| **Field Operator** | Maintain fleet in field | FO Mobile App |
| **Dispatch Operator** | Coordinate fleet & tasks | Mission Control |
| **Pilot Supervisor** | Manage pilot team | Mission Control |
| **General Manager** | Oversee metro operations | Mission Control, Analytics |
| **Head of Operations** | Plan deployments | Analytics, Planning tools |
| **Operations Support** | Handle issues & emergencies | Intercom, Mission Control, PagerDuty, Slack |
| **Engineering** | Build, debug, oncall response | Datadog, Foxglove, GitHub, all tools |
| **Operations Analyst** | Track effectiveness & improvements | Analytics dashboards, Data warehouse |
| **MRO Technician** | Repair robots | MRO Dashboard |
| **System Admin** | Manage users & config | Auth0, Admin panels |
| **Partner Platform** | Send delivery requests | API Integration |

---

## 2. Customer

**Profile**: End consumer ordering food/goods for delivery via partner apps (Uber Eats, DoorDash).

**Applications**: Uber Eats app, DoorDash app (native integration)

### User Journeys

#### 2.1 Track my delivery
> As a customer, I want to track my robot delivery so I know when it will arrive.

**Key Interactions:**
1. Customer views delivery status in partner app (Uber Eats / DoorDash)
2. App shows real-time robot location on map
3. App displays estimated arrival time (ETA)
4. ETA updates as robot progresses
5. Customer receives push notification when robot is arriving
6. Customer receives push notification when robot has arrived

---

#### 2.2 Retrieve my food from the robot
> As a customer, I want to open the robot and get my food when it arrives.

**Key Interactions:**
1. Customer receives "Robot arrived" notification in partner app
2. Customer goes to robot location
3. Customer taps "Open" button in partner app
4. Robot lid opens
5. Customer retrieves food from cargo bay
6. Customer closes lid
7. Delivery marked complete

**Edge Cases:**
- Lid won't open → Customer retries or contacts support via partner app
- Customer doesn't pick up → Reminder notifications sent, eventually may cancel
- Wrong location → Customer uses app to see robot's actual GPS location

---

#### 2.3 Report an issue with my delivery
> As a customer, I want to report a problem with my delivery.

**Key Interactions:**
1. Customer opens partner app (Uber Eats / DoorDash)
2. Customer navigates to order and selects "Help" or "Report Issue"
3. Issue routed to partner support or Coco support depending on type
4. Support investigates and resolves


---

## 3. Merchant

**Profile**: Restaurant or store staff responsible for loading orders into robots.

**Applications**: 
- **Merchant Portal** (`merchant.delivery.cocodelivery.com`) - View orders, delivery history, analytics
- **MX Manager** - Store configuration, payout schedules
- **Tablet** - In-store device for order notifications
- **Robot keypad** - Physical interaction for loading

### Loading Methods

| Method | Partner | How It Works |
|--------|---------|--------------|
| **Magic Lid** | Uber Eats | Touch the keypad → lid opens automatically |
| **Deviceless (2-digit PIN)** | DoorDash | Enter 2-digit PIN on keypad → backend validates → lid opens |
| **QR Code** | Various | Scan QR with phone → web app opens lid |

### User Journeys

#### 3.1 Load a robot with an order
> As a merchant, I want to load the correct order into the robot so the customer receives their food.

**Key Interactions:**
1. Merchant receives notification that robot is arriving (tablet/SMS)
2. Merchant receives notification that robot has arrived
3. Merchant identifies which order goes with this robot (order ID displayed)
4. Merchant opens lid:
   - **Uber Eats (Magic Lid)**: Touch keypad → lid opens
   - **DoorDash (Deviceless)**: Enter 2-digit PIN → lid opens
5. Merchant places food in cargo bay
6. Merchant closes lid firmly
7. System confirms successful load (lid closed, `hasFood = true`)
8. Robot departs for customer

**What's Missing Today:**
- [ ] Explicit confirmation to merchant that load was successful
- [ ] Clear error messaging if lid doesn't close properly

---

#### 3.2 Know when a robot is coming
> As a merchant, I want to know when a robot will arrive so I can have the order ready.

**Key Interactions:**
1. Order comes in via POS/tablet integration
2. Merchant sees order details and prepares food
3. Merchant receives "Robot on the way" notification with ETA
4. Merchant receives "Robot arrived" notification
5. Merchant goes to load the robot

**What's Missing Today:**
- [ ] Countdown timer showing robot ETA
- [ ] Notification if robot is delayed

---

#### 3.3 Report an issue with a robot
> As a merchant, I want to report when something is wrong with a robot or delivery.

**Key Interactions:**
1. Merchant identifies issue (robot not arriving, lid stuck, etc.)
2. Merchant contacts support via tablet or phone
3. Support creates appropriate ticket/task
4. Issue resolved (FO dispatched, delivery cancelled, etc.)

---

#### 3.4 View my delivery history and performance
> As a merchant, I want to see how my robot deliveries are performing.

**Key Interactions:**
1. Merchant logs into Merchant Portal
2. Views dashboard with delivery metrics
3. Reviews individual delivery history
4. Sees analytics (success rate, average times, etc.)

---

#### 3.5 Configure my store settings
> As a merchant, I want to configure my delivery preferences and store settings.

**Key Interactions:**
1. Merchant logs into MX Manager
2. Updates store hours
3. Configures delivery preferences
4. Views/manages payout schedule


---

## 4. Pilot

**Profile**: Remote operator who drives robots via teleoperation (DriveU system).

**Applications**: Pilot UI (`pilot.delivery.cocodelivery.com`), DriveU teleoperation system

### User Journeys

#### 4.1 Start my shift
> As a pilot, I want to clock in and become available for trip assignments.

**Key Interactions:**
1. Pilot logs into Pilot UI
2. Clicks "Start Shift"
3. Selects operating zone/hub
4. Status changes to `AVAILABLE`
5. Pilot now receives trip assignments

---

#### 4.2 Complete a delivery trip
> As a pilot, I want to drive a robot from merchant to customer to complete a delivery.

**Key Interactions:**
1. Pilot receives trip assignment notification
2. Pilot accepts trip (or rejects → reassigned to another pilot)
3. Pilot connects to robot via DriveU (video stream initializes)
4. Pilot starts trip → status: `IN_TRANSIT`
5. Pilot navigates robot to destination
6. Robot arrives → status: `AT_DESTINATION`
7. Pilot waits for customer/merchant interaction (lid cycle)
8. Lid cycle completes → trip complete
9. Pilot available for next assignment

**Trip Types:**
- **JITP** (Just-In-Time Pickup): Drive to merchant
- **DELIVERY**: Drive from merchant to customer
- **RETURN**: Return robot to parking lot
- **DEPLOYMENT**: Move robot to new service location

---

#### 4.3 Report an issue during a trip
> As a pilot, I want to report when something goes wrong so the issue can be resolved.

**Key Interactions:**
1. Pilot encounters issue during trip
2. Pilot opens issue reporting in Pilot UI
3. Pilot selects issue type and adds notes
4. System creates appropriate response (FO task, incident report, etc.)
5. Trip may be cancelled or reassigned

**Issue Types:**
- Robot stuck → FO task: `UNSTUCK_BOT`
- Robot flipped → FO task: `UNFLIP_BOT`
- Route blocked → Reroute or escalate
- Hardware issue → May ground robot
- Lid malfunction → Troubleshoot or request FO
- Collision (vehicle, person, object) → Incident report created
- Food stolen → Incident report created

---

#### 4.4 Take a break
> As a pilot, I want to take a scheduled break during my shift.

**Key Interactions:**
1. Pilot clicks "Start Break"
2. Status changes to `ON_BREAK`
3. Pilot stops receiving trip assignments
4. Pilot clicks "End Break"
5. Status returns to `AVAILABLE`

---

#### 4.5 End my shift
> As a pilot, I want to clock out at the end of my shift.

**Key Interactions:**
1. Pilot completes any active trip
2. Pilot clicks "End Shift"
3. Status changes to `OFFLINE`
4. Pilot no longer receives assignments

---

#### 4.6 Troubleshoot a lid issue
> As a pilot, I want to try to fix a lid that won't open or close.

**Key Interactions:**
1. Pilot notices lid issue (won't open for merchant/customer, or won't close)
2. Pilot attempts remote lid control (X to open, A to close on controller)
3. If successful → continue trip
4. If unsuccessful after 2-3 attempts → request FO assistance
5. FO task created, robot may be grounded


---

## 5. Field Operator (FO)

**Profile**: On-ground contractor who maintains fleet health and responds to field issues.

**Applications**: FO Mobile App (iOS/Android), Mission Control (limited view)

### Task Types Reference

| Category | Task Types |
|----------|------------|
| **Robot Service** | `BATTERY_SWAP`, `PARKING_LOT_BATTERY_SWAP`, `CLEAN_CAMERA`, `CLEAN_INSOLE`, `EMPTY_BOT`, `PUT_UP_FLAG`, `RETURN_TO_PARKING_LOT` |
| **Incident Response** | `UNSTUCK_BOT`, `UNFLIP_BOT`, `MOVE_BOT_FROM_CROSSWALK`, `BOT_HIT` |
| **Merchant Response** | `TABLET_ISSUE`, `QR_ISSUE` |
| **Rescue** | `DELIVERY_RESCUE`, `BOT_PICKUP`, `SWAP_BOT` |
| **Deployment** | `DEPLOY_BOTS`, `DEPLOY_BOTS_WITH_PILOT`, `PREPARE_BOTS_FOR_DEPLOYMENT`, `EOD_BOT_PICKUP` |

### User Journeys

#### 5.1 Start my shift
> As a field operator, I want to clock in and become available for tasks.

**Key Interactions:**
1. FO opens mobile app
2. Logs in with Auth0
3. Clicks "Clock In"
4. Selects assigned hub/zone
5. Status changes to `ONLINE`
6. FO now receives task assignments

---

#### 5.2 Complete a field task
> As a field operator, I want to complete an assigned task to keep the fleet running.

**Key Interactions:**
1. FO receives task notification (push notification)
2. FO views task details (type, location, robot info, priority)
3. FO accepts task (or rejects → reassigned)
4. FO navigates to task location (using in-app maps)
5. FO marks "Arrived"
6. FO performs required action(s)
7. FO marks subtask complete
8. If more subtasks → repeat step 6-7
9. FO marks task complete, adds notes if needed
10. FO available for next task

---

#### 5.3 Swap a robot's battery
> As a field operator, I want to replace a low battery so the robot can continue operating.

**Key Interactions:**
1. FO receives `BATTERY_SWAP` task
2. FO travels to robot location (with fresh battery)
3. FO arrives at robot
4. FO opens battery compartment
5. FO removes depleted battery
6. FO inserts fresh battery
7. FO closes compartment
8. FO marks task complete
9. Robot resumes operation with new battery

---

#### 5.4 Rescue a failed delivery
> As a field operator, I want to complete a delivery when the robot can't.

**Key Interactions:**
1. FO receives `DELIVERY_RESCUE` task (high priority)
2. FO travels to robot location
3. FO retrieves food from robot (opens lid manually or via app)
4. FO travels to customer location
5. FO delivers food to customer
6. FO marks delivery complete
7. Robot may need separate pickup task

---

#### 5.5 Deploy robots in the morning
> As a field operator, I want to deploy robots to the service area for the day.

**Key Interactions:**
1. FO receives `DEPLOY_BOTS` task
2. FO goes to storage/MRO location
3. FO prepares robots (checks, scans barcodes)
4. FO loads robots into van
5. FO drives to service area
6. FO unloads robots at designated parking lots
7. FO marks each robot as deployed
8. Robots become available for deliveries

---

#### 5.6 Pick up robots at end of day
> As a field operator, I want to collect robots and return them to storage.

**Key Interactions:**
1. FO receives `EOD_BOT_PICKUP` task
2. FO plans pickup route for assigned robots
3. FO travels to each robot location
4. FO loads robot into van
5. FO marks robot picked up
6. FO drives to storage location
7. FO unloads and checks in robots
8. Robots marked `OFF_DUTY`

---

#### 5.7 Respond to a stuck robot
> As a field operator, I want to free a robot that's stuck so it can continue operating.

**Key Interactions:**
1. FO receives `UNSTUCK_BOT` task (high priority)
2. FO travels to robot location
3. FO assesses situation
4. FO physically moves/frees the robot
5. FO verifies robot can move
6. FO marks task complete
7. Robot resumes operation (or needs further maintenance)


---

## 6. Dispatch Operator

**Profile**: Real-time coordinator who monitors fleet and manages task assignments.

**Applications**: Mission Control

### User Journeys

#### 6.1 Monitor fleet status
> As a dispatch operator, I want to see the real-time status of all robots and deliveries.

**Key Interactions:**
1. Dispatch logs into Mission Control
2. Views fleet map with robot locations
3. Reviews robot list (status, battery, health)
4. Monitors active deliveries queue
5. Watches for alerts and anomalies
6. Takes action on issues as they arise

---

#### 6.2 Handle a delayed delivery
> As a dispatch operator, I want to address a delivery that's running late.

**Key Interactions:**
1. Dispatch sees alert for delayed delivery
2. Reviews delivery details (robot status, pilot status, route)
3. Determines cause of delay
4. Takes appropriate action:
   - If pilot issue → reassign pilot
   - If robot issue → may reassign robot or create FO task
   - If route issue → coordinate reroute
5. Updates stakeholders if needed

---

#### 6.3 Assign an FO task
> As a dispatch operator, I want to create and assign a task to a field operator.

**Key Interactions:**
1. Dispatch identifies need for FO intervention
2. Creates task in Mission Control (selects type, location, priority)
3. Reviews available FOs (location, current tasks)
4. Assigns task to appropriate FO
5. Monitors task progress
6. Follows up if task is delayed or rejected

---

#### 6.4 Reserve a pilot for a specific task
> As a dispatch operator, I want to reserve a pilot so they're available for a specific assignment.

**Key Interactions:**
1. Dispatch identifies need to reserve pilot
2. Selects pilot in Mission Control
3. Clicks "Reserve"
4. Pilot stops receiving automatic assignments
5. Dispatch manually assigns specific task
6. After task complete, dispatch "Unreserves" pilot

---

#### 6.5 Escalate a critical issue
> As a dispatch operator, I want to escalate an issue I can't resolve to the appropriate team.

**Key Interactions:**
1. Dispatch encounters issue beyond their scope
2. Documents issue details
3. Posts in appropriate Slack channel (`#ops-oncall`)
4. Tags Operations Support or relevant team
5. Hands off issue while continuing to monitor

---

## 7. Pilot Supervisor

**Profile**: Manager responsible for pilot team performance and scheduling.

**Applications**: Mission Control

### User Journeys

#### 7.1 Monitor pilot performance
> As a pilot supervisor, I want to track how my pilots are performing.

**Key Interactions:**
1. Supervisor logs into Mission Control
2. Views pilot dashboard with metrics
3. Reviews individual pilot stats (trips completed, incidents, response times)
4. Identifies pilots needing coaching or recognition
5. Takes appropriate follow-up action

---

#### 7.2 End a pilot's cooldown
> As a pilot supervisor, I want to remove a pilot from cooldown so they can resume work.

**Key Interactions:**
1. Supervisor sees pilot in cooldown status
2. Reviews reason for cooldown
3. Determines if cooldown should end early
4. Clicks "End Cooldown" in Mission Control
5. Pilot returns to available status

---

#### 7.3 Handle pilot escalation
> As a pilot supervisor, I want to address issues escalated about or by pilots.

**Key Interactions:**
1. Supervisor receives escalation (Slack, Mission Control)
2. Reviews situation details
3. Contacts pilot if needed
4. Takes appropriate action (coaching, reassignment, etc.)
5. Documents outcome


---

## 8. General Manager (GM)

**Profile**: Operations leader responsible for a metro/service area.

**Applications**: Mission Control, Analytics dashboards, Slack

### User Journeys

#### 8.1 Review daily operations
> As a GM, I want to understand how operations performed today.

**Key Interactions:**
1. GM reviews overnight incident reports
2. Checks fleet health status
3. Reviews delivery metrics (volume, success rate, times)
4. Identifies any ongoing issues
5. Syncs with dispatch and FO teams
6. Addresses blockers

---

#### 8.2 Manage contractor relationships
> As a GM, I want to coordinate with FO contractors to ensure adequate coverage.

**Key Interactions:**
1. GM reviews FO staffing levels
2. Identifies coverage gaps
3. Communicates with contractor managers
4. Adjusts schedules as needed
5. Monitors FO performance metrics

---

#### 8.3 Handle merchant escalations
> As a GM, I want to resolve issues escalated by merchants.

**Key Interactions:**
1. GM receives merchant escalation
2. Reviews merchant history and issue details
3. Coordinates resolution (may involve support, FO, engineering)
4. Communicates with merchant
5. Documents resolution and any process improvements

---

#### 8.4 Report on area performance
> As a GM, I want to report on my area's performance to leadership.

**Key Interactions:**
1. GM pulls metrics from analytics dashboards
2. Compiles key KPIs (deliveries, success rate, incidents)
3. Identifies trends and issues
4. Prepares summary report
5. Shares with Head of Operations

---

## 9. Head of Operations

**Profile**: Strategic leader responsible for fleet deployment planning across all metros.

**Applications**: Analytics dashboards, Planning tools, Mission Control

### User Journeys

#### 9.1 Plan daily robot deployments
> As Head of Operations, I want to decide how many robots to deploy where based on expected demand.

**Key Interactions:**
1. Reviews historical demand data by zone/time
2. Analyzes upcoming factors (weather, events, day of week)
3. Assesses available fleet (healthy robots, maintenance schedule)
4. Creates deployment plan (robots per zone)
5. Communicates plan to GMs
6. Monitors execution and adjusts as needed

---

#### 9.2 Optimize fleet utilization
> As Head of Operations, I want to maximize the value we get from our robot fleet.

**Key Interactions:**
1. Reviews fleet utilization metrics
2. Identifies underutilized zones/times
3. Analyzes demand patterns
4. Adjusts deployment strategy
5. Tracks impact of changes

---

#### 9.3 Plan for capacity changes
> As Head of Operations, I want to plan for adding or reducing fleet capacity.

**Key Interactions:**
1. Reviews demand forecasts
2. Assesses current capacity vs. demand
3. Identifies gaps or excess
4. Plans fleet changes (new robots, retirements, reallocation)
5. Coordinates with MRO and procurement


---

## 10. Operations Support

**Profile**: Combined role handling customer/merchant support tickets AND fleet emergency response. On rotation for oncall coverage (24/7).

**Applications**: Intercom (tickets), Mission Control, PagerDuty, Slack (`#ops-oncall`), Foxglove

### User Journeys

#### 10.1 Help a customer who can't open the robot
> As operations support, I want to help a customer access their food when the lid won't open.

**Key Interactions:**
1. Receives ticket or escalation: "Can't open robot"
2. Looks up delivery in Mission Control
3. Verifies robot is at customer location
4. Attempts remote lid open command
5. If successful → customer retrieves food
6. If unsuccessful → may need to dispatch FO for rescue
7. Closes ticket with resolution

---

#### 10.2 Handle a missing delivery complaint
> As operations support, I want to investigate when a customer says their delivery never arrived.

**Key Interactions:**
1. Receives ticket: "Delivery never arrived"
2. Looks up delivery in Mission Control
3. Reviews delivery status and timeline
4. Checks robot location history
5. Determines what happened:
   - If delivered → provides proof of delivery
   - If cancelled → explains reason
   - If stuck → coordinates resolution
6. Processes refund if appropriate
7. Closes ticket

---

#### 10.3 Respond to a robot offline alert
> As operations support, I want to investigate and resolve a robot that's gone offline.

**Key Interactions:**
1. Receives PagerDuty alert: "Robot offline"
2. Checks robot status in Mission Control
3. Reviews last known location and state
4. Determines likely cause:
   - Connectivity issue → may recover on its own
   - Battery dead → create FO task: `BATTERY_SWAP`
   - Hardware failure → create FO task: `BOT_PICKUP`
5. Creates appropriate FO task if needed
6. Monitors resolution
7. Closes PagerDuty alert

---

#### 10.4 Respond to a robot blocking sidewalk
> As operations support, I want to urgently address a robot blocking public space.

**Key Interactions:**
1. Receives urgent alert or user report
2. Locates robot in Mission Control
3. Immediately creates high-priority FO task: `MOVE_BOT_FROM_CROSSWALK`
4. Contacts nearest available FO directly if needed
5. Monitors until robot is moved
6. Documents incident
7. Escalates to GM if pattern emerges

---

#### 10.5 Respond to merchant tablet offline
> As operations support, I want to help when a merchant's tablet stops working.

**Key Interactions:**
1. Receives Hexnode alert or merchant report
2. Contacts merchant to verify issue
3. Attempts remote troubleshooting
4. If unresolved → creates FO task: `TABLET_ISSUE`
5. Follows up until resolved
6. Closes ticket

---

#### 10.6 Coordinate multi-robot incident
> As operations support, I want to manage a situation affecting multiple robots.

**Key Interactions:**
1. Receives alerts for multiple robots
2. Assesses scope and impact
3. Prioritizes response (safety first, then active deliveries)
4. Coordinates FO resources
5. Communicates status to stakeholders via Slack
6. Escalates to GM/Head of Ops if major impact
7. Documents incident for post-mortem

---

#### 10.7 Investigate user-reported issue
> As operations support, I want to investigate an issue reported by a customer or merchant.

**Key Interactions:**
1. Receives report via ticket or Slack
2. Gathers details (robot serial, location, time, description)
3. Reviews robot data in Mission Control
4. Checks Foxglove for camera footage if needed
5. Determines root cause
6. Takes corrective action or creates follow-up task
7. Responds to reporter with findings

---

#### 10.8 Escalate to engineering
> As operations support, I want to escalate technical issues I can't resolve.

**Key Interactions:**
1. Identifies issue that appears to be software/system bug
2. Documents all relevant details (delivery ID, robot serial, timestamps, symptoms)
3. Posts in `#eng-oncall` Slack channel
4. Tags engineering oncall
5. Provides any logs or screenshots gathered
6. Hands off while maintaining ticket ownership
7. Follows up on resolution


---

## 11. Engineering

**Profile**: Software/hardware engineer building and maintaining the platform. Rotates through oncall for system alerts (24/7 coverage).

**Applications**: Datadog (logs, metrics, APM), Foxglove (robot recordings), GitHub, AWS Console, Kubernetes, Mission Control, all internal tools

### User Journeys

#### 11.1 Respond to a service error alert (oncall)
> As engineering oncall, I want to investigate and resolve a spike in service errors.

**Key Interactions:**
1. Receives PagerDuty alert: "Error rate elevated"
2. Opens Datadog APM to identify affected service
3. Reviews error logs to understand failure mode
4. Checks recent deployments for correlation
5. Determines fix:
   - If bad deploy → rollback
   - If infrastructure → scale/restart
   - If data issue → fix data
   - If external dependency → implement workaround
6. Verifies fix resolved the issue
7. Closes PagerDuty alert
8. Creates follow-up ticket if needed

---

#### 11.2 Investigate a watchdog alert (oncall)
> As engineering oncall, I want to understand why the watchdog system triggered an action.

**Key Interactions:**
1. Receives alert about watchdog action (e.g., auto-cancelled delivery)
2. Reviews watchdog logs in Datadog
3. Identifies which condition triggered
4. Checks if action was appropriate
5. If false positive → may need to tune watchdog rules
6. If legitimate → no action needed
7. Documents findings

**Watchdog Conditions:**
- `IsDeliveryLate` - Delivery past ETA
- `IsPickupLate` - Robot late to merchant
- `MissingETA` - No ETA available
- Lid open too long
- Various other health conditions

---

#### 11.3 Debug an issue for a specific delivery
> As an engineer, I want to understand what went wrong with a specific delivery.

**Key Interactions:**
1. Gets delivery ID from ops support or ticket
2. **Retrieve logs by delivery ID:**
   - Open Datadog Logs
   - Search: `@deliveryId:<delivery-id>` or `@attemptId:<attempt-id>`
   - Filter by time range of the delivery
   - Review logs from relevant services (deliveries, operations, dispatch-engine, state)
3. **Retrieve robot footage by delivery ID:**
   - Look up robot serial and trip timestamps from delivery record
   - Open Foxglove Data Platform
   - Search for recordings by robot serial and time range
   - Review camera footage, sensor data, robot state
4. **Correlate across systems:**
   - Check robot heartbeats around incident time
   - Review trip status transitions in operations service
   - Check dispatch engine decisions
   - Review any watchdog actions
5. Identify root cause
6. Document findings
7. Create fix or follow-up ticket

**Key Data Sources:**
| Data | Where to Find |
|------|---------------|
| Delivery/attempt logs | Datadog: `service:deliveries` |
| Trip/pilot logs | Datadog: `service:operations` |
| Dispatch decisions | Datadog: `service:dispatch-engine` |
| Robot state/heartbeats | Datadog: `service:state` |
| Robot camera footage | Foxglove: by robot serial + time |
| Robot sensor data | Foxglove: by robot serial + time |
| Robot state machine | Foxglove: by robot serial + time |

---

#### 11.4 Debug a robot-side issue
> As an engineer, I want to investigate an issue that may be robot firmware/software related.

**Key Interactions:**
1. Gets robot serial and time of incident
2. **Review Foxglove recordings:**
   - Open Foxglove Data Platform
   - Find recordings for robot serial around incident time
   - Review camera footage (what robot "saw")
   - Examine sensor data (GPS, IMU, lidar)
   - Check robot state machine transitions
   - Review commands sent to robot
3. **Correlate with backend:**
   - Check robot heartbeat data in Datadog
   - Review any commands sent from backend
   - Check DriveU connection status
4. Identify root cause (firmware bug, sensor failure, environmental, etc.)
5. If firmware issue → coordinate with robotics team
6. Document findings and create follow-up ticket

---

#### 11.5 Respond to infrastructure alert (oncall)
> As engineering oncall, I want to resolve infrastructure issues affecting services.

**Key Interactions:**
1. Receives alert (pod crashes, high memory, database issues)
2. Checks Kubernetes dashboard for service health
3. Reviews AWS CloudWatch for infrastructure metrics
4. Takes corrective action:
   - Restart unhealthy pods
   - Scale up resources
   - Clear stuck queues
   - Failover database connections
5. Monitors recovery
6. Documents incident

---

#### 11.6 Deploy a new feature
> As an engineer, I want to safely deploy new code to production.

**Key Interactions:**
1. Completes feature development and testing
2. Creates PR with changes
3. Gets code review approval
4. Merges to main branch
5. CI/CD deploys to staging
6. Verifies feature works in staging
7. Promotes to production
8. Monitors metrics and logs for issues
9. Rolls back if problems detected

---

#### 11.7 Investigate why a delivery was rejected
> As an engineer, I want to understand why the dispatch engine rejected a delivery request.

**Key Interactions:**
1. Gets delivery/quote request details
2. Search Datadog logs: `service:dispatch-engine @action:estimate`
3. Find the quote request by time and merchant/location
4. Review `LimitingFactor` returned:
   - `RobotAvailability` - No healthy robots available
   - `PilotAvailability` - No pilots on shift
   - `NotRobotAddressable` - Location outside service area
   - `Unroutable` - Can't calculate route
5. Drill into specific reason:
   - For robot availability: check which robots were considered and why rejected
   - For pilot availability: check pilot shift status
   - For routing: check maps service logs
6. Document findings


---

## 12. Operations Analyst

**Profile**: Data-focused role responsible for tracking fleet effectiveness, identifying improvement opportunities, and understanding what's preventing successful deliveries. Critical for maximizing revenue (which correlates directly with successful deliveries).

**Applications**: Analytics dashboards (Redshift/dbt), Mission Control, Data warehouse queries, BI tools

### User Journeys

#### 12.1 Understand why deliveries are failing
> As an operations analyst, I want to know the breakdown of reasons deliveries don't complete successfully.

**Key Interactions:**
1. Query delivery data for cancellation reasons
2. Categorize failures by type:
   - **Hardware issues**: Robot stuck, flipped, lid malfunction, battery
   - **Software issues**: System errors, routing failures
   - **Pilot issues**: Availability, errors, speed
   - **Merchant issues**: Long load, unresponsive, order problems
   - **Customer issues**: Unresponsive, cancelled, wrong address
   - **Environmental**: Weather, route blocked, terrain
3. Calculate failure rates by category
4. Identify trends over time
5. Prioritize categories with highest impact on revenue
6. Present findings to leadership with recommendations

**Key Questions:**
- What % of deliveries fail, and why?
- Which failure reasons are increasing/decreasing?
- Which failure reasons are preventable?
- What's the revenue impact of each failure category?

---

#### 12.2 Analyze robot fleet effectiveness by location
> As an operations analyst, I want to know which parking lots/zones have the most effective robots.

**Key Interactions:**
1. Query delivery data grouped by parking lot / deployment zone
2. Calculate metrics per location:
   - Deliveries completed per robot per day
   - Success rate
   - Average delivery time
   - Incidents per delivery
   - Time between deliveries (idle time)
3. Compare locations to identify top and bottom performers
4. Investigate factors driving differences:
   - Demand density
   - Route complexity
   - Robot health at that location
   - FO response times
5. Recommend reallocation or operational changes

**Key Questions:**
- Which parking lots generate the most deliveries per robot?
- Which zones have the highest success rates?
- Where are robots sitting idle most often?
- Should we redeploy robots from low-performing to high-performing areas?

---

#### 12.3 Understand robot unavailability
> As an operations analyst, I want to know why robots aren't available to take deliveries.

**Key Interactions:**
1. Query robot state data over time
2. Categorize unavailability reasons:
   - **Grounded**: Hardware issues, needs maintenance
   - **Low battery**: Waiting for swap
   - **In maintenance**: At MRO
   - **No pilot**: Pilot availability constraint
   - **Off duty**: Not deployed
   - **On trip**: Already doing a delivery (good!)
3. Calculate time spent in each state
4. Identify bottlenecks:
   - Are robots grounded too long before FO responds?
   - Are battery swaps taking too long?
   - Is MRO turnaround time too slow?
5. Quantify revenue impact of each unavailability type
6. Recommend process improvements

**Key Questions:**
- What % of robot-hours are spent unavailable vs. delivering?
- What's the average time a robot is grounded before being fixed?
- How much revenue are we losing to each unavailability reason?

---

#### 12.4 Analyze demand rejection reasons
> As an operations analyst, I want to know why we're rejecting delivery requests.

**Key Interactions:**
1. Query quote/demand data for rejections
2. Categorize by `LimitingFactor`:
   - `RobotAvailability` - No robots available
   - `PilotAvailability` - No pilots available
   - `NotRobotAddressable` - Outside service area
   - `Unroutable` - Can't calculate route
3. Calculate rejection rates by reason, time of day, zone
4. Identify patterns:
   - Are we rejecting more during peak hours?
   - Which zones have highest rejection rates?
   - Is pilot or robot availability the bigger constraint?
5. Quantify lost revenue from rejections
6. Recommend capacity changes (more robots, more pilots, zone expansion)

**Key Questions:**
- How many delivery requests do we reject?
- What's the primary constraint: robots or pilots?
- During which hours do we reject the most?
- How much revenue are we leaving on the table?

---

#### 12.5 Track robot servicing efficiency
> As an operations analyst, I want to understand how efficiently we're servicing robots.

**Key Interactions:**
1. Query FO task data
2. Calculate metrics:
   - Average time from task creation to completion
   - Average FO response time (creation to acceptance)
   - Average travel time to robot
   - Average task execution time
   - Tasks per FO per shift
3. Break down by task type (battery swap, unstuck, rescue, etc.)
4. Identify bottlenecks:
   - Are FOs taking too long to accept tasks?
   - Is travel time the biggest factor?
   - Are certain task types taking longer than expected?
5. Compare across zones/contractors
6. Recommend staffing or process changes

**Key Questions:**
- How long does a robot sit grounded before an FO arrives?
- Which task types take the longest?
- Are some FO contractors more efficient than others?
- Do we have enough FO coverage in each zone?

---

#### 12.6 Measure delivery time performance
> As an operations analyst, I want to understand our delivery time performance and what's slowing us down.

**Key Interactions:**
1. Query delivery time data
2. Break down total delivery time into phases:
   - Quote to robot assignment
   - JITP time (robot to merchant)
   - Load time (at merchant)
   - Delivery time (merchant to customer)
   - Dropoff time (customer pickup)
3. Calculate averages and distributions for each phase
4. Identify which phases have the most variance
5. Drill into slow phases:
   - Long JITP → robots deployed too far from merchants?
   - Long load time → merchant training needed?
   - Long delivery → route issues? pilot speed?
6. Recommend improvements

**Key Questions:**
- What's our average end-to-end delivery time?
- Which phase of delivery takes the longest?
- Where is there the most variance?
- How do we compare to our quoted ETAs?

---

#### 12.7 Build executive dashboard
> As an operations analyst, I want to provide leadership with a clear view of fleet performance.

**Key Interactions:**
1. Define key metrics for executive visibility:
   - Total deliveries (daily/weekly/monthly)
   - Success rate
   - Revenue
   - Fleet utilization
   - Rejection rate
   - Average delivery time
2. Build dashboard in BI tool
3. Set up automated refresh
4. Add trend lines and comparisons (vs. last week, last month)
5. Highlight anomalies and areas needing attention
6. Present to leadership regularly

---

#### 12.8 Investigate a drop in delivery volume
> As an operations analyst, I want to understand why delivery volume dropped.

**Key Interactions:**
1. Identify time period of drop
2. Check potential causes:
   - Demand side: Did partner platforms send fewer requests?
   - Supply side: Did we reject more requests?
   - Execution side: Did more deliveries fail?
3. Query data to isolate cause
4. If demand drop → check partner data, merchant status
5. If supply drop → check robot/pilot availability
6. If execution drop → check failure reasons
7. Report findings with root cause


---

## 13. MRO Technician

**Profile**: Maintenance, Repair, and Operations specialist who repairs and prepares robots.

**Applications**: MRO Dashboard, Mission Control, Device Service

### User Journeys

#### 13.1 Diagnose and repair a robot
> As an MRO technician, I want to fix a robot that came in with issues.

**Key Interactions:**
1. Robot arrives at MRO facility
2. Checks in robot in system
3. Reviews reported issues and robot history
4. Runs diagnostic tests
5. Identifies failed/failing components
6. Performs repairs (replace parts, recalibrate, update firmware)
7. Runs verification tests
8. Marks robot ready for deployment
9. Checks out robot

---

#### 13.2 Perform scheduled maintenance
> As an MRO technician, I want to complete routine maintenance on robots.

**Key Interactions:**
1. Reviews maintenance schedule
2. Identifies robots due for service
3. Performs standard maintenance checklist:
   - Clean cameras and sensors
   - Check battery health
   - Inspect wheels and motors
   - Verify lid mechanism
   - Update firmware if needed
4. Documents maintenance performed
5. Returns robot to available pool

---

#### 13.3 Prepare robots for deployment
> As an MRO technician, I want to prepare robots for field deployment.

**Key Interactions:**
1. Receives deployment preparation request
2. Selects robots from available pool
3. Performs pre-deployment checks
4. Ensures batteries are charged
5. Verifies all systems operational
6. Marks robots ready for deployment
7. Coordinates with FO team for pickup

---

## 14. System Administrator

**Profile**: Administrator managing users, roles, and system configuration.

**Applications**: Auth0, Admin panels, AWS IAM

### User Journeys

#### 14.1 Create a new user account
> As a system admin, I want to set up access for a new team member.

**Key Interactions:**
1. Receives request for new user access
2. Creates user in Auth0
3. Assigns appropriate role(s) based on job function
4. Sends credentials/invite to user
5. Verifies user can access required systems

**Role Assignments:**

| Job Function | Typical Roles |
|--------------|---------------|
| Pilot | `Pilots` |
| Field Operator | `FieldOps` |
| Dispatch | `Dispatch` |
| Operations Support | `Support`, `OpsAdmin` |
| GM | `OpsAdmin` |
| Engineer | `Engineering` |
| MRO Tech | `MROTech` |

---

#### 14.2 Modify user permissions
> As a system admin, I want to change a user's access level.

**Key Interactions:**
1. Receives request to modify access
2. Reviews current user roles
3. Adds or removes roles as needed
4. Verifies changes took effect
5. Notifies user of changes

---

#### 14.3 Audit user access
> As a system admin, I want to review who has access to what.

**Key Interactions:**
1. Pulls user list from Auth0
2. Reviews role assignments
3. Identifies any inappropriate access
4. Removes access for departed employees
5. Documents audit findings


---

## 15. Partner Platform

**Profile**: External delivery platform (Uber Eats, DoorDash) integrating via API.

**Applications**: API integration (webhooks)

### Integration Journeys

#### 15.1 Request a delivery quote
> As a partner platform, I want to check if Coco can fulfill a delivery and get pricing.

**Key Interactions:**
1. Partner sends quote request via API (pickup location, dropoff location, order details)
2. Coco checks robot availability
3. Coco calculates route and ETA
4. Coco returns quote (ETA, price) or rejection reason
5. Partner decides whether to proceed

**Rejection Reasons:**
- `RobotAvailability` - No robots available
- `PilotAvailability` - No pilots available
- `NotRobotAddressable` - Location not serviceable
- `Unroutable` - Can't calculate route

---

#### 15.2 Create a delivery
> As a partner platform, I want to book a robot delivery.

**Key Interactions:**
1. Partner accepts quote and sends create request
2. Coco creates delivery record
3. Coco assigns robot and schedules trip
4. Coco returns delivery ID and confirmation
5. Partner receives status webhooks as delivery progresses

---

#### 15.3 Receive delivery status updates
> As a partner platform, I want to know the current status of a delivery.

**Key Interactions:**
1. Coco sends webhook on status change
2. Partner receives and processes webhook
3. Partner updates their UI for customer

**Webhook Events:**
- `created` - Delivery created
- `transition` - Status changed (in transit, arrived, etc.)
- `completed` - Delivery successful
- `cancelled` - Delivery cancelled
- `driver-updated` - Robot info changed

---

#### 15.4 Cancel a delivery
> As a partner platform, I want to cancel a delivery that's no longer needed.

**Key Interactions:**
1. Partner sends cancel request via API
2. Coco processes cancellation
3. If food already loaded → triggers rescue/empty flow
4. Coco confirms cancellation
5. Partner receives cancelled webhook

---

## Summary: Key Improvement Tracking Questions

The Operations Analyst persona is critical for answering these business questions:

### Revenue & Delivery Success
- What % of deliveries complete successfully?
- What are the top reasons for delivery failures?
- How much revenue are we losing to each failure type?

### Fleet Effectiveness
- Which zones/parking lots are most productive?
- What's our robot utilization rate?
- How much time do robots spend unavailable vs. delivering?

### Capacity & Constraints
- Are we rejecting demand? Why?
- Is the constraint robots, pilots, or something else?
- Where should we add capacity?

### Operational Efficiency
- How long does it take to service a grounded robot?
- What's our average delivery time by phase?
- Where are the bottlenecks?

---

## Related Documents

- [[THE_LIFE_OF_A_ROBOT]] - Robot daily lifecycle
- [[THE_LIFE_OF_AN_ORDER]] - Delivery order lifecycle
- [[Delivery & Dispatch Flow]] - Dispatch engine details
- [[Field Op Performs Robot Maintenance Flow]] - FO task workflows
- [[Magic Lid Loading Flow]] - Merchant loading details

