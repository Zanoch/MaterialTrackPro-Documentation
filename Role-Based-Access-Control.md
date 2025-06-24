# Role-Based Access Control System - AWS-Manager

## Overview

The AWS-Manager implements a comprehensive role-based access control (RBAC) system using AWS Cognito User Groups and IAM policies. This system provides secure, role-specific access to tea processing operations across different locations and workflow stages.

## Architecture

### Two-Tier Permission System

**Admin Level Roles (3):**
- Full access to `/admin/*` endpoints
- Location or function-specific administrative capabilities
- Can access all operational endpoints

**User Level Roles (7):**
- Denied access to `/admin/*` endpoints via IAM policy
- Role-specific operational access
- Location or function-specific operational capabilities

### Implementation Components

1. **AWS Cognito User Pool**: Authentication and group membership
2. **AWS Cognito Identity Pool**: IAM role mapping
3. **IAM Policies**: Endpoint access control
4. **API Gateway**: Secure endpoint routing
5. **Frontend Role Routing**: Dynamic page loading based on user groups

---

## Complete Role Documentation

### **Admin Level Roles**

#### 1. MASTER-ADMIN
**Purpose**: System administration and user management

**Primary Workflow APIs:**
- `GET /admin/user` - Retrieve all system users
- `GET /admin/group` - Retrieve available user groups
- `POST /admin/user` - Create new users with group assignment
- `DELETE /admin/user` - Remove users from system

**Functionality**: Complete user management system including user creation, deletion, and group assignment

**UI Component**: `MasterAdmin.jsx`

---

#### 2. GRANDPASS-ADMIN
**Purpose**: Production planning and order plan management

**Primary Workflow APIs:**
- `GET /order/plan` - Retrieve production plans
- `POST /order/plan` - Upload new production plans (Excel files)
- `DELETE /order/plan` - Remove production plans

**Functionality**: Production plan management including Excel file upload, plan review, and plan deletion

**UI Component**: `GrandpassAdmin.jsx`

---

#### 3. SEEDUWA-ADMIN
**Purpose**: Shipment documentation and Central-Handler historical data access

**Primary Workflow APIs:**
- `GET /order` - View orders and shipments
- `GET /order/record/shipment` - Retrieve shipment records (frontend generates documents from this data)

**Historical Data APIs:**
- `GET /central/${selectedOption}` - Access Central-Handler historical records (tealine, blendsheet, etc.)

**Functionality**: Retrieve shipment data for frontend document generation + access to Central-Handler historical data for audit and review purposes

**UI Component**: `SeeduwaAdmin.jsx`

---

### **User Level Roles**

#### 4. GRANDPASS-OFFICER
**Purpose**: Order request initiation and production scheduling

**Primary Workflow APIs:**
- `GET /order` - View current orders
- `GET /order/plan` - View production plans
- `POST /order` - Create new order requests
- `POST /order/schedule` - Upload production schedules for filling operations

**Historical Data APIs:**
- `GET /order` (with historical filtering) - Order History tab functionality

**Functionality**: Create order requests, upload production schedules for DEV-MACHINE operations + AWS-Manager historical data access

**UI Component**: `GrandpassOfficer.jsx`

---

#### 5. GRANDPASS-SUPERVISOR
**Purpose**: Special request approval authority

**Primary Workflow APIs:**
- `GET /order` - View orders requiring approval
- `POST /order/shipment/event` - Create approval/denial events for special requests

**Functionality**: Handle approval/denial of special requests only (not standard workflow requests)

**UI Component**: `GrandpassSupervisor.jsx`

---

#### 6. SEEDUWA-OFFICER
**Purpose**: Order fulfillment status updates and trader request creation

**Primary Workflow APIs:**
- `GET /order` - View orders for status updates
- `GET /trader` - View trader requests
- `PUT /order/shipment` - Update shipment quantities
- `POST /order/shipment/event` - Create status update events (ready-to-ship or not)
- `POST /trader` - Create new trader requests

**Historical Data APIs:**
- `GET /order` (with historical filtering) - Order History tab functionality

**Functionality**: Update order requests with ready-to-ship status, create trader requests + AWS-Manager historical data access

**UI Component**: `SeeduwaOfficer.jsx`

---

#### 7. GRANDPASS-GATEPASS
**Purpose**: Shipment receiving and verification operations

**Primary Workflow APIs:**
- `GET /order` - View incoming shipments
- `POST /order/shipment/event` - Create shipment acceptance events
- `GET /order/record/shipment` - Retrieve shipment details for verification

**Historical Data APIs:**
- `GET /order` (with historical filtering) - Order History view mode functionality

**Functionality**: Accept shipments and verify using retrieved shipment records + AWS-Manager historical data access

**UI Component**: `GrandpassGatepass.jsx`

---

#### 8. GRANDPASS-DEV-MACHINE
**Purpose**: Tea box filling operations and production record management

**Primary Workflow APIs:**
- `GET /order/schedule` - Retrieve production schedules
- `GET /order/record/shipment` - Retrieve shipment records for filling operations
- `POST /order/record/parcel` - Create parcel allocation records during filling
- `PUT /order/schedule` - Update production schedules to completed status

**Functionality**: Fill tea boxes using retrieved production schedules, create parcel allocation records using retrieved shipment data

**UI Component**: `GrandpassDevMachine.jsx`

---

#### 9. TRADER-OFFICER
**Purpose**: Blendsheet batch processing and trader request management

**Primary Workflow APIs:**
- `GET /trader` - Retrieve blendsheet batches for review
- `PUT /trader` - Update trader request status, escalate complex cases

**Functionality**: Process trader requests, update blendsheet batch status, escalate complex requests to TRADER-SUPERVISOR

**UI Component**: `TraderOfficer.jsx`

---

#### 10. TRADER-SUPERVISOR
**Purpose**: Elevated trader request approval and oversight

**Primary Workflow APIs:**
- `GET /trader` - Retrieve elevated trader requests for review
- `PUT /trader` - Update status of requests elevated by TRADER-OFFICER

**Functionality**: Handle complex trader operations, approve/deny elevated trader requests, senior-level blendsheet batch decisions

**UI Component**: `TraderSupervisor.jsx`

---

## Tea Processing Workflow Integration

### Complete Workflow Chain:
1. **GRANDPASS-ADMIN** → Upload production plans (`POST /order/plan`)
2. **GRANDPASS-OFFICER** → Create order requests (`POST /order`) + upload production schedules (`POST /order/schedule`)
3. **GRANDPASS-SUPERVISOR** → Create approval/denial events for special requests (`POST /order/shipment/event`)
4. **SEEDUWA-OFFICER** → Create status update events (`POST /order/shipment/event`) + create trader requests (`POST /trader`)
5. **TRADER-OFFICER** → Update trader request status (`PUT /trader`), escalate complex cases
6. **TRADER-SUPERVISOR** → Update status of elevated requests (`PUT /trader`)
7. **[External Shipping Process]** → Physical shipment (separate project)
8. **SEEDUWA-ADMIN** → Retrieve shipment records (`GET /order/record/shipment`) for frontend document generation
9. **GRANDPASS-GATEPASS** → Create shipment acceptance events (`POST /order/shipment/event`), retrieve records for verification (`GET /order/record/shipment`)
10. **GRANDPASS-DEV-MACHINE** → Create parcel allocations (`POST /order/record/parcel`), update schedule completion (`PUT /order/schedule`)

---

## Historical Data Access

### AWS-Manager Historical Data Access
**Purpose**: Review past operations, audit trail, operational history

**Roles with Access:**
- **GRANDPASS-OFFICER**: Order History tab - past order requests and progression
- **SEEDUWA-OFFICER**: Order History tab - historical order status updates and trader activities
- **GRANDPASS-GATEPASS**: Order History view mode - past shipment receiving operations

**Implementation**: UI tabs/modes that filter historical data from standard `/order` endpoints

### Central-Handler Historical Data Access
**Purpose**: Access to Central-Handler system historical records for audit and analysis

**Roles with Access:**
- **SEEDUWA-ADMIN**: `/central/${selectedOption}` - access to tealine, blendsheet, flavorsheet, herbline, and other Central-Handler historical records

**Implementation**: Dynamic endpoint selection for different Central-Handler record types

---

## API Access Control Matrix

| Role | GET | POST | PUT | DELETE | Admin Access | Historical Access |
|------|-----|------|-----|--------|--------------|-------------------|
| MASTER-ADMIN | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| GRANDPASS-ADMIN | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| SEEDUWA-ADMIN | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ (Central-Handler) |
| GRANDPASS-OFFICER | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (AWS-Manager) |
| GRANDPASS-SUPERVISOR | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| SEEDUWA-OFFICER | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ (AWS-Manager) |
| GRANDPASS-GATEPASS | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (AWS-Manager) |
| GRANDPASS-DEV-MACHINE | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| TRADER-OFFICER | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| TRADER-SUPERVISOR | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |

---

## Security Implementation

### AWS Cognito Integration
- **User Pool**: Manages user authentication and group membership
- **Identity Pool**: Maps Cognito groups to IAM roles
- **JWT Tokens**: Contain group information for frontend routing

### IAM Policy Enforcement
```typescript
// Admin access only - denies user-level roles
Object.entries(roleLevels)
  .filter(([level]) => level !== "admin")
  .forEach(([_, role]) => {
    role.addToPolicy(
      new iam.PolicyStatement({
        actions: ["execute-api:Invoke"],
        effect: iam.Effect.DENY,
        resources: [api.arnForExecuteApi("*", "/admin/*")],
      })
    );
  });
```

### Frontend Role Routing
```javascript
// Dynamic component loading based on user group
fetchAuthSession()
  .then(({ tokens }) =>
    import(
      `./pages/${tokens.accessToken.payload["cognito:groups"][0]
        .split("-")
        .map((word) => word.replace(/(?<=[A-Z]).+/, (match) => match.toLowerCase()))
        .join("")}.jsx`
    )
  )
```

---

## Role Management

### User Creation Process
1. **MASTER-ADMIN** creates user via `POST /admin/user`
2. User assigned to specific Cognito group
3. IAM role automatically mapped based on group level
4. Frontend routes user to appropriate interface

### Group Assignment Rules
- **One group per user**: Each user belongs to exactly one Cognito group
- **Role-based routing**: Group name determines UI component
- **Permission inheritance**: Group level (admin/user) determines IAM permissions

### Available Groups
Retrieved via `GET /admin/group`:
- MASTER-ADMIN
- GRANDPASS-ADMIN, GRANDPASS-OFFICER, GRANDPASS-SUPERVISOR, GRANDPASS-GATEPASS, GRANDPASS-DEV-MACHINE
- SEEDUWA-ADMIN, SEEDUWA-OFFICER
- TRADER-OFFICER, TRADER-SUPERVISOR

---

## Integration Points

### Connected MTP Platform Projects
- **AWS-Manager**: Serverless backend handling order management and user administration
- **Central-Handler**: Node.js server managing tea processing operations
- **[Shipping Process Project]**: Separate project handling physical shipment operations (currently unavailable)

### Central-Handler System Integration
**Workflow Integration** (Primary Functions):
- **SEEDUWA-ADMIN**: `/order/record/shipment` for retrieving shipment data
- **GRANDPASS-GATEPASS**: `/order/record/shipment` for retrieving shipment verification data
- **GRANDPASS-DEV-MACHINE**: `/order/record/shipment` and `/order/record/parcel` for retrieving/creating production records

**Historical Integration** (Secondary Functions):
- **SEEDUWA-ADMIN**: `/central/${selectedOption}` for historical record access

---

## Administration Guide

### Adding New Users
1. Use MASTER-ADMIN role to access user management
2. Select appropriate group based on user's role and location
3. System automatically assigns correct permissions and UI access

### Troubleshooting Access Issues
1. **403 Errors**: Check user's group assignment and IAM role mapping
2. **Wrong UI**: Verify Cognito group name matches expected format
3. **Missing Endpoints**: Confirm role has appropriate HTTP method permissions

### Security Best Practices
1. **Principle of Least Privilege**: Users only have access to endpoints needed for their role
2. **Location-Based Access**: Users typically access operations for their specific location
3. **Approval Hierarchies**: Critical operations require supervisor-level approval
4. **Audit Trail**: Historical data access provides operational audit capabilities

---

## Notes

1. **Admin vs User Levels**: Determined by IAM policy configuration in CDK stack
2. **Role Specialization**: Each role has specific, non-overlapping operational responsibilities
3. **Workflow Dependencies**: Roles interact through defined handoff points in tea processing workflow
4. **Historical Access**: Separate from operational workflow, used for audit and review
5. **System Integration**: Seamless integration between AWS-Manager and Central-Handler through specific API endpoints
6. **Frontend Processing**: Document generation and complex data processing happens within the Admin-Panel React application using retrieved API data