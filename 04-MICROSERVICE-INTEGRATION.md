# Microservice Integration Guide - Local Government Services Auth System

## 📋 Table of Contents
1. [Integration Overview](#integration-overview)
2. [Government Services Architecture](#government-services-architecture)
3. [Service Registration](#service-registration)
4. [Authentication Flow](#authentication-flow)
5. [Authorization Resolution](#authorization-resolution)
6. [Implementation Examples](#implementation-examples)
7. [Best Practices](#best-practices)
8. [Caching Strategies](#caching-strategies)
9. [Error Handling](#error-handling)
10. [Monitoring & Observability](#monitoring--observability)

---

## Integration Overview

### Core Concept

The Central Auth System acts as the **Single Source of Truth** for all authentication and authorization decisions across your government services architecture. Citizens and employees authenticate once, and all department microservices verify their identity and permissions through this central system.

```
┌──────────────────────────────────────────────────────────────────────┐
│            GOVERNMENT SERVICES INTEGRATION PATTERN                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│             ┌─────────────────────┐                                  │
│             │  Citizen / Employee │                                  │
│             │  (Web/Mobile App)   │                                  │
│             └──────────┬──────────┘                                  │
│                        │                                             │
│              1. Login Request                                        │
│                        │                                             │
│                        ▼                                             │
│            ┌───────────────────────────┐                             │
│            │   CENTRAL AUTH SYSTEM     │                             │
│            │                           │                             │
│            │  • Validates credentials  │                             │
│            │  • Issues USER TOKEN      │                             │
│            │  • Returns user profile   │                             │
│            │  • Returns permissions    │                             │
│            └─────────────┬─────────────┘                             │
│                          │                                           │
│              2. Returns Token + Profile                              │
│                          │                                           │
│                          ▼                                           │
│             ┌─────────────────────┐                                  │
│             │  Citizen / Employee │  Stores token locally            │
│             └──────────┬──────────┘                                  │
│                        │                                             │
│   3. Service Request + Token (e.g., Submit Permit Application)       │
│                        │                                             │
│                        ▼                                             │
│            ┌───────────────────────────┐                             │
│            │   GOVERNMENT SERVICE      │                             │
│            │   (Permits Service)       │                             │
│            │                           │                             │
│            │  Extracts user token      │                             │
│            └─────────────┬─────────────┘                             │
│                          │                                           │
│   4. Verify Token + Check Authorization                              │
│      (SERVICE_TOKEN + USER_TOKEN + department context)               │
│                          │                                           │
│                          ▼                                           │
│            ┌───────────────────────────┐                             │
│            │   CENTRAL AUTH SYSTEM     │                             │
│            │                           │                             │
│            │  • Validate service token │                             │
│            │  • Validate user token    │                             │
│            │  • Check auth rules       │                             │
│            │  • Check department perms │                             │
│            │  • Return decision        │                             │
│            └─────────────┬─────────────┘                             │
│                          │                                           │
│   5. Auth Response                                                   │
│      {authorized, user, permissions, department_context}             │
│                          │                                           │
│                          ▼                                           │
│            ┌───────────────────────────┐                             │
│            │   GOVERNMENT SERVICE      │                             │
│            │                           │                             │
│            │  If authorized:           │                             │
│            │    Process request        │                             │
│            │    Log audit trail        │                             │
│            │  Else:                    │                             │
│            │    Return 403             │                             │
│            └───────────────────────────┘                             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Integration Points

| Integration Point | Description | Government Context |
|-------------------|-------------|-------------------|
| **Service Registration** | Register your microservice to get a service token | Each department service needs credentials |
| **Token Verification** | Call auth system to verify user tokens | Verify citizen or employee identity |
| **Authorization Check** | Get auth decision for specific route/action | Check department-scoped permissions |
| **User Context** | Receive user details and permissions | Citizen profile or employee profile |
| **Department Context** | Receive department-specific permissions | For employee multi-department access |
| **Audit Logging** | Log all citizen data access | Government compliance requirement |

---

## Government Services Architecture

### Department Microservices Landscape

```
┌──────────────────────────────────────────────────────────────────────────┐
│                 GOVERNMENT SERVICES ECOSYSTEM                             │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                      ┌──────────────────────────┐                        │
│                      │    CENTRAL AUTH SYSTEM   │                        │
│                      │                          │                        │
│                      │  • Citizen Identity      │                        │
│                      │  • Employee Identity     │                        │
│                      │  • Role Management       │                        │
│                      │  • Permission Control    │                        │
│                      │  • Service Auth          │                        │
│                      └────────────┬─────────────┘                        │
│                                   │                                      │
│         ┌─────────────────────────┼─────────────────────────┐            │
│         │                         │                         │            │
│         ▼                         ▼                         ▼            │
│   ┌───────────┐            ┌───────────┐            ┌───────────┐        │
│   │  Permits  │            │  Revenue  │            │  Public   │        │
│   │  Service  │            │  Service  │            │  Works    │        │
│   ├───────────┤            ├───────────┤            ├───────────┤        │
│   │ Building  │            │ Property  │            │ Road      │        │
│   │ permits,  │            │ tax,      │            │ repairs,  │        │
│   │ zoning,   │            │ business  │            │ utilities,│        │
│   │ planning  │            │ licenses  │            │ trash     │        │
│   └───────────┘            └───────────┘            └───────────┘        │
│                                                                          │
│         ┌─────────────────────────┬─────────────────────────┐            │
│         │                         │                         │            │
│         ▼                         ▼                         ▼            │
│   ┌───────────┐            ┌───────────┐            ┌───────────┐        │
│   │   Parks   │            │   Clerk   │            │  Human    │        │
│   │  Service  │            │  Service  │            │ Services  │        │
│   ├───────────┤            ├───────────┤            ├───────────┤        │
│   │ Facility  │            │ Records,  │            │ Social    │        │
│   │ booking,  │            │ elections,│            │ programs, │        │
│   │ events    │            │ licenses  │            │ benefits  │        │
│   └───────────┘            └───────────┘            └───────────┘        │
│                                                                          │
│                      SUPPORTING SERVICES                                 │
│         ┌─────────────────────────┬─────────────────────────┐            │
│         │                         │                         │            │
│         ▼                         ▼                         ▼            │
│   ┌───────────┐            ┌───────────┐            ┌───────────┐        │
│   │Notification│           │  Payment  │            │ Document  │        │
│   │  Service  │            │  Gateway  │            │  Storage  │        │
│   ├───────────┤            ├───────────┤            ├───────────┤        │
│   │ Email,    │            │ Credit,   │            │ Files,    │        │
│   │ SMS,      │            │ ACH,      │            │ forms,    │        │
│   │ push      │            │ checks    │            │ records   │        │
│   └───────────┘            └───────────┘            └───────────┘        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Architecture Patterns

#### Pattern 1: API Gateway Pattern (Recommended for Citizen Portal)

```
┌────────────────────────────────────────────────────────────────┐
│                API GATEWAY PATTERN (Citizen Portal)             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│     ┌─────────────┐                                            │
│     │   Citizen   │                                            │
│     │   Portal    │                                            │
│     └──────┬──────┘                                            │
│            │                                                   │
│            ▼                                                   │
│     ┌──────────────────────────────────────┐                   │
│     │        CITIZEN API GATEWAY            │                  │
│     │                                       │                  │
│     │  ┌─────────────────────┐             │                  │
│     │  │  Auth Middleware    │◄────────────┼────► Auth System │
│     │  │                     │             │                  │
│     │  │  • Extract token    │             │                  │
│     │  │  • Verify citizen   │             │                  │
│     │  │  • Check basic auth │             │                  │
│     │  │  • Inject user ctx  │             │                  │
│     │  └─────────────────────┘             │                  │
│     └───────────────┬──────────────────────┘                   │
│                     │                                          │
│       Trusted internal headers:                                │
│       X-Citizen-ID: 123                                        │
│       X-Citizen-Verified: true                                 │
│       X-Permissions: submit-apps,view-records                  │
│                     │                                          │
│     ┌───────────────┼───────────────┐                          │
│     │               │               │                          │
│     ▼               ▼               ▼                          │
│  ┌──────┐       ┌──────┐       ┌──────┐                        │
│  │Permit│       │Revenue│      │Parks │  ← Trust gateway       │
│  │ Svc  │       │ Svc   │      │ Svc  │    headers             │
│  └──────┘       └──────┘       └──────┘                        │
│                                                                │
│  Pros:                                                         │
│  • Single auth check per citizen request                       │
│  • Clean citizen experience                                    │
│  • Reduced auth system load                                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### Pattern 2: Service-Level Auth (For Employee/Admin Portals)

```
┌────────────────────────────────────────────────────────────────┐
│           SERVICE-LEVEL PATTERN (Employee Admin)                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│     ┌─────────────┐                                            │
│     │  Employee   │                                            │
│     │   Portal    │                                            │
│     └──────┬──────┘                                            │
│            │                                                   │
│            ▼                                                   │
│     ┌──────────────────────────────────────┐                   │
│     │       ADMIN API GATEWAY               │                  │
│     │       (Pass-through)                  │                  │
│     └───────────────┬──────────────────────┘                   │
│                     │                                          │
│                     │  Employee token passed through           │
│                     │                                          │
│     ┌───────────────┼───────────────┐                          │
│     │               │               │                          │
│     ▼               ▼               ▼                          │
│  ┌────────────────────────────────────────┐                   │
│  │         DEPARTMENT SERVICE              │                   │
│  │                                         │                   │
│  │  ┌─────────────────────┐               │                   │
│  │  │   Auth Middleware   │───────────────┼────► Auth System  │
│  │  │                     │               │                   │
│  │  │  • Verify employee  │               │     Check:        │
│  │  │  • Check dept perms │               │     • Token valid │
│  │  │  • Verify hierarchy │               │     • Has role    │
│  │  │  • Log audit trail  │               │     • Dept perms  │
│  │  └─────────────────────┘               │                   │
│  │  ┌─────────────────────┐               │                   │
│  │  │   Business Logic    │               │                   │
│  │  └─────────────────────┘               │                   │
│  └────────────────────────────────────────┘                   │
│                                                                │
│  Pros:                                                         │
│  • Department-specific auth rules                              │
│  • Granular permission control                                 │
│  • Better audit per department                                 │
│  • Employees see only their department data                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Service Registration

### Step 1: Request Service Credentials

Each government department microservice needs to be registered:

```bash
# Using curl (IT admin token required)
curl -X POST https://auth.gov.local/api/v1/service-clients \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "permits-service",
    "display_name": "Building & Planning Service",
    "department_id": "permits",
    "expires_at": "2025-12-31",
    "notes": "Handles building permits, zoning, planning applications"
  }'
```

### Step 2: Store Token Securely

**Response:**
```json
{
  "success": true,
  "data": {
    "service": {
      "id": 1,
      "name": "permits-service",
      "display_name": "Building & Planning Service",
      "department_id": "permits"
    },
    "token": "YWJjMTIzZGVmNDU2Nzg5MGFiY2RlZjEyMzQ1Njc4OTBh..."
  },
  "warning": "Save this token securely. It cannot be retrieved later."
}
```

⚠️ **CRITICAL:** Store the token immediately! It's only shown once.

**Government-Approved Storage Options:**

| Environment | Recommended Storage | Notes |
|-------------|---------------------|-------|
| Development | `.env` file (gitignored) | Local development only |
| Staging | Azure Key Vault / AWS Secrets | Test environment |
| Production | Azure Key Vault / HashiCorp Vault | Required for compliance |

### Step 3: Configure Your Service

```ini
# .env file for permits-service
AUTH_SYSTEM_URL=https://auth.gov.local/api/v1
AUTH_SERVICE_NAME=permits-service
AUTH_SERVICE_TOKEN=YWJjMTIzZGVmNDU2Nzg5MGFiY2RlZjEyMzQ1Njc4OTBh...
AUTH_DEPARTMENT_ID=permits
```

### Registered Government Services

| Service Name | Department | Purpose |
|--------------|------------|---------|
| `permits-service` | permits | Building permits, zoning, planning |
| `revenue-service` | revenue | Property tax, business licenses |
| `public-works-service` | public-works | Road repairs, utilities, trash |
| `parks-service` | parks | Facility bookings, events, programs |
| `clerk-service` | clerk | Records, elections, vital records |
| `human-services-service` | human-services | Social programs, benefits |
| `notification-service` | (shared) | Email, SMS, push notifications |
| `payment-gateway` | (shared) | Payment processing |
| `document-service` | (shared) | File storage, form management |

---

## Authentication Flow

### Citizen Token Verification

**Endpoint:** `POST /api/v1/auth/token-verify`

**Request (Citizen submitting permit application):**
```http
POST /api/v1/auth/token-verify HTTP/1.1
Host: auth.gov.local
Authorization: Bearer <permits-service-token>
Content-Type: application/json

{
  "service": "permits-service",
  "token": "1|citizen_token_abc123...",
  "method": "POST",
  "path": "/applications",
  "route_name": "applications.store"
}
```

**Response (200 OK - Citizen Authorized):**
```json
{
  "success": true,
  "data": {
    "authorized": true,
    "granted_by": "permissions_any",
    "required_permissions": ["submit applications"],
    "user_type": "citizen",
    "user": {
      "id": 123,
      "name": "Maria Garcia",
      "email": "maria.garcia@email.com",
      "phone": "+1-555-0123",
      "national_id_verified": true,
      "roles": ["verified-citizen"],
      "permissions": [
        "submit applications",
        "view own records",
        "make payments",
        "track status"
      ],
      "verification_status": {
        "email": true,
        "phone": true,
        "identity": true
      },
      "address": {
        "city": "Springfield",
        "postal_code": "62701"
      }
    }
  }
}
```

### Employee Token Verification

**Request (Employee processing permits):**
```http
POST /api/v1/auth/token-verify HTTP/1.1
Host: auth.gov.local
Authorization: Bearer <permits-service-token>
Content-Type: application/json

{
  "service": "permits-service",
  "token": "2|employee_token_xyz...",
  "method": "PUT",
  "path": "/admin/applications/456",
  "route_name": "admin.applications.update",
  "department_id": "permits"
}
```

**Response (200 OK - Employee Authorized):**
```json
{
  "success": true,
  "data": {
    "authorized": true,
    "granted_by": "permissions_any",
    "required_permissions": ["process applications"],
    "user_type": "employee",
    "user": {
      "id": 50,
      "name": "Sarah Johnson",
      "email": "sarah.johnson@gov.springfield.us",
      "employee_id": "EMP-2024-0150",
      "roles": ["clerk"],
      "permissions": [
        "view citizen profiles",
        "view applications",
        "process applications"
      ]
    },
    "department_context": {
      "department_id": "permits",
      "department_name": "Building & Planning",
      "effective_roles": ["clerk"],
      "effective_permissions": [
        "view citizen profiles",
        "view applications",
        "process applications"
      ],
      "hierarchy": {
        "reports_to": ["supervisor", "department-head"],
        "can_manage": []
      }
    }
  }
}
```

**Response (403 Forbidden - Wrong Department):**
```json
{
  "success": false,
  "data": {
    "authorized": false,
    "granted_by": "deny",
    "required_permissions": ["process applications"],
    "message": "User does not have required permissions in this department",
    "department_id": "permits",
    "user_departments": ["revenue"]
  }
}
```

---

## Authorization Resolution

### Resolution Flow for Government Services

```
┌─────────────────────────────────────────────────────────────────┐
│            GOVERNMENT AUTH RESOLUTION FLOW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Input: service, method, path, user_type, department_id         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. DETERMINE USER TYPE                                   │   │
│  │                                                          │   │
│  │    if user.user_type == 'citizen':                       │   │
│  │        context = 'citizen'                               │   │
│  │        permissions = citizen_permissions                 │   │
│  │    else if user.user_type == 'employee':                 │   │
│  │        context = 'employee'                              │   │
│  │        permissions = get_dept_permissions(department_id) │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. SUPER ADMIN CHECK (Employees Only)                    │   │
│  │                                                          │   │
│  │    if user has 'super-admin' role:                       │   │
│  │        return AUTHORIZED (bypass all rules)              │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. LOAD AUTH RULES                                       │   │
│  │                                                          │   │
│  │    SELECT * FROM auth_rules                              │   │
│  │    WHERE service = :service                              │   │
│  │      AND is_active = true                                │   │
│  │      AND (user_type = :user_type OR user_type = 'any')   │   │
│  │      AND method IN (:method, 'ANY')                      │   │
│  │    ORDER BY priority DESC, id ASC                        │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 4. MATCH RULES                                           │   │
│  │                                                          │   │
│  │    Match by route_name first (exact match)               │   │
│  │    Then match by path_regex (pattern match)              │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 5. EVALUATE MATCHED RULE                                 │   │
│  │                                                          │   │
│  │    FOR CITIZENS:                                         │   │
│  │      Check: permissions_any against citizen permissions  │   │
│  │      Example: ["submit applications"]                    │   │
│  │                                                          │   │
│  │    FOR EMPLOYEES:                                        │   │
│  │      Check: roles_any OR permissions_any                 │   │
│  │      Against: department-scoped permissions              │   │
│  │      Must have permission IN the specified department    │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 6. LOG AUDIT TRAIL                                       │   │
│  │                                                          │   │
│  │    Log: user_id, action, resource, authorized,           │   │
│  │         department_id, ip_address, timestamp             │   │
│  │                                                          │   │
│  │    Required for government compliance!                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Sample Auth Rules for Government Services

```sql
-- Citizen Rules for Permits Service
INSERT INTO auth_rules (service, method, path_dsl, user_type, permissions_any, description) VALUES
('permits-service', 'POST', '/applications', 'citizen', '["submit applications"]', 'Citizens submit permit apps'),
('permits-service', 'GET', '/applications', 'citizen', '["view own records"]', 'Citizens view their applications'),
('permits-service', 'GET', '/applications/{id}', 'citizen', '["view own records"]', 'Citizens view single application'),
('permits-service', 'GET', '/applications/{id}/status', 'citizen', '["track status"]', 'Citizens track status'),
('permits-service', 'POST', '/applications/{id}/documents', 'citizen', '["submit applications"]', 'Citizens upload documents'),
('permits-service', 'POST', '/applications/{id}/payment', 'citizen', '["make payments"]', 'Citizens make payment');

-- Employee Rules for Permits Service
INSERT INTO auth_rules (service, method, path_dsl, user_type, permissions_any, description) VALUES
('permits-service', 'GET', '/admin/applications', 'employee', '["view applications"]', 'Staff view all applications'),
('permits-service', 'GET', '/admin/applications/{id}', 'employee', '["view applications"]', 'Staff view application detail'),
('permits-service', 'PUT', '/admin/applications/{id}', 'employee', '["process applications"]', 'Staff process application'),
('permits-service', 'POST', '/admin/applications/{id}/approve', 'employee', '["approve applications"]', 'Staff approve application'),
('permits-service', 'POST', '/admin/applications/{id}/deny', 'employee', '["deny applications"]', 'Staff deny application'),
('permits-service', 'POST', '/admin/applications/{id}/assign', 'employee', '["assign applications"]', 'Staff reassign application'),
('permits-service', 'GET', '/admin/citizens/{id}', 'employee', '["view citizen profiles"]', 'Staff view citizen profile');

-- Inspector Rules
INSERT INTO auth_rules (service, method, path_dsl, user_type, roles_any, description) VALUES
('permits-service', 'GET', '/inspections', 'employee', '["inspector"]', 'Inspectors view assigned'),
('permits-service', 'POST', '/inspections/{id}/complete', 'employee', '["inspector"]', 'Inspectors complete inspection'),
('permits-service', 'POST', '/inspections/{id}/schedule', 'employee', '["inspector","supervisor"]', 'Schedule inspection');
```

---

## Implementation Examples

### Python/Django Middleware (Recommended for Government Services)

```python
# auth_middleware.py
import httpx
import hashlib
import json
from django.conf import settings
from django.http import JsonResponse
from functools import wraps
import time
import logging

logger = logging.getLogger('auth')

AUTH_SYSTEM_URL = settings.AUTH_SYSTEM_URL
AUTH_SERVICE_NAME = settings.AUTH_SERVICE_NAME
AUTH_SERVICE_TOKEN = settings.AUTH_SERVICE_TOKEN
AUTH_DEPARTMENT_ID = settings.AUTH_DEPARTMENT_ID

# Simple cache
auth_cache = {}
CACHE_TTL = 60  # seconds


class GovAuthMiddleware:
    """
    Authentication middleware for government services.
    Verifies citizen/employee tokens through the central auth system.
    """
    
    # Public paths that don't require authentication
    PUBLIC_PATHS = [
        '/health',
        '/api/v1/public/',
    ]
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Skip auth for public paths
        if self._is_public_path(request.path):
            return self.get_response(request)
        
        # Extract token
        auth_header = request.META.get('HTTP_AUTHORIZATION', '')
        if not auth_header.startswith('Bearer '):
            return JsonResponse(
                {'error': 'Missing authorization token'},
                status=401
            )
        
        user_token = auth_header[7:]
        
        # Verify token and get auth result
        auth_result = self._verify_token(
            user_token=user_token,
            method=request.method,
            path=request.path,
            department_id=request.META.get('HTTP_X_DEPARTMENT_ID', AUTH_DEPARTMENT_ID)
        )
        
        if auth_result is None:
            return JsonResponse(
                {'error': 'Authentication service unavailable'},
                status=503
            )
        
        if not auth_result.get('authorized'):
            # Log unauthorized access attempt
            self._log_auth_failure(request, auth_result)
            
            return JsonResponse({
                'error': 'Forbidden',
                'message': auth_result.get('message', 'Insufficient permissions'),
                'required_permissions': auth_result.get('required_permissions', [])
            }, status=403)
        
        # Attach user info to request
        request.gov_user = auth_result.get('user', {})
        request.gov_user_type = auth_result.get('user_type')
        request.gov_permissions = auth_result.get('user', {}).get('permissions', [])
        request.gov_department_context = auth_result.get('department_context')
        
        # Log successful access for audit
        self._log_auth_success(request, auth_result)
        
        response = self.get_response(request)
        return response
    
    def _is_public_path(self, path):
        return any(path.startswith(p) for p in self.PUBLIC_PATHS)
    
    def _verify_token(self, user_token, method, path, department_id=None):
        """Verify token with central auth system, with caching."""
        
        # Create cache key (hash the token for security)
        token_hash = hashlib.sha256(user_token.encode()).hexdigest()[:16]
        cache_key = f"{token_hash}:{method}:{path}:{department_id or ''}"
        
        # Check cache
        cached = auth_cache.get(cache_key)
        if cached and cached['expires'] > time.time():
            return cached['result']
        
        # Call auth system
        try:
            payload = {
                'service': AUTH_SERVICE_NAME,
                'token': user_token,
                'method': method,
                'path': path,
            }
            
            if department_id:
                payload['department_id'] = department_id
            
            response = httpx.post(
                f"{AUTH_SYSTEM_URL}/auth/token-verify",
                json=payload,
                headers={
                    'Authorization': f'Bearer {AUTH_SERVICE_TOKEN}',
                    'Content-Type': 'application/json'
                },
                timeout=5.0
            )
            
            if response.status_code == 401:
                return {'authorized': False, 'message': 'Invalid token'}
            
            result = response.json().get('data', {})
            
            # Cache successful results
            if result.get('authorized'):
                auth_cache[cache_key] = {
                    'result': result,
                    'expires': time.time() + CACHE_TTL
                }
            
            return result
            
        except httpx.TimeoutException:
            logger.error('Auth system timeout')
            return None
        except Exception as e:
            logger.error(f'Auth system error: {e}')
            return None
    
    def _log_auth_success(self, request, auth_result):
        """Log successful authentication for audit."""
        user = auth_result.get('user', {})
        logger.info(json.dumps({
            'event': 'auth_success',
            'user_id': user.get('id'),
            'user_type': auth_result.get('user_type'),
            'method': request.method,
            'path': request.path,
            'ip_address': self._get_client_ip(request),
            'department': auth_result.get('department_context', {}).get('department_id'),
        }))
    
    def _log_auth_failure(self, request, auth_result):
        """Log failed authentication attempt."""
        logger.warning(json.dumps({
            'event': 'auth_failure',
            'method': request.method,
            'path': request.path,
            'ip_address': self._get_client_ip(request),
            'reason': auth_result.get('message'),
            'required_permissions': auth_result.get('required_permissions'),
        }))
    
    def _get_client_ip(self, request):
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0].strip()
        return request.META.get('REMOTE_ADDR')


def require_permission(permission):
    """
    Decorator for additional permission checks on views.
    Use when auth rules alone aren't sufficient.
    """
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(request, *args, **kwargs):
            if not hasattr(request, 'gov_permissions'):
                return JsonResponse({'error': 'Not authenticated'}, status=401)
            
            if permission not in request.gov_permissions:
                return JsonResponse({
                    'error': 'Forbidden',
                    'required_permission': permission
                }, status=403)
            
            return view_func(request, *args, **kwargs)
        return wrapper
    return decorator


def require_citizen(view_func):
    """Decorator to ensure only citizens can access."""
    @wraps(view_func)
    def wrapper(request, *args, **kwargs):
        if request.gov_user_type != 'citizen':
            return JsonResponse({
                'error': 'This endpoint is for citizens only'
            }, status=403)
        return view_func(request, *args, **kwargs)
    return wrapper


def require_employee(view_func):
    """Decorator to ensure only employees can access."""
    @wraps(view_func)
    def wrapper(request, *args, **kwargs):
        if request.gov_user_type != 'employee':
            return JsonResponse({
                'error': 'This endpoint is for government employees only'
            }, status=403)
        return view_func(request, *args, **kwargs)
    return wrapper
```

**Usage in Django Views:**
```python
from django.views import View
from django.http import JsonResponse
from .auth_middleware import require_permission, require_citizen, require_employee

class CitizenApplicationsView(View):
    """Citizen views their applications."""
    
    @require_citizen
    def get(self, request):
        citizen_id = request.gov_user['id']
        applications = Application.objects.filter(citizen_id=citizen_id)
        return JsonResponse({'applications': list(applications.values())})
    
    @require_citizen
    @require_permission('submit applications')
    def post(self, request):
        citizen = request.gov_user
        # Create new application
        application = Application.objects.create(
            citizen_id=citizen['id'],
            citizen_name=citizen['name'],
            # ...
        )
        return JsonResponse({'application_id': application.id}, status=201)


class AdminApplicationsView(View):
    """Employee processes applications."""
    
    @require_employee
    @require_permission('view applications')
    def get(self, request):
        department = request.gov_department_context['department_id']
        applications = Application.objects.filter(department=department)
        return JsonResponse({'applications': list(applications.values())})
    
    @require_employee
    @require_permission('approve applications')
    def post(self, request, application_id):
        employee = request.gov_user
        department = request.gov_department_context
        
        application = Application.objects.get(id=application_id)
        application.status = 'approved'
        application.approved_by = employee['id']
        application.approved_at = timezone.now()
        application.save()
        
        # Log audit trail
        AuditLog.objects.create(
            user_id=employee['id'],
            user_type='employee',
            action='approve_application',
            entity_type='Application',
            entity_id=application_id,
            department_id=department['department_id'],
        )
        
        return JsonResponse({'status': 'approved'})
```

### Node.js/Express Middleware

```javascript
// auth-middleware.js
const axios = require('axios');
const crypto = require('crypto');

const AUTH_SYSTEM_URL = process.env.AUTH_SYSTEM_URL;
const AUTH_SERVICE_NAME = process.env.AUTH_SERVICE_NAME;
const AUTH_SERVICE_TOKEN = process.env.AUTH_SERVICE_TOKEN;
const AUTH_DEPARTMENT_ID = process.env.AUTH_DEPARTMENT_ID;

// Cache
const authCache = new Map();
const CACHE_TTL = 60000; // 1 minute

// Public paths (no auth required)
const PUBLIC_PATHS = ['/health', '/api/v1/public'];

async function govAuthMiddleware(req, res, next) {
  // Skip public paths
  if (PUBLIC_PATHS.some(p => req.path.startsWith(p))) {
    return next();
  }
  
  try {
    // Extract token
    const authHeader = req.headers.authorization;
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'Missing authorization token' });
    }
    
    const userToken = authHeader.substring(7);
    const departmentId = req.headers['x-department-id'] || AUTH_DEPARTMENT_ID;
    
    // Cache key
    const tokenHash = crypto.createHash('sha256').update(userToken).digest('hex').substring(0, 16);
    const cacheKey = `${tokenHash}:${req.method}:${req.path}:${departmentId || ''}`;
    
    // Check cache
    const cached = authCache.get(cacheKey);
    if (cached && cached.expires > Date.now()) {
      req.govUser = cached.result.user;
      req.govUserType = cached.result.user_type;
      req.govPermissions = cached.result.user?.permissions || [];
      req.govDepartmentContext = cached.result.department_context;
      return next();
    }
    
    // Call auth system
    const payload = {
      service: AUTH_SERVICE_NAME,
      token: userToken,
      method: req.method,
      path: req.path,
    };
    
    if (departmentId) {
      payload.department_id = departmentId;
    }
    
    const response = await axios.post(
      `${AUTH_SYSTEM_URL}/auth/token-verify`,
      payload,
      {
        headers: {
          'Authorization': `Bearer ${AUTH_SERVICE_TOKEN}`,
          'Content-Type': 'application/json'
        },
        timeout: 5000
      }
    );
    
    const authResult = response.data.data;
    
    // Check authorization
    if (!authResult.authorized) {
      console.warn('Auth failure:', {
        path: req.path,
        method: req.method,
        required: authResult.required_permissions
      });
      
      return res.status(403).json({
        error: 'Forbidden',
        message: authResult.message || 'Insufficient permissions',
        required_permissions: authResult.required_permissions
      });
    }
    
    // Cache result
    authCache.set(cacheKey, {
      result: authResult,
      expires: Date.now() + CACHE_TTL
    });
    
    // Attach to request
    req.govUser = authResult.user;
    req.govUserType = authResult.user_type;
    req.govPermissions = authResult.user?.permissions || [];
    req.govDepartmentContext = authResult.department_context;
    
    // Log for audit
    console.log('Auth success:', {
      user_id: authResult.user?.id,
      user_type: authResult.user_type,
      path: req.path,
      method: req.method
    });
    
    next();
    
  } catch (error) {
    if (error.response?.status === 401) {
      return res.status(401).json({ error: 'Invalid token' });
    }
    console.error('Auth error:', error.message);
    return res.status(503).json({ error: 'Authentication service unavailable' });
  }
}

// Permission check middleware
function requirePermission(permission) {
  return (req, res, next) => {
    if (!req.govPermissions.includes(permission)) {
      return res.status(403).json({
        error: 'Forbidden',
        required_permission: permission
      });
    }
    next();
  };
}

// User type check middleware
function requireCitizen(req, res, next) {
  if (req.govUserType !== 'citizen') {
    return res.status(403).json({ error: 'Citizens only' });
  }
  next();
}

function requireEmployee(req, res, next) {
  if (req.govUserType !== 'employee') {
    return res.status(403).json({ error: 'Government employees only' });
  }
  next();
}

module.exports = {
  govAuthMiddleware,
  requirePermission,
  requireCitizen,
  requireEmployee
};
```

---

## Best Practices

### 1. Government Compliance

```
┌─────────────────────────────────────────────────────────────┐
│              GOVERNMENT COMPLIANCE REQUIREMENTS              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  AUDIT LOGGING (Required):                                  │
│  • Log ALL citizen data access                              │
│  • Log employee actions with timestamps                     │
│  • Include: who, what, when, where (IP)                     │
│  • Retain logs per local retention policy                   │
│  • Make logs tamper-proof (append-only)                     │
│                                                             │
│  DATA PROTECTION:                                           │
│  • Encrypt citizen PII at rest                              │
│  • Use TLS 1.2+ for all communications                      │
│  • Minimize data in auth responses                          │
│  • Mask sensitive data in logs (SSN, etc.)                  │
│                                                             │
│  ACCESS CONTROL:                                            │
│  • Employees see only their department data                 │
│  • Citizens see only their own records                      │
│  • Implement "need to know" principle                       │
│  • Regular access reviews                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. Token Security

```
┌─────────────────────────────────────────────────────────────┐
│                    TOKEN SECURITY                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ DO:                                                     │
│  • Store service tokens in approved secret managers         │
│  • Rotate service tokens annually                           │
│  • Use short-lived citizen tokens (24h max)                 │
│  • Implement token refresh for mobile apps                  │
│  • Hash tokens before using as cache keys                   │
│                                                             │
│  ❌ DON'T:                                                  │
│  • Log full token values                                    │
│  • Store tokens in source code                              │
│  • Share tokens between environments                        │
│  • Use same token for multiple services                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. Error Messages

```
┌─────────────────────────────────────────────────────────────┐
│              CITIZEN-FRIENDLY ERROR MESSAGES                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  BAD: "403 Forbidden - missing permission: view_citizen"    │
│                                                             │
│  GOOD: "You don't have access to this page. If you          │
│         believe this is an error, please contact            │
│         support@gov.springfield.us"                         │
│                                                             │
│  BAD: "Token validation failed: JWT signature invalid"      │
│                                                             │
│  GOOD: "Your session has expired. Please log in again."     │
│                                                             │
│  BAD: "Database connection timeout in auth_middleware"      │
│                                                             │
│  GOOD: "We're experiencing technical difficulties.          │
│         Please try again in a few minutes."                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Caching Strategies

### Multi-Level Cache for Government Services

```
┌─────────────────────────────────────────────────────────────┐
│                  MULTI-LEVEL CACHING                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Request                                                    │
│     │                                                       │
│     ▼                                                       │
│  ┌─────────────────┐                                        │
│  │  L1: In-Memory  │  TTL: 30-60 seconds                   │
│  │  (per instance) │  Size: 1000 entries                   │
│  │                 │  Use: High-frequency requests          │
│  └────────┬────────┘                                        │
│           │ miss                                            │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │  L2: Redis      │  TTL: 5-15 minutes                    │
│  │  (shared)       │  Use: Cross-instance sharing          │
│  └────────┬────────┘                                        │
│           │ miss                                            │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │  Auth System    │  Source of truth                      │
│  └─────────────────┘                                        │
│                                                             │
│  Cache Key Format:                                          │
│  gov:auth:{token_hash}:{method}:{path_hash}:{dept}         │
│                                                             │
│  IMPORTANT:                                                 │
│  • Never cache authorization failures                       │
│  • Invalidate on role/permission changes                    │
│  • Short TTL for employee permissions (more dynamic)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Cache Invalidation Events

```python
# Events that should invalidate auth cache
INVALIDATION_EVENTS = [
    'citizen.permissions.changed',
    'employee.role.changed',
    'employee.department.changed',
    'role.permissions.changed',
    'auth_rule.changed',
    'user.logout',
    'user.password.changed',
    'user.deactivated'
]
```

---

## Monitoring & Observability

### Key Metrics for Government Services

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `auth_requests_total` | Total auth verifications | N/A |
| `auth_request_duration_ms` | Auth system latency | p99 > 500ms |
| `auth_failures_by_type` | Failed by reason | Spike detection |
| `auth_cache_hit_ratio` | Cache effectiveness | < 70% |
| `citizen_data_access_total` | Citizen profile views | Audit reporting |
| `employee_actions_total` | Employee actions | Audit reporting |

### Structured Logging for Audit

```json
{
  "timestamp": "2024-01-25T14:30:00Z",
  "level": "INFO",
  "service": "permits-service",
  "event": "citizen_data_access",
  "data": {
    "user_id": 50,
    "user_type": "employee",
    "user_email": "sarah.johnson@gov.springfield.us",
    "action": "view_citizen_profile",
    "citizen_id": 123,
    "method": "GET",
    "path": "/admin/citizens/123",
    "department": "permits",
    "ip_address": "10.0.1.50",
    "latency_ms": 45,
    "authorized": true
  }
}
```

---

## Summary Checklist

### Before Deploying Government Service

- [ ] Service registered with IT admin
- [ ] Service token stored in approved secret manager
- [ ] Auth middleware implemented with caching
- [ ] Audit logging enabled for all citizen data access
- [ ] Error messages are citizen-friendly
- [ ] Circuit breaker implemented for auth system calls
- [ ] Health check includes auth system dependency
- [ ] Rate limiting configured appropriately
- [ ] Employee-only endpoints protected
- [ ] Department context properly passed
- [ ] Security review completed
- [ ] Penetration testing passed
- [ ] Compliance review signed off
