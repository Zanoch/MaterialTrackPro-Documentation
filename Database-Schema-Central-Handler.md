# Central-Handler Database Schema Documentation

## Overview

The Central-Handler database is a dedicated PostgreSQL instance named `tea_management` that manages tea processing operations, inventory tracking, and allocation management for the MTP Platform. It operates independently from the AWS-Manager database and integrates through API calls.

## Database Information

- **Database Name**: `tea_management`
- **Database Engine**: PostgreSQL 14.12+
- **Character Encoding**: UTF-8
- **Locale**: en_US.UTF-8
- **Primary Schema**: `public`

## Schema Structure

### Public Schema (`public`)

The public schema handles all tea processing operations, from raw materials to finished products, including allocation tracking for shipments.

## Core Processing Tables

### Base Material Tables

#### `tealine`

**Purpose**: Tea leaf inventory and incoming raw materials

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Unique item identifier |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Creation timestamp (Unix ms) |
| `invoice_no` | `text` | NULLABLE | Invoice number |
| `grade` | `text` | NULLABLE | Tea grade classification |
| `no_of_bags` | `integer` | NOT NULL | Expected number of bags |
| `weight_per_bag` | `numeric(5,2)` | NOT NULL | Expected weight per bag |
| `broker` | `text` | NULLABLE | Broker information |
| `garden` | `text` | NULLABLE | Tea garden source |
| `purchase_order` | `text` | NULLABLE | Purchase order reference |

**Primary Key**: `item_code`, `created_ts`
**Foreign Key**: `item_code`, `created_ts` → `item(item_code, created_ts)`

#### `blendsheet`

**Purpose**: Tea blending formulations and specifications

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `blendsheet_no` | `text` | PRIMARY KEY, NOT NULL | Unique blendsheet identifier |
| `standard` | `text` | NOT NULL | Quality standard |
| `grade` | `text` | NULLABLE | Final blend grade |
| `remarks` | `text` | NOT NULL | Additional notes |
| `no_of_batches` | `integer` | DEFAULT 1, NOT NULL | Expected batch count |
| `total` | `numeric(8,2)` | NOT NULL | Total blend quantity |
| `blend_code` | `text` | NULLABLE | Blend classification code |

**Primary Key**: `blendsheet_no`

#### `flavorsheet`

**Purpose**: Flavor processing specifications

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `flavorsheet_no` | `text` | PRIMARY KEY, NOT NULL | Unique flavorsheet identifier |
| `flavor_code` | `text` | NOT NULL | Flavor classification |
| `remarks` | `text` | NOT NULL | Processing notes |

**Primary Key**: `flavorsheet_no`

#### `herbline`

**Purpose**: Herbal ingredient management with restricted storage

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Unique item identifier |
| `item_name` | `text` | NOT NULL | Herbal ingredient name |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Creation timestamp (Unix ms) |
| `purchase_order` | `text` | NULLABLE | Purchase order reference |
| `weight` | `numeric` | NULLABLE | Expected weight |

**Primary Key**: `item_code`, `created_ts`
**Foreign Key**: `item_code`, `created_ts` → `item(item_code, created_ts)`

#### `blendbalance`

**Purpose**: Blend inventory balance management

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Unique item identifier |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Creation timestamp (Unix ms) |
| `blend_code` | `text` | NOT NULL | Blend classification |
| `transfer_id` | `text` | NOT NULL | Transfer reference |
| `weight` | `numeric` | NOT NULL | Expected total weight |

**Primary Key**: `created_ts`, `item_code`
**Foreign Key**: `item_code`, `created_ts` → `item(item_code, created_ts)`

## Record Tables (Processing Operations)

### `tealine_record`

**Purpose**: Individual tea bag processing records

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Reference to tealine |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Reference to tealine |
| `received_ts` | `bigint` | NOT NULL, DEFAULT epoch_ms | Processing timestamp |
| `store_location` | `text` | NOT NULL | Storage location |
| `bag_weight` | `numeric(5,2)` | NOT NULL | Bag weight (tare) |
| `gross_weight` | `numeric(5,2)` | NOT NULL | Total weight |
| `barcode` | `uuid` | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique record identifier |
| `status` | `text` | NOT NULL | Processing status |
| `remaining` | `numeric(5,2)` | NOT NULL | Remaining quantity |

**Primary Key**: `barcode`
**Foreign Key**: `item_code`, `created_ts` → `tealine(item_code, created_ts)`
**Valid Status**: ACCEPTED, IN_PROCESS, PROCESSED, DISPATCHED

### `blendsheet_record`

**Purpose**: Blend processing records

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Reference to batch |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Reference to batch |
| `received_ts` | `bigint` | NOT NULL, DEFAULT epoch_ms | Processing timestamp |
| `store_location` | `text` | NOT NULL | Storage location |
| `bag_weight` | `numeric(5,2)` | NOT NULL | Bag weight (tare) |
| `gross_weight` | `numeric(5,2)` | NOT NULL | Total weight |
| `barcode` | `uuid` | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique record identifier |
| `status` | `text` | NOT NULL | Processing status |
| `remaining` | `numeric(5,2)` | NOT NULL | Remaining quantity |

**Primary Key**: `barcode`
**Foreign Key**: `item_code`, `created_ts` → `blendsheet_batch(item_code, created_ts)`
**Valid Status**: ACCEPTED, IN_PROCESS, PROCESSED, DISPATCHED

### `flavorsheet_record`

**Purpose**: Flavor processing records

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Reference to batch |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Reference to batch |
| `store_location` | `text` | NOT NULL | Storage location |
| `gross_weight` | `numeric(5,2)` | NOT NULL | Total weight |
| `barcode` | `uuid` | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique record identifier |
| `status` | `text` | DEFAULT 'ACCEPTED', NOT NULL | Processing status |
| `bag_weight` | `numeric(5,2)` | NOT NULL | Bag weight (tare) |
| `received_ts` | `bigint` | NOT NULL, DEFAULT epoch_ms | Processing timestamp |
| `remaining` | `numeric(5,2)` | NOT NULL | Remaining quantity |

**Primary Key**: `barcode`

### `herbline_record`

**Purpose**: Herbal processing records with location restrictions

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Reference to herbline |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Reference to herbline |
| `received_ts` | `bigint` | NOT NULL, DEFAULT epoch_ms | Processing timestamp |
| `store_location` | `text` | NOT NULL | Restricted storage location |
| `bag_weight` | `numeric(5,2)` | NOT NULL | Bag weight (tare) |
| `gross_weight` | `numeric(5,2)` | NOT NULL | Total weight |
| `reference` | `text` | NOT NULL | Processing reference |
| `barcode` | `uuid` | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique record identifier |
| `status` | `text` | DEFAULT 'ACCEPTED', NOT NULL | Processing status |
| `remaining` | `numeric(5,2)` | NOT NULL | Remaining quantity |

**Primary Key**: `barcode`
**Foreign Key**: `item_code`, `created_ts` → `herbline(item_code, created_ts)`
**Valid Status**: ACCEPTED, IN_PROCESS, PROCESSED, DISPATCHED
**Location Restriction**: Must use `herbline_section = true` locations

### `blendbalance_record`

**Purpose**: Balance processing records

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Reference to blendbalance |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Reference to blendbalance |
| `received_ts` | `bigint` | NOT NULL, DEFAULT epoch_ms | Processing timestamp |
| `store_location` | `text` | NOT NULL | Storage location |
| `bag_weight` | `numeric(5,2)` | NOT NULL | Bag weight (tare) |
| `gross_weight` | `numeric(5,2)` | NOT NULL | Total weight |
| `barcode` | `uuid` | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique record identifier |
| `status` | `text` | NOT NULL | Processing status |
| `remaining` | `numeric(5,2)` | NOT NULL | Remaining quantity |

**Primary Key**: `barcode`
**Foreign Key**: `item_code`, `created_ts` → `blendbalance(item_code, created_ts)`
**Valid Status**: ACCEPTED, IN_PROCESS, PROCESSED, DISPATCHED

## Batch Management Tables

### `blendsheet_batch`

**Purpose**: Batch tracking with trader approval workflow

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Unique batch identifier |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Batch creation timestamp |
| `blendsheet_no` | `text` | NOT NULL | Reference to blendsheet |
| `completed` | `boolean` | DEFAULT false, NOT NULL | Completion status |
| `humidity` | `real` | NULLABLE | Humidity measurement |
| `density` | `real` | NULLABLE | Density measurement |
| `trader_status` | `text` | NULLABLE | Trader approval status |

**Primary Key**: `item_code`, `created_ts`
**Foreign Keys**: 
- `blendsheet_no` → `blendsheet(blendsheet_no)`
- `item_code`, `created_ts` → `item(item_code, created_ts)`

**Valid Trader Status**: TRADER_PENDING, TRADER_REQUESTED, TRADER_ELEVATED, TRADER_ALLOWED, TRADER_BLOCKED

### `flavorsheet_batch`

**Purpose**: Flavor batch management

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Unique batch identifier |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Batch creation timestamp |
| `flavorsheet_no` | `text` | NOT NULL | Reference to flavorsheet |
| `completed` | `boolean` | DEFAULT false, NOT NULL | Completion status |

**Primary Key**: `item_code`, `created_ts`

## Allocation Tables

### Blendsheet Ingredient Allocation (New)

#### `tealine_allocate_blendsheet`

**Purpose**: Allocation of tealine records to blendsheet batches for ingredient traceability

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `barcode` | `uuid` | PRIMARY KEY, NOT NULL | Reference to tealine record |
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Reference to blendsheet batch |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Reference to blendsheet batch |
| `amount` | `numeric(5,2)` | NOT NULL | Allocated amount |

**Primary Key**: `barcode`, `item_code`, `created_ts`
**Foreign Keys**: 
- `barcode` → `tealine_record(barcode)`
- `item_code`, `created_ts` → `blendsheet_batch(item_code, created_ts)`

#### `blendbalance_allocate_blendsheet`

**Purpose**: Allocation of blendbalance records to blendsheet batches for ingredient traceability

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `barcode` | `uuid` | PRIMARY KEY, NOT NULL | Reference to blendbalance record |
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Reference to blendsheet batch |
| `created_ts` | `bigint` | PRIMARY KEY, NOT NULL | Reference to blendsheet batch |
| `amount` | `numeric(5,2)` | NOT NULL | Allocated amount |

**Primary Key**: `barcode`, `item_code`, `created_ts`
**Foreign Keys**: 
- `barcode` → `blendbalance_record(barcode)`
- `item_code`, `created_ts` → `blendsheet_batch(item_code, created_ts)`

### Shipment Integration

### `tealine_allocate_shipment`

**Purpose**: Tea allocation to AWS-Manager shipments

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `barcode` | `uuid` | PRIMARY KEY, NOT NULL | Reference to tealine record |
| `request_code` | `bigint` | PRIMARY KEY, NOT NULL | AWS-Manager request code |
| `shipment_code` | `integer` | PRIMARY KEY, NOT NULL | AWS-Manager shipment code |
| `amount` | `numeric(5,2)` | NOT NULL | Allocated amount |
| `remaining` | `numeric(5,2)` | NOT NULL | Remaining after allocation |

**Primary Key**: `barcode`, `request_code`, `shipment_code`
**Foreign Key**: `barcode` → `tealine_record(barcode)`
**Constraint**: `remaining <= amount AND remaining >= 0`

### `blendsheet_allocate_shipment`

**Purpose**: Blend allocation to AWS-Manager shipments

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `barcode` | `uuid` | PRIMARY KEY, NOT NULL | Reference to blendsheet record |
| `request_code` | `bigint` | PRIMARY KEY, NOT NULL | AWS-Manager request code |
| `shipment_code` | `integer` | PRIMARY KEY, NOT NULL | AWS-Manager shipment code |
| `amount` | `numeric(5,2)` | NOT NULL | Allocated amount |
| `remaining` | `numeric(5,2)` | NOT NULL | Remaining after allocation |

**Primary Key**: `shipment_code`, `request_code`, `barcode`
**Foreign Key**: `barcode` → `blendsheet_record(barcode)`
**Constraint**: `remaining <= amount AND remaining >= 0`

### `tealine_allocate_parcel`

**Purpose**: Tea allocation to production parcels

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `barcode` | `uuid` | PRIMARY KEY, NOT NULL | Reference to tealine record |
| `request_code` | `bigint` | PRIMARY KEY, NOT NULL | AWS-Manager request code |
| `shipment_code` | `integer` | PRIMARY KEY, NOT NULL | AWS-Manager shipment code |
| `schedule_code` | `bigint` | PRIMARY KEY, NOT NULL | AWS-Manager schedule code |
| `amount` | `numeric(5,2)` | NOT NULL | Allocated amount |

**Primary Key**: `barcode`, `request_code`, `shipment_code`, `schedule_code`
**Foreign Key**: `shipment_code`, `request_code`, `barcode` → `tealine_allocate_shipment(...)`

### `blendsheet_allocate_parcel`

**Purpose**: Blend allocation to production parcels

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `barcode` | `uuid` | PRIMARY KEY, NOT NULL | Reference to blendsheet record |
| `request_code` | `bigint` | PRIMARY KEY, NOT NULL | AWS-Manager request code |
| `shipment_code` | `integer` | PRIMARY KEY, NOT NULL | AWS-Manager shipment code |
| `schedule_code` | `bigint` | PRIMARY KEY, NOT NULL | AWS-Manager schedule code |
| `amount` | `numeric(5,2)` | NOT NULL | Allocated amount |

**Primary Key**: `barcode`, `request_code`, `shipment_code`, `schedule_code`
**Foreign Key**: `shipment_code`, `request_code`, `barcode` → `blendsheet_allocate_shipment(...)`

## Mixture & Formulation Tables

### `blendsheet_mix_tealine`

**Purpose**: Tea components in blend formulations

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `blendsheet_no` | `text` | PRIMARY KEY, NOT NULL | Reference to blendsheet |
| `mixture_code` | `text` | PRIMARY KEY, NOT NULL | Tea mixture identifier |
| `no_of_bags` | `integer` | NOT NULL | Required number of bags |

**Primary Key**: `blendsheet_no`, `mixture_code`
**Foreign Key**: `blendsheet_no` → `blendsheet(blendsheet_no)`

### `blendsheet_mix_blendbalance`

**Purpose**: Balance components in blend formulations

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `blendsheet_no` | `text` | PRIMARY KEY, NOT NULL | Reference to blendsheet |
| `mixture_code` | `text` | PRIMARY KEY, NOT NULL | Balance mixture identifier |
| `weight` | `numeric(1000,2)` | NOT NULL | Required weight |

**Primary Key**: `blendsheet_no`, `mixture_code`
**Foreign Key**: `blendsheet_no` → `blendsheet(blendsheet_no)`

### `flavorsheet_mix`

**Purpose**: Flavor mixture components

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `flavorsheet_no` | `text` | PRIMARY KEY, NOT NULL | Reference to flavorsheet |
| `mixture_code` | `text` | PRIMARY KEY, NOT NULL | Mixture identifier |
| `weight` | `numeric(1000,2)` | NOT NULL | Required weight |

**Primary Key**: `flavorsheet_no`, `mixture_code`
**Foreign Key**: `flavorsheet_no` → `flavorsheet(flavorsheet_no)`

## Support Tables

### `store_location`

**Purpose**: Storage location management with herbline restrictions

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `location_name` | `text` | PRIMARY KEY, NOT NULL | Unique location identifier |
| `herbline_section` | `boolean` | DEFAULT false, NOT NULL | Herbline storage authorization |

**Primary Key**: `location_name`

**Business Rules**:
- `herbline_section = true`: Authorized for herbline storage only
- `herbline_section = false`: Standard storage for other materials
- Location segregation enforced by triggers

### `item`

**Purpose**: Base item tracking with timestamps

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `item_code` | `text` | PRIMARY KEY, NOT NULL | Unique item identifier |
| `created_ts` | `bigint` | PRIMARY KEY, DEFAULT epoch_ms, NOT NULL | Creation timestamp |

**Primary Key**: `item_code`, `created_ts`

## Database Functions

### `scan_record(tname text, barcode uuid) RETURNS json`

**Purpose**: Comprehensive record scanning with automatic joins and allocation calculation

**Features**:
- Dynamic table validation for `{tname}_record` existence
- Dynamic allocation table discovery for `{tname}_allocate_*` tables
- Comprehensive allocation calculation across all allocation types
- Parent table joining (either `{tname}` or `{tname}_batch`)
- Status determination based on total allocation amounts
- Returns enriched JSON with complete record information

**Usage**: Powers `/app/scan` and `/app/shipment/record` endpoints

**Logic**:
1. Validates record table existence
2. Dynamically discovers all allocation tables for the material type
3. Excludes parcel allocations to prevent double-counting
4. Calculates total allocations across all allocation types
5. Determines parent table (batch vs base table)
6. Performs dynamic joins and returns comprehensive data

**Allocation Calculation**:
```
remaining = original_quantity - (shipment_allocations + blendsheet_allocations + ...other_allocations)
```

**Hierarchical Allocation Flow**:
```
record → allocate_shipment → allocate_parcel (refinement)
      └→ allocate_blendsheet (parallel processing)
```

- **Shipment Allocations**: Records allocated to shipments (reduces original inventory)
- **Blendsheet Allocations**: Records allocated to ingredient processing (reduces original inventory)
- **Parcel Allocations**: Refinements of shipment allocations (excluded from calculation)
- **Design**: Prevents double-counting while supporting parallel allocation workflows

**Dynamic Table Discovery**:
- Automatically finds all `{tname}_allocate_*` tables except `{tname}_allocate_parcel`
- Future-proof: automatically includes new allocation types without code changes
- Unified calculation: single query handles all allocation types efficiently

**Why Exclude Parcels?**:
- Parcel allocations represent subdivisions of shipment allocations, not additional consumption
- Including parcels would double-count allocated materials
- Maintains accurate inventory tracking and business logic alignment

## Trigger Functions

### `record_status() RETURNS trigger`

**Purpose**: Auto-initialize record status and remaining quantities

**Behavior**:
- Sets `status = 'ACCEPTED'`
- Calculates `remaining = gross_weight - bag_weight`
- Applied to all record tables on INSERT

### `record_tealine() RETURNS trigger`

**Purpose**: Validate tealine bag count limits

**Behavior**:
- Checks current bag count against `tealine.no_of_bags`
- Raises error if limit exceeded
- Applied to `tealine_record` on INSERT

### `record_blendbalance() RETURNS trigger`

**Purpose**: Validate blendbalance weight limits

**Behavior**:
- Checks cumulative weight against `blendbalance.weight`
- Calculates available weight from existing records
- Raises error if weight limit exceeded
- Applied to `blendbalance_record` on INSERT

### `record_location(herbline_flag) RETURNS trigger`

**Purpose**: Validate storage location restrictions

**Parameters**:
- `herbline_flag`: 'true' for herbline, 'false' for others

**Behavior**:
- For herbline: Requires `herbline_section = true`
- For others: Requires `herbline_section = false`
- Enforces storage segregation compliance
- Applied to all record tables on INSERT/UPDATE

### `record_shipment() RETURNS trigger`

**Purpose**: Initialize shipment allocation records

**Behavior**:
- Sets `remaining = amount` for new allocations
- Applied to allocation tables on INSERT

## Data Types & Patterns

### Timestamp Management
- **Storage**: Unix timestamps in milliseconds (`bigint`)
- **Default**: `date_part('epoch'::text, now()) * (1000)::double precision`
- **Usage**: Creation tracking, processing timestamps

### Barcode System
- **Type**: UUID v4 automatically generated
- **Purpose**: Unique record identification across systems
- **Usage**: Cross-system integration, allocation tracking

### Weight & Quantity Precision
- **Weights**: `numeric(5,2)` for processing weights
- **Large Quantities**: `numeric(1000,2)` for bulk operations
- **Business Logic**: Net weight = gross_weight - bag_weight

### Status Enumerations

**Record Status**:
- `ACCEPTED`: Initial state after creation
- `IN_PROCESS`: Currently being processed
- `PROCESSED`: Processing completed, no remaining quantity
- `DISPATCHED`: Final state, shipped/allocated

**Trader Status**:
- `TRADER_PENDING`: Awaiting trader review
- `TRADER_REQUESTED`: Trader review requested
- `TRADER_ELEVATED`: Escalated to higher authority
- `TRADER_ALLOWED`: Approved by trader
- `TRADER_BLOCKED`: Rejected by trader

## Allocation Support Matrix

| Material Type | Blendsheet Allocation | Shipment Allocation | Parcel Allocation | Notes |
|---------------|---------------------|-------------------|------------------|-------|
| **Tealine** | ✅ | ✅ | ✅ | Full allocation support with ingredient traceability |
| **Blendsheet** | ❌ | ✅ | ✅ | Shipment allocation support only |
| **Flavorsheet** | ❌ | ❌ | ❌ | Record tracking only |
| **Herbline** | ❌ | ❌ | ❌ | Record tracking only |
| **Blendbalance** | ✅ | ❌ | ❌ | Blendsheet allocation support only |

## Integration Points

### AWS-Manager Integration
- Allocation tables bridge to AWS-Manager shipments
- Request/shipment codes provide foreign key relationships
- Status synchronization through API calls

### Storage Compliance
- Herbline location restrictions enforced at database level
- Trigger-based validation prevents compliance violations
- Location segregation supports regulatory requirements

## Best Practices

### Performance Optimization
- Index frequently queried columns (item_code, created_ts, barcode)
- Use prepared statements for complex joins
- Monitor trigger performance on high-volume operations

### Data Integrity
- Rely on triggers for business rule enforcement
- Use transactions for multi-table operations
- Validate allocation constraints through database functions

### Monitoring & Maintenance
- Monitor allocation table growth and performance
- Track trigger execution times
- Audit storage location compliance

---

## Related Documentation

- [Database Overview](./Database-Overview.md)
- [Database Schema - AWS Manager](./Database-Schema-AWS-Manager.md)
- [Database Integration Patterns](./Database-Integration-Patterns.md)
- [API Documentation - Central Handler](./API-Documentation-Central-Handler.md)