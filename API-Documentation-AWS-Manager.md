# AWS-Manager API Documentation

## Overview

The AWS-Manager is a serverless AWS CDK application that provides API Gateway endpoints through Lambda functions for tea order management, user administration, and integration with the Central-Handler system. It uses PostgreSQL database and AWS Cognito for user management.

## Architecture

The system consists of several Lambda functions:
- **order-handler**: Main API orchestrator handling order management
- **central-handler**: Proxy to Central-Handler system
- **trader-handler**: Trader-specific operations (proxies to Central-Handler)
- **barcode-handler**: Shipment and parcel operations (proxies to Central-Handler)
- **cognito-handler**: User authentication and management

## Base URL Structure

All endpoints are served through AWS API Gateway with the following route structure:
- `/order/*` - Order management operations
- `/central/*` - Direct proxy to Central-Handler (path rewritten to `/app/*`)
- `/trader/*` - Trader operations
- `/admin/*` - Administrative operations

## Authentication & Authorization

### Authentication System
- **AWS Cognito User Pool**: User authentication and group membership management
- **AWS Cognito Identity Pool**: Maps user groups to IAM roles
- **JWT Tokens**: Contain group information for frontend role routing

### Role-Based Access Control (RBAC)
The system implements a two-tier role hierarchy with 10 specific user groups:

**Admin Level Roles (3):**
- `MASTER-ADMIN`: User management system access
- `GRANDPASS-ADMIN`: Production plan management
- `SEEDUWA-ADMIN`: Shipment documentation + Central-Handler historical data access

**User Level Roles (7):**
- `GRANDPASS-OFFICER`: Order requests + production scheduling + AWS-Manager historical data
- `GRANDPASS-SUPERVISOR`: Special request approvals
- `SEEDUWA-OFFICER`: Order status updates + trader request creation + AWS-Manager historical data
- `GRANDPASS-GATEPASS`: Shipment receiving + verification + AWS-Manager historical data
- `GRANDPASS-DEV-MACHINE`: Tea box filling + production record management
- `TRADER-OFFICER`: Blendsheet batch processing + trader request management
- `TRADER-SUPERVISOR`: Elevated trader request approvals

### Access Control Implementation
- **Admin Roles**: Full access to `/admin/*` endpoints + all operational endpoints
- **User Roles**: IAM policy denies access to `/admin/*` endpoints
- **Endpoint Permissions**: Role-specific access to operational endpoints
- **Historical Data Access**: Select roles can access AWS-Manager or Central-Handler historical data

For complete role documentation, see [Role-Based Access Control Documentation](./Role-Based-Access-Control.md).

### CORS Configuration
- CORS enabled for cross-origin requests from Admin-Panel frontend

## Error Handling

Standard HTTP status codes:
- 200: Success
- 404: Not Found
- 500: Internal Server Error

---

## Order Management Routes (`/order`)

### Order Planning

#### GET `/order/plan`
Retrieves order plans with allocation and request information.

**Query Parameters:**
- `current_date`: Date in YYYY-MM-DD format (TEMPORARILY IGNORED - returns all plans)

**Response:**
```json
[
  {
    "order_code": "string",
    "product_name": "string",
    "requirement": "number",
    "plan_start": "date",
    "plan_end": "date",
    "allowed": "number",
    "requests": [
      {
        "request_code": "bigint",
        "shipment_code": "integer",
        "quantity": "number",
        "status": "string"
      }
    ]
  }
]
```

#### POST `/order/plan`
Creates new order plans from raw planning data.

**Request Body:**
```json
[
  {
    "lot_code": "string",
    "order_code": "string",
    "item_code": "string",
    "production_date": "string",
    "stock": "string",
    "requirement": "string",
    "location": "string",
    "product_code": "string",
    "product_name": "string",
    "item_name": "string",
    "sub_location": "string",
    "plan_start": "string",
    "plan_end": "string"
  }
]
```

**Response:**
```json
[
  {
    "order_code": "string",
    "product_name": "string",
    "requirement": "number",
    "plan_start": "date",
    "plan_end": "date"
  }
]
```

#### DELETE `/order/plan`
Deletes an order plan if no requests exist for it.

**Query Parameters:**
- `order_code`: Order code to delete

**Response:**
```json
{
  "success": "boolean"
}
```

### Order Requests and Shipments

#### GET `/order`
Retrieves order requests with shipment events and filtering options.

**Query Parameters:**
- `status`: Filter by shipment status (optional)
- `timestamp`: Filter by event timestamp (optional)
- `shipment_vehicle`: Filter by vehicle (optional)

**Response:**
```json
[
  {
    "request_code": "bigint",
    "order_code": "string",
    "product_name": "string",
    "requirement": "number",
    "comments": "string",
    "shipments": [
      {
        "shipment_code": "integer",
        "quantity": "number",
        "events": [
          {
            "status": "string",
            "timestamp": "bigint",
            "shipment_vehicle": "string (optional)",
            "shipment_remarks": "string (optional)",
            "order_remarks": "string (optional)"
          }
        ]
      }
    ]
  }
]
```

#### POST `/order`
Creates new order requests with automatic shipment generation.

**Request Body:**
```json
[
  {
    "order_code": "string",
    "product_name": "string",
    "requirement": "number",
    "comments": "string (optional)",
    "status": "string"
  }
]
```

**Valid status values:** `APPROVAL_REQUESTED`, `ORDER_REQUESTED`

**Response:**
```json
[
  {
    "request_code": "bigint",
    "order_code": "string",
    "product_name": "string",
    "requirement": "number",
    "comments": "string",
    "shipments": [
      {
        "shipment_code": "integer",
        "quantity": "number",
        "events": [
          {
            "status": "string",
            "timestamp": "bigint"
          }
        ]
      }
    ]
  }
]
```

### Shipment Management

#### PUT `/order/shipment`
Updates shipment quantities and creates new shipments for remaining quantities.

**Request Body:**
```json
[
  {
    "request_code": "bigint",
    "shipment_code": "integer",
    "quantity": "number"
  }
]
```

**Response:**
```json
[
  {
    "request_code": "bigint",
    "shipments": [
      {
        "shipment_code": "integer",
        "quantity": "number",
        "events": [
          {
            "status": "string",
            "timestamp": "bigint"
          }
        ]
      }
    ]
  }
]
```

### Shipment Events

#### POST `/order/shipment/event`
Creates shipment status events with automatic workflow handling.

**Request Body:**
```json
[
  {
    "request_code": "bigint",
    "shipment_code": "integer",
    "status": "string",
    "shipment_vehicle": "string (required for SHIPMENT_DISPATCHED)",
    "shipment_remarks": "string (required for SHIPMENT_ACCEPTED)",
    "order_remarks": "string (required for ORDER_NOT_READY)"
  }
]
```

**Valid status values:**
- `APPROVAL_ALLOWED` (auto-creates ORDER_REQUESTED event)
- `SHIPMENT_ACCEPTED`
- `SHIPMENT_DISPATCHED`
- `ORDER_NOT_READY`

**Response:**
```json
[
  {
    "request_code": "bigint",
    "shipments": [
      {
        "shipment_code": "integer",
        "events": [
          {
            "status": "string",
            "timestamp": "bigint",
            "shipment_vehicle": "string (optional)",
            "shipment_remarks": "string (optional)",
            "order_remarks": "string (optional)"
          }
        ]
      }
    ]
  }
]
```

### Scheduling

#### GET `/order/schedule`
Retrieves schedules for a specific date with accepted shipments.

**Query Parameters:**
- `schedule_date`: Date in YYYY-MM-DD format

**Response:**
```json
[
  {
    "schedule_code": "bigint",
    "schedule_date": "date",
    "order_code": "string",
    "shift": "string",
    "section": "string",
    "quantity": "integer",
    "completed": "boolean",
    "accepted_shipments": [
      {
        "request_code": "bigint",
        "shipment_code": "integer"
      }
    ]
  }
]
```

#### POST `/order/schedule`
Creates new production schedules.

**Request Body:**
```json
[
  {
    "schedule_date": "string",
    "order_code": "string",
    "shift": "string",
    "section": "string",
    "quantity": "integer"
  }
]
```

**Valid shift values:** `NIGHT`, `NIGHT & DAY`, `DAY`

**Response:**
```json
[
  {
    "schedule_code": "bigint",
    "schedule_date": "date",
    "order_code": "string",
    "shift": "string",
    "section": "string",
    "quantity": "integer"
  }
]
```

#### PUT `/order/schedule`
Marks schedules as completed.

**Request Body:**
```json
[
  {
    "schedule_code": "bigint"
  }
]
```

**Response:**
```json
[
  {
    "schedule_code": "bigint",
    "schedule_date": "date",
    "order_code": "string",
    "shift": "string",
    "section": "string",
    "quantity": "integer",
    "completed": "boolean"
  }
]
```

---

## Record Management Routes (`/order/record`)

### Shipment Records

#### GET `/order/record/shipment`
**Integration:** Central-Handler via barcode-handler

Retrieves shipment records from Central-Handler system.

**Query Parameters:**
- `request_code`: Request code
- `shipment_code`: Shipment code

**Response:**
```json
[
  {
    "table": "string",
    "request_code": "string",
    "shipment_code": "integer",
    "amount": "number",
    "remaining": "number",
    "barcode": "uuid",
    "... additional fields from Central-Handler scan_record() function"
  }
]
```

#### PUT `/order/record/shipment`
**Integration:** Central-Handler via barcode-handler

Updates shipment records in Central-Handler system.

**Request Body:**
```json
[
  {
    "table_name": "string",
    "barcode": "uuid",
    "request_code": "bigint",
    "shipment_code": "integer",
    "amount": "number"
  }
]
```

**Response:**
```json
[
  {
    "barcode": "uuid"
  }
]
```

### Parcel Records

#### GET `/order/record/parcel`
**Integration:** Central-Handler via barcode-handler

Retrieves parcel records from Central-Handler system.

**Query Parameters:**
- `schedule_code`: Schedule code

**Response:**
```json
[
  {
    "barcode": "uuid",
    "request_code": "bigint",
    "shipment_code": "integer",
    "schedule_code": "bigint",
    "amount": "number",
    "tbl": "string"
  }
]
```

#### POST `/order/record/parcel`
**Integration:** Central-Handler via barcode-handler

Creates parcel allocations in Central-Handler system.

**Request Body:**
```json
[
  {
    "table_name": "string",
    "barcode": "uuid",
    "request_code": "bigint",
    "shipment_code": "integer",
    "schedule_code": "bigint",
    "reduced_by": "number"
  }
]
```

**Response:**
```json
[
  {
    "barcode": "uuid"
  }
]
```

---

## Trader Routes (`/trader`)

### Blendsheet Batch Management

#### GET `/trader`
**Integration:** Central-Handler via trader-handler

Retrieves blendsheet batches for trader review.

**Query Parameters:**
- `trader_status`: Filter by trader status (optional)

**Response:**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "blendsheet_no": "string",
    "humidity": "number",
    "density": "number",
    "trader_status": "string",
    "blend_code": "string",
    "remarks": "string"
  }
]
```

#### PUT `/trader`
**Integration:** Central-Handler via trader-handler

Updates trader status for blendsheet batches.

**Request Body:**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "trader_status": "string"
  }
]
```

**Response:**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "trader_status": "string"
  }
]
```

---

## Admin Routes (`/admin`)

### User Management

#### GET `/admin/user`
Retrieves all users from the admin database.

**Response:**
```json
[
  {
    "sub": "uuid",
    "username": "string",
    "group": "string"
  }
]
```

#### POST `/admin/user`
**Integration:** AWS Cognito via cognito-handler

Creates a new user in both Cognito and admin database.

**Request Body:**
```json
{
  "username": "string",
  "password": "string",
  "group": "string"
}
```

**Response:**
```json
{
  "sub": "uuid",
  "username": "string",
  "group": "string"
}
```

#### DELETE `/admin/user`
**Integration:** AWS Cognito via cognito-handler

Deletes a user from both admin database and Cognito.

**Query Parameters:**
- `sub`: User sub (UUID)

**Response:**
```json
{
  "success": "boolean"
}
```

### Group Management

#### GET `/admin/group`
**Integration:** AWS Cognito via cognito-handler

Retrieves available user groups from Cognito.

**Response:**
```json
[
  "group_name_1",
  "group_name_2"
]
```

---

## Central-Handler Proxy Routes (`/central/*`)

All routes under `/central/*` are automatically proxied to the Central-Handler system with the path rewritten from `/central/*` to `/app/*`.

**Examples:**
- `GET /central/tealine` → `GET /app/tealine` (Central-Handler)
- `POST /central/admin/tealine` → `POST /app/admin/tealine` (Central-Handler)
- `PUT /central/blendsheet/batch` → `PUT /app/blendsheet/batch` (Central-Handler)

For complete documentation of available endpoints, see the [Central-Handler API Documentation](./API-Documentation-Central-Handler.md).

---

## Data Types & Validation

### Common Data Types
- `bigint`: Large integers (request codes, timestamps)
- `integer`: Regular integers (quantities, shipment codes)
- `uuid`: UUID v4 format (user sub, barcodes)
- `number`: Decimal numbers (quantities in kg)
- `string`: Text fields
- `date`: Date in YYYY-MM-DD format
- `boolean`: True/false values

### Status Values
- **Order Request Status**: APPROVAL_REQUESTED, ORDER_REQUESTED
- **Shipment Event Status**: APPROVAL_ALLOWED, SHIPMENT_ACCEPTED, SHIPMENT_DISPATCHED, ORDER_NOT_READY
- **Trader Status**: TRADER_PENDING, TRADER_REQUESTED, TRADER_ELEVATED, TRADER_ALLOWED, TRADER_BLOCKED
- **Shift Values**: NIGHT, NIGHT & DAY, DAY

### Validation Rules
- All fields are required unless marked as optional
- Quantities are stored internally in grams (multiply by 1000)
- Timestamps are Unix timestamps in milliseconds
- Date formats must be valid PostgreSQL dates
- Status values must match predefined enums
- Password validation follows Cognito User Pool policy

---

## Integration Architecture

### Central-Handler Integration
The AWS-Manager integrates with Central-Handler through multiple channels:

1. **Direct Proxy** (`/central/*`): Complete passthrough for all Central-Handler endpoints
2. **Lambda Functions**: Specialized handlers for specific operations:
   - **trader-handler**: Calls Central-Handler trader endpoints
   - **barcode-handler**: Calls Central-Handler shipment/parcel/scan endpoints

### Database Schema
- **Public Schema**: Order management (order_plan, order_request, order_shipment, order_shipment_event, order_schedule)
- **Admin Schema**: User management (user table)
- **Database Instance**: Uses its own separate `tea_management` database instance, independent from Central-Handler

### AWS Services Integration
- **API Gateway**: RESTful API endpoints with CORS support
- **Lambda**: Serverless compute for all API operations
- **Cognito**: User authentication and authorization
- **Secrets Manager**: Secure storage of database credentials and API keys
- **RDS PostgreSQL**: Separate database instance with same name as Central-Handler

---

## Environment Variables

### order-handler
- `DB_SECRET`: AWS Secrets Manager secret ID for database credentials
- `CENTRAL_HANDLER`: Lambda function name for Central-Handler proxy
- `TRADER_HANDLER`: Lambda function name for trader operations
- `BARCODE_HANDLER`: Lambda function name for barcode operations
- `COGNITO_HANDLER`: Lambda function name for Cognito operations
- `CORS_ORIGINS`: Allowed CORS origins

### Integration Lambda Functions
- `CENTRAL_HANDLER_API_SECRET`: Secrets Manager secret ID for Central-Handler API
- `CENTRAL_HANDLER_API_KEY`: Key name in secret for Central-Handler host URL

### cognito-handler
- `USER_POOL_ID`: AWS Cognito User Pool ID

---

## Error Handling & Workflows

### Automatic Workflows
1. **APPROVAL_ALLOWED**: Automatically creates ORDER_REQUESTED event
2. **Order Request Creation**: Automatically creates corresponding shipment
3. **Shipment Updates**: Creates new shipments for remaining quantities

### Error Propagation
- Lambda function errors are propagated as HTTP 500 responses
- Central-Handler integration errors include original error messages
- Cognito errors include validation details for password policies

### Data Consistency
- Database transactions ensure atomic operations
- Foreign key constraints maintain referential integrity
- Duplicate prevention through EXISTS checks

---

## Notes

1. All quantities are stored in grams internally but exposed as kg in API responses
2. Timestamps are Unix timestamps in milliseconds
3. Integration with Central-Handler maintains data consistency through API communication between separate database instances
4. AWS Lambda functions provide scalable, serverless architecture
5. Cognito integration provides enterprise-grade authentication
6. CORS headers are managed at API Gateway level for reliability
7. Database connections are established per Lambda invocation for optimal performance