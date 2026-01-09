# Database Schema - Local Government Services Central Authentication System

## 📋 Table of Contents
1. [Overview](#overview)
2. [Entity Relationship Diagram](#entity-relationship-diagram)
3. [Table Definitions](#table-definitions)
4. [Index Strategy](#index-strategy)
5. [Data Types & Constraints](#data-types--constraints)
6. [Migration Order](#migration-order)
7. [Best Practices](#best-practices)

---

## Overview

The database schema is designed for a **Local Government Services Platform** where **citizens** are the primary users accessing various government services. The schema supports:

- **Citizen Identity** - Primary user profile for residents
- **Government Employees** - Staff processing citizen requests
- **Multi-Department Access** - Role-based access per department
- **Service-to-Service Auth** - Microservice authentication
- **Audit Compliance** - Complete logging for government regulations

### Database Requirements

| Requirement | Recommendation |
|-------------|----------------|
| **Engine** | PostgreSQL 14+ (recommended) or MySQL 8+ |
| **Character Set** | UTF-8 (utf8mb4 for MySQL) |
| **Collation** | utf8mb4_unicode_ci |
| **InnoDB** | Required for MySQL (FK support) |
| **Encryption** | At-rest encryption required for citizen data |

### Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Normalization** | Minimize data redundancy |
| **Foreign Key Integrity** | Enforce relationships at DB level |
| **Indexing Strategy** | Optimize common query patterns |
| **Soft Deletes** | Audit trail for compliance |
| **Timestamps** | Track creation/modification |
| **Citizen Privacy** | Separate PII from operational data |

---

## Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           ENTITY RELATIONSHIP DIAGRAM                                │
│                    Local Government Services Auth System                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────┐                                           ┌──────────────┐        │
│  │   sessions   │                                           │    otps      │        │
│  ├──────────────┤                                           ├──────────────┤        │
│  │ id (PK)      │                                           │ id (PK)      │        │
│  │ user_id (FK) │───────────────────┐                       │ email        │        │
│  │ ip_address   │                   │                       │ phone        │        │
│  │ user_agent   │                   │                       │ otp          │        │
│  │ payload      │                   │                       │ type         │        │
│  │ last_activity│                   │                       │ expires_at   │        │
│  └──────────────┘                   │                       │ used         │        │
│                                     │                       └──────────────┘        │
│  ┌──────────────┐                   │                                               │
│  │ user_devices │                   │                       ┌──────────────┐        │
│  ├──────────────┤                   │                       │service_clients│       │
│  │ id (PK)      │                   │                       ├──────────────┤        │
│  │ user_id (FK) │───────────────────┤                       │ id (PK)      │        │
│  │ device_id    │                   │                       │ name         │        │
│  │ platform     │                   │                       │ token_hash   │        │
│  │ model        │                   │                       │ is_active    │        │
│  │ fcm_token    │                   │                       │ expires_at   │        │
│  │ last_seen_at │                   │                       │ notes        │        │
│  └──────────────┘                   │                       │ last_used_at │        │
│                                     ▼                       │ use_count    │        │
│                              ┌──────────────┐               └──────────────┘        │
│                              │   citizens   │  ◄── PRIMARY USER TABLE               │
│                              ├──────────────┤                                       │
│                              │ id (PK)      │                                       │
│                              │ national_id  │  ◄── Government ID (optional)         │
│                              │ name         │                                       │
│                              │ email (UQ)   │                                       │
│                              │ phone        │                                       │
│                              │ password     │                                       │
│                              │ date_of_birth│                                       │
│                              │ address      │                                       │
│                              │ city         │                                       │
│                              │ postal_code  │                                       │
│                              │ email_       │                                       │
│                              │ verified_at  │                                       │
│                              │ phone_       │                                       │
│                              │ verified_at  │                                       │
│                              │ identity_    │                                       │
│                              │ verified_at  │                                       │
│                              │ user_type    │  ◄── 'citizen' or 'employee'          │
│                              │ is_active    │                                       │
│                              └──────┬───────┘                                       │
│                                     │                                               │
│           ┌─────────────────────────┼─────────────────────────┐                     │
│           │                         │                         │                     │
│           ▼                         ▼                         ▼                     │
│  ┌─────────────────┐       ┌─────────────────┐      ┌──────────────────┐           │
│  │ model_has_roles │       │model_has_perms  │      │ user_role_dept   │           │
│  ├─────────────────┤       ├─────────────────┤      ├──────────────────┤           │
│  │ role_id (FK)    │       │ permission_id   │      │ id (PK)          │           │
│  │ model_type      │       │ model_type      │      │ user_id (FK)     │           │
│  │ model_id (FK)   │       │ model_id (FK)   │      │ role_id (FK)     │───┐       │
│  └────────┬────────┘       └────────┬────────┘      │ department_id(FK)│─┐ │       │
│           │                         │               │ metadata (JSON)  │ │ │       │
│           │                         │               │ is_active        │ │ │       │
│           │                         │               └──────────────────┘ │ │       │
│           │                         │                                    │ │       │
│           │                         │               ┌──────────────────┐ │ │       │
│           │                         │               │   departments    │◄┘ │       │
│           │                         │               ├──────────────────┤   │       │
│           │                         │               │ id (PK)          │   │       │
│           │                         │               │ code (UQ)        │   │       │
│           │                         │               │ name (UQ)        │   │       │
│           │                         │               │ description      │   │       │
│           │                         │               │ metadata (JSON)  │   │       │
│           │                         │               │ is_active        │   │       │
│           │                         │               └──────────────────┘   │       │
│           │                         │                         ▲            │       │
│           │                         │                         │            │       │
│           │                         │               ┌──────────────────┐   │       │
│           │                         │               │ role_hierarchy   │   │       │
│           │                         │               ├──────────────────┤   │       │
│           │                         │               │ id (PK)          │   │       │
│           │                         │               │ higher_role_id   │───┤       │
│           │                         │               │ lower_role_id    │───┤       │
│           │                         │               │ department_id(FK)│───┘       │
│           │                         │               │ metadata (JSON)  │           │
│           │                         │               │ is_active        │           │
│           ▼                         ▼               └──────────────────┘           │
│      ┌──────────────┐       ┌──────────────┐                                       │
│      │    roles     │       │ permissions  │                                       │
│      ├──────────────┤       ├──────────────┤                                       │
│      │ id (PK)      │       │ id (PK)      │                                       │
│      │ name         │       │ name         │                                       │
│      │ guard_name   │       │ guard_name   │                                       │
│      │ description  │       │ description  │                                       │
│      │ is_system    │       │ is_system    │                                       │
│      └──────┬───────┘       └──────┬───────┘                                       │
│             │                      │                                               │
│             └──────────┬───────────┘                                               │
│                        ▼                                                           │
│              ┌─────────────────────┐                                               │
│              │ role_has_permissions│                                               │
│              ├─────────────────────┤                                               │
│              │ permission_id (FK)  │                                               │
│              │ role_id (FK)        │                                               │
│              └─────────────────────┘                                               │
│                                                                                    │
│                                                                                    │
│  ┌──────────────────────────┐       ┌──────────────────────────┐                   │
│  │ personal_access_tokens   │       │       auth_rules         │                   │
│  ├──────────────────────────┤       ├──────────────────────────┤                   │
│  │ id (PK)                  │       │ id (PK)                  │                   │
│  │ tokenable_type           │       │ service                  │                   │
│  │ tokenable_id             │       │ method                   │                   │
│  │ name                     │       │ path_dsl                 │                   │
│  │ token (UQ)               │       │ path_regex               │                   │
│  │ abilities (JSON)         │       │ route_name               │                   │
│  │ last_used_at             │       │ roles_any (JSON)         │                   │
│  │ expires_at               │       │ permissions_any (JSON)   │                   │
│  └──────────────────────────┘       │ permissions_all (JSON)   │                   │
│                                     │ is_active                │                   │
│                                     │ priority                 │                   │
│  ┌──────────────────────────┐       └──────────────────────────┘                   │
│  │      audit_logs          │                                                      │
│  ├──────────────────────────┤                                                      │
│  │ id (PK)                  │       ┌──────────────────────────┐                   │
│  │ user_id                  │       │  citizen_verifications   │                   │
│  │ user_type                │       ├──────────────────────────┤                   │
│  │ action                   │       │ id (PK)                  │                   │
│  │ entity_type              │       │ citizen_id (FK)          │                   │
│  │ entity_id                │       │ verification_type        │                   │
│  │ old_values (JSON)        │       │ verification_status      │                   │
│  │ new_values (JSON)        │       │ verified_by              │                   │
│  │ ip_address               │       │ verified_at              │                   │
│  │ user_agent               │       │ document_reference       │                   │
│  │ service_name             │       │ notes                    │                   │
│  │ created_at               │       └──────────────────────────┘                   │
│  └──────────────────────────┘                                                      │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Table Definitions

### 1. `citizens` - Core User Table (Citizens & Employees)

```sql
CREATE TABLE citizens (
    id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    
    -- Identity
    national_id         VARCHAR(50) NULL UNIQUE,       -- Government-issued ID
    name                VARCHAR(255) NOT NULL,
    email               VARCHAR(255) NOT NULL UNIQUE,
    phone               VARCHAR(20) NULL,
    password            VARCHAR(255) NOT NULL,
    
    -- Profile (for citizens)
    date_of_birth       DATE NULL,
    address             VARCHAR(500) NULL,
    city                VARCHAR(100) NULL,
    postal_code         VARCHAR(20) NULL,
    
    -- Verification Status
    email_verified_at   TIMESTAMP NULL,
    phone_verified_at   TIMESTAMP NULL,
    identity_verified_at TIMESTAMP NULL,              -- National ID verified
    
    -- User Type
    user_type           ENUM('citizen', 'employee') DEFAULT 'citizen',
    employee_id         VARCHAR(50) NULL,              -- For gov employees
    department_primary  VARCHAR(255) NULL,             -- Primary department
    
    -- Account Status
    is_active           BOOLEAN DEFAULT TRUE,
    remember_token      VARCHAR(100) NULL,
    
    -- Timestamps
    created_at          TIMESTAMP NULL,
    updated_at          TIMESTAMP NULL,
    deleted_at          TIMESTAMP NULL,                -- Soft delete
    
    INDEX idx_citizens_email (email),
    INDEX idx_citizens_national_id (national_id),
    INDEX idx_citizens_phone (phone),
    INDEX idx_citizens_user_type (user_type),
    INDEX idx_citizens_deleted (deleted_at)
);
```

**Purpose:** Stores identity information for both citizens and government employees.

| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED | Auto-increment primary key |
| `national_id` | VARCHAR(50) | Government-issued ID number (SSN, passport, etc.) |
| `name` | VARCHAR(255) | Full name |
| `email` | VARCHAR(255) | Unique email address (login identifier) |
| `phone` | VARCHAR(20) | Phone number for OTP verification |
| `password` | VARCHAR(255) | Hashed password (Argon2/bcrypt) |
| `date_of_birth` | DATE | Birth date (for citizen verification) |
| `address` | VARCHAR(500) | Street address |
| `city` | VARCHAR(100) | City/municipality |
| `postal_code` | VARCHAR(20) | Postal/ZIP code |
| `email_verified_at` | TIMESTAMP | When email was verified |
| `phone_verified_at` | TIMESTAMP | When phone was verified |
| `identity_verified_at` | TIMESTAMP | When national ID was verified |
| `user_type` | ENUM | 'citizen' or 'employee' |
| `employee_id` | VARCHAR(50) | Government employee ID (for employees) |

**Why single table for citizens and employees?**
- Simplifies authentication (one login endpoint)
- Employees may also be citizens of the jurisdiction
- Unified audit trail
- Role-based differentiation handles access control

---

### 2. `departments` - Government Departments

```sql
CREATE TABLE departments (
    id              VARCHAR(50) PRIMARY KEY,           -- e.g., 'revenue', 'permits'
    code            VARCHAR(20) NOT NULL UNIQUE,       -- Short code: 'REV', 'PRM'
    name            VARCHAR(255) NOT NULL UNIQUE,      -- "Revenue & Finance"
    description     TEXT NULL,
    parent_id       VARCHAR(50) NULL,                  -- For sub-departments
    metadata        JSON NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    INDEX idx_dept_code (code),
    INDEX idx_dept_parent (parent_id),
    INDEX idx_dept_active (is_active),
    
    FOREIGN KEY (parent_id) REFERENCES departments(id) ON DELETE SET NULL
);
```

**Purpose:** Define government departments/agencies.

| Column | Type | Description |
|--------|------|-------------|
| `id` | VARCHAR(50) | Human-readable ID (e.g., 'revenue') |
| `code` | VARCHAR(20) | Short code for display (e.g., 'REV') |
| `name` | VARCHAR(255) | Full department name |
| `description` | TEXT | Description of department functions |
| `parent_id` | VARCHAR(50) | For hierarchical departments |
| `metadata` | JSON | Additional department info |

**Example Departments:**
```sql
INSERT INTO departments (id, code, name) VALUES
('revenue', 'REV', 'Revenue & Finance'),
('permits', 'PRM', 'Building & Planning'),
('public-works', 'PWK', 'Public Works'),
('parks', 'PRK', 'Parks & Recreation'),
('clerk', 'CLK', 'City Clerk'),
('safety', 'SAF', 'Public Safety'),
('hr', 'HR', 'Human Resources'),
('it', 'IT', 'Information Technology');
```

---

### 3. `sessions` - Session Storage

```sql
CREATE TABLE sessions (
    id              VARCHAR(255) PRIMARY KEY,
    user_id         BIGINT UNSIGNED NULL,
    ip_address      VARCHAR(45) NULL,
    user_agent      TEXT NULL,
    payload         LONGTEXT NOT NULL,
    last_activity   INT NOT NULL,
    
    INDEX idx_sessions_user_id (user_id),
    INDEX idx_sessions_last_activity (last_activity)
);
```

**Purpose:** Server-side session storage for web portal applications.

---

### 4. `otps` - One-Time Passwords

```sql
CREATE TABLE otps (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email           VARCHAR(255) NULL,
    phone           VARCHAR(20) NULL,
    otp             VARCHAR(10) NOT NULL,
    type            ENUM('email_verification', 'phone_verification', 
                         'password_reset', 'login_2fa') NOT NULL,
    expires_at      TIMESTAMP NOT NULL,
    used            BOOLEAN DEFAULT FALSE,
    attempts        INT DEFAULT 0,                     -- Rate limiting
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    INDEX idx_otps_email_type (email, type),
    INDEX idx_otps_phone_type (phone, type),
    INDEX idx_otps_expires (expires_at)
);
```

**Purpose:** Stores OTP codes for various verification purposes.

| Column | Type | Description |
|--------|------|-------------|
| `email` | VARCHAR(255) | Target email address |
| `phone` | VARCHAR(20) | Target phone number |
| `otp` | VARCHAR(10) | 6-digit OTP code |
| `type` | ENUM | Purpose of OTP |
| `expires_at` | TIMESTAMP | When OTP expires (typically 10 minutes) |
| `used` | BOOLEAN | Whether OTP has been consumed |
| `attempts` | INT | Failed verification attempts (rate limiting) |

**OTP Types:**
- `email_verification` - New account email verification
- `phone_verification` - Phone number verification
- `password_reset` - Password recovery
- `login_2fa` - Two-factor authentication for employees

---

### 5. `personal_access_tokens` - API Tokens

```sql
CREATE TABLE personal_access_tokens (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tokenable_type  VARCHAR(255) NOT NULL,             -- 'App\Models\Citizen'
    tokenable_id    BIGINT UNSIGNED NOT NULL,
    name            VARCHAR(255) NOT NULL,             -- 'citizen-mobile', 'employee-portal'
    token           VARCHAR(64) NOT NULL UNIQUE,       -- SHA-256 hashed token
    abilities       JSON NULL,                         -- ['*'], ['read-only']
    last_used_at    TIMESTAMP NULL,
    expires_at      TIMESTAMP NULL,
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    INDEX idx_pat_tokenable (tokenable_type, tokenable_id),
    INDEX idx_pat_expires_at (expires_at)
);
```

**Purpose:** API authentication tokens for citizens and employees.

**Token Abilities for Government Services:**
```json
// Citizen abilities (default)
["submit-applications", "view-own-records", "make-payments", "track-status"]

// Employee abilities (based on role)
["view-citizen-data", "process-applications", "approve-requests", "generate-reports"]

// Full access
["*"]
```

---

### 6. `roles` - Role Definitions

```sql
CREATE TABLE roles (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    guard_name      VARCHAR(255) NOT NULL DEFAULT 'web',
    description     TEXT NULL,
    is_system       BOOLEAN DEFAULT FALSE,             -- System roles can't be deleted
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    UNIQUE INDEX idx_roles_name_guard (name, guard_name)
);
```

**Purpose:** Define named role groupings for citizens and employees.

**Government-Specific Roles:**
```sql
-- Citizen Roles
INSERT INTO roles (name, guard_name, description, is_system) VALUES
('citizen', 'citizen', 'Standard citizen account', true),
('verified-citizen', 'citizen', 'Identity-verified citizen', true),
('business-owner', 'citizen', 'Registered business owner', false);

-- Employee Roles (common across departments)
INSERT INTO roles (name, guard_name, description, is_system) VALUES
('super-admin', 'employee', 'Full system access', true),
('department-head', 'employee', 'Department leadership', true),
('supervisor', 'employee', 'Team supervisor', true),
('clerk', 'employee', 'General processing clerk', true),
('inspector', 'employee', 'Field inspector', true),
('case-worker', 'employee', 'Case management', true),
('receptionist', 'employee', 'Front desk staff', true),
('auditor', 'employee', 'Read-only audit access', true);
```

---

### 7. `permissions` - Permission Definitions

```sql
CREATE TABLE permissions (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    guard_name      VARCHAR(255) NOT NULL DEFAULT 'web',
    description     TEXT NULL,
    category        VARCHAR(100) NULL,                 -- Group for UI display
    is_system       BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    UNIQUE INDEX idx_perms_name_guard (name, guard_name),
    INDEX idx_perms_category (category)
);
```

**Purpose:** Define atomic permission units.

**Government-Specific Permissions:**
```sql
-- Citizen Permissions
INSERT INTO permissions (name, guard_name, category, description) VALUES
('submit applications', 'citizen', 'applications', 'Submit new applications'),
('view own records', 'citizen', 'records', 'View own submitted records'),
('make payments', 'citizen', 'payments', 'Make online payments'),
('track status', 'citizen', 'applications', 'Track application status'),
('update profile', 'citizen', 'profile', 'Update own profile'),
('request records', 'citizen', 'records', 'Request public records');

-- Employee Permissions - Citizens
INSERT INTO permissions (name, guard_name, category, description) VALUES
('view citizen profiles', 'employee', 'citizens', 'View citizen information'),
('edit citizen profiles', 'employee', 'citizens', 'Edit citizen information'),
('verify citizen identity', 'employee', 'citizens', 'Verify national ID');

-- Employee Permissions - Applications
INSERT INTO permissions (name, guard_name, category, description) VALUES
('view applications', 'employee', 'applications', 'View submitted applications'),
('process applications', 'employee', 'applications', 'Process/update applications'),
('approve applications', 'employee', 'applications', 'Approve applications'),
('deny applications', 'employee', 'applications', 'Deny applications'),
('assign applications', 'employee', 'applications', 'Assign to other employees');

-- Employee Permissions - Reports
INSERT INTO permissions (name, guard_name, category, description) VALUES
('view reports', 'employee', 'reports', 'View department reports'),
('generate reports', 'employee', 'reports', 'Generate custom reports'),
('export data', 'employee', 'reports', 'Export data to files');

-- Employee Permissions - Admin
INSERT INTO permissions (name, guard_name, category, description) VALUES
('manage employees', 'employee', 'admin', 'Manage employee accounts'),
('manage roles', 'employee', 'admin', 'Manage roles and permissions'),
('manage departments', 'employee', 'admin', 'Manage department settings'),
('view audit logs', 'employee', 'admin', 'View system audit logs'),
('manage service clients', 'employee', 'admin', 'Manage API service clients');
```

---

### 8. `role_has_permissions` - Role-Permission Junction

```sql
CREATE TABLE role_has_permissions (
    permission_id   BIGINT UNSIGNED NOT NULL,
    role_id         BIGINT UNSIGNED NOT NULL,
    
    PRIMARY KEY (permission_id, role_id),
    
    FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
);
```

**Purpose:** Many-to-many relationship between roles and permissions.

---

### 9. `model_has_roles` - Global Role Assignment

```sql
CREATE TABLE model_has_roles (
    role_id         BIGINT UNSIGNED NOT NULL,
    model_type      VARCHAR(255) NOT NULL,
    model_id        BIGINT UNSIGNED NOT NULL,
    
    PRIMARY KEY (role_id, model_id, model_type),
    
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    
    INDEX idx_mhr_model (model_id, model_type)
);
```

**Purpose:** Assign global roles to users (not department-specific).

**Usage:**
- Citizens get `citizen` or `verified-citizen` role here
- Super-admins get global `super-admin` role here
- Department-specific roles use `user_role_dept` table

---

### 10. `model_has_permissions` - Direct Permission Assignment

```sql
CREATE TABLE model_has_permissions (
    permission_id   BIGINT UNSIGNED NOT NULL,
    model_type      VARCHAR(255) NOT NULL,
    model_id        BIGINT UNSIGNED NOT NULL,
    
    PRIMARY KEY (permission_id, model_id, model_type),
    
    FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE,
    
    INDEX idx_mhp_model (model_id, model_type)
);
```

**Purpose:** Assign permissions directly to users (bypassing roles).

---

### 11. `user_role_dept` - User-Role-Department Assignment

```sql
CREATE TABLE user_role_dept (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    role_id         BIGINT UNSIGNED NOT NULL,
    department_id   VARCHAR(50) NOT NULL,
    
    assigned_by     BIGINT UNSIGNED NULL,              -- Who assigned this role
    assigned_at     TIMESTAMP NULL,
    expires_at      TIMESTAMP NULL,                    -- Temporary assignments
    metadata        JSON NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    UNIQUE INDEX idx_urd_unique (user_id, role_id, department_id),
    INDEX idx_urd_dept_role (department_id, role_id),
    INDEX idx_urd_expires (expires_at),
    
    FOREIGN KEY (user_id) REFERENCES citizens(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE CASCADE,
    FOREIGN KEY (assigned_by) REFERENCES citizens(id) ON DELETE SET NULL
);
```

**Purpose:** Assign roles to employees within specific departments.

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | BIGINT UNSIGNED | Employee receiving the role |
| `role_id` | BIGINT UNSIGNED | Role being assigned |
| `department_id` | VARCHAR(50) | Department context for the role |
| `assigned_by` | BIGINT UNSIGNED | Who made the assignment |
| `expires_at` | TIMESTAMP | For temporary assignments |

**Example:**
```sql
-- Sarah is a clerk in Revenue and an inspector in Permits
INSERT INTO user_role_dept (user_id, role_id, department_id) VALUES
(5, 4, 'revenue'),    -- Sarah as 'clerk' in 'Revenue'
(5, 5, 'permits');    -- Sarah as 'inspector' in 'Permits'
```

---

### 12. `role_hierarchy` - Role Inheritance Per Department

```sql
CREATE TABLE role_hierarchy (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    higher_role_id  BIGINT UNSIGNED NOT NULL,
    lower_role_id   BIGINT UNSIGNED NOT NULL,
    department_id   VARCHAR(50) NOT NULL,
    metadata        JSON NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    UNIQUE INDEX idx_rh_unique (higher_role_id, lower_role_id, department_id),
    INDEX idx_rh_dept_higher (department_id, higher_role_id),
    
    FOREIGN KEY (higher_role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (lower_role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE CASCADE
);
```

**Purpose:** Define role management hierarchies per department.

**Example Hierarchy (Revenue Department):**
```sql
INSERT INTO role_hierarchy (higher_role_id, lower_role_id, department_id) VALUES
(2, 3, 'revenue'),    -- department-head > supervisor
(3, 4, 'revenue'),    -- supervisor > clerk
(3, 6, 'revenue');    -- supervisor > case-worker
```

---

### 13. `service_clients` - Microservice Authentication

```sql
CREATE TABLE service_clients (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(255) NOT NULL UNIQUE,      -- 'permits-service', 'revenue-service'
    display_name    VARCHAR(255) NULL,                 -- Human-friendly name
    token_hash      VARCHAR(128) NOT NULL,             -- SHA-256 hex
    
    department_id   VARCHAR(50) NULL,                  -- Associated department
    allowed_paths   JSON NULL,                         -- Restrict to certain paths
    rate_limit      INT DEFAULT 1000,                  -- Requests per minute
    
    is_active       BOOLEAN DEFAULT TRUE,
    expires_at      TIMESTAMP NULL,
    notes           VARCHAR(512) NULL,
    
    last_used_at    TIMESTAMP NULL,
    use_count       BIGINT UNSIGNED DEFAULT 0,
    
    created_by      BIGINT UNSIGNED NULL,
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    INDEX idx_sc_name (name),
    INDEX idx_sc_active (is_active),
    INDEX idx_sc_dept (department_id),
    
    FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE SET NULL,
    FOREIGN KEY (created_by) REFERENCES citizens(id) ON DELETE SET NULL
);
```

**Purpose:** Register and authenticate government microservices.

**Example Service Clients:**
```sql
INSERT INTO service_clients (name, display_name, department_id, token_hash) VALUES
('permits-service', 'Permits & Licensing Service', 'permits', 'sha256_hash_here'),
('revenue-service', 'Revenue & Tax Service', 'revenue', 'sha256_hash_here'),
('public-works-service', 'Public Works Service', 'public-works', 'sha256_hash_here'),
('parks-service', 'Parks & Recreation Service', 'parks', 'sha256_hash_here'),
('notification-service', 'Notification Service', NULL, 'sha256_hash_here'),
('payment-gateway', 'Payment Processing Gateway', NULL, 'sha256_hash_here');
```

---

### 14. `auth_rules` - Dynamic Authorization Rules

```sql
CREATE TABLE auth_rules (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    
    -- Matching criteria
    service         VARCHAR(255) NOT NULL,             -- Service name
    method          VARCHAR(10) DEFAULT 'GET',         -- HTTP method or 'ANY'
    path_dsl        VARCHAR(255) NULL,                 -- Human-friendly pattern
    path_regex      VARCHAR(255) NULL,                 -- Compiled regex
    route_name      VARCHAR(255) NULL,                 -- Named route match
    
    -- Authorization requirements
    roles_any       JSON NULL,                         -- ["clerk", "supervisor"]
    permissions_any JSON NULL,                         -- ["view applications"]
    permissions_all JSON NULL,                         -- ["process applications", "approve applications"]
    
    -- For citizens vs employees
    user_type       ENUM('any', 'citizen', 'employee') DEFAULT 'any',
    
    -- Rule config
    is_active       BOOLEAN DEFAULT TRUE,
    priority        INT DEFAULT 100,
    description     TEXT NULL,
    
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    INDEX idx_ar_service_method (service, method),
    INDEX idx_ar_service_route (service, route_name),
    INDEX idx_ar_service_priority (service, priority)
);
```

**Purpose:** Define per-service, per-route authorization rules.

**Example Rules for Permits Service:**
```sql
INSERT INTO auth_rules (service, method, path_dsl, user_type, permissions_any, description) VALUES
-- Citizen endpoints
('permits-service', 'POST', '/applications', 'citizen', '["submit applications"]', 'Citizens submit permit applications'),
('permits-service', 'GET', '/applications/{id}', 'citizen', '["view own records"]', 'Citizens view their applications'),
('permits-service', 'GET', '/applications/{id}/status', 'citizen', '["track status"]', 'Citizens track application status'),

-- Employee endpoints
('permits-service', 'GET', '/admin/applications', 'employee', '["view applications"]', 'Staff view all applications'),
('permits-service', 'PUT', '/admin/applications/{id}', 'employee', '["process applications"]', 'Staff process applications'),
('permits-service', 'POST', '/admin/applications/{id}/approve', 'employee', '["approve applications"]', 'Staff approve applications'),
('permits-service', 'POST', '/admin/applications/{id}/deny', 'employee', '["deny applications"]', 'Staff deny applications'),
('permits-service', 'POST', '/admin/applications/{id}/assign', 'employee', '["assign applications"]', 'Staff assign to others'),

-- Inspection endpoints
('permits-service', 'GET', '/inspections', 'employee', '["view applications"]', 'Inspectors view assigned inspections'),
('permits-service', 'POST', '/inspections/{id}/complete', 'employee', '["process applications"]', 'Inspectors complete inspections');
```

---

### 15. `user_devices` - Device Registration

```sql
CREATE TABLE user_devices (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    device_id       VARCHAR(255) NULL,
    platform        VARCHAR(50) NULL,                  -- ios, android, web
    model           VARCHAR(100) NULL,
    os_version      VARCHAR(50) NULL,
    app_version     VARCHAR(50) NULL,
    fcm_token       VARCHAR(500) NULL,                 -- Push notifications
    last_seen_at    TIMESTAMP NULL,
    last_ip         VARCHAR(45) NULL,
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    
    UNIQUE INDEX idx_ud_user_device (user_id, device_id),
    INDEX idx_ud_device_id (device_id),
    INDEX idx_ud_fcm_token (fcm_token),
    
    FOREIGN KEY (user_id) REFERENCES citizens(id) ON DELETE CASCADE
);
```

**Purpose:** Track citizen and employee devices for notifications and security.

---

### 16. `citizen_verifications` - Identity Verification Records

```sql
CREATE TABLE citizen_verifications (
    id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    citizen_id          BIGINT UNSIGNED NOT NULL,
    
    verification_type   ENUM('email', 'phone', 'identity', 'address', 'business') NOT NULL,
    verification_status ENUM('pending', 'verified', 'rejected', 'expired') DEFAULT 'pending',
    
    -- Verification details
    submitted_value     VARCHAR(255) NULL,             -- What was submitted for verification
    document_reference  VARCHAR(255) NULL,             -- Reference to uploaded document
    
    -- Processing
    verified_by         BIGINT UNSIGNED NULL,          -- Employee who verified
    verified_at         TIMESTAMP NULL,
    rejection_reason    TEXT NULL,
    
    -- Validity
    valid_until         TIMESTAMP NULL,                -- Some verifications expire
    
    notes               TEXT NULL,
    created_at          TIMESTAMP NULL,
    updated_at          TIMESTAMP NULL,
    
    INDEX idx_cv_citizen (citizen_id),
    INDEX idx_cv_type_status (verification_type, verification_status),
    
    FOREIGN KEY (citizen_id) REFERENCES citizens(id) ON DELETE CASCADE,
    FOREIGN KEY (verified_by) REFERENCES citizens(id) ON DELETE SET NULL
);
```

**Purpose:** Track citizen identity verification history for compliance.

---

### 17. `audit_logs` - Comprehensive Audit Trail

```sql
CREATE TABLE audit_logs (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    
    -- Who
    user_id         BIGINT UNSIGNED NULL,
    user_type       ENUM('citizen', 'employee', 'service', 'system') NOT NULL,
    user_email      VARCHAR(255) NULL,                 -- Denormalized for query
    
    -- What
    action          VARCHAR(255) NOT NULL,             -- 'login', 'view_citizen', 'approve_permit'
    entity_type     VARCHAR(255) NULL,                 -- 'Citizen', 'Application', 'Payment'
    entity_id       VARCHAR(255) NULL,
    
    -- Details
    old_values      JSON NULL,
    new_values      JSON NULL,
    description     TEXT NULL,
    
    -- Context
    ip_address      VARCHAR(45) NULL,
    user_agent      TEXT NULL,
    service_name    VARCHAR(255) NULL,                 -- Which microservice
    department_id   VARCHAR(50) NULL,
    
    -- Metadata
    metadata        JSON NULL,
    severity        ENUM('info', 'warning', 'critical') DEFAULT 'info',
    
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_audit_user (user_id),
    INDEX idx_audit_action (action),
    INDEX idx_audit_entity (entity_type, entity_id),
    INDEX idx_audit_created (created_at),
    INDEX idx_audit_severity (severity),
    INDEX idx_audit_service (service_name),
    INDEX idx_audit_dept (department_id)
) ENGINE=InnoDB;
-- Consider partitioning by date for large deployments
```

**Purpose:** Comprehensive audit logging for government compliance.

**Audit Actions to Log:**
```
Authentication:
- login_success, login_failure
- logout
- password_change, password_reset
- otp_sent, otp_verified, otp_failed

Citizen Data Access:
- view_citizen_profile
- edit_citizen_profile
- verify_citizen_identity

Application Processing:
- view_application
- process_application
- approve_application
- deny_application
- assign_application

Administrative:
- create_employee
- modify_employee_role
- modify_permissions
- create_service_client
```

---

## Index Strategy

### Primary Query Patterns

| Query Pattern | Optimal Index |
|---------------|---------------|
| Login by email | `citizens.email` (unique) |
| Login by phone | `citizens.phone` |
| Find by national ID | `citizens.national_id` (unique) |
| Token lookup | `personal_access_tokens.token` (unique) |
| User's global roles | `model_has_roles.model_id + model_type` |
| Roles in department | `user_role_dept.department_id + role_id` |
| Service auth | `service_clients.name` |
| Auth rules | `auth_rules.service + method + priority` |
| Audit by date | `audit_logs.created_at` |
| Audit by citizen | `audit_logs.entity_type + entity_id` |

### Composite Indexes

```sql
-- User role department lookups
CREATE INDEX idx_urd_user_dept ON user_role_dept(user_id, department_id, is_active);

-- Role hierarchy traversal
CREATE INDEX idx_rh_traverse ON role_hierarchy(department_id, higher_role_id, is_active);

-- Auth rule matching
CREATE INDEX idx_ar_match ON auth_rules(service, method, is_active, priority DESC);

-- Audit queries
CREATE INDEX idx_audit_citizen_access ON audit_logs(entity_type, entity_id, created_at DESC);
CREATE INDEX idx_audit_employee_actions ON audit_logs(user_id, created_at DESC);
```

---

## Migration Order

Execute migrations in this order to respect foreign key dependencies:

```
1. citizens                     -- Core users table
2. departments                  -- Government departments
3. sessions                     -- Session storage
4. otps                         -- OTP codes
5. permissions                  -- Permission definitions
6. roles                        -- Role definitions
7. role_has_permissions         -- Role-permission junction
8. model_has_permissions        -- Direct user permissions
9. model_has_roles              -- Global role assignment
10. personal_access_tokens      -- API tokens
11. user_role_dept              -- Department role assignment
12. role_hierarchy              -- Role hierarchy
13. service_clients             -- Microservice auth
14. auth_rules                  -- Authorization rules
15. user_devices                -- Device tracking
16. citizen_verifications       -- Identity verification
17. audit_logs                  -- Audit trail
```

---

## Best Practices

### 1. Citizen Data Protection

```sql
-- Consider encryption for sensitive fields
-- Use database-level encryption or application-level
-- Example: national_id could be encrypted at rest

-- Data retention policy
-- Implement automatic deletion of old OTPs
DELETE FROM otps WHERE expires_at < NOW() - INTERVAL 30 DAY;

-- Audit log retention (keep for compliance period)
-- Partition audit_logs by month for easier management
```

### 2. Soft Deletes for Citizens

```sql
-- Never hard delete citizen records
-- Use deleted_at for soft deletes
-- Maintain audit trail even for deleted accounts

-- Query active citizens only
SELECT * FROM citizens WHERE deleted_at IS NULL;
```

### 3. Audit Everything

```sql
-- Log all access to citizen data
-- Include who, what, when, where (IP)
-- Store both old and new values for changes
-- Use severity levels for alerting
```

### 4. Token Security

```sql
-- Always hash tokens before storage
-- Use SHA-256 for token hashing
-- Implement token rotation for service clients
-- Set appropriate expiration times
```

### 5. Department Isolation

```sql
-- Employees should only see data for their departments
-- Implement row-level security where possible
-- Use department_id in all employee queries
```

### 6. Performance Considerations

```sql
-- Partition audit_logs by date
ALTER TABLE audit_logs PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027)
);

-- Archive old sessions
DELETE FROM sessions WHERE last_activity < UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY);

-- Regular ANALYZE for query optimization
ANALYZE TABLE citizens, auth_rules, audit_logs;
```

---

## Government Compliance Notes

### Data Protection Requirements

1. **Encryption at Rest** - All citizen PII must be encrypted
2. **Encryption in Transit** - TLS 1.2+ required
3. **Access Logging** - All access to citizen data logged
4. **Data Minimization** - Only collect necessary information
5. **Right to Access** - Citizens can request their data
6. **Right to Correction** - Citizens can request corrections
7. **Data Retention** - Follow local government retention policies
8. **Breach Notification** - Have procedures in place

### Audit Requirements

1. **Complete Trail** - Every action logged
2. **Tamper-Proof** - Audit logs should be append-only
3. **Long Retention** - Keep logs per compliance requirements
4. **Searchable** - Support audit investigations
5. **Reportable** - Generate compliance reports
