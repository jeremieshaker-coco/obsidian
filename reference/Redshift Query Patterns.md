---
tags:
  - database
  - redshift
  - analytics
  - sql
  - best-practices
---
# Redshift Query Patterns

Common query patterns and best practices for analyzing data in the [[Redshift Data Warehouse]].

## Cross-Schema Joins

### Delivery with Merchant Configuration

Join operational delivery data with merchant config:

```sql
SELECT 
  d.id AS delivery_id,
  d.created_at,
  a.status AS attempt_status,
  m.name AS merchant_name,
  m.timezone,
  pc.store_name,
  pc.load_time,
  pc.salesforce_account_id
FROM deliveriesv3prod_rds_public.deliveries d
LEFT JOIN deliveriesv3prod_rds_public.attempts a ON d.latest_attempt_id = a.id
JOIN config_rds_public.merchant m ON d.partner_id = m.merchant_id
LEFT JOIN deliveries_rds_public.partner_configs pc ON d.partner_id = pc.partner_id
WHERE d.created_at >= CURRENT_DATE - 7
  AND m.is_test_merchant = FALSE
  AND d._fivetran_deleted = FALSE;
```

### Delivery with Robot Location

```sql
SELECT 
  d.id,
  d.hub_id,
  m.name AS merchant_name,
  rl.name AS parking_lot_name,
  rl.latitude AS parking_lat,
  rl.longitude AS parking_lng
FROM deliveriesv3prod_rds_public.deliveries d
JOIN config_rds_public.merchant m ON d.partner_id = m.merchant_id
LEFT JOIN config_rds_public.robot_location rl 
  ON m.parking_lot_id = rl.robot_location_id
WHERE d.created_at >= CURRENT_DATE - 1
  AND d._fivetran_deleted = FALSE;
```

## Analytics Queries

### Daily Delivery Volume by Merchant

```sql
SELECT 
  DATE_TRUNC('day', d.created_at) AS delivery_date,
  pc.store_name,
  pc.city,
  COUNT(*) AS delivery_count,
  COUNT(CASE WHEN a.status = 'DELIVERED' THEN 1 END) AS delivered_count,
  AVG(EXTRACT(EPOCH FROM (d.updated_at - d.created_at))/60) AS avg_duration_minutes
FROM deliveriesv3prod_rds_public.deliveries d
LEFT JOIN deliveriesv3prod_rds_public.attempts a ON d.latest_attempt_id = a.id
LEFT JOIN deliveries_rds_public.partner_configs pc ON d.partner_id = pc.partner_id
WHERE d.created_at >= CURRENT_DATE - 30
  AND d._fivetran_deleted = FALSE
GROUP BY 1, 2, 3
ORDER BY 1 DESC, 4 DESC;
```

### Provider Mix Analysis

```sql
SELECT 
  a.provider,
  COUNT(*) AS total_deliveries,
  COUNT(CASE WHEN a.status = 'DELIVERED' THEN 1 END) AS successful,
  COUNT(CASE WHEN a.status = 'CANCELLED' THEN 1 END) AS cancelled,
  ROUND(100.0 * COUNT(CASE WHEN a.status = 'DELIVERED' THEN 1 END) / COUNT(*), 2) AS success_rate
FROM deliveriesv3prod_rds_public.deliveries d
LEFT JOIN deliveriesv3prod_rds_public.attempts a ON d.latest_attempt_id = a.id
WHERE d.created_at >= CURRENT_DATE - 7
  AND d._fivetran_deleted = FALSE
GROUP BY 1
ORDER BY 2 DESC;
```

### Customer Behavior Analysis

From [[Coco Marketplace Analytics Schema]]:

```sql
-- Daily Active Users
SELECT 
  DATE_TRUNC('day', timestamp) AS date,
  COUNT(DISTINCT user_id) AS dau,
  COUNT(DISTINCT anonymous_id) AS sessions
FROM coco_marketplace_prod_analytics.tracks
WHERE timestamp >= CURRENT_DATE - 30
  AND user_id IS NOT NULL
GROUP BY 1
ORDER BY 1 DESC;

-- Feature Usage
SELECT 
  event,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_id) AS unique_users
FROM coco_marketplace_prod_analytics.tracks
WHERE timestamp >= CURRENT_DATE - 7
  AND event LIKE '%Button Clicked%'
GROUP BY 1
ORDER BY 2 DESC;
```

### Pilot Operations Analysis

From [[Delivery Platform Events Schema]]:

```sql
-- Troubleshooting Steps by Pilot
SELECT 
  user_id AS pilot_id,
  DATE_TRUNC('day', timestamp) AS date,
  COUNT(*) AS troubleshooting_events,
  COUNT(DISTINCT trip_id) AS unique_trips,
  COUNT(CASE WHEN event = 'Pilot Hard Reset' THEN 1 END) AS hard_resets,
  COUNT(CASE WHEN event = 'Pilot Route Issue' THEN 1 END) AS route_issues
FROM delivery_platform_prod.pilot_troubleshooting_step
WHERE timestamp >= CURRENT_DATE - 30
GROUP BY 1, 2
ORDER BY 2 DESC, 3 DESC;

-- Incident Types
SELECT 
  incident_type,
  COUNT(*) AS incident_count,
  COUNT(DISTINCT pilot_id) AS pilots_reporting
FROM delivery_platform_prod.pilot_incident_report
WHERE timestamp >= CURRENT_DATE - 30
GROUP BY 1
ORDER BY 2 DESC;
```

## Event Funnel Analysis

### Order Tracker Funnel

```sql
WITH tracker_events AS (
  SELECT 
    order_id,
    MIN(CASE WHEN event = 'Order Tracker Open' THEN timestamp END) AS opened_at,
    MIN(CASE WHEN event = 'Order Tracker FAQ Open' THEN timestamp END) AS faq_at,
    MIN(CASE WHEN event = 'Order Tracker Rating Submit' THEN timestamp END) AS rating_at
  FROM delivery_platform_prod.tracks
  WHERE timestamp >= CURRENT_DATE - 7
    AND order_id IS NOT NULL
    AND event LIKE 'Order Tracker%'
  GROUP BY 1
)
SELECT 
  COUNT(*) AS total_opens,
  COUNT(faq_at) AS faq_views,
  COUNT(rating_at) AS ratings_submitted,
  ROUND(100.0 * COUNT(faq_at) / COUNT(*), 2) AS faq_rate,
  ROUND(100.0 * COUNT(rating_at) / COUNT(*), 2) AS rating_rate
FROM tracker_events;
```

### Unlock Success Rate

```sql
WITH unlock_attempts AS (
  SELECT 
    delivery_id,
    MIN(timestamp) AS attempt_time,
    MAX(CASE WHEN event = 'Deliveries Unlock Coco Success' THEN 1 ELSE 0 END) AS succeeded,
    MAX(CASE WHEN event = 'Deliveries Unlock Coco Fail' THEN 1 ELSE 0 END) AS failed
  FROM delivery_platform_prod.tracks
  WHERE timestamp >= CURRENT_DATE - 7
    AND event LIKE '%Unlock Coco%'
  GROUP BY 1
)
SELECT 
  COUNT(*) AS total_attempts,
  SUM(succeeded) AS successful_unlocks,
  SUM(failed) AS failed_unlocks,
  ROUND(100.0 * SUM(succeeded) / COUNT(*), 2) AS success_rate
FROM unlock_attempts;
```

## Salesforce Integration Queries

From [[Salesforce Data Sync]]:

### Merchant Onboarding Status

```sql
SELECT 
  sa.name AS merchant,
  sa.city_hub__c AS hub,
  sc.status AS onboarding_status,
  sc.estimated_activation_date__c AS target_launch,
  sc.actual_activation_date__c AS actual_launch,
  sc.number_of_locations__c AS location_count,
  ARRAY_REMOVE(ARRAY[
    CASE WHEN sc.doordash__c = 1 THEN 'DoorDash' END,
    CASE WHEN sc.ubereats__c = 1 THEN 'UberEats' END,
    CASE WHEN sc.grub__c = 1 THEN 'Grubhub' END,
    CASE WHEN sc.native__c = 1 THEN 'Native' END
  ], NULL) AS platforms
FROM deliveries_rds_public.salesforce_cases sc
JOIN deliveries_rds_public.salesforce_accounts sa ON sc.accountid = sa.id
WHERE sc.type = 'Onboarding'
  AND sc._fivetran_deleted = FALSE
ORDER BY 
  COALESCE(sc.actual_activation_date__c, sc.estimated_activation_date__c) DESC;
```

### Revenue Pipeline

```sql
SELECT 
  sa.name AS account,
  sa.brand__c AS brand,
  so.stagename,
  so.closedate,
  so.locs__c AS locations,
  sq.sbqq__subscriptionterm__c AS term_months,
  SUM(sql.sbqq__netprice__c * sql.sbqq__quantity__c) AS total_value
FROM deliveries_rds_public.salesforce_opportunities so
JOIN deliveries_rds_public.salesforce_accounts sa ON so.accountid = sa.id
LEFT JOIN deliveries_rds_public.salesforce_sbqq_quotes sq ON so.sbqq__primaryquote__c = sq.id
LEFT JOIN deliveries_rds_public.salesforce_sbqq_quotelines sql ON sq.id = sql.sbqq__quote__c
WHERE so._fivetran_deleted = FALSE
  AND so.stagename NOT IN ('Closed Lost', 'Cancelled')
GROUP BY 1, 2, 3, 4, 5, 6
ORDER BY 7 DESC;
```

## Performance Best Practices

### Use Distribution Keys

Redshift distributes data across nodes. Query performance improves when joining on distribution keys.

```sql
-- Good: Join on distribution key (likely id)
SELECT d.*, a.*
FROM deliveriesv3prod_rds_public.deliveries d
LEFT JOIN deliveriesv3prod_rds_public.delivery_attempts da ON d.id = da.delivery_id
LEFT JOIN deliveriesv3prod_rds_public.attempts a ON da.attempt_id = a.id;

-- Less optimal: Join on non-distribution key
SELECT d.*, m.*
FROM deliveriesv3prod_rds_public.deliveries d
JOIN config_rds_public.merchant m ON d.partner_id = m.merchant_id;
```

### Filter on Sort Keys

Tables are often sorted by timestamp columns. Filter on these for better performance.

```sql
-- Good: Filter on sort key (created_at)
SELECT *
FROM deliveriesv3prod_rds_public.deliveries
WHERE created_at >= '2024-01-01'
  AND created_at < '2024-02-01'
  AND _fivetran_deleted = FALSE;

-- Less optimal: No time filter
SELECT *
FROM deliveriesv3prod_rds_public.deliveries
WHERE partner_id = 'some_partner'
  AND _fivetran_deleted = FALSE;
```

### Use Column Subset

Only select columns you need:

```sql
-- Good: Select specific columns
SELECT id, created_at, latest_attempt_id
FROM deliveriesv3prod_rds_public.deliveries
WHERE created_at >= CURRENT_DATE - 7
  AND _fivetran_deleted = FALSE;

-- Bad: Select all columns
SELECT *
FROM deliveriesv3prod_rds_public.deliveries
WHERE created_at >= CURRENT_DATE - 7;
```

### Handle Fivetran Soft Deletes

Always filter out soft-deleted records:

```sql
SELECT *
FROM config_rds_public.merchant
WHERE _fivetran_deleted = FALSE;
```

### Use EXPLAIN for Query Plans

```sql
EXPLAIN 
SELECT d.id, COUNT(a.id) AS attempt_count
FROM deliveriesv3prod_rds_public.deliveries d
LEFT JOIN deliveriesv3prod_rds_public.delivery_attempts da ON d.id = da.delivery_id
LEFT JOIN deliveriesv3prod_rds_public.attempts a ON da.attempt_id = a.id
WHERE d.created_at >= CURRENT_DATE - 7
  AND d._fivetran_deleted = FALSE
GROUP BY 1;
```

## Common Pitfalls

### JSON Column Access

Segment event properties are often stored as JSON:

```sql
-- Parse JSON properties
SELECT 
  delivery_id,
  JSON_EXTRACT_PATH_TEXT(properties, 'merchant_name') AS merchant_name,
  JSON_EXTRACT_PATH_TEXT(properties, 'delivery_status') AS status
FROM delivery_platform_prod.tracks
WHERE event = 'Order Tracker Open';
```

### Timezone Handling

```sql
-- Convert UTC to merchant timezone
SELECT 
  d.id,
  d.created_at AT TIME ZONE 'UTC' AT TIME ZONE m.timezone AS created_at_local,
  m.timezone
FROM deliveriesv3prod_rds_public.deliveries d
JOIN config_rds_public.merchant m ON d.partner_id = m.merchant_id
WHERE d._fivetran_deleted = FALSE;
```

### V3 Schema Differences

V3 schema has structural differences from V2:

```sql
-- V3: No current_attempt_status on deliveries - join to attempts
SELECT 
  d.id,
  d.latest_attempt_id,
  a.status AS current_status,
  a.provider
FROM deliveriesv3prod_rds_public.deliveries d
LEFT JOIN deliveriesv3prod_rds_public.attempts a ON d.latest_attempt_id = a.id
WHERE d._fivetran_deleted = FALSE;

-- V3: Get all attempts for a delivery (many-to-many)
SELECT 
  d.id,
  a.id AS attempt_id,
  a.status,
  a.provider,
  a.created_at
FROM deliveriesv3prod_rds_public.deliveries d
LEFT JOIN deliveriesv3prod_rds_public.delivery_attempts da ON d.id = da.delivery_id
LEFT JOIN deliveriesv3prod_rds_public.attempts a ON da.attempt_id = a.id
WHERE d.id = 'del_123'
ORDER BY a.created_at;

-- Note: Partner configs still in legacy schema
SELECT 
  d.id,
  pc.store_name,
  pc.load_time
FROM deliveriesv3prod_rds_public.deliveries d
LEFT JOIN deliveries_rds_public.partner_configs pc ON d.partner_id = pc.partner_id;
```

## Schema Reference Notes

**Current Schema (V3):** `deliveriesv3prod_rds_public`
- Primary schema for delivery operations
- Uses string IDs (`delivery_id`, `attempt_id`)
- Has `customer` and `dropoff` tables
- Uses `delivery_attempts` join table for many-to-many relationship
- No `current_attempt_status` - use `latest_attempt_id` + join

**Legacy Data:** `deliveries_rds_public`
- Partner configs (`partner_configs`, `partners`)
- Salesforce sync tables (`salesforce_*`)
- Historical delivery data (pre-V3 migration)
- OLO/DoorDash integration tables

**Config Data:** `config_rds_public`
- Merchant and robot location configurations
- Integration settings
- Tags and audit history

## Related Concepts

- [[Redshift Data Warehouse]] - Overview
- [[Deliveries V3 RDS Schema]] - Current schema (primary)
- [[Deliveries RDS Schema]] - Legacy schema (partner configs, Salesforce)
- [[Config RDS Schema]] - Configuration data
- [[Delivery Platform Events Schema]] - Event data
- [[Segment Analytics Integration]] - Event collection

