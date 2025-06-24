# MTP Platform Database Overview

## Executive Summary

The MTP (Tea Management) Platform operates on a **distributed database architecture** with two separate PostgreSQL database servers, each running their own instance of a database named `tea_management`. The systems achieve integration through **inter-connected API routes** rather than shared database access, providing system independence while maintaining data consistency.

## Architecture Overview

### Distributed Database Design

```
┌─────────────────────────┐    API Integration    ┌─────────────────────────┐
│     AWS-Manager         │ ←──────────────────→ │    Central-Handler      │
│   Database Server       │                      │   Database Server       │
│                         │                      │                         │
│  tea_management (DB)    │                      │  tea_management (DB)    │
│  ├── admin schema       │                      │  └── public schema      │
│  └── public schema      │                      │      (processing)       │
│      (orders)           │                      │                         │
└─────────────────────────┘                      └─────────────────────────┘
```

### System Responsibilities

| System | Database Focus | Primary Functions |
|--------|----------------|-------------------|
| **AWS-Manager** | Order Management & User Admin | Order planning, requests, shipments, scheduling, user management |
| **Central-Handler** | Tea Processing & Inventory | Raw material processing, blending, batching, allocation tracking, ingredient traceability |

## Database Specifications

### Common Characteristics
- **Database Engine**: PostgreSQL 14+
- **Database Name**: `tea_management` (both instances)
- **Character Encoding**: UTF-8
- **Locale**: en_US.UTF-8

### AWS-Manager Database
- **Primary Schema**: `public` (order management)
- **Secondary Schema**: `admin` (user management) 
- **Integration**: AWS Cognito, API Gateway, Lambda functions
- **Key Features**: Auto-incrementing sequences, event tracking, foreign key relationships

### Central-Handler Database  
- **Primary Schema**: `public` (processing operations)
- **Integration**: Direct PostgreSQL access, AWS API calls
- **Key Features**: Complex triggers, validation functions, allocation tracking, storage location management

## Integration Architecture

### API-Based Data Exchange

**AWS-Manager → Central-Handler**
- Proxy routes (`/central/*` → `/app/*`)
- Specialized Lambda functions for specific operations
- Real-time processing requests

**Central-Handler → AWS-Manager**  
- Shipment API integration via environment variable `SHIPMENT_API`
- AWS SigV4 authenticated API calls
- Status synchronization and allocation updates

### Data Consistency Mechanisms

1. **API-Level Transactions**: Synchronous calls for critical operations
2. **Cross-System Identifiers**: Request codes, shipment codes, barcode UUIDs
3. **Status Synchronization**: Coordinated workflow states
4. **Allocation Tracking**: Central-Handler maintains shipment assignments

## Key Business Workflows

### Order-to-Production Flow
```
Order Planning (AWS) → Order Requests (AWS) → Shipment Allocation (AWS) 
       ↓
Processing Records (Central) → Batch Management (Central) → Allocation (Central)
       ↓
Status Updates (Central) → Shipment Events (AWS) → Completion (AWS)
```

### Material Processing Flow
```
Raw Materials → Processing Records → Batch Creation → Quality Control → Allocation
                                            ↓
                                   Blendsheet Ingredient Allocation
                                   (Traceability Support)
```

## Advantages of Distributed Architecture

### 1. **System Independence**
- Independent scaling and performance optimization
- Isolated failure domains and maintenance windows
- Technology stack flexibility per system requirements

### 2. **Security & Compliance**
- Separate credential management and access control
- Storage segregation for regulatory compliance (herbline restrictions)
- Network-level isolation capabilities

### 3. **Performance Benefits**
- Reduced database contention and query optimization per use case
- Independent backup strategies and recovery procedures
- Optimized indexing for specific workloads

### 4. **Development & Deployment**
- Independent development cycles and testing
- Separate deployment pipelines and rollback capabilities
- Team autonomy and reduced coordination overhead

## Challenges & Mitigation Strategies

### 1. **Data Consistency**
- **Challenge**: Maintaining consistency across separate databases
- **Mitigation**: API-based synchronization with transactional boundaries, retry mechanisms

### 2. **Network Dependencies**
- **Challenge**: Cross-system operations depend on network connectivity
- **Mitigation**: Circuit breakers, timeout handling, graceful degradation

### 3. **Monitoring Complexity**
- **Challenge**: Tracking operations across multiple systems
- **Mitigation**: Distributed tracing, correlation IDs, centralized logging

## Best Practices

### 1. **API Design**
- Use idempotent operations where possible
- Implement proper error handling and retry logic
- Version APIs for backward compatibility

### 2. **Data Management**
- Coordinate backup schedules across systems
- Implement data reconciliation procedures
- Monitor cross-system data consistency

### 3. **Performance Optimization**
- Cache frequently accessed cross-system data
- Use asynchronous processing where appropriate
- Monitor and optimize API call patterns

## Future Considerations

### 1. **Scalability Enhancements**
- Consider read replicas for reporting workloads
- Evaluate message queues for asynchronous integration
- Plan for horizontal scaling strategies

### 2. **Observability Improvements**
- Implement distributed tracing across API calls
- Add comprehensive health checks and monitoring
- Create cross-system performance dashboards

### 3. **Integration Evolution**
- Evaluate event-driven architecture patterns
- Consider database replication for read operations
- Plan for eventual consistency patterns where appropriate

---

## Related Documentation

- [Database Schema - AWS Manager](./Database-Schema-AWS-Manager.md)
- [Database Schema - Central Handler](./Database-Schema-Central-Handler.md)
- [Database Integration Patterns](./Database-Integration-Patterns.md)
- [Comprehensive Database Documentation](./Comprehensive-Database-Documentation.md)