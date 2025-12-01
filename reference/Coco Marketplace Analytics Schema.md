---
tags:
  - database
  - redshift
  - analytics
  - segment
---
# Coco Marketplace Analytics Schema

**Namespace:** `coco_marketplace_prod_analytics`

This schema contains user behavior and analytics data collected through [[Segment Analytics Integration]]. It tracks customer interactions with the Coco marketplace applications.

## Core Tables

### `tracks`
Generic event tracking table for all user actions.

**Key columns:**
- `id` - Unique event identifier
- `event` - Event name (e.g., "Button Clicked", "Page Viewed")
- `user_id` - Authenticated user identifier
- `anonymous_id` - Anonymous session identifier
- `timestamp` - Event occurrence time
- `context_*` - Device, browser, and session context
- `event_text` - Additional event details

### `screen_rendered`
Mobile app screen view events.

**Key columns:**
- `anonymous_id` - Session identifier
- `context_app_version` - App version
- `context_device_model` - Device model
- `event_text` - Screen identifier

### `users`
User profile data synchronized from app.

**Key columns:**
- `id` - User identifier
- `first_name` - User's first name
- `created_at` - Account creation timestamp
- `install_type` - App installation source
- `attribution_channel` - Marketing attribution
- `attribution_source` - Referral source

### `user_created`
Event fired when new user accounts are created.

## Event Context Fields

All events include rich context:
- **Device:** `context_device_type`, `context_device_manufacturer`, `context_device_model`
- **Location:** `context_timezone`, `context_locale`
- **Network:** `context_network_wifi`, `context_network_cellular`
- **App:** `context_app_name`, `context_app_version`, `context_app_build`
- **Web:** `context_page_url`, `context_page_path`, `context_user_agent`

## Related Concepts

- [[Segment Analytics Integration]] - How events are collected
- [[Delivery Platform Events Schema]] - Server-side and operational events
- [[Tracker UI]] - Primary source of customer-facing events
- [[Merchant UI]] - Source of merchant portal events

