---
tags:
  - database
  - redshift
  - analytics
  - data-warehouse
---
# Redshift Data Warehouse

The Coco delivery platform uses Amazon Redshift as its data warehouse for analytics and business intelligence. Redshift aggregates data from multiple operational databases and event streams.

## Schema Namespaces

The warehouse is organized into five main schema namespaces:

1. **[[Coco Marketplace Analytics Schema]]** - User behavior and analytics events from Segment
2. **[[Config RDS Schema]]** - Configuration and settings for merchants and robots
3. **[[Deliveries RDS Schema]]** - Legacy delivery system data (operational)
4. **[[Deliveries V3 RDS Schema]]** - Current delivery system data (operational)
5. **[[Delivery Platform Events Schema]]** - Event tracking for user interactions and system operations

## Data Pipeline

Data flows into Redshift through:
- **Fivetran** - Syncs operational RDS databases (see `_fivetran_synced` columns)
- **Segment** - Streams analytics events in real-time
- **[[Salesforce Data Sync]]** - CRM data integration

## Use Cases

- Business intelligence and reporting
- Customer behavior analysis
- Operational metrics and KPIs
- [[Pilot UI]] and [[Mission Control UI]] analytics
- [[Delivery]] lifecycle tracking
- Merchant performance analysis

## Related Services

- [[Deliveries Service]] - Source of delivery operational data
- [[Operations Service]] - Source of task and trip data
- [[Config Service]] - Source of configuration data

