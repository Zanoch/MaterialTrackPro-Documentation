# MTP Platform - Comprehensive API Documentation

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture & Integration](#architecture--integration)
3. [Authentication & Authorization](#authentication--authorization)
4. [AWS-Manager API](#aws-manager-api)
5. [Central-Handler API](#central-handler-api)
6. [Data Types & Validation](#data-types--validation)
7. [Integration Patterns](#integration-patterns)
8. [Error Handling](#error-handling)

---

## System Overview

The MTP Platform consists of two main components working together to manage tea processing operations:

- **AWS-Manager**: A serverless AWS CDK application providing order management, user administration, and shipment tracking
- **Central-Handler**: A Node.js server managing tea processing operations including tealine, blendsheet, flavorsheet, herbline, and blendbalance

Both systems operate separate PostgreSQL database instances (each named `tea_management`) and integrate through AWS API Gateway for seamless data exchange.

---

## Architecture & Integration

### AWS-Manager Architecture
- **API Gateway**: RESTful endpoints with CORS support
- **Lambda Functions**: Serverless compute for API operations
  - order-handler: Main API orchestrator
  - central-handler: Proxy to Central-Handler
  - trader-handler: Trader operations
  - barcode-handler: Shipment/parcel operations
  - cognito-handler: User authentication
- **AWS Cognito**: User authentication and authorization
- **RDS PostgreSQL**: Shared database with Central-Handler

### Central-Handler Architecture  
- **Node.js Server**: Main application server
- **PostgreSQL Integration**: Direct database operations
- **AWS API Gateway Client**: Integration with AWS-Manager

### Integration Points
1. **Shared Database**: Both systems use `tea_management` PostgreSQL database
2. **API Proxying**: AWS-Manager proxies requests to Central-Handler via `/central/*` routes
3. **Specialized Handlers**: Lambda functions call specific Central-Handler endpoints
4. **Real-time Sync**: Shipment and order data synchronized between systems

---

## Authentication & Authorization

### AWS-Manager
- **AWS Cognito User Pool**: User authentication and group membership management
- **AWS Cognito Identity Pool**: Maps user groups to IAM roles
- **Role-based access control**: 10 specific user groups with role-based endpoint access
- **Two-tier hierarchy**: Admin level (3 roles) with full `/admin/*` access, User level (7 roles) denied `/admin/*` access
- **Historical data access**: Select roles can access AWS-Manager or Central-Handler historical records
- **CORS enabled** for cross-origin requests from Admin-Panel

**Role Summary:**
- **Admin Roles**: MASTER-ADMIN, GRANDPASS-ADMIN, SEEDUWA-ADMIN
- **User Roles**: GRANDPASS-OFFICER, GRANDPASS-SUPERVISOR, SEEDUWA-OFFICER, GRANDPASS-GATEPASS, GRANDPASS-DEV-MACHINE, TRADER-OFFICER, TRADER-SUPERVISOR

For complete role documentation and API access patterns, see [Role-Based Access Control Documentation](./Role-Based-Access-Control.md).

### Central-Handler
- **CORS headers** support
- **Supported methods**: GET, OPTIONS, POST, PUT
- **Integration**: Accessible via AWS-Manager proxy routes (`/central/*` → `/app/*`)

---

## AWS-Manager API

### Base URL Structure
- `/order/*` - Order management operations
- `/central/*` - Direct proxy to Central-Handler (rewritten to `/app/*`)
- `/trader/*` - Trader operations
- `/admin/*` - Administrative operations

### Order Management Routes (`/order`)

#### Order Planning

**GET `/order/plan`**
Retrieves order plans with allocation and request information.

Query Parameters:
- `current_date`: Date in YYYY-MM-DD format (TEMPORARILY IGNORED - returns all plans)

Response:
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

**POST `/order/plan`**
Creates new order plans from raw planning data.

Request Body:
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

**DELETE `/order/plan`**
Deletes an order plan if no requests exist.

Query Parameters:
- `order_code`: Order code to delete

#### Order Requests and Shipments

**GET `/order`**
Retrieves order requests with shipment events.

Query Parameters:
- `status`: Filter by shipment status (optional)
- `timestamp`: Filter by event timestamp (optional)
- `shipment_vehicle`: Filter by vehicle (optional)

**POST `/order`**
Creates new order requests with automatic shipment generation.

Valid status values: `APPROVAL_REQUESTED`, `ORDER_REQUESTED`

#### Shipment Management

**PUT `/order/shipment`**
Updates shipment quantities and creates new shipments for remaining quantities.

**POST `/order/shipment/event`**
Creates shipment status events with automatic workflow handling.

Valid status values:
- `APPROVAL_ALLOWED` (auto-creates ORDER_REQUESTED event)
- `SHIPMENT_ACCEPTED`
- `SHIPMENT_DISPATCHED`
- `ORDER_NOT_READY`

#### Scheduling

**GET `/order/schedule`**
Retrieves schedules for a specific date.

**POST `/order/schedule`**
Creates new production schedules.

Valid shift values: `NIGHT`, `NIGHT & DAY`, `DAY`

**PUT `/order/schedule`**
Marks schedules as completed.

### Record Management Routes (`/order/record`)

**Integration with Central-Handler via barcode-handler**

#### Shipment Records
- **GET `/order/record/shipment`**: Retrieves shipment records
- **PUT `/order/record/shipment`**: Updates shipment records

#### Parcel Records  
- **GET `/order/record/parcel`**: Retrieves parcel records
- **POST `/order/record/parcel`**: Creates parcel allocations

### Trader Routes (`/trader`)

**Integration with Central-Handler via trader-handler**

- **GET `/trader`**: Retrieves blendsheet batches for trader review
- **PUT `/trader`**: Updates trader status for blendsheet batches

### Admin Routes (`/admin`)

#### User Management
- **GET `/admin/user`**: Retrieves all users
- **POST `/admin/user`**: Creates new user (integrates with Cognito)
- **DELETE `/admin/user`**: Deletes user (integrates with Cognito)

#### Group Management
- **GET `/admin/group`**: Retrieves available user groups from Cognito

### Central-Handler Proxy Routes (`/central/*`)

All routes under `/central/*` are automatically proxied to Central-Handler with path rewriting from `/central/*` to `/app/*`.

Examples:
- `GET /central/tealine` → `GET /app/tealine`
- `POST /central/admin/tealine` → `POST /app/admin/tealine`

---

## Central-Handler API

### Base URL Structure
- `/app/admin` - Administrative operations
- `/app/trader` - Trader-specific operations
- `/app` - General user operations

### Admin Routes (`/app/admin`)

#### Tealine Management
- **POST `/app/admin/tealine`**: Creates new tealine item
- **GET `/app/admin/tealine`**: Retrieves tealine items with optional filtering

#### Blendsheet Management
- **POST `/app/admin/blendsheet`**: Creates new blendsheet with mixtures
- **GET `/app/admin/blendsheet`**: Retrieves blendsheet items with batch information

#### Flavorsheet Management
- **POST `/app/admin/flavorsheet`**: Creates new flavorsheet with mixtures
- **GET `/app/admin/flavorsheet`**: Retrieves flavorsheet items

#### Herbline Management
- **POST `/app/admin/herbline`**: Creates new herbline item
- **GET `/app/admin/herbline`**: Retrieves herbline items

#### Blendbalance Management
- **POST `/app/admin/blendbalance`**: Creates new blendbalance item
- **GET `/app/admin/blendbalance`**: Retrieves blendbalance items

#### Location Management
- **POST `/app/admin/location`**: Creates new storage location

### Trader Routes (`/app/trader`)

#### Blendsheet Batch Management
- **GET `/app/trader/blendsheet/batch`**: Retrieves batches for trader review
- **PUT `/app/trader/blendsheet/batch`**: Updates trader status

### User Routes (`/app`)

#### Tealine Operations
- **GET `/app/tealine`**: Retrieves incomplete tealine items
- **POST `/app/tealine/record`**: Creates new tealine record
- **PUT `/app/tealine/record`**: Updates tealine record status
- **POST `/app/tealine/record/blendsheet`**: Allocates tealine record to blendsheet batch for ingredient traceability
- **POST `/app/tealine/record/shipment`**: Allocates record to shipment
- **POST `/app/tealine/record/parcel`**: Allocates record to parcel

#### Blendsheet Operations
- **GET `/app/blendsheet`**: Retrieves incomplete blendsheet items
- **PUT `/app/blendsheet`**: Updates blendsheet batch count
- **POST `/app/blendsheet/batch`**: Creates new blendsheet batch
- **GET `/app/blendsheet/batch`**: Retrieves incomplete batches with ingredient allocation data
- **PUT `/app/blendsheet/batch`**: Updates batch (completion or trader preparation)
- **POST `/app/blendsheet/record`**: Creates new blendsheet record
- **PUT `/app/blendsheet/record`**: Updates blendsheet record status
- **POST `/app/blendsheet/record/shipment`**: Allocates record to shipment
- **POST `/app/blendsheet/record/parcel`**: Allocates record to parcel

#### Flavorsheet Operations
- **GET `/app/flavorsheet`**: Retrieves flavorsheets without created batches
- **POST `/app/flavorsheet/batch`**: Creates new flavorsheet batch
- **GET `/app/flavorsheet/batch`**: Retrieves incomplete batches
- **PUT `/app/flavorsheet/batch`**: Updates batch completion status
- **POST `/app/flavorsheet/record`**: Creates new flavorsheet record
- **PUT `/app/flavorsheet/record`**: Updates flavorsheet record status

#### Herbline Operations
- **GET `/app/herbline`**: Retrieves herbline items
- **POST `/app/herbline/record`**: Creates new herbline record
- **PUT `/app/herbline/record`**: Updates herbline record to DISPATCHED

#### Blendbalance Operations
- **GET `/app/blendbalance`**: Retrieves incomplete blendbalance items
- **POST `/app/blendbalance/record`**: Creates new blendbalance record
- **PUT `/app/blendbalance/record`**: Updates blendbalance record status
- **POST `/app/blendbalance/record/blendsheet`**: Allocates blendbalance record to blendsheet batch for ingredient traceability

#### Utility Operations
- **GET `/app/location`**: Retrieves storage locations
- **GET `/app/scan`**: Scans barcode to retrieve comprehensive record information

#### Shipment Operations (AWS-Manager Integration)
- **GET `/app/shipment`**: Retrieves available shipments from AWS-Manager
- **PUT `/app/shipment`**: Updates shipment quantity in AWS-Manager
- **POST `/app/shipment/event`**: Creates shipment dispatch events in AWS-Manager
- **GET `/app/shipment/record`**: Retrieves shipment records
- **GET `/app/parcel/record`**: Retrieves parcel records

---

## Data Types & Validation

### Common Data Types
- `bigint`: Large integers (request codes, timestamps)
- `integer`: Regular integers (quantities, shipment codes)
- `uuid`: UUID v4 format (user sub, barcodes)
- `number`: Decimal numbers (quantities in kg/grams)
- `string`: Text fields
- `date`: Date in YYYY-MM-DD format
- `boolean`: True/false values

### Status Values

#### Order & Shipment Status
- **Order Request Status**: APPROVAL_REQUESTED, ORDER_REQUESTED
- **Shipment Event Status**: APPROVAL_ALLOWED, SHIPMENT_ACCEPTED, SHIPMENT_DISPATCHED, ORDER_NOT_READY
- **Record Status**: ACCEPTED, IN_PROCESS, PROCESSED, DISPATCHED

#### Trader Status
- TRADER_PENDING, TRADER_REQUESTED, TRADER_ELEVATED, TRADER_ALLOWED, TRADER_BLOCKED

#### Shift Values
- NIGHT, NIGHT & DAY, DAY

### Validation Rules
- All fields are required unless marked as optional
- Quantities stored internally in grams (multiply by 1000 for AWS-Manager)
- Timestamps are Unix timestamps in milliseconds
- Date formats must be valid PostgreSQL dates
- Status values must match predefined enums
- Barcode values are auto-generated UUIDs
- Weight values must be positive numbers

---

## Integration Patterns

### 1. Direct Database Sharing
Both systems share the `tea_management` PostgreSQL database:
- **Public Schema**: Order management tables (AWS-Manager)
- **Admin Schema**: User management (AWS-Manager)
- **Processing Tables**: Tea processing operations (Central-Handler)

### 2. API Proxying
AWS-Manager acts as a gateway:
```
Client → AWS-Manager (/central/*) → Central-Handler (/app/*)
```

### 3. Specialized Lambda Integration
Dedicated Lambda functions for specific operations:
- **trader-handler**: Calls Central-Handler trader endpoints
- **barcode-handler**: Calls Central-Handler shipment/parcel/scan endpoints

### 4. Real-time Synchronization
- Shipment data flows from AWS-Manager to Central-Handler
- Order requests trigger shipment creation
- Status updates propagate between systems

### 5. Allocation Workflows
- **Shipment Allocation**: Records allocated to AWS-Manager shipments for order fulfillment
- **Blendsheet Allocation**: Records allocated to blendsheet batches for ingredient traceability
- **Parcel Allocation**: Refinement of shipment allocations for production scheduling
- **Dynamic Scanning**: Enhanced `scan_record()` function with comprehensive allocation calculation

#### Enhanced scan_record() Function
The `scan_record()` function uses dynamic allocation discovery to calculate remaining quantities:

**Allocation Calculation Formula:**
```
remaining = original_quantity - (shipment_allocations + blendsheet_allocations + ...other_allocations)
```

**Critical Design Decision:**
- **Parcel allocations are EXCLUDED** from the calculation to prevent double-counting
- **Rationale**: Parcel allocations represent subdivisions of shipment allocations, not additional consumption
- **Business Logic**: Including parcels would double-count allocated materials and distort inventory tracking

**Dynamic Discovery:**
- Automatically finds all `{table_name}_allocate_*` tables except `{table_name}_allocate_parcel`
- Future-proof: automatically includes new allocation types without code changes
- Unified calculation: single query handles all allocation types efficiently

### 6. AWS Services Integration
- **API Gateway**: RESTful API endpoints
- **Lambda**: Serverless compute
- **Cognito**: Authentication and authorization
- **Secrets Manager**: Secure credential storage
- **RDS PostgreSQL**: Shared database

---

## Error Handling

### Standard HTTP Status Codes
- **200**: Success
- **400**: Bad Request (validation errors)
- **404**: Not Found
- **500**: Internal Server Error

### AWS-Manager Error Handling
- Lambda function errors propagated as HTTP 500
- Central-Handler integration errors include original messages
- Cognito errors include validation details
- Database transactions ensure atomic operations

### Central-Handler Error Handling
- Descriptive error messages for validation failures
- CORS enabled for error responses
- Database constraint violations handled gracefully

### Automatic Workflows
1. **APPROVAL_ALLOWED**: Automatically creates ORDER_REQUESTED event
2. **Order Request Creation**: Automatically creates corresponding shipment
3. **Shipment Updates**: Creates new shipments for remaining quantities

### Data Consistency
- Database transactions ensure atomic operations
- Foreign key constraints maintain referential integrity
- Duplicate prevention through EXISTS checks
- Shared database ensures data consistency between systems

---

## Database Functions

### scan_record(table_name, barcode)
Sophisticated PostgreSQL function that:
1. Validates table existence
2. Handles shipment allocations with remaining quantity calculations
3. Joins with parent tables automatically
4. Returns enriched data with complete record information

Used by `/app/scan` and `/app/shipment/record` endpoints.

---

## Environment Variables

### AWS-Manager Configuration
- `DB_SECRET`: Database credentials (Secrets Manager)
- `CENTRAL_HANDLER`: Lambda function name for Central-Handler proxy
- `TRADER_HANDLER`: Lambda function name for trader operations
- `BARCODE_HANDLER`: Lambda function name for barcode operations
- `COGNITO_HANDLER`: Lambda function name for Cognito operations
- `CORS_ORIGINS`: Allowed CORS origins
- `USER_POOL_ID`: AWS Cognito User Pool ID

### Central-Handler Configuration
- `SHIPMENT_API`: Base URL for AWS-Manager integration
- Database connection via standard PostgreSQL pool

---

## Additional Endpoints

*For enhanced dashboard functionality, search capabilities, and performance optimizations, see:*

**📊 [Dashboard API Endpoints Documentation](Dashboard-API-Endpoints.md)**

This separate document covers **dashboard-specific endpoints** implemented to address critical performance bottlenecks identified during comprehensive testing of 17 dashboard pages. These endpoints solve specific dashboard requirements while maintaining complete backward compatibility.

### Performance Optimization Endpoints
- **`GET /app/tealine/pending-with-calculations`**: Replaces 542 API calls with single optimized call (99.8% reduction, 30+ seconds → 1.5 seconds)
- **`GET /app/admin/tealine/inventory-complete`**: Complete inventory with allocation tracking (93% performance improvement)

### Search Enhancement Endpoints  
- **Flavorsheet Search**: `GET /app/admin/flavorsheet/search`, `/app/flavorsheet/search`, `/app/flavorsheet/batch/search`
- **Herbline Search**: `GET /app/admin/herbline/search`, `/app/herbline/search`
- **Features**: Server-side pattern matching, case-insensitive search, combined filtering

### Pagination Enhancement
- **`GET /app/admin/blendsheet/paginated`**: Handles large datasets (818+ items) with memory-efficient pagination

### Analytics & Intelligence Enhancement
- **`GET /app/flavorsheet/stats`**: Production statistics and completion analytics for dashboard KPIs
- **`GET /app/flavorsheet/:flavorsheetNo/composition`**: Detailed ingredient composition analysis for quality control
- **`GET /app/herbline/analytics`**: Inventory analytics and completion metrics for procurement planning
- **`GET /app/blendsheet/summary`**: Comprehensive blendsheet analytics with calculated weights, batch tracking, and production efficiency metrics

### Key Design Principles
- **Dashboard-First Design**: Optimized specifically for dashboard frontend performance requirements
- **100% Backward Compatibility**: Original endpoints remain unchanged and fully functional
- **Separation-Based Approach**: Enhanced endpoints use path extensions (e.g., `/search`, `/paginated`, `/inventory-complete`)
- **Zero Risk Migration**: Frontend teams can adopt enhanced endpoints progressively without breaking existing functionality

These endpoints directly implement solutions from the dashboard requirements document (`Claude/Extras/dashboard-api-requirements-v3.md`) and transform previously unusable dashboard pages into responsive, real-time interfaces.

---

---

## Device-Panel API Usage Patterns

### Blendbalance Creation Workflow
**New Feature (2025-07-08)**: Device-Panel cleanup operations integration

#### Workflow Pattern
Device-Panel implements a specialized blendbalance creation workflow with real-time barcode printing:

```javascript
// Step 1: Order Plan Selection (Device-Panel → AWS-Manager)
GET /order/plan
// Returns orders with item_code (blend_code) for selection

// Step 2: Store Location Setup (Device-Panel → Central-Handler)  
GET /app/location?herbline_section=0
// Gets default non-herbline location for bag storage

// Step 3: Blendbalance Creation (Device-Panel → Central-Handler)
// Now performed during selection phase
POST /app/admin/blendbalance {
  item_code: "user_input",           // Manual blendbalance identifier
  blend_code: "from_order_plan",     // Extracted from selected order
  transfer_id: "user_input"          // Manual transfer identifier
  // Note: weight parameter removed to enable real-time processing
}

// Step 4: Real-Time Bag Records (Device-Panel → Central-Handler)
// Each bag creates immediate record with barcode printing
POST /app/blendbalance/record {
  item_code: "matching_blendbalance",
  created_ts: "from_step3_response", 
  store_location: "default_location",
  gross_weight: "loadcell_reading",
  bag_weight: "manual_tare_weight"
}

// Step 5: Immediate Barcode Printing (Device-Panel Hardware)
app.serialTx("printer", {
  "Transfer ID": transfer_id,
  "Blend Balance No": item_code,
  "Scanned At": current_timestamp,
  "Location": store_location,
  "Gross Weight": gross_weight,
  "Bag Weight": bag_weight,
  "Net Weight": net_weight,
  "Bag No": bag_id,
  barcode: `blendbalance:${barcode}`
})
```

#### Key Integration Points
- **Order Plan Integration**: Uses AWS-Manager `order_plan.item_code` as blend_code source
- **Hardware Integration**: Loadcell weight readings via Electron serial communication
- **Real-Time Processing**: Immediate record creation with barcode printing (v2025.07.09)
- **Circular Dependency Resolution**: Weight calculated dynamically to enable real-time workflow
- **Default Location Strategy**: First non-herbline location used automatically

#### Database Schema Impact
**AWS-Manager Changes**:
- `order_plan` table: Added `item_code` column with NOT NULL constraint
- Automated item_code population from `order_plan_raw` with regex processing
- Updated `POST /order/plan` endpoint to include item_code processing

## Notes

1. **Quantity Handling**: AWS-Manager stores quantities in grams internally but exposes as kg; Central-Handler uses direct weight values
2. **Timestamp Format**: Unix timestamps in milliseconds across both systems
3. **UUID Generation**: Automatic UUID generation for barcodes and user IDs
4. **Authentication**: Enterprise-grade through AWS Cognito
5. **Scalability**: Serverless architecture for AWS-Manager, Node.js server for Central-Handler
6. **Data Flow**: Bidirectional integration through API communication between separate database instances ensuring consistency
7. **CORS Management**: Handled at API Gateway level for reliability
8. **Performance**: Optimized database connections and Lambda cold starts
9. **Device-Panel Integration**: Hardware-integrated workflows for specialized tea processing operations