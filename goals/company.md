# Company Goals

## What is Coco?

Coco is a **last-mile delivery service** that deploys **human-operated sidewalk robots** to transport food, groceries, and small goods from merchants to customers. Unlike fully autonomous delivery companies, Coco robots are remotely driven by **pilots**—human operators who control the robots in real-time using live video feeds.

## Value Proposition

Coco delivers **more reliably, sustainably, and consistently** than traditional courier-based delivery:

- **Reliability**: Human pilots make real-time decisions; technology provides safety features (emergency brakes, collision avoidance)
- **Sustainability**: Electric robots on sidewalks vs. cars/scooters on roads
- **Consistency**: Predictable delivery experience without the variability of gig-economy couriers

## How It Works

1. **Customer orders** through a partner platform (Uber Eats, DoorDash, Wolt)
2. **Dispatch Engine** assigns a robot and calculates route
3. **Pilot is assigned** and remotely drives the robot to merchant
4. **Merchant loads** the robot's secure compartment
5. **Pilot drives** the robot on sidewalks to the customer's address
6. **Customer unlocks** the robot via PIN code and retrieves their order
7. **Robot returns** to a parking lot or storage location (PODS)

## Key Roles

| Role | Description |
|------|-------------|
| **Pilot** | Remote operator who drives robots via Pilot UI (video feed + joystick) |
| **Field Operator (FO)** | Physical operator who deploys/retrieves robots, uses iOS app |
| **MRO Team** | Maintenance, Repairs, and Operations |

## Business Model

Coco operates as a **delivery fulfillment partner** for major food delivery platforms. Revenue comes from per-delivery fees.

### Supply & Demand Dynamics

Coco's business is a **two-sided matching problem**:

- **Supply**: Our robot fleet and pilot workforce—we control this
- **Demand**: Delivery requests from partners—we don't control this, but we need to understand it

#### Partner Integration Models

| Partner | How Demand Works | Coco's Visibility |
|---------|------------------|-------------------|
| **Uber Eats** | Coco publishes fleet heartbeat data → Uber decides when to send us requests for specific robots | **Opaque** - Uber's selection criteria is a black box |
| **DoorDash** | DoorDash broadcasts a chunk of their demand → Coco decides whether to quote/accept | **More transparent** - We see what's available (though broadcast criteria unclear) |
| **Direct Merchants** | Merchants use Coco for first-party delivery | **Full visibility** |

In all cases, Coco can **accept or reject** delivery requests. Understanding our accept/reject patterns is critical to knowing if we're missing out on demand.

#### Location Matters

Robot location significantly impacts demand received, especially for Uber. A robot parked near a high-demand merchant cluster will see more offers than one in a quiet area. This creates a feedback loop:
- Better positioning → more demand → higher utilization → better unit economics

## Key Concepts

| Term | Definition |
|------|------------|
| **Delivery** | A request from a partner to move goods from merchant → customer |
| **Attempt** | A single try to complete a delivery; max 2 per delivery (robot first, then "rescue" by human courier if needed) |
| **Trip** | The movement of a robot (pickup trip, dropoff trip, return trip) |
| **Demand** | An order/delivery request in the dispatch system |
| **Parking Lot** | Location near merchants where robots wait for assignments |
| **Storage/PODS** | Overnight storage location for robots |

## Strategic Goals

*TODO: Define current company-level OKRs and strategic priorities*

---

## Why This Matters for Engineering

Understanding the business model is critical for making good technical decisions:

### Supply Side (Fleet Efficiency)
1. **Pilot efficiency = unit economics** - Every second a pilot waits is lost productivity
2. **Robot utilization = capacity** - Idle robots don't generate revenue
3. **Delivery success rate = partner trust** - Failed deliveries hurt platform relationships

### Demand Side (Order Flow)
4. **Demand visibility = strategic decisions** - Understanding what demand we receive, accept, and reject tells us if we're missing opportunities
5. **Location-based demand patterns** - Knowing which areas generate more offers informs fleet positioning and expansion strategy
6. **Accept/reject analysis** - Are we rejecting demand because we're at capacity, or because of suboptimal matching? These are very different problems

### Observability
7. **You can't improve what you can't measure** - Both supply efficiency and demand capture require systematic tracking to optimize
