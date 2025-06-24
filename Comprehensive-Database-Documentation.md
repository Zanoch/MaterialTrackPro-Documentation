# MTP Platform - Comprehensive Database Documentation

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture & Design](#architecture--design)
3. [AWS-Manager Database](#aws-manager-database)
4. [Central-Handler Database](#central-handler-database)
5. [Integration Patterns](#integration-patterns)
6. [Data Flow & Workflows](#data-flow--workflows)
7. [Performance & Optimization](#performance--optimization)
8. [Security & Compliance](#security--compliance)
9. [Monitoring & Maintenance](#monitoring--maintenance)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## System Overview

The MTP (Tea Management) Platform operates on a **distributed database architecture** consisting of two separate PostgreSQL database servers, each running their own instance of a database named `tea_management`. This architecture provides system independence while maintaining data consistency through well-designed API integration patterns.

### Key Components

- **AWS-Manager Database**: Order management and user administration
- **Central-Handler Database**: Tea processing operations and inventory tracking
- **API Integration Layer**: Real-time data synchronization and workflow coordination

### Business Domain

The platform manages the complete tea processing lifecycle:
- Raw material procurement (tealine, herbline)
- Blending and flavor processing operations
- Quality control and trader approval workflows
- Order management and shipment tracking
- Production scheduling and allocation management

---

## Architecture & Design

### Distributed Database Pattern

```
┌─────────────────────────────────┐    API Integration    ┌─────────────────────────────────┐
│         AWS-Manager             │ ←──────────────────→ │       Central-Handler           │
│    PostgreSQL Instance          │                      │    PostgreSQL Instance          │
│                                 │                      │                                 │
│    tea_management Database      │                      │    tea_management Database      │
│    ├── admin schema            │                      │    └── public schema            │
│    │   └── User management     │                      │        ├── Processing tables    │
│    └── public schema           │                      │        ├── Allocation tables    │
│        ├── Order planning      │                      │        ├── Batch management     │
│        ├── Request management  │                      │        └── Storage management   │
│        ├── Shipment tracking   │                      │                                 │
│        └── Production schedule │                      │                                 │
└─────────────────────────────────┘                      └─────────────────────────────────┘
```

### Design Principles

**1. System Independence**
- Each database can scale independently
- Isolated failure domains prevent cascading failures
- Independent maintenance and upgrade cycles

**2. Data Consistency**
- API-based synchronization with transactional boundaries
- Eventual consistency with reconciliation mechanisms
- Cross-system identifiers for relationship mapping

**3. Performance Optimization**
- Optimized queries specific to each system's use cases
- Reduced database contention through separation
- Independent backup and recovery strategies

---

## AWS-Manager Database

### Purpose & Scope

The AWS-Manager database handles order lifecycle management and user administration, integrating with AWS services for authentication and serverless operations.

### Database Specifications

- **Database Name**: `tea_management`
- **Engine**: PostgreSQL 14.17+
- **Character Encoding**: UTF-8
- **Schemas**: `admin`, `public`

### Schema: Admin (`admin`)

#### `admin.user`
User account management with AWS Cognito integration.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `sub` | `uuid` | PRIMARY KEY | AWS Cognito User Pool subject |
| `username` | `text` | NOT NULL | User login identifier |
| `group` | `text` | NOT NULL | Permission group assignment |

### Schema: Public (`public`)

#### Order Planning Tables

**`order_plan`**
Production planning master data.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `order_code` | `text` | PRIMARY KEY | Unique order identifier |
| `product_name` | `text` | NOT NULL | Product specification |
| `requirement` | `integer` | NOT NULL | Required quantity |
| `plan_start` | `date` | NOT NULL | Planned start date |
| `plan_end` | `date` | NOT NULL | Planned completion date |

**`order_plan_raw`**
Staging table for bulk planning data imports.

#### Order Management Tables

**`order_request`**
Customer order requests with auto-generated tracking codes.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `request_code` | `bigint` | PRIMARY KEY, AUTO INCREMENT | Unique request identifier |
| `order_code` | `text` | NOT NULL | Reference to order plan |
| `product_name` | `text` | NOT NULL | Product specification |
| `requirement` | `integer` | NOT NULL | Requested quantity |
| `comments` | `text` | NULLABLE | Additional information |

**`order_shipment`**
Shipment allocations for fulfillment tracking.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `request_code` | `bigint` | PRIMARY KEY | Request reference |
| `shipment_code` | `integer` | PRIMARY KEY, DEFAULT 1 | Shipment number |
| `quantity` | `integer` | NULLABLE | Allocated quantity |

**`order_shipment_event`**
Comprehensive event tracking for shipment lifecycle.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `request_code` | `bigint` | PRIMARY KEY | Request reference |
| `shipment_code` | `integer` | PRIMARY KEY | Shipment reference |
| `status` | `text` | PRIMARY KEY | Event status type |
| `event_ts` | `bigint` | DEFAULT epoch_ms | Event timestamp |
| `shipment_vehicle` | `text` | NULLABLE | Transportation details |
| `order_remarks` | `text` | NULLABLE | Order-specific notes |
| `shipment_remarks` | `text` | NULLABLE | Shipment-specific notes |

**Valid Status Values**: APPROVAL_REQUESTED, ORDER_REQUESTED, APPROVAL_ALLOWED, SHIPMENT_ACCEPTED, SHIPMENT_DISPATCHED, ORDER_NOT_READY

**`order_schedule`**
Production scheduling with shift and section management.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `schedule_code` | `bigint` | PRIMARY KEY | Unique schedule identifier |
| `schedule_date` | `date` | NULLABLE | Production date |
| `order_code` | `text` | NULLABLE | Order reference |
| `shift` | `text` | NULLABLE | Production shift |
| `section` | `text` | NULLABLE | Production section |
| `quantity` | `integer` | NULLABLE | Scheduled quantity |
| `completed` | `boolean` | DEFAULT false | Completion status |

**Valid Shift Values**: NIGHT, DAY, NIGHT & DAY

---

## Central-Handler Database

### Purpose & Scope

The Central-Handler database manages tea processing operations from raw materials to finished products, including allocation tracking for AWS-Manager integration.

### Database Specifications

- **Database Name**: `tea_management`
- **Engine**: PostgreSQL 14.12+ / 15.6+
- **Character Encoding**: UTF-8
- **Primary Schema**: `public`

### Core Processing Tables

#### Raw Material Management

**`tealine`**
Tea leaf inventory and incoming raw materials.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY | Unique item identifier |
| `created_ts` | `bigint` | PRIMARY KEY | Creation timestamp |
| `invoice_no` | `text` | NULLABLE | Invoice reference |
| `grade` | `text` | NULLABLE | Tea grade classification |
| `no_of_bags` | `integer` | NOT NULL | Expected bag count |
| `weight_per_bag` | `numeric(5,2)` | NOT NULL | Expected bag weight |
| `broker` | `text` | NULLABLE | Broker information |
| `garden` | `text` | NULLABLE | Tea garden source |
| `purchase_order` | `text` | NULLABLE | Purchase reference |

**`herbline`**
Herbal ingredient management with storage restrictions.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY | Unique item identifier |
| `item_name` | `text` | NOT NULL | Ingredient name |
| `created_ts` | `bigint` | PRIMARY KEY | Creation timestamp |
| `purchase_order` | `text` | NULLABLE | Purchase reference |
| `weight` | `numeric` | NULLABLE | Expected weight |

**Storage Restriction**: Herbline items require specialized storage locations with `herbline_section = true`.

**`blendbalance`**
Blend inventory balance management.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY | Unique item identifier |
| `created_ts` | `bigint` | PRIMARY KEY | Creation timestamp |
| `blend_code` | `text` | NOT NULL | Blend classification |
| `weight` | `numeric(1000,2)` | NOT NULL | Available weight |
| `transfer_id` | `text` | NOT NULL | Transfer reference |

#### Formulation Management

**`blendsheet`**
Tea blending formulations and specifications.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `blendsheet_no` | `text` | PRIMARY KEY | Unique blendsheet identifier |
| `standard` | `text` | NOT NULL | Quality standard |
| `grade` | `text` | NULLABLE | Final blend grade |
| `remarks` | `text` | NOT NULL | Additional notes |
| `no_of_batches` | `integer` | DEFAULT 1 | Expected batch count |
| `total` | `numeric(8,2)` | NOT NULL | Total blend quantity |
| `blend_code` | `text` | NULLABLE | Blend classification |

**`flavorsheet`**
Flavor processing specifications.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `flavorsheet_no` | `text` | PRIMARY KEY | Unique identifier |
| `flavor_code` | `text` | NOT NULL | Flavor classification |
| `remarks` | `text` | NOT NULL | Processing notes |

#### Processing Records

All processing record tables follow a consistent pattern:

**Common Record Structure**:
- `item_code`, `created_ts`: Reference to parent item
- `received_ts`: Processing timestamp (auto-generated)
- `store_location`: Storage location (validated by triggers)
- `bag_weight`: Tare weight
- `gross_weight`: Total weight
- `barcode`: UUID identifier (auto-generated)
- `status`: Processing status (auto-set to 'ACCEPTED')
- `remaining`: Available quantity (calculated as gross - bag weight)

**Record Tables**:
- `tealine_record`: Tea processing records
- `blendsheet_record`: Blend processing records
- `flavorsheet_record`: Flavor processing records
- `herbline_record`: Herbal processing records (location restricted)
- `blendbalance_record`: Balance processing records

**Valid Status Values**: ACCEPTED, IN_PROCESS, PROCESSED, DISPATCHED

#### Batch Management

**`blendsheet_batch`**
Batch tracking with trader approval workflow.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY | Batch identifier |
| `created_ts` | `bigint` | PRIMARY KEY | Creation timestamp |
| `blendsheet_no` | `text` | NOT NULL | Formulation reference |
| `completed` | `boolean` | DEFAULT false | Completion status |
| `humidity` | `real` | NULLABLE | Quality measurement |
| `density` | `real` | NULLABLE | Quality measurement |
| `trader_status` | `text` | NULLABLE | Approval status |

**Valid Trader Status**: TRADER_PENDING, TRADER_REQUESTED, TRADER_ELEVATED, TRADER_ALLOWED, TRADER_BLOCKED

**`flavorsheet_batch`**
Flavor batch management (simplified workflow).

#### Allocation Tables

**Blendsheet Ingredient Allocation**:
- ✅ **`tealine_allocate_blendsheet`**: Tealine blendsheet ingredient allocation to batches
- ✅ **`blendbalance_allocate_blendsheet`**: Blendbalance blendsheet ingredient allocation to batches

**Shipment Allocation Support** (AWS-Manager Integration):
- ✅ **`tealine_allocate_shipment`**: Tea shipment allocations
- ✅ **`blendsheet_allocate_shipment`**: Blend shipment allocations

**Parcel Allocation Support** (AWS-Manager Integration):
- ✅ **`tealine_allocate_parcel`**: Tea parcel allocations
- ✅ **`blendsheet_allocate_parcel`**: Blend parcel allocations

**Allocation Table Structure**:
```sql
*_allocate_shipment:
- barcode (uuid): Record reference
- request_code (bigint): AWS-Manager request
- shipment_code (integer): AWS-Manager shipment
- amount (numeric): Allocated quantity
- remaining (numeric): Remaining after allocation

*_allocate_parcel:
- barcode (uuid): Record reference  
- request_code (bigint): AWS-Manager request
- shipment_code (integer): AWS-Manager shipment
- schedule_code (bigint): AWS-Manager schedule
- amount (numeric): Allocated quantity

*_allocate_blendsheet:
- barcode (uuid): Record reference
- item_code (text): Blendsheet batch identifier
- created_ts (bigint): Blendsheet batch timestamp
- amount (numeric): Allocated quantity for ingredient traceability
```

#### Mixture & Formulation Tables

**`blendsheet_mix_tealine`**
Tea components in blend formulations.

**`blendsheet_mix_blendbalance`**
Balance components in blend formulations.

**`flavorsheet_mix`**
Flavor mixture components.

#### Support Tables

**`store_location`**
Storage location management with herbline restrictions.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `location_name` | `text` | PRIMARY KEY | Location identifier |
| `herbline_section` | `boolean` | DEFAULT false | Herbline authorization flag |

**Business Rules**:
- `herbline_section = true`: Authorized for herbline storage only
- `herbline_section = false`: Standard storage for other materials
- Location segregation enforced by triggers

**`item`**
Base item tracking with creation timestamps.

### Database Functions

#### `scan_record(tname text, barcode uuid) RETURNS json`

Comprehensive record scanning with automatic joins and allocation calculation.

**Features**:
- Dynamic table validation for `{tname}_record` existence
- Dynamic allocation table discovery for `{tname}_allocate_*` tables
- Comprehensive allocation calculation across all allocation types
- Parent table joining (either `{tname}` or `{tname}_batch`)
- Status determination based on total allocation amounts
- Returns enriched JSON with complete record information

**Usage**: Powers `/app/scan` and `/app/shipment/record` endpoints

**Allocation Calculation**: Dynamically discovers all allocation tables and calculates remaining quantities from **shipment and blendsheet allocations** (excludes parcel allocations). This supports both traditional shipment workflows and ingredient traceability:

**Enhanced Allocation Flow**:
```
record → shipment_allocation → parcel_allocation (refinement)
      └→ blendsheet_allocation (parallel ingredient processing)
```

**Design Benefits**:
- **Ingredient Traceability**: Complete tracking from raw materials to finished blendsheet batches
- **Parallel Processing**: Materials can be allocated to both shipments and blendsheet processing
- **Accurate Accounting**: Parcel allocations excluded to prevent double-counting
- **Future-Proof**: Dynamic discovery automatically includes new allocation types

### Trigger Functions

**`record_status()`**
Auto-initialize record status and remaining quantities.

**`record_tealine()`**
Validate tealine bag count against expected limits.

**`record_blendbalance()`**
Validate blendbalance weight against available capacity.

**`record_location(herbline_flag)`**
Validate storage location restrictions:
- `herbline_flag = 'true'`: Requires `herbline_section = true`
- `herbline_flag = 'false'`: Requires `herbline_section = false`

**`record_shipment()`**
Initialize shipment allocation records with remaining quantities.

---

## Integration Patterns

### API-Based Integration Architecture

#### AWS-Manager → Central-Handler

**Proxy Pattern**:
```
Client → AWS API Gateway → Lambda (order-handler) → Central-Handler
                            ↓
               Path Rewrite: /central/* → /app/*
```

**Specialized Lambda Functions**:
- **trader-handler**: Blendsheet trader operations
- **barcode-handler**: Shipment/parcel/scan operations
- **central-handler**: General proxy operations

#### Central-Handler → AWS-Manager

**Direct API Integration**:
```
Central-Handler → AWS SigV4 Auth → AWS API Gateway → Lambda → AWS-Manager DB
```

**Key Endpoints**:
- `GET /order`: Retrieve available shipments
- `PUT /order/shipment`: Update shipment quantities
- `POST /order/shipment/event`: Create status events

### Cross-System Data Relationships

| Data Type | AWS-Manager | Central-Handler | Integration Purpose |
|-----------|-------------|-----------------|-------------------|
| **Request Code** | `order_request.request_code` | `*_allocate_shipment.request_code` | Order tracking |
| **Shipment Code** | `order_shipment.shipment_code` | `*_allocate_shipment.shipment_code` | Shipment linking |
| **Schedule Code** | `order_schedule.schedule_code` | `*_allocate_parcel.schedule_code` | Production scheduling |
| **Barcode UUID** | Referenced via API | `*_record.barcode` | Record identification |

### Status Synchronization

#### Order Workflow Integration

**AWS-Manager Order States**:
```
APPROVAL_REQUESTED → APPROVAL_ALLOWED → ORDER_REQUESTED → 
SHIPMENT_ACCEPTED → SHIPMENT_DISPATCHED
```

**Central-Handler Processing States**:
```
ACCEPTED → IN_PROCESS → PROCESSED → DISPATCHED
```

**Synchronization Points**:
- `ORDER_REQUESTED` enables allocation eligibility
- `SHIPMENT_ACCEPTED` enables processing operations
- `PROCESSED` status updates shipment quantities
- `DISPATCHED` triggers shipment completion events

---

## Data Flow & Workflows

### Order-to-Production Workflow

```
1. Order Planning (AWS-Manager)
   ├── order_plan creation
   ├── order_request generation
   └── order_shipment allocation
           ↓ (API Integration)
2. Processing Operations (Central-Handler)
   ├── Raw material records (*_record)
   ├── Batch creation and management
   ├── Quality control and trader approval
   └── Allocation to shipments (*_allocate_shipment)
           ↓ (API Callbacks)
3. Fulfillment Tracking (AWS-Manager)
   ├── Shipment quantity updates
   ├── Status event creation
   └── Production scheduling
```

### Material Processing Workflows

#### Tealine & Blendbalance Processing (with Blendsheet Ingredient Allocation)
```
Raw Materials → Processing Records → Batch Management → Quality Control → Allocation
     ↓                ↓                    ↓                ↓              ↓
  tealine/         *_record           *_batch         Trader          *_allocate_*
  blendbalance    creation           creation        Review          shipment/parcel
                                          ↓
                                 Blendsheet Ingredient Allocation
                                          ↓
                               *_allocate_blendsheet
```

#### Herbline Processing (Specialized Storage)
```
Herbal Materials → Processing Records → Quality Control → Status Tracking
       ↓                  ↓                  ↓               ↓
    herbline          herbline_record       Review         DISPATCHED
  (restricted)         creation           Process         (record only)
   storage
```

### Allocation Workflow

```
1. Processing Record Creation (Central-Handler)
   └── *_record with remaining quantity
           ↓
2. Parallel Allocation Paths (Central-Handler)
   ├── Shipment Allocation:
   │   ├── *_allocate_shipment creation
   │   └── Integration with AWS-Manager shipments
   └── Blendsheet Allocation:
       ├── *_allocate_blendsheet creation  
       └── Ingredient traceability for batches
           ↓
3. Quantity & Status Synchronization
   ├── Remaining quantity calculation (all allocations)
   ├── Status determination based on total allocations
   └── Cross-system quantity updates (shipment path)
```

---

## Performance & Optimization

### Database Performance Strategies

#### Indexing Strategy

**AWS-Manager Indexes**:
- Primary keys: Auto-indexed
- Foreign keys: `request_code` relationships
- Temporal queries: `event_ts`, `schedule_date`
- Status filtering: `status` columns

**Central-Handler Indexes**:
- Composite keys: `item_code`, `created_ts`
- Barcode lookups: `barcode` UUID columns
- Location queries: `store_location`
- Status filtering: `status`, `trader_status`

#### Query Optimization

**Common Query Patterns**:
- Record scanning with joins via `scan_record()` function
- Allocation queries with quantity calculations
- Status-based filtering for workflows
- Temporal range queries for reporting

**Optimization Techniques**:
- Prepared statements for repeated queries
- Partial indexes for status-based queries
- Materialized views for complex aggregations
- Connection pooling for high-volume operations

### API Performance

#### Caching Strategies

**Local Caching**:
- User authentication tokens
- Location and status mappings
- Frequently accessed lookup data

**Cross-System Caching**:
- Shipment availability data
- Allocation summaries
- Real-time status information

#### Batch Processing

**Bulk Operations**:
- Mass allocation updates
- Batch status synchronization
- Scheduled reconciliation jobs

**Asynchronous Processing**:
- Event queuing for status updates
- Deferred consistency operations
- Background reconciliation tasks

---

## Security & Compliance

### Authentication & Authorization

#### AWS-Manager Security

**AWS Cognito Integration**:
- User Pool authentication
- IAM role-based access control
- JWT token validation
- Group-based permissions

**Database Security**:
- AWS Secrets Manager for credentials
- VPC isolation for database access
- SSL/TLS encryption in transit
- Audit logging for sensitive operations

#### Central-Handler Security

**API Authentication**:
- AWS SigV4 request signing
- Temporary credential rotation
- Cross-system authentication

**Database Protection**:
- Connection encryption
- Access control at schema level
- Audit trails for data modifications

### Compliance Requirements

#### Storage Segregation

**Herbline Compliance**:
- Mandatory storage location restrictions
- Trigger-enforced validation
- Audit trail for location assignments
- Segregation from general tea storage

**Data Protection**:
- Encryption at rest and in transit
- Access logging and monitoring
- Data retention policies
- Backup encryption and security

### Security Best Practices

**API Security**:
- Rate limiting and throttling
- Input validation and sanitization
- Error message sanitization
- CORS configuration

**Database Security**:
- Principle of least privilege
- Regular security updates
- Monitoring for unusual access patterns
- Backup security and testing

---

## Monitoring & Maintenance

### System Monitoring

#### Database Monitoring

**Performance Metrics**:
- Query execution times
- Connection pool utilization
- Disk I/O and storage usage
- Lock contention and deadlocks

**Business Metrics**:
- Allocation consistency
- Status synchronization lag
- Processing workflow completion rates
- Error rates and types

#### API Integration Monitoring

**Cross-System Metrics**:
- API call latency and success rates
- Integration point availability
- Data synchronization delays
- Error propagation tracking

**Health Checks**:
- Database connectivity status
- API endpoint availability
- Integration workflow functionality
- Data consistency validation

### Maintenance Procedures

#### Database Maintenance

**Regular Tasks**:
- Index optimization and statistics updates
- Backup verification and testing
- Log rotation and cleanup
- Performance baseline monitoring

**Scheduled Operations**:
- Data reconciliation jobs
- Archived data cleanup
- Capacity planning reviews
- Security audit procedures

#### Integration Maintenance

**API Management**:
- Endpoint health monitoring
- Authentication token rotation
- Error rate threshold monitoring
- Performance baseline tracking

**Data Consistency**:
- Cross-system reconciliation
- Orphaned record cleanup
- Status consistency validation
- Allocation accuracy verification

---

## Troubleshooting Guide

### Common Issues

#### Database Connection Issues

**Symptoms**:
- Connection timeout errors
- Pool exhaustion warnings
- Authentication failures

**Diagnosis**:
```sql
-- Check active connections
SELECT datname, count(*) FROM pg_stat_activity GROUP BY datname;

-- Monitor long-running queries
SELECT query, state, query_start 
FROM pg_stat_activity 
WHERE query_start < now() - interval '5 minutes';
```

**Solutions**:
- Verify connection pool configuration
- Check network connectivity
- Validate credentials and permissions
- Review resource limits

#### Data Synchronization Issues

**Symptoms**:
- Allocation quantity mismatches
- Status synchronization delays
- Missing records in allocation tables

**Diagnosis**:
```sql
-- Check allocation consistency
SELECT a.request_code, a.shipment_code, 
       SUM(a.amount) as allocated,
       s.quantity as shipment_quantity
FROM *_allocate_shipment a
JOIN order_shipment s ON a.request_code = s.request_code 
                      AND a.shipment_code = s.shipment_code
GROUP BY a.request_code, a.shipment_code, s.quantity
HAVING SUM(a.amount) != s.quantity;
```

**Solutions**:
- Run reconciliation procedures
- Check API call error logs
- Verify trigger function execution
- Review cross-system identifier mapping

#### Performance Issues

**Symptoms**:
- Slow query execution
- High database CPU utilization
- API timeout errors

**Diagnosis**:
```sql
-- Identify slow queries
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Check index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

**Solutions**:
- Optimize slow queries
- Add appropriate indexes
- Review connection pool settings
- Implement query result caching

### Emergency Procedures

#### System Recovery

**Database Recovery**:
1. Assess scope of data loss or corruption
2. Stop application services
3. Restore from latest verified backup
4. Replay transaction logs if available
5. Verify data integrity
6. Resume services with monitoring

**Integration Recovery**:
1. Identify affected systems
2. Switch to manual/degraded mode
3. Implement temporary workarounds
4. Restore API connectivity
5. Run data reconciliation
6. Resume normal operations

#### Data Repair Procedures

**Allocation Inconsistencies**:
1. Identify discrepancies through comparison queries
2. Create backup of affected data
3. Implement corrective SQL scripts
4. Verify corrections through reconciliation
5. Update monitoring to prevent recurrence

**Status Synchronization Issues**:
1. Identify out-of-sync records
2. Determine correct status through business rules
3. Update statuses with proper audit trails
4. Trigger dependent workflow updates
5. Monitor for propagation of corrections

---

## Related Documentation

- [Database Overview](./Database-Overview.md)
- [Database Schema - AWS Manager](./Database-Schema-AWS-Manager.md)
- [Database Schema - Central Handler](./Database-Schema-Central-Handler.md)
- [Database Integration Patterns](./Database-Integration-Patterns.md)
- [API Documentation - AWS Manager](./API-Documentation-AWS-Manager.md)
- [API Documentation - Central Handler](./API-Documentation-Central-Handler.md)
- [Comprehensive API Documentation](./Comprehensive-API-Documentation.md)