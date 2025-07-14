# Central-Handler API Documentation

## Overview

The Central-Handler is a Node.js web server that manages tea processing operations including tealine, blendsheet, flavorsheet, herbline, and blendbalance management. It connects to its own PostgreSQL database instance (`tea_management`) and integrates with AWS-Manager through API calls.

## Base URL Structure

The API has three main route groups:
- `/app/admin` - Administrative operations
- `/app/trader` - Trader-specific operations  
- `/app` - General user operations

## Authentication & Authorization

The API uses CORS headers and supports the following methods: GET, OPTIONS, POST, PUT

## Error Handling

Standard HTTP status codes are returned:
- 400: Bad Request (validation errors)
- 404: Not Found
- 500: Internal Server Error

---

## Admin Routes (`/app/admin`)

### Tealine Management

#### POST `/app/admin/tealine`
Creates a new tealine item.

**Request Body:**
```json
{
  "item_code": "string",
  "invoice_no": "string (optional)",
  "grade": "string (optional)",
  "no_of_bags": "integer",
  "weight_per_bag": "number",
  "broker": "string (optional)",
  "garden": "string (optional)",
  "purchase_order": "string (optional)"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint"
}
```

#### GET `/app/admin/tealine`
Retrieves tealine items with optional filtering.

**Query Parameters (recommended filters):**
- `item_code`: Item code
- `created_ts`: Creation timestamp

**Response (without parameters):**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "broker": "string",
    "garden": "string",
    "purchase_order": "string"
  }
]
```

**Note:** Weight is calculated dynamically as the sum of `(gross_weight - bag_weight)` from all related `blendbalance_record` entries.

**Response (with parameters):**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "invoice_no": "string",
  "grade": "string",
  "no_of_bags": "integer",
  "broker": "string",
  "garden": "string",
  "purchase_order": "string",
  "record_list": [
    {
      "id": "integer",
      "received_ts": "bigint",
      "store_location": "string",
      "bag_weight": "number",
      "gross_weight": "number",
      "barcode": "uuid"
    }
  ]
}
```

### Blendsheet Management

#### POST `/app/admin/blendsheet`
Creates a new blendsheet with mixtures.

**Request Body:**
```json
{
  "blendsheet_no": "string",
  "standard": "string",
  "grade": "string (optional)",
  "remarks": "string",
  "total": "number",
  "blend_code": "string (optional)",
  "mixtures": {
    "tealine": [
      {
        "mixture_code": "string",
        "no_of_bags": "integer"
      }
    ],
    "blendbalance": [
      {
        "mixture_code": "string",
        "weight": "number"
      }
    ]
  }
}
```

**Response:**
```json
{
  "blendsheet_no": "string"
}
```

#### GET `/app/admin/blendsheet`
Retrieves blendsheet items with batch information.

**Query Parameters (recommended filters):**
- `blendsheet_no`: Blendsheet number

**Response (without parameters):**
```json
[
  {
    "blendsheet_no": "string",
    "standard": "string",
    "no_of_batches": "integer",
    "created_batches": "integer"
  }
]
```

**Response (with parameters):**
```json
{
  "blendsheet_no": "string",
  "standard": "string",
  "grade": "string",
  "remarks": "string",
  "total": "number",
  "blend_code": "string",
  "no_of_batches": "integer",
  "mixtures": {
    "tealine": [
      {
        "mixture_code": "string",
        "no_of_bags": "integer"
      }
    ],
    "blendbalance": [
      {
        "mixture_code": "string",
        "weight": "number"
      }
    ]
  },
  "created_batches": "integer"
}
```

### Flavorsheet Management

#### POST `/app/admin/flavorsheet`
Creates a new flavorsheet with mixtures.

**Request Body:**
```json
{
  "flavorsheet_no": "string",
  "flavor_code": "string",
  "remarks": "string",
  "mixtures": [
    {
      "mixture_code": "string",
      "weight": "number"
    }
  ]
}
```

**Response:**
```json
{
  "flavorsheet_no": "string"
}
```

#### GET `/app/admin/flavorsheet`
Retrieves flavorsheet items.

**Query Parameters (recommended filters):**
- `flavorsheet_no`: Flavorsheet number

**Response (without parameters):**
```json
[
  {
    "flavorsheet_no": "string",
    "flavor_code": "string",
    "batch_created": "boolean"
  }
]
```

**Response (with parameters):**
```json
{
  "flavorsheet_no": "string",
  "flavor_code": "string",
  "remarks": "string",
  "mixtures": [
    {
      "mixture_code": "string",
      "weight": "number"
    }
  ],
  "batch_created": "boolean"
}
```

### Herbline Management

#### POST `/app/admin/herbline`
Creates a new herbline item.

**Request Body:**
```json
{
  "item_code": "string",
  "item_name": "string",
  "purchase_order": "string",
  "weight": "number"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint"
}
```

#### GET `/app/admin/herbline`
Retrieves herbline items.

**Query Parameters (recommended filters):**
- `item_code`: Item code
- `created_ts`: Creation timestamp

**Response (without parameters):**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "item_name": "string",
    "purchase_order": "string",
    "weight": "number"
  }
]
```

**Response (with parameters):**
```json
{
  "item_code": "string",
  "item_name": "string",
  "created_ts": "bigint",
  "purchase_order": "string",
  "weight": "number",
  "record_list": [
    {
      "received_ts": "bigint",
      "store_location": "string",
      "bag_weight": "number",
      "gross_weight": "number",
      "reference": "string",
      "barcode": "uuid",
      "remaining": "number"
    }
  ]
}
```

### Blendbalance Management

#### POST `/app/admin/blendbalance`
Creates a new blendbalance item.

**Request Body:**
```json
{
  "item_code": "string",
  "blend_code": "string",
  "transfer_id": "string"
}
```

**Note:** Weight parameter removed in v2025.07.09 to enable real-time barcode printing. Weight is now calculated dynamically from individual records.

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint"
}
```

#### GET `/app/admin/blendbalance`
Retrieves blendbalance items.

**Query Parameters (recommended filters):**
- `created_ts`: Creation timestamp
- `item_code`: Item code

**Response (without parameters):**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "blend_code": "string",
    "weight": "number",
    "transfer_id": "string"
  }
]
```

**Response (with parameters):**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "blend_code": "string",
  "weight": "number",
  "transfer_id": "string",
  "record_list": [
    {
      "id": "integer",
      "received_ts": "bigint",
      "store_location": "string",
      "bag_weight": "number",
      "gross_weight": "number",
      "barcode": "uuid"
    }
  ]
}
```

### Location Management

#### POST `/app/admin/location`
Creates a new storage location.

**Request Body:**
```json
{
  "location_name": "string",
  "herbline_section": "boolean"
}
```

**Response:**
```json
{
  "location": "string"
}
```

---

## Trader Routes (`/app/trader`)

### Blendsheet Batch Management

#### GET `/app/trader/blendsheet/batch`
Retrieves blendsheet batches for trader review.

**Query Parameters:**
- `trader_status`: Filter by trader status (optional, default: "TRADER_PENDING")

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

#### PUT `/app/trader/blendsheet/batch`
Updates trader status for a blendsheet batch.

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "trader_status": "string"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "trader_status": "string"
}
```

---

## User Routes (`/app`)

### Tealine Operations

#### GET `/app/tealine`
Retrieves incomplete tealine items (where recorded bags < expected bags).

**Query Parameters (recommended filters):**
- `item_code`: Item code
- `created_ts`: Creation timestamp

**Response (without parameters):**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "broker": "string",
    "garden": "string",
    "purchase_order": "string"
  }
]
```

**Response (with parameters):**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "invoice_no": "string",
  "grade": "string",
  "no_of_bags": "integer",
  "broker": "string",
  "garden": "string",
  "purchase_order": "string",
  "record_list": [
    {
      "id": "integer",
      "received_ts": "bigint",
      "store_location": "string",
      "bag_weight": "number",
      "gross_weight": "number",
      "barcode": "uuid"
    }
  ]
}
```

#### POST `/app/tealine/record`
Creates a new tealine record (bag entry).

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "string",
  "store_location": "string",
  "gross_weight": "number",
  "bag_weight": "number"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "string",
  "id": "integer",
  "received_ts": "bigint",
  "barcode": "uuid"
}
```

#### PUT `/app/tealine/record`
Updates tealine record status with conditional logic.

**Request Body:**
```json
{
  "barcode": "uuid",
  "reduced_by": "number (optional)"
}
```

**Status Logic:**
- **Without `reduced_by`**: Status becomes `"DISPATCHED"`
- **With `reduced_by`**: Status becomes `"PROCESSED"` if remaining quantity equals 0, otherwise `"IN_PROCESS"`

**Response:**
```json
{
  "barcode": "uuid",
  "status": "string"
}
```

#### POST `/app/tealine/record/shipment`
Allocates tealine record to shipment.

**Request Body:**
```json
{
  "barcode": "uuid",
  "request_code": "bigint",
  "shipment_code": "integer",
  "amount": "number"
}
```

**Response:**
```json
{
  "barcode": "uuid"
}
```

#### POST `/app/tealine/record/parcel`
Allocates tealine record to parcel.

**Request Body:**
```json
{
  "barcode": "uuid",
  "request_code": "bigint",
  "shipment_code": "integer",
  "schedule_code": "bigint",
  "reduced_by": "number"
}
```

**Response:**
```json
{
  "barcode": "uuid"
}
```

### Blendsheet Operations

#### GET `/app/blendsheet`
Retrieves incomplete blendsheet items (where created batches < expected batches).

**Query Parameters (recommended filters):**
- `blendsheet_no`: Blendsheet number

**Response (without parameters):**
```json
[
  {
    "blendsheet_no": "string",
    "standard": "string",
    "no_of_batches": "integer",
    "created_batches": "integer"
  }
]
```

**Response (with parameters):**
```json
{
  "blendsheet_no": "string",
  "standard": "string",
  "grade": "string",
  "remarks": "string",
  "total": "number",
  "blend_code": "string",
  "no_of_batches": "integer",
  "mixtures": {
    "tealine": [
      {
        "mixture_code": "string",
        "no_of_bags": "integer"
      }
    ],
    "blendbalance": [
      {
        "mixture_code": "string",
        "weight": "number"
      }
    ]
  },
  "created_batches": "integer"
}
```

#### PUT `/app/blendsheet`
Updates blendsheet batch count.

**Request Body:**
```json
{
  "blendsheet_no": "string",
  "no_of_batches": "integer"
}
```

**Response:**
```json
{
  "blendsheet_no": "string",
  "no_of_batches": "integer"
}
```

#### POST `/app/blendsheet/batch`
Creates a new blendsheet batch.

**Request Body:**
```json
{
  "blendsheet_no": "string"
}
```

**Response:**
```json
{
  "blendsheet_no": "string",
  "item_code": "string",
  "created_ts": "bigint"
}
```

#### GET `/app/blendsheet/batch`
Retrieves incomplete blendsheet batches.

**Query Parameters (recommended filters):**
- `item_code`: Item code
- `created_ts`: Creation timestamp

**Response (without parameters):**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "blendsheet_no": "string",
    "humidity": "number",
    "density": "number",
    "trader_status": "string",
    "remarks": "string"
  }
]
```

**Response (with parameters):**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "blendsheet_no": "string",
  "humidity": "number",
  "density": "number",
  "trader_status": "string",
  "standard": "string",
  "remarks": "string",
  "total": "number",
  "blend_code": "string",
  "record_list": [
    {
      "id": "integer",
      "received_ts": "bigint",
      "store_location": "string",
      "bag_weight": "number",
      "gross_weight": "number",
      "barcode": "uuid"
    }
  ],
  "allocations": {
    "tealine": [
      {
        "barcode": "uuid",
        "amount": "number"
      }
    ],
    "blendbalance": [
      {
        "barcode": "uuid",
        "amount": "number"
      }
    ]
  }
}
```

**Note:** The `allocations` field contains ingredient traceability data. For batches created before the ingredient traceability update, this field will contain empty arrays.

#### PUT `/app/blendsheet/batch`
Updates blendsheet batch with one of two mutually exclusive operations:

##### Option 1 - Mark batch as completed

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "completed": "boolean"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "completed": "boolean"
}
```

##### Option 2 - Prepare batch for trader requests (sets trader_status to "TRADER_PENDING")

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "humidity": "number",
  "density": "number"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "humidity": "number",
  "density": "number"
}
```

#### POST `/app/blendsheet/record`
Creates a new blendsheet record.

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "store_location": "string",
  "gross_weight": "number",
  "bag_weight": "number"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "id": "integer",
  "received_ts": "bigint",
  "barcode": "uuid"
}
```

#### PUT `/app/blendsheet/record`
Updates blendsheet record status with conditional logic.

**Request Body:**
```json
{
  "barcode": "uuid",
  "reduced_by": "number (optional)"
}
```

**Status Logic:**
- **Without `reduced_by`**: Status becomes `"DISPATCHED"`
- **With `reduced_by`**: Status becomes `"PROCESSED"` if remaining quantity equals 0, otherwise `"IN_PROCESS"`

**Response:**
```json
{
  "barcode": "uuid",
  "status": "string"
}
```

#### POST `/app/blendsheet/record/shipment`
Allocates blendsheet record to shipment.

**Request Body:**
```json
{
  "barcode": "uuid",
  "request_code": "bigint",
  "shipment_code": "integer",
  "amount": "number"
}
```

**Response:**
```json
{
  "barcode": "uuid"
}
```

#### POST `/app/blendsheet/record/parcel`
Allocates blendsheet record to parcel.

**Request Body:**
```json
{
  "barcode": "uuid",
  "request_code": "bigint",
  "shipment_code": "integer",
  "schedule_code": "bigint",
  "reduced_by": "number"
}
```

**Response:**
```json
{
  "barcode": "uuid"
}
```

### Blendsheet Ingredient Allocation Operations

#### POST `/app/tealine/record/blendsheet`
Allocates tealine record to blendsheet batch for ingredient traceability.

**Request Body:**
```json
{
  "barcode": "uuid",
  "item_code": "string",
  "created_ts": "bigint",
  "amount": "number"
}
```

**Response:**
```json
{
  "barcode": "uuid"
}
```

#### POST `/app/blendbalance/record/blendsheet`
Allocates blendbalance record to blendsheet batch for ingredient traceability.

**Request Body:**
```json
{
  "barcode": "uuid",
  "item_code": "string",
  "created_ts": "bigint",
  "amount": "number"
}
```

**Response:**
```json
{
  "barcode": "uuid"
}
```

### Flavorsheet Operations

#### GET `/app/flavorsheet`
Retrieves flavorsheets without created batches.

**Query Parameters (recommended filters):**
- `flavorsheet_no`: Flavorsheet number

**Response (without parameters):**
```json
[
  {
    "flavorsheet_no": "string",
    "flavor_code": "string"
  }
]
```

**Response (with parameters):**
```json
{
  "flavorsheet_no": "string",
  "flavor_code": "string",
  "remarks": "string",
  "mixtures": [
    {
      "mixture_code": "string",
      "weight": "number"
    }
  ],
  "batch_created": "boolean"
}
```

#### POST `/app/flavorsheet/batch`
Creates a new flavorsheet batch.

**Request Body:**
```json
{
  "flavorsheet_no": "string"
}
```

**Response:**
```json
{
  "flavorsheet_no": "string",
  "item_code": "string",
  "created_ts": "bigint"
}
```

#### GET `/app/flavorsheet/batch`
Retrieves incomplete flavorsheet batches.

**Query Parameters (recommended filters):**
- `item_code`: Item code
- `created_ts`: Creation timestamp

**Response (without parameters):**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "flavorsheet_no": "string",
    "remarks": "string"
  }
]
```

**Response (with parameters):**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "flavorsheet_no": "string",
  "flavor_code": "string",
  "remarks": "string",
  "record_list": [
    {
      "id": "integer",
      "received_ts": "bigint",
      "store_location": "string",
      "bag_weight": "number",
      "gross_weight": "number",
      "barcode": "uuid"
    }
  ]
}
```

#### PUT `/app/flavorsheet/batch`
Updates flavorsheet batch completion status.

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "completed": "boolean"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "completed": "boolean"
}
```

#### POST `/app/flavorsheet/record`
Creates a new flavorsheet record.

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "store_location": "string",
  "gross_weight": "number",
  "bag_weight": "number"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "id": "integer",
  "received_ts": "bigint",
  "barcode": "uuid"
}
```

#### PUT `/app/flavorsheet/record`
Updates flavorsheet record status with conditional logic.

**Request Body:**
```json
{
  "barcode": "uuid",
  "reduced_by": "number (optional)"
}
```

**Status Logic:**
- **Without `reduced_by`**: Status becomes `"DISPATCHED"`
- **With `reduced_by`**: Status becomes `"PROCESSED"` if remaining quantity equals 0, otherwise `"IN_PROCESS"`

**Response:**
```json
{
  "barcode": "uuid",
  "status": "string"
}
```

### Herbline Operations

#### GET `/app/herbline`
Retrieves herbline items.

**Query Parameters (recommended filters):**
- `item_code`: Item code
- `created_ts`: Creation timestamp

**Response (without parameters):**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "item_name": "string",
    "purchase_order": "string",
    "weight": "number"
  }
]
```

**Response (with parameters):**
```json
{
  "item_code": "string",
  "item_name": "string",
  "created_ts": "bigint",
  "purchase_order": "string",
  "weight": "number",
  "record_list": [
    {
      "received_ts": "bigint",
      "store_location": "string",
      "bag_weight": "number",
      "gross_weight": "number",
      "reference": "string",
      "barcode": "uuid",
      "remaining": "number"
    }
  ]
}
```

#### POST `/app/herbline/record`
Creates a new herbline record.

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "number",
  "store_location": "string",
  "gross_weight": "number",
  "bag_weight": "number",
  "reference": "string"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "received_ts": "bigint",
  "barcode": "uuid"
}
```

#### PUT `/app/herbline/record`
Updates herbline record status to DISPATCHED.

**Request Body:**
```json
{
  "barcode": "uuid"
}
```

**Response:**
```json
{
  "barcode": "uuid",
  "status": "string"
}
```

### Blendbalance Operations

#### GET `/app/blendbalance`
Retrieves blendbalance items with dynamically calculated weight.

**Note:** Previously filtered incomplete items, now returns all items with calculated weight from records.

**Query Parameters (recommended filters):**
- `created_ts`: Creation timestamp
- `item_code`: Item code

**Response (without parameters):**
```json
[
  {
    "item_code": "string",
    "created_ts": "bigint",
    "blend_code": "string",
    "weight": "number",
    "transfer_id": "string"
  }
]
```

**Response (with parameters):**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "blend_code": "string",
  "weight": "number",
  "transfer_id": "string",
  "record_list": [
    {
      "id": "integer",
      "received_ts": "bigint",
      "store_location": "string",
      "bag_weight": "number",
      "gross_weight": "number",
      "barcode": "uuid"
    }
  ]
}
```

#### POST `/app/blendbalance/record`
Creates a new blendbalance record.

**Request Body:**
```json
{
  "item_code": "string",
  "created_ts": "string",
  "store_location": "string",
  "gross_weight": "number",
  "bag_weight": "number"
}
```

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "string",
  "id": "integer",
  "received_ts": "bigint",
  "barcode": "uuid"
}
```

#### PUT `/app/blendbalance/record`
Updates blendbalance record status with conditional logic.

**Request Body:**
```json
{
  "barcode": "uuid",
  "reduced_by": "number (optional)"
}
```

**Status Logic:**
- **Without `reduced_by`**: Status becomes `"DISPATCHED"`
- **With `reduced_by`**: Status becomes `"PROCESSED"` if remaining quantity equals 0, otherwise `"IN_PROCESS"`

**Response:**
```json
{
  "barcode": "uuid",
  "status": "string"
}
```

### Utility Operations

#### GET `/app/location`
Retrieves storage locations.

**Query Parameters:**
- `herbline_section`: Filter by herbline section (optional, boolean)

**Response:**
```json
[
  {
    "store_location": "string",
    "herbline_section": "boolean"
  }
]
```

#### GET `/app/scan`
Scans barcode to retrieve comprehensive record information. This endpoint uses the `scan_record()` database function which intelligently joins data from the specified record table with its parent table (e.g., tealine, blendsheet, etc.) and any related shipment allocation data to provide a complete view of the record.

**Query Parameters:**
- `table_name`: Table to scan (tealine, blendsheet, flavorsheet, herbline, blendbalance)
- `barcode`: UUID barcode to scan

**Response:**
```json
{
  "item_code": "string",
  "created_ts": "bigint",
  "received_ts": "bigint",
  "store_location": "string",
  "bag_weight": "number",
  "gross_weight": "number",
  "barcode": "uuid",
  "status": "string",
  "remaining": "number",
  "... additional fields from parent table based on `table_name` value"
}
```

### Shipment Operations

#### GET `/app/shipment`
**Integration:** AWS-Manager `/order` endpoint

Retrieves available shipments that are ready for processing (filtered from AWS-Manager order system).

**Response:**
```json
[
  {
    "request_code": "bigint",
    "order_code": "string",
    "product_name": "string",
    "requirement": "number",
    "comments": "string",
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
```

**Note:** Returns only shipments with ORDER_READY status that are not TRADER_BLOCKED or SHIPMENT_DISPATCHED. Integrates with AWS-Manager order system for shipment availability.

#### PUT `/app/shipment`
**Integration:** AWS-Manager `/order/shipment` endpoint

Updates shipment quantity in the AWS-Manager order system.

**Request Body:**
```json
{
  "request_code": "bigint",
  "shipment_code": "integer",
  "quantity": "number"
}
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

#### POST `/app/shipment/event`
**Integration:** AWS-Manager `/order/shipment/event` endpoint

Creates shipment dispatch events in the AWS-Manager order system.

**Request Body:**
```json
[
  {
    "request_code": "bigint",
    "shipment_code": "integer",
    "status": "SHIPMENT_DISPATCHED"
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

#### GET `/app/shipment/record`
Retrieves shipment records for specific request and shipment codes. This endpoint uses the `scan_record()` function to provide enriched data by joining shipment allocation tables with their corresponding record and parent tables.

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
    "... additional fields from parent table based on `table` value"
  }
]
```

#### GET `/app/parcel/record`
Retrieves parcel records for specific schedule code from both tealine and blendsheet allocate parcel tables.

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

---

## Data Types & Validation

### Common Data Types
- `bigint`: Unix timestamp in milliseconds
- `uuid`: UUID v4 format
- `number`: Decimal numbers with precision
- `integer`: Whole numbers
- `string`: Text fields

### Status Values
- **Record Status**: ACCEPTED, IN_PROCESS, PROCESSED, DISPATCHED
- **Trader Status**: TRADER_PENDING, TRADER_REQUESTED, TRADER_ELEVATED, TRADER_ALLOWED, TRADER_BLOCKED

### Validation Rules
- All fields are required unless marked as optional
- Weight values must be positive numbers
- Barcode values are auto-generated UUIDs
- Timestamps are auto-generated in milliseconds
- Status values must match predefined enums

---

## External Dependencies

### AWS API Gateway Integration
The application integrates with an external shipment management system via AWS API Gateway:
- **Environment Variable**: `SHIPMENT_API` (base URL)
- **Authentication**: AWS SigV4 signing
- **Region**: us-east-1
- **Service**: execute-api

### Database Connection
- **Database**: PostgreSQL
- **Schema**: tea_management
- **Connection**: Standard pg pool connection

---

## Database Function: scan_record()

The `scan_record(table_name, barcode)` function is a sophisticated PostgreSQL function that:

1. **Validates table existence**: Checks if the specified `{table_name}_record` table exists
2. **Handles comprehensive allocations**: Automatically discovers and processes all allocation tables for the material type, including shipment and blendsheet allocations
3. **Joins with parent tables**: Automatically determines and joins with the appropriate parent table (e.g., `tealine`, `blendsheet`) or batch table (e.g., `tealine_batch`, `blendsheet_batch`) based on the table structure
4. **Returns enriched data**: Returns the first row containing complete record information including data from parent tables and calculated fields

⚠️ **IMPORTANT ALLOCATION BEHAVIOR**: The `scan_record()` function uses **dynamic allocation discovery** to find all relevant allocation tables (`{table_name}_allocate_*`) and includes them in remaining quantity calculations, **EXCEPT** parcel allocation tables which are excluded to prevent double-counting.

**Allocation Flow Explanation:**
The allocation system follows both hierarchical and parallel allocation flows:
```
tealine_record → tealine_allocate_shipment → tealine_allocate_parcel (refinement)
              └→ tealine_allocate_blendsheet (parallel processing)
```

- **Shipment Allocation**: Records are allocated to shipments from the original inventory
- **Blendsheet Allocation**: Records are allocated to blendsheet batches for ingredient processing (parallel to shipments)
- **Parcel Allocation**: Parcels are created from already-shipped inventory (refinements of shipment allocations)
- **Design Rationale**: Both shipment and blendsheet allocations represent actual material consumption and are included in remaining calculations, while parcel allocations are excluded to prevent double-counting

**Why Not Use `shipment_allocate_parcel`?**
A unified `shipment_allocate_parcel` table cannot be used because tealine and blendsheet can generate identical barcodes for different bags, which would violate primary key constraints.

**Remaining Quantity Calculation:**
- `scan_record()` calculates: `original_quantity - (shipment_allocations + blendsheet_allocations + ...other_allocations)`
- Parcel allocations are excluded to avoid double-counting since they represent subdivisions of already-allocated shipment quantities
- *other_allocations* represents future allocation types that may be added

This function is used by both `/app/scan` and `/app/shipment/record` endpoints to provide comprehensive record views.

---

## Notes

1. All timestamps are stored as Unix timestamps in milliseconds
2. Barcodes are automatically generated as UUIDs for all record entries
3. The API uses database triggers for validation and automatic field population
4. External shipment API calls are authenticated using AWS SigV4 signatures
5. CORS is enabled for all origins with specific allowed methods
6. Error responses include descriptive messages for validation failures
7. Shipment operations integrate with AWS-Manager for order management workflow