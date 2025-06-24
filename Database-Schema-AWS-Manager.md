# AWS-Manager Database Schema Documentation

## Overview

The AWS-Manager database is a dedicated PostgreSQL instance named `tea_management` that manages order operations and user administration for the MTP Platform. It operates independently from the Central-Handler database and integrates through API calls.

## Database Information

- **Database Name**: `tea_management`
- **Database Engine**: PostgreSQL 14.17+
- **Character Encoding**: UTF-8
- **Locale**: en_US.UTF-8
- **Primary Schemas**: `admin`, `public`

## Schema Structure

### Admin Schema (`admin`)

The admin schema handles user management and integrates with AWS Cognito for authentication.

#### Tables

##### `admin.user`

**Purpose**: User account management with AWS Cognito integration

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `sub` | `uuid` | PRIMARY KEY, NOT NULL | AWS Cognito User Pool subject identifier |
| `username` | `text` | NOT NULL | User login name |
| `group` | `text` | NOT NULL | User role/permission group |

**Primary Key**: `sub`

**Integration**: 
- Synchronized with AWS Cognito User Pool
- `sub` field maps directly to Cognito user subject
- `group` field corresponds to Cognito user groups

---

### Public Schema (`public`)

The public schema handles all order management operations including planning, requests, shipments, and scheduling.

#### Core Tables

##### `order_plan`

**Purpose**: Production planning data for tea orders

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `order_code` | `text` | PRIMARY KEY, NOT NULL | Unique order identifier |
| `product_name` | `text` | NOT NULL | Product being ordered |
| `requirement` | `integer` | NOT NULL | Required quantity |
| `plan_start` | `date` | NOT NULL | Planned start date |
| `plan_end` | `date` | NOT NULL | Planned completion date |

**Primary Key**: `order_code`

##### `order_plan_raw`

**Purpose**: Raw import data for order planning before processing

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `lot_code` | `text` | NOT NULL | Lot identifier |
| `order_code` | `text` | NOT NULL | Order identifier |
| `item_code` | `text` | NOT NULL | Item identifier |
| `production_date` | `text` | NOT NULL | Production date (string format) |
| `stock` | `text` | NOT NULL | Stock information |
| `requirement` | `text` | NOT NULL | Requirement (string format) |
| `location` | `text` | NOT NULL | Storage location |
| `product_code` | `text` | NOT NULL | Product code |
| `product_name` | `text` | NOT NULL | Product name |
| `item_name` | `text` | NOT NULL | Item name |
| `sub_location` | `text` | NOT NULL | Sub-location details |
| `plan_start` | `text` | NOT NULL | Start date (string format) |
| `plan_end` | `text` | NOT NULL | End date (string format) |

**Usage**: Staging table for bulk imports that get processed into `order_plan`

##### `order_request`

**Purpose**: Customer order requests with auto-generated identifiers

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `request_code` | `bigint` | PRIMARY KEY, AUTO INCREMENT | Unique request identifier |
| `order_code` | `text` | NOT NULL | Reference to order plan |
| `product_name` | `text` | NOT NULL | Product being requested |
| `requirement` | `integer` | NOT NULL | Requested quantity |
| `comments` | `text` | NULLABLE | Additional comments |

**Primary Key**: `request_code`
**Sequence**: `order_request_request_code_seq` (starts at 1, increment 1)

##### `order_shipment`

**Purpose**: Shipment allocations for order requests

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `request_code` | `bigint` | PRIMARY KEY, NOT NULL | Reference to order request |
| `shipment_code` | `integer` | PRIMARY KEY, DEFAULT 1, NOT NULL | Shipment number for request |
| `quantity` | `integer` | NULLABLE | Allocated quantity |

**Primary Key**: `request_code`, `shipment_code`
**Foreign Key**: `request_code` → `order_request.request_code`

##### `order_shipment_event`

**Purpose**: Event tracking for shipment lifecycle with timestamps

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `request_code` | `bigint` | PRIMARY KEY, NOT NULL | Reference to order request |
| `shipment_code` | `integer` | PRIMARY KEY, NOT NULL | Shipment identifier |
| `status` | `text` | PRIMARY KEY, NOT NULL | Event status |
| `event_ts` | `bigint` | NOT NULL, DEFAULT epoch_ms | Event timestamp (Unix milliseconds) |
| `shipment_vehicle` | `text` | NULLABLE | Vehicle information |
| `order_remarks` | `text` | NULLABLE | Order-related remarks |
| `shipment_remarks` | `text` | NULLABLE | Shipment-related remarks |

**Primary Key**: `request_code`, `shipment_code`, `status`
**Foreign Key**: `request_code` → `order_request.request_code`
**Default Timestamp**: Current epoch time in milliseconds

**Valid Status Values**:
- `APPROVAL_REQUESTED`
- `ORDER_REQUESTED` 
- `APPROVAL_ALLOWED`
- `SHIPMENT_ACCEPTED`
- `SHIPMENT_DISPATCHED`
- `ORDER_NOT_READY`

##### `order_schedule`

**Purpose**: Production scheduling with shift and section management

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `schedule_code` | `bigint` | PRIMARY KEY, NOT NULL | Unique schedule identifier |
| `schedule_date` | `date` | NULLABLE | Scheduled production date |
| `order_code` | `text` | NULLABLE | Reference to order |
| `shift` | `text` | NULLABLE | Production shift |
| `section` | `text` | NULLABLE | Production section |
| `quantity` | `integer` | NULLABLE | Scheduled quantity |
| `completed` | `boolean` | DEFAULT false | Completion status |

**Primary Key**: `schedule_code`

**Valid Shift Values**:
- `NIGHT`
- `DAY`
- `NIGHT & DAY`

## Relationships & Foreign Keys

### Foreign Key Relationships

```
order_request (request_code) ←─┐
                               ├─ order_shipment (request_code)
                               └─ order_shipment_event (request_code)
```

### Data Flow Relationships

```
order_plan_raw → [Processing] → order_plan
order_plan → order_request → order_shipment → order_shipment_event
order_request → order_schedule
```

## Sequences

### `order_request_request_code_seq`
- **Type**: Auto-incrementing sequence
- **Start Value**: 1
- **Increment**: 1
- **Usage**: Generates unique `request_code` values for `order_request` table

## Data Types & Patterns

### Timestamp Handling
- **Storage**: Unix timestamps in milliseconds (`bigint`)
- **Default**: `date_part('epoch'::text, now()) * (1000)::double precision`
- **Usage**: Event tracking, audit trails

### Identifier Patterns
- **Order Codes**: Text-based identifiers
- **Request Codes**: Auto-incrementing bigint
- **Shipment Codes**: Integer, defaults to 1 per request
- **User IDs**: UUID format for Cognito integration

### Quantity Management
- **Storage**: Integer values (quantities in base units)
- **Validation**: Business logic enforced at application level
- **Integration**: Converted to/from different units for API responses

## Integration Points

### AWS Cognito Integration
- `admin.user` table synchronized with Cognito User Pool
- User creation/deletion triggers Cognito operations
- Group management through Cognito groups

### Central-Handler Integration
- Order requests and shipments provide data for allocation
- Shipment events receive updates from processing operations
- API-based synchronization maintains consistency

### API Gateway Integration
- All tables support RESTful operations through Lambda functions
- CORS-enabled for cross-origin requests
- Error handling propagates database constraints

## Best Practices

### Data Integrity
- Use transactions for multi-table operations
- Validate foreign key relationships
- Implement proper error handling for constraint violations

### Performance Optimization
- Index frequently queried columns
- Use prepared statements for repeated queries
- Monitor sequence performance for high-volume inserts

### Security Considerations
- Secure database credentials through AWS Secrets Manager
- Use IAM roles for database access
- Implement proper access controls at schema level

---

## Related Documentation

- [Database Overview](./Database-Overview.md)
- [Database Schema - Central Handler](./Database-Schema-Central-Handler.md)
- [Database Integration Patterns](./Database-Integration-Patterns.md)
- [API Documentation - AWS Manager](./API-Documentation-AWS-Manager.md)