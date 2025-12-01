---
tags:
  - frontend
  - ui
  - customer
  - legacy
---
# Tracker UI

(Legacy) A customer-facing interface to track the real-time status of a [[Delivery]]. It connects to the [[Deliveries Service]]. The primary flow for end-customer delivery tracking is now directly through partner apps like DoorDash and Uber Eats.

## Features

- Real-time delivery status updates
- Live map with robot location
- Estimated time of arrival (ETA)
- Self-service robot unlock
- FAQ and support contact
- Delivery rating and feedback

## Analytics

Customer interactions tracked via [[Segment Analytics Integration]]:
- Order tracker page views and navigation
- FAQ interactions
- Robot unlock events
- Delivery ratings and feedback
- App download banner interactions

Events stored in [[Delivery Platform Events Schema]]:
- `order_tracker_open` - Customer views tracking page
- `order_tracker_faq_open` - FAQ item expanded
- `order_tracker_rating_submit` - Rating submitted
- `order_unlock_coco` - Customer unlocks robot
- `app_banner_viewed/clicked/dismissed` - Marketing banner interactions

## Related Concepts

- [[Deliveries Service]] - Backend API
- [[Customer Unlocks Robot for Pickup Flow]] - Unlock workflow
- [[Track a Delivery Flow]] - Overall tracking flow
- [[Delivery Platform Events Schema]] - Analytics data
