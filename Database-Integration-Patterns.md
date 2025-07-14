# Database Integration Patterns

## Overview

The MTP Platform implements a **distributed database architecture** where AWS-Manager and Central-Handler operate separate PostgreSQL instances with the same database name (`tea_management`). This document details the integration patterns, data flow mechanisms, and synchronization strategies that maintain consistency across the distributed system.

## Architecture Patterns

### 1. Separated Database Pattern

```
┌─────────────────────────┐                    ┌─────────────────────────┐
│     AWS-Manager         │                    │    Central-Handler      │
│   PostgreSQL Instance   │                    │   PostgreSQL Instance   │
│                         │                    │                         │
│  tea_management         │◄──── API Calls ───►│  tea_management         │
│  ├── admin             │                    │  └── public             │
│  └── public            │                    │      (processing)       │
│      (orders)          │                    │                         │
└─────────────────────────┘                    └─────────────────────────┘
```

**Benefits**:
- Independent scaling and performance optimization
- Isolated failure domains
- Separate backup and maintenance windows
- Technology stack flexibility

**Challenges**:
- Data consistency across systems
- Network dependency for operations
- Complex monitoring and debugging

## Integration Mechanisms

### 1. API-Based Integration

#### AWS-Manager → Central-Handler

**Proxy Pattern (`/central/*`)**
```
Client Request → AWS API Gateway → Lambda (order-handler) → Central-Handler
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

**Environment Configuration**:
- `SHIPMENT_API`: Base URL for AWS-Manager
- AWS credentials for SigV4 signing
- Region: us-east-1, Service: execute-api

### 2. Data Synchronization Patterns

#### Order-to-Processing Flow

```
1. Order Creation (AWS-Manager)
   ├── order_plan
   ├── order_request  
   └── order_shipment
           ↓ (API Call)
2. Processing Allocation (Central-Handler)
   ├── *_allocate_shipment
   └── *_allocate_parcel
           ↓ (API Callback)
3. Status Updates (AWS-Manager)
   └── order_shipment_event
```

#### Processing-to-Shipment Flow

```
1. Material Processing (Central-Handler)
   ├── tealine_record / blendsheet_record
   ├── batch creation
   └── allocation to shipments
           ↓ (API Call)
2. Shipment Updates (AWS-Manager)
   ├── quantity adjustments
   └── status events
           ↓ (API Callback)  
3. Completion Tracking (Central-Handler)
   └── allocation status updates
```

## Cross-System Data Relationships

### 1. Shared Identifiers

| Identifier Type | AWS-Manager | Central-Handler | Usage |
|----------------|-------------|-----------------|-------|
| **Request Code** | `order_request.request_code` | `*_allocate_shipment.request_code` | Order tracking |
| **Shipment Code** | `order_shipment.shipment_code` | `*_allocate_shipment.shipment_code` | Shipment linking |
| **Schedule Code** | `order_schedule.schedule_code` | `*_allocate_parcel.schedule_code` | Production scheduling |
| **Barcode UUID** | Referenced via API | `*_record.barcode` | Record identification |

### 2. Status Synchronization

#### Order Workflow States

**AWS-Manager States**:
```
APPROVAL_REQUESTED → APPROVAL_ALLOWED → ORDER_REQUESTED → 
SHIPMENT_ACCEPTED → SHIPMENT_DISPATCHED
```

**Central-Handler Processing States**:
```
ACCEPTED → IN_PROCESS → PROCESSED → DISPATCHED
```

**Synchronization Points**:
- `ORDER_REQUESTED` triggers allocation eligibility
- `SHIPMENT_ACCEPTED` enables processing start
- `PROCESSED` status updates shipment quantities
- `DISPATCHED` triggers shipment completion

#### Trader Approval Workflow

**Central-Handler Trader States**:
```
TRADER_PENDING → TRADER_REQUESTED → TRADER_ELEVATED → 
TRADER_ALLOWED / TRADER_BLOCKED
```

**AWS-Manager Integration**:
- `TRADER_ALLOWED` enables shipment processing
- `TRADER_BLOCKED` prevents shipment allocation
- Status changes sync via trader-handler Lambda

## Allocation Integration Patterns

### 1. Supported Allocation Types

**Allocation Support**:
- ✅ **Tealine**: Shipment + Parcel + Blendsheet allocation
- ✅ **Blendsheet**: Shipment + Parcel allocation
- ✅ **Blendbalance**: Blendsheet allocation only
- ❌ **Flavorsheet**: Record tracking only
- ❌ **Herbline**: Record tracking only

### 2. Allocation Data Flow

#### Shipment Allocation Process

```
1. Order Shipment Created (AWS-Manager)
   └── order_shipment (request_code, shipment_code, quantity)
           ↓ (Available via API)
2. Processing Record Created (Central-Handler)  
   └── tealine_record / blendsheet_record (barcode, remaining)
           ↓ (Manual/System Allocation)
3. Shipment Allocation (Central-Handler)
   └── *_allocate_shipment (barcode, request_code, shipment_code, amount)
           ↓ (API Update)
4. Quantity Update (AWS-Manager)
   └── order_shipment.quantity -= allocated_amount
```

#### Parcel Allocation Process

```
1. Production Schedule (AWS-Manager)
   └── order_schedule (schedule_code, order_code, quantity)
           ↓ (Production Planning)
2. Parcel Allocation (Central-Handler)
   └── *_allocate_parcel (barcode, request_code, shipment_code, schedule_code)
           ↓ (Reduces Shipment Allocation)
3. Shipment Allocation Update
   └── *_allocate_shipment.remaining -= parcel_amount
```

## API Integration Endpoints

### 1. Central-Handler → AWS-Manager

| Endpoint | Purpose | Data Flow |
|----------|---------|-----------|
| `GET /order` | Retrieve available shipments | Shipment data for allocation |
| `PUT /order/shipment` | Update shipment quantities | Allocation quantity changes |
| `POST /order/shipment/event` | Create status events | Processing status updates |

### 2. AWS-Manager → Central-Handler

| Endpoint Pattern | Purpose | Data Flow |
|------------------|---------|-----------|
| `GET /central/shipment` | Retrieve shipment records | Record data with allocations |
| `PUT /central/shipment` | Update shipment allocations | Allocation modifications |
| `GET /central/scan` | Scan record details | Comprehensive record info |

## Data Consistency Strategies

### 1. Transactional Boundaries

**Within Each System**:
- Database transactions ensure ACID properties
- Triggers maintain referential integrity
- Constraints prevent invalid states

**Across Systems**:
- API calls provide eventual consistency
- Retry mechanisms handle temporary failures
- Compensation logic for partial failures

### 2. Conflict Resolution

**Optimistic Concurrency**:
- Timestamp-based change detection
- Last-write-wins for status updates
- Allocation conflicts resolved by quantities

**Error Handling**:
- API timeouts with exponential backoff
- Circuit breaker patterns for resilience
- Dead letter queues for failed operations

### 3. Data Reconciliation

**Scheduled Reconciliation**:
- Daily allocation vs shipment quantity checks
- Status consistency validation
- Orphaned record detection and cleanup

**Real-time Validation**:
- Allocation constraint checking
- Quantity limit enforcement
- Status transition validation

## Performance Optimization Patterns

### 1. Caching Strategies

**Local Caching**:
- Frequently accessed lookup data
- User authentication tokens
- Location and status mappings

**Distributed Caching**:
- Cross-system identifier mappings
- Aggregated allocation summaries
- Real-time status dashboards

### 2. Batch Processing

**Bulk Operations**:
- Batch allocation updates
- Mass status synchronization
- Scheduled reconciliation jobs

**Event Queuing**:
- Asynchronous status updates
- Deferred consistency operations
- Peak load smoothing

## Monitoring & Observability

### 1. Cross-System Tracing

**Correlation Identifiers**:
- Request IDs across API calls
- Barcode tracking through systems
- User session correlation

**Distributed Tracing**:
- API call timing and latency
- Database operation performance
- Error propagation tracking

### 2. Health Monitoring

**System Health Checks**:
- Database connectivity status
- API endpoint availability
- Integration point functionality

**Business Logic Monitoring**:
- Allocation consistency checks
- Status synchronization lag
- Data quality metrics

## Security Considerations

### 1. Inter-System Authentication

**AWS SigV4 Signing**:
- Request authentication and integrity
- IAM role-based access control
- Temporary credential rotation

**API Gateway Security**:
- CORS configuration for browser access
- Rate limiting and throttling
- Request/response validation

### 2. Data Protection

**Encryption in Transit**:
- HTTPS for all API communications
- TLS certificate validation
- Secure credential transmission

**Database Security**:
- Separate credential management
- Network isolation where possible
- Audit logging for sensitive operations

## Best Practices

### 1. API Design

**Idempotency**:
- Safe retry of failed operations
- Duplicate request handling
- Consistent response behavior

**Versioning**:
- Backward compatibility maintenance
- Gradual deprecation strategies
- Version-specific error handling

### 2. Error Handling

**Graceful Degradation**:
- Fallback to cached data
- Reduced functionality modes
- User notification strategies

**Recovery Procedures**:
- Automatic retry with backoff
- Manual intervention workflows
- Data repair and reconciliation

### 3. Testing Strategies

**Integration Testing**:
- End-to-end workflow validation
- Cross-system data consistency
- Failure scenario simulation

**Performance Testing**:
- Load testing under realistic conditions
- Latency and throughput measurement
- Resource utilization monitoring

## Future Evolution Patterns

### 1. Event-Driven Architecture

**Potential Migration**:
- Message queues for async operations
- Event sourcing for audit trails
- CQRS for read/write separation

### 2. Microservices Considerations

**Service Boundaries**:
- Domain-driven design principles
- API gateway consolidation
- Service mesh implementation

### 3. Cloud-Native Patterns

**Containerization**:
- Docker containers for deployment
- Kubernetes orchestration
- Service discovery mechanisms

---

## Related Documentation

- [Database Overview](./Database-Overview.md)
- [Database Schema - AWS Manager](./Database-Schema-AWS-Manager.md)
- [Database Schema - Central Handler](./Database-Schema-Central-Handler.md)
- [API Documentation - AWS Manager](./API-Documentation-AWS-Manager.md)
- [API Documentation - Central Handler](./API-Documentation-Central-Handler.md)