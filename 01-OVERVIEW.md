# Central Authentication & Authorization System - Local Government Services Platform

## 📋 Table of Contents
1. [Executive Summary](#executive-summary)
2. [System Vision & Goals](#system-vision--goals)
3. [Core Concepts](#core-concepts)
4. [Architecture Overview](#architecture-overview)
5. [Key Features](#key-features)
6. [Design Principles](#design-principles)
7. [Use Cases](#use-cases)

---

## Executive Summary

This document describes a **Central Authentication and Authorization System** designed to serve as the **core identity and access management (IAM) hub** for a **Local Government Services Platform**. The system provides a single source of truth for:

- **Citizen Identity Management** - Citizen profiles, credentials, verification
- **Authentication** - Token-based auth, OTP verification, session management
- **Authorization** - Role-based (RBAC), Permission-based, Department-scoped access control
- **Service-to-Service Auth** - Secure communication between government microservices
- **Multi-Department Support** - Optional department-scoped permissions for government employees

### Target Users

| User Type | Description | Example Roles |
|-----------|-------------|---------------|
| **Citizens** | Residents accessing government services | Resident, Business Owner, Property Owner |
| **Government Employees** | Department staff processing requests | Clerk, Inspector, Case Worker |
| **Department Managers** | Staff with elevated department access | Supervisor, Department Head |
| **System Administrators** | IT staff managing the platform | Super Admin, IT Admin |

The design is **framework-agnostic** and can be implemented in any language (PHP/Laravel, Python/Django, Node.js, Go, Java/Spring, etc.) while maintaining the same database schema, API contracts, and architectural patterns.

---

## System Vision & Goals

### Primary Goals

```
┌─────────────────────────────────────────────────────────────────────────┐
│           LOCAL GOVERNMENT SERVICES - AUTH SYSTEM VISION                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                 │
│  │  Citizen    │    │   Citizen   │    │ Gov Staff   │                 │
│  │  Mobile App │    │  Web Portal │    │   Portal    │                 │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                 │
│         │                  │                  │                         │
│         └──────────────────┼──────────────────┘                         │
│                            ▼                                            │
│              ┌─────────────────────────────┐                           │
│              │       API GATEWAY           │                           │
│              │     (Load Balancer)         │                           │
│              └─────────────┬───────────────┘                           │
│                            │                                            │
│         ┌──────────────────┼──────────────────┐                        │
│         │                  │                  │                         │
│         ▼                  ▼                  ▼                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                 │
│  │  Permits &  │    │   Revenue   │    │   Public    │                 │
│  │  Licensing  │    │   & Taxes   │    │   Works     │                 │
│  │  (Python)   │    │   (Java)    │    │  (Node.js)  │                 │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                 │
│         │                  │                  │                         │
│         └──────────────────┼──────────────────┘                         │
│                            │                                            │
│                            ▼                                            │
│         ┌──────────────────────────────────────┐                       │
│         │   ★ CENTRAL AUTH SYSTEM ★            │                       │
│         │                                      │                        │
│         │  • Citizen Identity                  │                        │
│         │  • Employee Management               │                        │
│         │  • Authentication                    │                        │
│         │  • Authorization                     │                        │
│         │  • Token Validation                  │                        │
│         │  • Permission Resolution             │                        │
│         │  • Service Client Auth               │                        │
│         │  • Department Hierarchy              │                        │
│         │  • Audit Trail                       │                        │
│         └──────────────────────────────────────┘                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Design Goals

| Goal | Description |
|------|-------------|
| **One Citizen, One Identity** | Single profile across all government services |
| **Stateless Authentication** | JWT/token-based auth enabling horizontal scaling |
| **Fine-grained Authorization** | Permission-level control with role inheritance |
| **Service Isolation** | Microservices authenticate via service tokens |
| **Complete Audit Trail** | Every access logged for compliance |
| **High Availability** | Designed for 99.9%+ uptime (critical for gov services) |
| **Framework Agnostic** | Can be implemented in any language/framework |
| **Citizen Privacy** | Data protection and compliance with regulations |

---

## Core Concepts

### 1. Authentication vs Authorization

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│   AUTHENTICATION (AuthN)          AUTHORIZATION (AuthZ)        │
│   ═══════════════════════         ═════════════════════        │
│                                                                │
│   "WHO are you?"                  "WHAT can you do?"           │
│                                                                │
│   • Login with credentials        • Permission checks          │
│   • National ID verification      • Role verification          │
│   • Token validation              • Access control             │
│   • OTP verification              • Department-level policies  │
│                                                                │
│   Citizens:                       Citizens:                    │
│   • Email + Password              • Submit applications        │
│   • Phone + OTP                   • View own records           │
│   • National ID verification      • Make payments              │
│                                                                │
│   Employees:                      Employees:                   │
│   • Gov email + Password          • Process applications       │
│   • 2FA required                  • Access citizen data        │
│                                   • Approve/Deny requests      │
│                                                                │
│   RESULT: Identity Token          RESULT: Access Decision      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2. Identity Hierarchy

```
                    ┌─────────────────┐
                    │   PERMISSION    │
                    │  (Atomic Unit)  │
                    │                 │
                    │ e.g., "view     │
                    │   tax records"  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
       ┌─────────────┐              ┌─────────────┐
       │    ROLE     │              │   DIRECT    │
       │             │              │ ASSIGNMENT  │
       │ Collection  │              │             │
       │ of Perms    │              │ User has    │
       │             │              │ permission  │
       └──────┬──────┘              │ directly    │
              │                     └──────┬──────┘
              │                            │
              └────────────┬───────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   CITIZEN   │
                    │  or STAFF   │
                    │             │
                    │ • Roles     │
                    │ • Direct    │
                    │   Perms     │
                    │ • Dept      │
                    │   Context   │
                    └─────────────┘
```

### 3. User Types in Government Context

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER TYPE MATRIX                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CITIZENS (External Users)                                      │
│  ═════════════════════════                                      │
│                                                                 │
│  • Primary users of the platform                                │
│  • Self-register via citizen portal                             │
│  • Verify identity via email/phone/national ID                  │
│  • Access their own data and applications                       │
│  • Cannot access other citizens' data                           │
│                                                                 │
│  Typical Permissions:                                           │
│  ├── submit applications                                        │
│  ├── view own records                                           │
│  ├── make payments                                              │
│  ├── track application status                                   │
│  └── update own profile                                         │
│                                                                 │
│                                                                 │
│  GOVERNMENT EMPLOYEES (Internal Users)                          │
│  ═════════════════════════════════════                          │
│                                                                 │
│  • Managed by HR / IT department                                │
│  • Assigned to specific department(s)                           │
│  • Role varies by department and position                       │
│  • Can access citizen data (based on role)                      │
│  • Subject to audit logging                                     │
│                                                                 │
│  Example Roles:                                                 │
│  ├── Clerk - Process applications                               │
│  ├── Inspector - Conduct field inspections                      │
│  ├── Case Worker - Manage citizen cases                         │
│  ├── Supervisor - Approve decisions, manage team                │
│  └── Department Head - Full department access                   │
│                                                                 │
│                                                                 │
│  ADMINISTRATORS (System Users)                                  │
│  ═════════════════════════════                                  │
│                                                                 │
│  • IT staff managing the platform                               │
│  • Create/manage employee accounts                              │
│  • Configure roles and permissions                              │
│  • Monitor system health and security                           │
│                                                                 │
│  Roles:                                                         │
│  ├── Super Admin - Full system access                           │
│  ├── IT Admin - User/role management                            │
│  └── Security Admin - Audit, security config                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Multi-Department Model (Optional)

```
┌─────────────────────────────────────────────────────────────────┐
│                 MULTI-DEPARTMENT MODEL                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   EMPLOYEE can have DIFFERENT ROLES in DIFFERENT DEPARTMENTS    │
│                                                                 │
│   ┌───────────┐                                                 │
│   │ EMPLOYEE  │                                                 │
│   │   Sarah   │                                                 │
│   └─────┬─────┘                                                 │
│         │                                                       │
│         ├─────────────────────┬─────────────────────┐           │
│         │                     │                     │           │
│         ▼                     ▼                     ▼           │
│   ┌───────────┐         ┌───────────┐         ┌───────────┐    │
│   │  Revenue  │         │  Permits  │         │   Parks   │    │
│   │   Dept    │         │   Dept    │         │    Dept   │    │
│   │           │         │           │         │           │    │
│   │ Role:     │         │ Role:     │         │ Role:     │    │
│   │ TAX CLERK │         │ INSPECTOR │         │  (none)   │    │
│   └───────────┘         └───────────┘         └───────────┘    │
│                                                                 │
│   Sarah processes taxes in Revenue Dept                         │
│   Sarah conducts inspections for Permits Dept                   │
│   Sarah has NO access to Parks Dept                             │
│                                                                 │
│                                                                 │
│   CITIZENS typically don't have department-scoped roles         │
│   They access services, not departments                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5. Role Hierarchy Concept (For Employees)

```
┌─────────────────────────────────────────────────────────────────┐
│              ROLE HIERARCHY (Per Department)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Roles can INHERIT permissions from subordinate roles          │
│   Higher roles can MANAGE employees with lower roles            │
│                                                                 │
│                    ┌───────────────┐                           │
│                    │  SUPER-ADMIN  │ Level 0 (IT)              │
│                    │               │                            │
│                    │ All perms +   │                            │
│                    │ manages all   │                            │
│                    └───────┬───────┘                            │
│                            │                                    │
│              ┌─────────────┴─────────────┐                     │
│              │                           │                      │
│              ▼                           ▼                      │
│      ┌───────────────┐          ┌───────────────┐              │
│      │  DEPARTMENT   │          │   DEPARTMENT  │ Level 1      │
│      │    HEAD       │          │    HEAD       │              │
│      │  (Revenue)    │          │  (Permits)    │              │
│      └───────┬───────┘          └───────┬───────┘              │
│              │                           │                      │
│              ▼                           ▼                      │
│      ┌───────────────┐          ┌───────────────┐              │
│      │  SUPERVISOR   │          │   SUPERVISOR  │ Level 2      │
│      │               │          │               │              │
│      │ Approves work │          │ Reviews cases │              │
│      └───────┬───────┘          └───────┬───────┘              │
│              │                           │                      │
│              ▼                           ▼                      │
│      ┌───────────────┐          ┌───────────────┐              │
│      │    CLERK      │          │   INSPECTOR   │ Level 3      │
│      │               │          │               │              │
│      │ Processes     │          │ Conducts      │              │
│      │ applications  │          │ inspections   │              │
│      └───────────────┘          └───────────────┘              │
│                                                                 │
│   A SUPERVISOR automatically inherits all CLERK permissions     │
│   A SUPERVISOR can assign/reassign work to CLERKs               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     SYSTEM ARCHITECTURE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                              CLIENTS                                        │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│    │ Citizen  │    │ Citizen  │    │ Employee │    │  Third   │            │
│    │   App    │    │  Portal  │    │  Portal  │    │  Party   │            │
│    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘            │
│         │               │               │               │                   │
│         └───────────────┴───────────────┴───────────────┘                   │
│                                   │                                         │
│                                   ▼                                         │
│                    ┌──────────────────────────┐                             │
│                    │       API GATEWAY        │                             │
│                    │  • Rate Limiting         │                             │
│                    │  • Load Balancing        │                             │
│                    │  • SSL Termination       │                             │
│                    └────────────┬─────────────┘                             │
│                                 │                                           │
│         ┌───────────────────────┼───────────────────────┐                   │
│         │                       │                       │                   │
│         ▼                       ▼                       ▼                   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │                 │    │                 │    │                 │         │
│  │ PERMITS SERVICE │    │ REVENUE SERVICE │    │ PUBLIC WORKS    │         │
│  │   (Python)      │    │   (Java)        │    │  (Node.js)      │         │
│  │                 │    │                 │    │                 │         │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘         │
│           │                      │                      │                   │
│           │    SERVICE TOKEN     │    SERVICE TOKEN     │                   │
│           │         +            │         +            │                   │
│           │    USER TOKEN        │    USER TOKEN        │                   │
│           │                      │                      │                   │
│           └──────────────────────┼──────────────────────┘                   │
│                                  │                                          │
│                                  ▼                                          │
│                    ┌──────────────────────────────────┐                     │
│                    │                                  │                     │
│                    │     CENTRAL AUTH SYSTEM          │                     │
│                    │                                  │                     │
│                    │  ┌────────────────────────────┐  │                     │
│                    │  │     REST API LAYER         │  │                     │
│                    │  │  • Auth Endpoints          │  │                     │
│                    │  │  • Citizen Management      │  │                     │
│                    │  │  • Employee Management     │  │                     │
│                    │  │  • Role/Permission Mgmt    │  │                     │
│                    │  │  • Token Verify            │  │                     │
│                    │  └────────────────────────────┘  │                     │
│                    │                                  │                     │
│                    │  ┌────────────────────────────┐  │                     │
│                    │  │     SERVICE LAYER          │  │                     │
│                    │  │  • AuthService             │  │                     │
│                    │  │  • AuthorizationResolver   │  │                     │
│                    │  │  • ServiceClientAuth       │  │                     │
│                    │  └────────────────────────────┘  │                     │
│                    │                                  │                     │
│                    │  ┌────────────────────────────┐  │                     │
│                    │  │     DATA LAYER             │  │                     │
│                    │  │  • Citizens                │  │                     │
│                    │  │  • Employees               │  │                     │
│                    │  │  • Roles & Permissions     │  │                     │
│                    │  │  • Auth Rules              │  │                     │
│                    │  │  • Service Clients         │  │                     │
│                    │  │  • Sessions & Tokens       │  │                     │
│                    │  └────────────────────────────┘  │                     │
│                    │                                  │                     │
│                    └────────────────┬─────────────────┘                     │
│                                     │                                       │
│                                     ▼                                       │
│                    ┌──────────────────────────────────┐                     │
│                    │         DATABASE                 │                     │
│                    │    (PostgreSQL/MySQL)            │                     │
│                    └──────────────────────────────────┘                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Authentication Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AUTHENTICATION FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. CITIZEN LOGIN FLOW                                                      │
│  ═════════════════════                                                      │
│                                                                             │
│  Citizen App               Auth System                Database              │
│    │                          │                         │                   │
│    │  POST /auth/login        │                         │                   │
│    │  {email, password}       │                         │                   │
│    │─────────────────────────►│                         │                   │
│    │                          │  Verify credentials     │                   │
│    │                          │────────────────────────►│                   │
│    │                          │                         │                   │
│    │                          │◄────────────────────────│                   │
│    │                          │  Citizen found, valid   │                   │
│    │                          │                         │                   │
│    │                          │  Generate Token         │                   │
│    │                          │  Load Roles/Perms       │                   │
│    │                          │────────────────────────►│                   │
│    │                          │                         │                   │
│    │  {token, citizen_data}   │◄────────────────────────│                   │
│    │◄─────────────────────────│                         │                   │
│    │                          │                         │                   │
│                                                                             │
│  2. SERVICE REQUEST FLOW (Citizen → Permits Service)                        │
│  ═══════════════════════════════════════════════════                        │
│                                                                             │
│  Citizen App           Permits Service             Auth System              │
│    │                          │                          │                  │
│    │  POST /permits/apply     │                          │                  │
│    │  + Bearer Token          │                          │                  │
│    │─────────────────────────►│                          │                  │
│    │                          │                          │                  │
│    │                          │  POST /auth/token-verify │                  │
│    │                          │  {service, user_token,   │                  │
│    │                          │   method, path}          │                  │
│    │                          │─────────────────────────►│                  │
│    │                          │                          │                  │
│    │                          │  {authorized: true,      │                  │
│    │                          │   citizen: {...}}        │                  │
│    │                          │◄─────────────────────────│                  │
│    │                          │                          │                  │
│    │                          │  Process permit          │                  │
│    │                          │  application             │                  │
│    │◄─────────────────────────│                          │                  │
│    │  Application #12345      │                          │                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Features

### 1. Token-Based Authentication
- **Personal Access Tokens** - Long-lived API tokens
- **Session Tokens** - Short-lived auth tokens
- **Token Abilities** - Scope-limited tokens (read-only, etc.)
- **Token Expiration** - Configurable TTL per token type

### 2. OTP System
- **Email Verification** - Citizen account activation
- **Phone Verification** - Mobile number confirmation
- **Password Reset** - Secure password recovery
- **6-digit Codes** - User-friendly format
- **Time-Limited** - Auto-expiration (10 minutes)

### 3. Role-Based Access Control (RBAC)
- **Citizen Roles** - Limited (standard citizen, verified citizen, business owner)
- **Employee Roles** - Granular (clerk, inspector, supervisor, manager, admin)
- **Permissions** - Atomic access rights
- **Role Assignment** - Direct or through hierarchy
- **Guard Support** - Separate guards for citizen/employee

### 4. Dynamic Authorization Rules
- **Path Patterns** - DSL-based route matching
- **Route Names** - Named route support
- **Method Filtering** - HTTP method restrictions
- **Priority System** - Rule precedence
- **Per-Service Rules** - Each microservice has its own rules

### 5. Service-to-Service Authentication
- **Service Clients** - Registered government microservices
- **Hashed Tokens** - Secure SHA-256 storage
- **Expiration Support** - Time-limited access
- **Usage Tracking** - Audit capabilities

### 6. Role Hierarchy (For Employees)
- **Parent-Child Roles** - Inheritance chains
- **Department-Scoped** - Different hierarchies per department
- **Circular Detection** - Prevents invalid configurations
- **Permission Inheritance** - Automatic permission flow

### 7. Multi-Device Support
- **Device Registration** - Track citizen devices
- **FCM Integration** - Push notification tokens
- **Last Activity** - Session monitoring
- **Device Management** - List/revoke devices

### 8. Government-Specific Features
- **National ID Integration** - Verify citizen identity
- **Audit Logging** - Complete access trail for compliance
- **Data Privacy** - GDPR/local regulation compliance
- **Employee 2FA** - Required for staff accounts

---

## Design Principles

### 1. Separation of Concerns
```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   Authentication    ≠    Authorization    ≠    Identity │
│                                                         │
│   "Prove who       "What can they       "Store citizen  │
│    you are"         do?"                 profile"       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2. Principle of Least Privilege
- Citizens access only their own data
- Employees access only assigned departments
- Time-limited tokens
- Scoped permissions

### 3. Defense in Depth
- Multiple validation layers
- Service + User authentication
- Rule-based + Permission-based checks
- Network isolation for auth system

### 4. Stateless Design
- No server-side sessions for API (tokens carry context)
- Horizontal scaling friendly
- Cacheable authorization responses

### 5. Audit Everything
- Login attempts (success and failure)
- Permission checks
- Data access by employees
- Admin actions

### 6. Citizen-First Design
- Simple registration process
- Clear error messages
- Accessible interface
- Multi-language support capability

---

## Use Cases

### Use Case 1: Citizen Registration
```
Actor: Citizen
Precondition: None
Flow:
  1. Citizen provides email, name, password
  2. System validates email format, password strength
  3. System sends verification OTP to email
  4. Citizen enters OTP
  5. System creates verified citizen account
  6. System returns access token
Postcondition: Citizen has active account and session
```

### Use Case 2: Citizen Submits Permit Application
```
Actor: Citizen
Precondition: Citizen is logged in
Flow:
  1. Citizen navigates to Permits Service
  2. Citizen submits building permit application
  3. Permits Service calls Auth System to verify citizen token
  4. Auth System confirms citizen identity and permissions
  5. Permits Service processes application
  6. Application is assigned to relevant department
Postcondition: Permit application created and tracked
```

### Use Case 3: Employee Processes Application
```
Actor: Government Employee (Clerk)
Precondition: Employee has "process permits" permission in Permits Dept
Flow:
  1. Employee logs into staff portal
  2. Employee views assigned permit applications
  3. For each action, Auth System verifies employee permissions
  4. Employee reviews and approves/denies application
  5. System logs action with employee ID and timestamp
  6. Citizen is notified of decision
Postcondition: Permit approved/denied, audit trail recorded
```

### Use Case 4: API Request from Microservice
```
Actor: Permits Microservice
Precondition: Service has valid service client token
Flow:
  1. Service receives request with citizen token
  2. Service calls auth system's verify endpoint
  3. Auth system validates service token
  4. Auth system validates citizen token
  5. Auth system checks authorization rules
  6. Auth system returns decision + citizen permissions
Postcondition: Service knows if request is allowed
```

### Use Case 5: Supervisor Reviews Employee Work
```
Actor: Department Supervisor
Precondition: Supervisor has role hierarchy over Clerks
Flow:
  1. Supervisor logs in
  2. System loads department and subordinate employees
  3. Supervisor reviews work by team members
  4. Supervisor can reassign cases if needed
  5. All access logged in audit trail
Postcondition: Supervisor has oversight of team work
```

---

## Government Services Covered

This auth system is designed to support services across all government departments:

| Department | Example Services |
|------------|-----------------|
| **Revenue & Finance** | Property Tax, Business Licensing, Utility Bills |
| **Building & Planning** | Building Permits, Zoning Requests, Inspections |
| **Public Works** | Pothole Reports, Street Light Issues, Sidewalk Permits |
| **Parks & Recreation** | Facility Reservations, Program Registration, Event Permits |
| **Clerk's Office** | Birth/Death Certificates, Marriage Licenses, Public Records |
| **Public Safety** | Police Reports, Fire Permits, Emergency Alerts |
| **Human Services** | Social Services, Senior Programs, Youth Services |
| **Transportation** | Parking Permits, Bus Passes, Traffic Complaints |

---

## Next Steps

See the following documents for detailed implementation:

1. **[00-WHY-THIS-APPROACH.md](./00-WHY-THIS-APPROACH.md)** - Architecture decision analysis
2. **[02-DATABASE-SCHEMA.md](./02-DATABASE-SCHEMA.md)** - Complete database design
3. **[03-API-SPECIFICATION.md](./03-API-SPECIFICATION.md)** - API endpoints and contracts
4. **[04-MICROSERVICE-INTEGRATION.md](./04-MICROSERVICE-INTEGRATION.md)** - Integration patterns
5. **[05-SECURITY-BEST-PRACTICES.md](./05-SECURITY-BEST-PRACTICES.md)** - Security guidelines
6. **[06-DJANGO-IMPLEMENTATION.md](./06-DJANGO-IMPLEMENTATION.md)** - Django reference implementation
