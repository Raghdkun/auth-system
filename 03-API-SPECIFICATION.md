# API Specification - Local Government Services Authentication System

## 📋 Table of Contents
1. [API Design Principles](#api-design-principles)
2. [Authentication Methods](#authentication-methods)
3. [API Endpoints](#api-endpoints)
4. [Request/Response Formats](#requestresponse-formats)
5. [Error Handling](#error-handling)
6. [Rate Limiting](#rate-limiting)

---

## API Design Principles

### RESTful Conventions

| Principle | Implementation |
|-----------|---------------|
| **Resource-based URLs** | `/api/v1/citizens`, `/api/v1/departments` |
| **HTTP Methods** | GET (read), POST (create), PUT (update), DELETE (remove) |
| **Stateless** | Each request contains all necessary information |
| **Versioned** | `/api/v1/`, `/api/v2/` |
| **JSON** | Request and response bodies in JSON |

### URL Structure

```
https://auth.gov.local/api/v1/{resource}[/{id}][/{action}]

Examples:
  POST   /api/v1/auth/login
  GET    /api/v1/citizens
  GET    /api/v1/citizens/123
  PUT    /api/v1/citizens/123
  DELETE /api/v1/citizens/123
  POST   /api/v1/employees/123/departments/assign
```

### Government API Considerations

| Consideration | Implementation |
|---------------|----------------|
| **Audit Logging** | All requests logged with user, action, timestamp |
| **Data Minimization** | Only return necessary citizen data |
| **Consent Tracking** | Track data access consent where required |
| **Accessibility** | Clear error messages, consistent structure |

---

## Authentication Methods

### 1. Citizen Token (Sanctum/Bearer)

```http
Authorization: Bearer <citizen_token>
```

**Token Format:** `{token_id}|{plain_token}`

**Usage:** Citizen authentication for self-service portals (web/mobile).

### 2. Employee Token (Sanctum/Bearer)

```http
Authorization: Bearer <employee_token>
```

**Usage:** Government employee authentication for admin portals.

**Note:** Employees may require 2FA for sensitive operations.

### 3. Service Client Token (Microservices)

```http
Authorization: Bearer <service_token>
Content-Type: application/json

{
  "service": "permits-service"
}
```

**Usage:** Government microservice-to-auth-system communication.

### 4. Combined (Service + User Token)

For token verification from microservices:

```http
Authorization: Bearer <service_token>
Content-Type: application/json

{
  "service": "permits-service",
  "token": "<citizen_or_employee_token>",
  "method": "POST",
  "path": "/applications"
}
```

---

## API Endpoints

### Citizen Authentication Endpoints

#### POST `/api/v1/auth/citizen/login`

Authenticate citizen and receive access token.

**Request:**
```json
{
  "email": "citizen@email.com",
  "password": "secure_password",
  "client_type": "web|mobile",
  "device": {
    "device_id": "unique-device-id",
    "platform": "ios|android|web",
    "model": "iPhone 14 Pro",
    "os_version": "17.2",
    "app_version": "1.0.0"
  },
  "fcm_token": "firebase-cloud-messaging-token"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "token": "1|abc123xyz...",
    "token_type": "Bearer",
    "expires_at": "2024-02-15T10:30:00Z",
    "user": {
      "id": 1,
      "user_type": "citizen",
      "name": "Maria Garcia",
      "email": "maria.garcia@email.com",
      "phone": "+1-555-0123",
      "email_verified_at": "2024-01-15T10:30:00Z",
      "phone_verified_at": "2024-01-16T08:00:00Z",
      "identity_verified_at": "2024-01-20T14:30:00Z",
      "address": "456 Residential Ave",
      "city": "Springfield",
      "postal_code": "62701",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-20T14:30:00Z",
      
      "roles": [
        {
          "id": 1,
          "name": "verified-citizen",
          "guard_name": "citizen",
          "permissions": [
            {"id": 1, "name": "submit applications", "guard_name": "citizen"},
            {"id": 2, "name": "view own records", "guard_name": "citizen"},
            {"id": 3, "name": "make payments", "guard_name": "citizen"},
            {"id": 4, "name": "track status", "guard_name": "citizen"}
          ]
        }
      ],
      
      "permissions": [
        "submit applications",
        "view own records",
        "make payments",
        "track status",
        "update profile",
        "request records"
      ],
      
      "verification_status": {
        "email": true,
        "phone": true,
        "identity": true,
        "address": false
      },
      
      "active_applications": 2,
      "pending_payments": 1
    }
  }
}
```

**Error Response (401 Unauthorized):**
```json
{
  "success": false,
  "message": "Invalid credentials",
  "errors": null
}
```

---

#### POST `/api/v1/auth/citizen/register`

Register new citizen account.

**Request:**
```json
{
  "name": "John Smith",
  "email": "john.smith@email.com",
  "phone": "+1-555-0456",
  "password": "SecurePassword123!",
  "password_confirmation": "SecurePassword123!",
  "date_of_birth": "1985-06-15",
  "address": "123 Main Street",
  "city": "Springfield",
  "postal_code": "62701",
  "terms_accepted": true,
  "privacy_policy_accepted": true
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "message": "Registration successful. Please verify your email.",
  "data": {
    "user": {
      "id": 100,
      "name": "John Smith",
      "email": "john.smith@email.com",
      "email_verified_at": null,
      "created_at": "2024-01-25T10:00:00Z"
    },
    "verification_required": true,
    "verification_sent_to": "john.smith@email.com"
  }
}
```

---

#### POST `/api/v1/auth/citizen/verify-email`

Verify citizen email address.

**Request:**
```json
{
  "email": "john.smith@email.com",
  "otp": "123456"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Email verified successfully",
  "data": {
    "email_verified_at": "2024-01-25T10:15:00Z",
    "token": "1|new_token_after_verification..."
  }
}
```

---

### Employee Authentication Endpoints

#### POST `/api/v1/auth/employee/login`

Authenticate government employee (Step 1).

**Request:**
```json
{
  "email": "sarah.johnson@gov.springfield.us",
  "password": "secure_password",
  "client_type": "web"
}
```

**Response (200 OK - 2FA Required):**
```json
{
  "success": true,
  "message": "2FA verification required",
  "data": {
    "requires_2fa": true,
    "2fa_token": "temp_token_for_2fa_step...",
    "2fa_sent_to": "sarah.j***@gov.springfield.us",
    "expires_in": 600
  }
}
```

---

#### POST `/api/v1/auth/employee/verify-2fa`

Complete employee login with 2FA (Step 2).

**Request:**
```json
{
  "2fa_token": "temp_token_for_2fa_step...",
  "otp": "789012"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "token": "2|employee_token_xyz...",
    "token_type": "Bearer",
    "user": {
      "id": 50,
      "user_type": "employee",
      "name": "Sarah Johnson",
      "email": "sarah.johnson@gov.springfield.us",
      "employee_id": "EMP-2024-0150",
      "department_primary": "permits",
      "created_at": "2022-03-15T08:00:00Z",
      
      "global_roles": [
        {
          "id": 4,
          "name": "clerk",
          "guard_name": "employee"
        }
      ],
      
      "departments": [
        {
          "department": {
            "id": "permits",
            "code": "PRM",
            "name": "Building & Planning",
            "is_active": true
          },
          "direct_roles": [
            {
              "id": 4,
              "name": "clerk",
              "guard_name": "employee",
              "permissions": [
                {"id": 10, "name": "view applications"},
                {"id": 11, "name": "process applications"}
              ]
            }
          ],
          "effective_roles": [
            {
              "id": 4,
              "name": "clerk",
              "guard_name": "employee",
              "is_inherited": false
            }
          ],
          "effective_permissions": [
            "view citizen profiles",
            "view applications",
            "process applications"
          ],
          "hierarchy_info": {
            "reports_to": ["supervisor"],
            "can_be_managed_by": ["supervisor", "department-head"]
          }
        },
        {
          "department": {
            "id": "revenue",
            "code": "REV",
            "name": "Revenue & Finance",
            "is_active": true
          },
          "direct_roles": [
            {
              "id": 6,
              "name": "case-worker",
              "guard_name": "employee"
            }
          ],
          "effective_permissions": [
            "view citizen profiles",
            "view applications",
            "process applications"
          ]
        }
      ],
      
      "all_permissions": [
        "view citizen profiles",
        "view applications",
        "process applications"
      ],
      
      "summary": {
        "total_departments": 2,
        "total_roles": 2,
        "total_permissions": 5,
        "pending_tasks": 12
      }
    }
  }
}
```

---

### Common Authentication Endpoints

#### POST `/api/v1/auth/logout`

Revoke current access token.

**Headers:**
```http
Authorization: Bearer <token>
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

---

#### POST `/api/v1/auth/refresh-token`

Generate new token (revokes current token).

**Headers:**
```http
Authorization: Bearer <token>
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "token": "2|new_token_xyz...",
    "token_type": "Bearer",
    "expires_at": "2024-02-25T10:30:00Z"
  }
}
```

---

#### GET `/api/v1/auth/me`

Get current authenticated user details.

**Headers:**
```http
Authorization: Bearer <token>
```

**Response (200 OK - Citizen):**
```json
{
  "success": true,
  "data": {
    "user_type": "citizen",
    "user": {
      "id": 1,
      "name": "Maria Garcia",
      "email": "maria.garcia@email.com",
      "roles": ["verified-citizen"],
      "permissions": ["submit applications", "view own records", "make payments"],
      "verification_status": {
        "email": true,
        "phone": true,
        "identity": true
      }
    }
  }
}
```

**Response (200 OK - Employee):**
```json
{
  "success": true,
  "data": {
    "user_type": "employee",
    "user": {
      "id": 50,
      "name": "Sarah Johnson",
      "employee_id": "EMP-2024-0150",
      "departments": [...],
      "permissions": [...]
    }
  }
}
```

---

#### POST `/api/v1/auth/forgot-password`

Request password reset OTP.

**Request:**
```json
{
  "email": "citizen@email.com",
  "user_type": "citizen"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Password reset code sent to your email",
  "data": {
    "sent_to": "cit***@email.com",
    "expires_in": 600
  }
}
```

---

#### POST `/api/v1/auth/reset-password`

Reset password using OTP.

**Request:**
```json
{
  "email": "citizen@email.com",
  "otp": "123456",
  "password": "NewSecurePassword123!",
  "password_confirmation": "NewSecurePassword123!"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Password reset successfully. Please login with your new password."
}
```

---

### Token Verification Endpoint (Microservices)

#### POST `/api/v1/auth/token-verify` ⭐ Key Microservice Endpoint

Verify user token and authorize action from a microservice.

**Headers:**
```http
Authorization: Bearer <service_client_token>
Content-Type: application/json
```

**Request (Citizen accessing permits service):**
```json
{
  "service": "permits-service",
  "token": "1|citizen_token_abc123...",
  "method": "POST",
  "path": "/applications",
  "route_name": "applications.store"
}
```

**Response (200 OK - Authorized Citizen):**
```json
{
  "success": true,
  "data": {
    "authorized": true,
    "granted_by": "permissions_any",
    "required_permissions": ["submit applications"],
    "user_type": "citizen",
    "user": {
      "id": 1,
      "name": "Maria Garcia",
      "email": "maria.garcia@email.com",
      "national_id_verified": true,
      "roles": ["verified-citizen"],
      "permissions": ["submit applications", "view own records", "make payments"]
    }
  }
}
```

**Request (Employee accessing admin endpoint):**
```json
{
  "service": "permits-service",
  "token": "2|employee_token_xyz...",
  "method": "PUT",
  "path": "/admin/applications/456",
  "route_name": "admin.applications.update",
  "department_id": "permits"
}
```

**Response (200 OK - Authorized Employee):**
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
      "permissions": ["view applications", "process applications"]
    },
    "department_context": {
      "department_id": "permits",
      "department_name": "Building & Planning",
      "effective_roles": ["clerk"],
      "effective_permissions": [
        "view citizen profiles",
        "view applications",
        "process applications"
      ]
    }
  }
}
```

**Response (403 Forbidden - Unauthorized):**
```json
{
  "success": false,
  "data": {
    "authorized": false,
    "granted_by": "deny",
    "required_permissions": ["approve applications"],
    "message": "User does not have required permissions for this action"
  }
}
```

---

### Citizen Management Endpoints

#### GET `/api/v1/citizens`

List citizens (employees only, requires permissions).

**Headers:**
```http
Authorization: Bearer <employee_token>
```

**Query Parameters:**
```
?page=1
&per_page=15
&search=garcia
&verified=true
&city=Springfield
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": 1,
        "name": "Maria Garcia",
        "email": "maria.garcia@email.com",
        "phone": "+1-555-0123",
        "city": "Springfield",
        "email_verified_at": "2024-01-15T10:30:00Z",
        "identity_verified_at": "2024-01-20T14:30:00Z",
        "created_at": "2024-01-01T00:00:00Z"
      }
    ],
    "current_page": 1,
    "last_page": 10,
    "per_page": 15,
    "total": 145
  }
}
```

---

#### GET `/api/v1/citizens/{id}`

Get citizen details (employee access or own profile).

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "citizen": {
      "id": 1,
      "national_id": "***-**-1234",
      "name": "Maria Garcia",
      "email": "maria.garcia@email.com",
      "phone": "+1-555-0123",
      "date_of_birth": "1985-06-15",
      "address": "456 Residential Ave",
      "city": "Springfield",
      "postal_code": "62701",
      "email_verified_at": "2024-01-15T10:30:00Z",
      "phone_verified_at": "2024-01-16T08:00:00Z",
      "identity_verified_at": "2024-01-20T14:30:00Z",
      "roles": ["verified-citizen"],
      "verifications": [
        {
          "type": "email",
          "status": "verified",
          "verified_at": "2024-01-15T10:30:00Z"
        },
        {
          "type": "phone",
          "status": "verified",
          "verified_at": "2024-01-16T08:00:00Z"
        },
        {
          "type": "identity",
          "status": "verified",
          "verified_at": "2024-01-20T14:30:00Z",
          "verified_by": "Sarah Johnson"
        }
      ]
    }
  }
}
```

---

#### PUT `/api/v1/citizens/{id}`

Update citizen profile (own profile or employee with permission).

**Request:**
```json
{
  "name": "Maria Garcia-Smith",
  "phone": "+1-555-0124",
  "address": "789 New Address Dr",
  "city": "Springfield",
  "postal_code": "62702"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Profile updated successfully",
  "data": {
    "citizen": {...}
  }
}
```

---

#### POST `/api/v1/citizens/{id}/verify-identity`

Verify citizen identity (employees with permission).

**Request:**
```json
{
  "verification_type": "identity",
  "status": "verified",
  "document_reference": "DL-2024-0123456",
  "notes": "Driver's license verified in person"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Citizen identity verified successfully",
  "data": {
    "verification": {
      "id": 45,
      "citizen_id": 1,
      "verification_type": "identity",
      "verification_status": "verified",
      "verified_by": 50,
      "verified_at": "2024-01-25T14:30:00Z"
    }
  }
}
```

---

### Employee Management Endpoints

#### GET `/api/v1/employees`

List employees (admin access).

**Query Parameters:**
```
?page=1
&per_page=15
&department=permits
&role=clerk
&search=johnson
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": 50,
        "employee_id": "EMP-2024-0150",
        "name": "Sarah Johnson",
        "email": "sarah.johnson@gov.springfield.us",
        "department_primary": "permits",
        "departments": [
          {"id": "permits", "name": "Building & Planning", "role": "clerk"},
          {"id": "revenue", "name": "Revenue & Finance", "role": "case-worker"}
        ],
        "is_active": true,
        "created_at": "2022-03-15T08:00:00Z"
      }
    ],
    "current_page": 1,
    "last_page": 5,
    "per_page": 15,
    "total": 65
  }
}
```

---

#### POST `/api/v1/employees`

Create employee account (HR/IT admin).

**Request:**
```json
{
  "name": "John Davis",
  "email": "john.davis@gov.springfield.us",
  "employee_id": "EMP-2024-0200",
  "department_primary": "public-works",
  "password": "TempPassword123!",
  "send_welcome_email": true,
  "departments": [
    {"department_id": "public-works", "role_id": 5}
  ]
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "message": "Employee created successfully",
  "data": {
    "employee": {
      "id": 100,
      "employee_id": "EMP-2024-0200",
      "name": "John Davis",
      "email": "john.davis@gov.springfield.us",
      "departments": [...],
      "created_at": "2024-01-25T10:00:00Z"
    },
    "welcome_email_sent": true,
    "temporary_password_expires": "2024-01-26T10:00:00Z"
  }
}
```

---

### Department Management Endpoints

#### GET `/api/v1/departments`

List all departments.

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": "permits",
        "code": "PRM",
        "name": "Building & Planning",
        "description": "Handles building permits, zoning, and planning applications",
        "is_active": true,
        "employee_count": 15,
        "metadata": {
          "office_location": "City Hall, 2nd Floor",
          "phone": "+1-555-GOV-PERM"
        }
      },
      {
        "id": "revenue",
        "code": "REV",
        "name": "Revenue & Finance",
        "description": "Property taxes, business licenses, utility billing",
        "is_active": true,
        "employee_count": 22
      },
      {
        "id": "public-works",
        "code": "PWK",
        "name": "Public Works",
        "description": "Roads, utilities, infrastructure maintenance",
        "is_active": true,
        "employee_count": 45
      }
    ]
  }
}
```

---

#### POST `/api/v1/departments`

Create new department (admin only).

**Request:**
```json
{
  "id": "transportation",
  "code": "TRN",
  "name": "Transportation",
  "description": "Public transit, traffic management, parking",
  "metadata": {
    "office_location": "Transportation Center",
    "phone": "+1-555-GOV-TRAN"
  }
}
```

---

#### GET `/api/v1/departments/{id}/employees`

Get employees in department.

---

#### GET `/api/v1/departments/{id}/roles`

Get roles used in department.

---

### User-Role-Department Assignment Endpoints

#### POST `/api/v1/user-role-dept/assign`

Assign role to employee in specific department.

**Request:**
```json
{
  "user_id": 100,
  "role_id": 5,
  "department_id": "permits",
  "metadata": {
    "assigned_by": 50,
    "reason": "Transferred from Public Works"
  }
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Role assigned successfully",
  "data": {
    "assignment": {
      "id": 250,
      "user_id": 100,
      "role_id": 5,
      "department_id": "permits",
      "assigned_by": 50,
      "assigned_at": "2024-01-25T14:00:00Z"
    }
  }
}
```

---

#### POST `/api/v1/user-role-dept/remove`

Remove role from employee in department.

**Request:**
```json
{
  "user_id": 100,
  "role_id": 5,
  "department_id": "public-works"
}
```

---

#### POST `/api/v1/user-role-dept/transfer`

Transfer employee between departments.

**Request:**
```json
{
  "user_id": 100,
  "from_department_id": "public-works",
  "to_department_id": "permits",
  "from_role_id": 5,
  "to_role_id": 4,
  "effective_date": "2024-02-01",
  "reason": "Department restructuring"
}
```

---

### Role Management Endpoints

#### GET `/api/v1/roles`

List all roles.

**Response:**
```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": 1,
        "name": "citizen",
        "guard_name": "citizen",
        "description": "Standard citizen account",
        "is_system": true,
        "permissions": [
          {"id": 1, "name": "submit applications"},
          {"id": 2, "name": "view own records"}
        ]
      },
      {
        "id": 2,
        "name": "verified-citizen",
        "guard_name": "citizen",
        "description": "Identity-verified citizen",
        "is_system": true
      },
      {
        "id": 3,
        "name": "super-admin",
        "guard_name": "employee",
        "description": "Full system access",
        "is_system": true
      },
      {
        "id": 4,
        "name": "clerk",
        "guard_name": "employee",
        "description": "General processing clerk",
        "is_system": true
      }
    ]
  }
}
```

---

#### POST `/api/v1/roles`

Create a new role.

**Request:**
```json
{
  "name": "senior-inspector",
  "guard_name": "employee",
  "description": "Senior field inspector with approval rights",
  "permissions": ["view applications", "process applications", "approve applications"]
}
```

---

### Role Hierarchy Endpoints

#### POST `/api/v1/role-hierarchy`

Create hierarchy relationship.

**Request:**
```json
{
  "higher_role_id": 3,
  "lower_role_id": 4,
  "department_id": "permits",
  "metadata": {
    "created_by": 1
  }
}
```

---

#### GET `/api/v1/role-hierarchy/tree?department_id=permits`

Get hierarchy tree for department.

**Response:**
```json
{
  "success": true,
  "data": {
    "department": {
      "id": "permits",
      "name": "Building & Planning"
    },
    "tree": [
      {
        "role": {"id": 2, "name": "department-head"},
        "permissions": [...],
        "employee_count": 1,
        "children": [
          {
            "role": {"id": 3, "name": "supervisor"},
            "permissions": [...],
            "employee_count": 3,
            "children": [
              {
                "role": {"id": 4, "name": "clerk"},
                "permissions": [...],
                "employee_count": 8,
                "children": []
              },
              {
                "role": {"id": 5, "name": "inspector"},
                "permissions": [...],
                "employee_count": 4,
                "children": []
              }
            ]
          }
        ]
      }
    ]
  }
}
```

---

### Service Client Endpoints

#### GET `/api/v1/service-clients`

List all registered service clients.

**Response:**
```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": 1,
        "name": "permits-service",
        "display_name": "Permits & Licensing Service",
        "department_id": "permits",
        "is_active": true,
        "expires_at": "2025-12-31T23:59:59Z",
        "last_used_at": "2024-01-25T14:30:00Z",
        "use_count": 125430
      },
      {
        "id": 2,
        "name": "revenue-service",
        "display_name": "Revenue & Tax Service",
        "department_id": "revenue",
        "is_active": true
      },
      {
        "id": 3,
        "name": "notification-service",
        "display_name": "Notification Service",
        "department_id": null,
        "is_active": true
      }
    ]
  }
}
```

---

#### POST `/api/v1/service-clients`

Register new service client.

**Request:**
```json
{
  "name": "parks-service",
  "display_name": "Parks & Recreation Service",
  "department_id": "parks",
  "expires_at": "2025-12-31",
  "notes": "Parks facility booking and event management service"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "message": "Service client created",
  "data": {
    "service": {
      "id": 5,
      "name": "parks-service",
      "display_name": "Parks & Recreation Service",
      "department_id": "parks",
      "is_active": true,
      "expires_at": "2025-12-31T23:59:59Z"
    },
    "token": "YWJjMTIz...plain_token..."
  },
  "warning": "Save this token securely. It cannot be retrieved later."
}
```

---

#### POST `/api/v1/service-clients/{id}/rotate-token`

Rotate service token.

**Response:**
```json
{
  "success": true,
  "data": {
    "service": {...},
    "token": "new_plain_token..."
  },
  "warning": "Save this token securely. Previous token is now invalid."
}
```

---

### Authorization Rules Endpoints

#### GET `/api/v1/auth-rules`

List all authorization rules.

**Query Parameters:**
```
?service=permits-service
&user_type=citizen
&is_active=true
```

---

#### POST `/api/v1/auth-rules`

Create authorization rule.

**Request (Citizen rule):**
```json
{
  "service": "permits-service",
  "method": "POST",
  "path_dsl": "/applications",
  "user_type": "citizen",
  "permissions_any": ["submit applications"],
  "priority": 100,
  "is_active": true,
  "description": "Citizens can submit permit applications"
}
```

**Request (Employee rule):**
```json
{
  "service": "permits-service",
  "method": ["GET", "PUT"],
  "path_dsl": "/admin/applications/{id}",
  "user_type": "employee",
  "permissions_any": ["view applications", "process applications"],
  "priority": 100,
  "is_active": true,
  "description": "Staff can view and process applications"
}
```

---

#### POST `/api/v1/auth-rules/test`

Test rule path matching.

**Request:**
```json
{
  "path_dsl": "/applications/{id}/documents/*",
  "test_path": "/applications/123/documents/456"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "path_dsl": "/applications/{id}/documents/*",
    "path_regex": "#^/applications/[^/]+/documents/[^/]+$#",
    "test_path": "/applications/123/documents/456",
    "matches": true,
    "compiled_successfully": true
  }
}
```

---

### Audit Log Endpoints

#### GET `/api/v1/audit-logs`

Query audit logs (admin/auditor access).

**Query Parameters:**
```
?page=1
&per_page=50
&user_id=50
&action=view_citizen_profile
&entity_type=Citizen
&entity_id=123
&date_from=2024-01-01
&date_to=2024-01-31
&severity=critical
&service_name=permits-service
&department_id=permits
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": 10500,
        "user_id": 50,
        "user_type": "employee",
        "user_email": "sarah.johnson@gov.springfield.us",
        "action": "view_citizen_profile",
        "entity_type": "Citizen",
        "entity_id": "123",
        "description": "Viewed citizen profile for permit application processing",
        "ip_address": "192.168.1.100",
        "service_name": "permits-service",
        "department_id": "permits",
        "severity": "info",
        "created_at": "2024-01-25T14:30:00Z"
      }
    ],
    "current_page": 1,
    "last_page": 210,
    "per_page": 50,
    "total": 10485
  }
}
```

---

#### GET `/api/v1/audit-logs/citizen/{citizen_id}`

Get all audit logs for a specific citizen (who accessed their data).

**Response:**
```json
{
  "success": true,
  "data": {
    "citizen_id": 123,
    "citizen_name": "Maria Garcia",
    "access_logs": [
      {
        "accessed_by": "Sarah Johnson (Employee)",
        "action": "view_citizen_profile",
        "service": "permits-service",
        "department": "Building & Planning",
        "timestamp": "2024-01-25T14:30:00Z",
        "reason": "Permit application processing"
      }
    ],
    "total_accesses": 15,
    "unique_employees": 3
  }
}
```

---

## Request/Response Formats

### Standard Success Response

```json
{
  "success": true,
  "message": "Optional success message",
  "data": {
    // Response payload
  }
}
```

### Standard Error Response

```json
{
  "success": false,
  "message": "Error description",
  "errors": {
    "field_name": ["Validation error message"]
  }
}
```

### Pagination Format

```json
{
  "data": [...],
  "current_page": 1,
  "last_page": 10,
  "per_page": 15,
  "total": 150,
  "from": 1,
  "to": 15,
  "links": {
    "first": "https://auth.gov.local/api/v1/resource?page=1",
    "last": "https://auth.gov.local/api/v1/resource?page=10",
    "prev": null,
    "next": "https://auth.gov.local/api/v1/resource?page=2"
  }
}
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Government Context |
|------|---------|-------------------|
| `200` | OK | Successful request |
| `201` | Created | Resource created (new application, account) |
| `204` | No Content | Successful deletion |
| `400` | Bad Request | Validation errors |
| `401` | Unauthorized | Missing/invalid/expired token |
| `403` | Forbidden | Insufficient permissions for action |
| `404` | Not Found | Resource doesn't exist |
| `409` | Conflict | Duplicate record (email already registered) |
| `422` | Unprocessable Entity | Business logic error |
| `429` | Too Many Requests | Rate limited |
| `500` | Internal Server Error | Server error (logged for review) |

### Error Examples

**401 Unauthorized:**
```json
{
  "success": false,
  "message": "Your session has expired. Please log in again.",
  "errors": null
}
```

**403 Forbidden (Citizen):**
```json
{
  "success": false,
  "message": "You do not have permission to access this resource",
  "errors": {
    "detail": "Viewing other citizens' records requires employee access"
  }
}
```

**403 Forbidden (Employee):**
```json
{
  "success": false,
  "message": "Insufficient permissions",
  "errors": {
    "required_permissions": ["approve applications"],
    "user_permissions": ["view applications", "process applications"],
    "department": "permits"
  }
}
```

**409 Conflict:**
```json
{
  "success": false,
  "message": "Registration failed",
  "errors": {
    "email": ["This email is already registered. Please login or reset your password."]
  }
}
```

**422 Validation Error:**
```json
{
  "success": false,
  "message": "The given data was invalid",
  "errors": {
    "email": ["Please enter a valid email address."],
    "password": ["Password must be at least 12 characters with uppercase, lowercase, and number."],
    "phone": ["Please enter a valid phone number."]
  }
}
```

---

## Rate Limiting

### Default Limits

| Endpoint Type | Limit | Notes |
|---------------|-------|-------|
| Citizen Login | 5 requests/minute/IP | Prevent brute force |
| Employee Login | 5 requests/minute/IP | Prevent brute force |
| Password Reset | 3 requests/hour/email | Prevent abuse |
| Registration | 3 requests/hour/IP | Prevent spam |
| Token Verify | 1000 requests/minute/service | High volume microservice calls |
| Citizen API | 60 requests/minute/user | Normal citizen usage |
| Employee API | 120 requests/minute/user | Higher for employee workload |

### Rate Limit Headers

```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1705312800
Retry-After: 30
```

### Rate Limited Response (429)

```json
{
  "success": false,
  "message": "Too many requests. Please wait before trying again.",
  "errors": {
    "retry_after": 30,
    "retry_after_human": "30 seconds"
  }
}
```

---

## Security Headers

All API responses include:

```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'none'
Cache-Control: no-store, no-cache, must-revalidate
```

---

## Government-Specific Endpoints

### GET `/api/v1/my/applications`

Citizen: Get all my submitted applications across services.

**Response:**
```json
{
  "success": true,
  "data": {
    "applications": [
      {
        "id": "PRM-2024-0001",
        "service": "permits-service",
        "type": "Building Permit",
        "status": "under_review",
        "submitted_at": "2024-01-20T10:00:00Z",
        "last_updated": "2024-01-22T14:30:00Z",
        "assigned_to": "Building & Planning Department"
      },
      {
        "id": "REV-2024-0015",
        "service": "revenue-service",
        "type": "Business License",
        "status": "pending_payment",
        "submitted_at": "2024-01-18T09:00:00Z",
        "amount_due": 150.00
      }
    ],
    "total": 2
  }
}
```

### GET `/api/v1/my/payments`

Citizen: Get payment history.

### GET `/api/v1/my/documents`

Citizen: Get uploaded documents.

### POST `/api/v1/my/data-request`

Citizen: Request a copy of all personal data (GDPR/data privacy).

**Request:**
```json
{
  "request_type": "export",
  "format": "pdf"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Your data export request has been submitted. You will receive an email within 30 days.",
  "data": {
    "request_id": "DR-2024-0001",
    "estimated_completion": "2024-02-25"
  }
}
```
