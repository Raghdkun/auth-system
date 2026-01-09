# Security Best Practices - Local Government Services Auth System

## 📋 Table of Contents
1. [Security Principles](#security-principles)
2. [Government Compliance Requirements](#government-compliance-requirements)
3. [Password Security](#password-security)
4. [Token Security](#token-security)
5. [API Security](#api-security)
6. [Citizen Data Protection](#citizen-data-protection)
7. [Attack Prevention](#attack-prevention)
8. [Audit & Compliance](#audit--compliance)
9. [Incident Response](#incident-response)

---

## Security Principles

### Defense in Depth for Government Systems

```
┌────────────────────────────────────────────────────────────────┐
│          DEFENSE IN DEPTH - GOVERNMENT SERVICES                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Layer 1: Network Security                                     │
│  ├── Government-grade firewall rules                           │
│  ├── VPN/Private government network                            │
│  ├── DDoS protection (critical infrastructure)                 │
│  ├── TLS 1.2+ encryption (mandatory)                           │
│  └── Network segmentation (departments)                        │
│                                                                │
│  Layer 2: Application Security                                 │
│  ├── Input validation (all citizen inputs)                     │
│  ├── Output encoding (prevent XSS)                             │
│  ├── Authentication middleware                                 │
│  ├── Authorization checks (role + department)                  │
│  └── API rate limiting                                         │
│                                                                │
│  Layer 3: Data Security (Citizen PII)                          │
│  ├── Encryption at rest (AES-256)                              │
│  ├── Encryption in transit (TLS 1.2+)                          │
│  ├── Secure key management (HSM/KMS)                           │
│  ├── Data masking in logs                                      │
│  └── Database-level encryption                                 │
│                                                                │
│  Layer 4: Monitoring & Detection                               │
│  ├── Intrusion detection systems                               │
│  ├── Anomaly detection (citizen data access)                   │
│  ├── Complete audit logging                                    │
│  ├── Real-time alerting                                        │
│  └── SIEM integration                                          │
│                                                                │
│  Layer 5: Physical Security (Government)                       │
│  ├── Secure data center facilities                             │
│  ├── Access controls for server rooms                          │
│  ├── Secure disposal of hardware                               │
│  └── Geographic redundancy                                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Principle of Least Privilege

```
┌────────────────────────────────────────────────────────────────┐
│       LEAST PRIVILEGE FOR GOVERNMENT EMPLOYEES                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Citizens:                                                     │
│  • Access only their own records                               │
│  • Cannot view other citizens' data                            │
│  • Limited to citizen-appropriate permissions                  │
│  • Verified citizens get additional permissions                │
│                                                                │
│  Government Employees:                                         │
│  • Access only their department's data                         │
│  • Permissions scoped to department                            │
│  • Role hierarchy enforced within department                   │
│  • Regular access reviews (quarterly)                          │
│  • Access removed immediately on departure                     │
│  • Just-in-time access for elevated privileges                 │
│                                                                │
│  Service Accounts:                                             │
│  • Scoped to specific department service                       │
│  • Time-limited tokens with auto-expiry                        │
│  • Separate tokens per environment                             │
│  • Regular rotation (annually minimum)                         │
│                                                                │
│  Administrators:                                               │
│  • Require 2FA for all admin actions                           │
│  • Admin actions logged with extra detail                      │
│  • Separate admin accounts (no dual-use)                       │
│  • Time-based access restrictions                              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Government Compliance Requirements

### Regulatory Framework

```
┌────────────────────────────────────────────────────────────────┐
│            GOVERNMENT COMPLIANCE REQUIREMENTS                   │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  FEDERAL REQUIREMENTS (if applicable):                         │
│  ├── FedRAMP (Federal Risk and Authorization)                  │
│  ├── FISMA (Federal Information Security Management Act)       │
│  ├── CJIS (Criminal Justice Information Services)              │
│  └── HIPAA (if health data involved)                           │
│                                                                │
│  STATE/LOCAL REQUIREMENTS:                                     │
│  ├── State data protection laws                                │
│  ├── Public records retention requirements                     │
│  ├── Freedom of Information Act (FOIA)                         │
│  └── State privacy regulations                                 │
│                                                                │
│  GENERAL DATA PROTECTION:                                      │
│  ├── GDPR/CCPA principles (where applicable)                   │
│  ├── Data minimization                                         │
│  ├── Purpose limitation                                        │
│  ├── Storage limitation                                        │
│  └── Transparency                                              │
│                                                                │
│  ACCESSIBILITY:                                                │
│  ├── Section 508 compliance                                    │
│  ├── WCAG 2.1 AA standards                                     │
│  └── Accessible error messages                                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Data Classification for Government

| Classification | Examples | Protection Requirements |
|---------------|----------|-------------------------|
| **Public** | Business hours, department names | Standard web security |
| **Internal** | Employee names, office locations | Authentication required |
| **Confidential** | Citizen PII, applications | Encryption + access control |
| **Restricted** | SSN, financial data, investigations | Max protection, audit logging |

### Citizen Data Categories

```
┌────────────────────────────────────────────────────────────────┐
│                CITIZEN DATA PROTECTION LEVELS                   │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  HIGHLY SENSITIVE (Maximum Protection):                        │
│  • Social Security Numbers / National ID                       │
│  • Financial account numbers                                   │
│  • Tax information                                             │
│  • Criminal records                                            │
│  • Medical information                                         │
│  • Immigration status                                          │
│  Requirements: Encrypted, masked in logs, limited access       │
│                                                                │
│  SENSITIVE (High Protection):                                  │
│  • Full name + address combination                             │
│  • Date of birth                                               │
│  • Phone numbers                                               │
│  • Email addresses                                             │
│  • Employment information                                      │
│  Requirements: Encrypted at rest, access logged                │
│                                                                │
│  STANDARD (Normal Protection):                                 │
│  • Application status                                          │
│  • Permit numbers                                              │
│  • Payment amounts (not account details)                       │
│  • Communication preferences                                   │
│  Requirements: Access control, audit trail                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Password Security

### Hashing Requirements

```
┌────────────────────────────────────────────────────────────────┐
│                    PASSWORD HASHING                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  REQUIRED: Argon2id (OWASP/NIST recommended)                   │
│  ═══════════════════════════════════════════                   │
│                                                                │
│  Parameters for Citizen Passwords:                             │
│  • Memory: 64 MB (65536 KB)                                    │
│  • Iterations: 3                                               │
│  • Parallelism: 4                                              │
│  • Salt: 16 bytes (auto-generated)                             │
│  • Hash length: 32 bytes                                       │
│                                                                │
│  Parameters for Employee Passwords (Higher Security):          │
│  • Memory: 128 MB                                              │
│  • Iterations: 4                                               │
│  • Parallelism: 4                                              │
│  • Salt: 16 bytes                                              │
│  • Hash length: 32 bytes                                       │
│                                                                │
│  ALTERNATIVE: bcrypt (legacy systems)                          │
│  • Cost factor: 12 (minimum)                                   │
│                                                                │
│  ❌ PROHIBITED:                                                │
│  • MD5, SHA-1, SHA-256 without salt                            │
│  • Plain text storage                                          │
│  • Reversible encryption                                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Password Policy for Government Services

```
┌────────────────────────────────────────────────────────────────┐
│                    PASSWORD POLICY                              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  CITIZEN ACCOUNTS:                                             │
│  • Minimum length: 12 characters                               │
│  • Maximum length: 128 characters                              │
│  • Check against breached password database                    │
│  • Password strength meter (zxcvbn, score >= 3)                │
│  • No forced periodic changes (per NIST 800-63B)               │
│  • Allow all characters including spaces                       │
│                                                                │
│  EMPLOYEE ACCOUNTS (Higher Requirements):                      │
│  • Minimum length: 14 characters                               │
│  • Maximum length: 128 characters                              │
│  • Check against breached passwords + common patterns          │
│  • Password strength score >= 4                                │
│  • 2FA mandatory for all employees                             │
│  • Annual password change (security policy)                    │
│  • No password reuse (last 12 passwords)                       │
│                                                                │
│  ADMIN ACCOUNTS:                                               │
│  • Minimum length: 16 characters                               │
│  • Mandatory 2FA (hardware key preferred)                      │
│  • Separate from regular employee account                      │
│  • Session timeout: 15 minutes                                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Implementation Example

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

class GovPasswordHasher:
    """Government-compliant password hasher."""
    
    # Citizen settings
    CITIZEN_HASHER = PasswordHasher(
        time_cost=3,
        memory_cost=65536,
        parallelism=4,
        hash_len=32,
        salt_len=16
    )
    
    # Employee settings (higher security)
    EMPLOYEE_HASHER = PasswordHasher(
        time_cost=4,
        memory_cost=131072,  # 128 MB
        parallelism=4,
        hash_len=32,
        salt_len=16
    )
    
    @classmethod
    def hash_citizen_password(cls, password: str) -> str:
        return cls.CITIZEN_HASHER.hash(password)
    
    @classmethod
    def hash_employee_password(cls, password: str) -> str:
        return cls.EMPLOYEE_HASHER.hash(password)
    
    @classmethod
    def verify_password(cls, hash: str, password: str) -> bool:
        """Verify password against hash (auto-detects settings)."""
        try:
            # Argon2 hash contains parameters, so standard verify works
            PasswordHasher().verify(hash, password)
            return True
        except VerifyMismatchError:
            return False
    
    @classmethod
    def needs_rehash(cls, hash: str, user_type: str) -> bool:
        """Check if password needs rehashing due to policy change."""
        hasher = cls.EMPLOYEE_HASHER if user_type == 'employee' else cls.CITIZEN_HASHER
        return hasher.check_needs_rehash(hash)
```

---

## Token Security

### Token Generation for Government Services

```
┌────────────────────────────────────────────────────────────────┐
│                    SECURE TOKEN GENERATION                      │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Citizen Tokens:                                               │
│  ═══════════════                                               │
│  • 256 bits (32 bytes) of entropy                              │
│  • SHA-256 hash for storage                                    │
│  • 24-hour expiration (web)                                    │
│  • 7-day expiration (mobile with refresh)                      │
│  • Revoke on password change                                   │
│                                                                │
│  Employee Tokens:                                              │
│  ════════════════                                              │
│  • 256 bits of entropy                                         │
│  • 8-hour expiration (work day)                                │
│  • Require 2FA for issuance                                    │
│  • Auto-revoke at end of day (optional)                        │
│  • Revoke on any security event                                │
│                                                                │
│  Service Tokens:                                               │
│  ═══════════════                                               │
│  • 384+ bits (48 bytes) of entropy                             │
│  • SHA-256 hash for storage                                    │
│  • 1-year expiration with rotation                             │
│  • Unique per service per environment                          │
│  • Logged usage (count, last used)                             │
│                                                                │
│  OTP Codes (2FA):                                              │
│  ═══════════════                                               │
│  • 6 digits (cryptographically random)                         │
│  • 10-minute expiration                                        │
│  • Max 3 verification attempts                                 │
│  • New OTP invalidates previous                                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Token Implementation

```python
import secrets
import hashlib
import base64
from datetime import datetime, timedelta

class GovTokenGenerator:
    """Government-compliant token generator."""
    
    @staticmethod
    def generate_citizen_token():
        """Generate citizen access token (256-bit)."""
        random_bytes = secrets.token_bytes(32)
        plain_token = base64.urlsafe_b64encode(random_bytes).decode('utf-8')
        token_hash = hashlib.sha256(plain_token.encode()).hexdigest()
        return {
            'plain': plain_token,
            'hash': token_hash,
            'expires_at': datetime.utcnow() + timedelta(hours=24)
        }
    
    @staticmethod
    def generate_employee_token():
        """Generate employee access token (256-bit, shorter expiry)."""
        random_bytes = secrets.token_bytes(32)
        plain_token = base64.urlsafe_b64encode(random_bytes).decode('utf-8')
        token_hash = hashlib.sha256(plain_token.encode()).hexdigest()
        return {
            'plain': plain_token,
            'hash': token_hash,
            'expires_at': datetime.utcnow() + timedelta(hours=8)  # Work day
        }
    
    @staticmethod
    def generate_service_token():
        """Generate service token (384-bit)."""
        random_bytes = secrets.token_bytes(48)
        plain_token = base64.urlsafe_b64encode(random_bytes).decode('utf-8')
        token_hash = hashlib.sha256(plain_token.encode()).hexdigest()
        return {
            'plain': plain_token,
            'hash': token_hash,
            'expires_at': datetime.utcnow() + timedelta(days=365)
        }
    
    @staticmethod
    def generate_2fa_otp():
        """Generate 6-digit OTP for 2FA."""
        otp = f"{secrets.randbelow(1000000):06d}"
        return {
            'otp': otp,
            'expires_at': datetime.utcnow() + timedelta(minutes=10),
            'max_attempts': 3
        }
    
    @staticmethod
    def verify_token(plain_token: str, stored_hash: str) -> bool:
        """Constant-time token verification."""
        computed_hash = hashlib.sha256(plain_token.encode()).hexdigest()
        return secrets.compare_digest(computed_hash, stored_hash)
```

### Token Storage Guidelines

| Storage | Citizens | Employees | Notes |
|---------|----------|-----------|-------|
| HTTP-only Cookie | ✅ Web | ✅ Web | Secure, SameSite=Strict |
| iOS Keychain | ✅ Mobile | ✅ Mobile | Encrypted at OS level |
| Android Keystore | ✅ Mobile | ✅ Mobile | Hardware-backed |
| localStorage | ❌ Never | ❌ Never | XSS vulnerable |
| sessionStorage | ⚠️ Limited | ⚠️ Limited | Lost on tab close |

---

## API Security

### Request Validation for Government APIs

```
┌────────────────────────────────────────────────────────────────┐
│           REQUEST VALIDATION - GOVERNMENT APIs                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Headers:                                                      │
│  □ Validate Authorization header format                        │
│  □ Validate Content-Type matches expected                      │
│  □ Check X-Request-ID for tracing                              │
│  □ Validate department headers (employees)                     │
│                                                                │
│  Citizen Input Validation:                                     │
│  □ Validate email format                                       │
│  □ Validate phone number format                                │
│  □ Validate national ID format (if provided)                   │
│  □ Sanitize name fields (prevent XSS)                          │
│  □ Validate date formats                                       │
│  □ Validate address components                                 │
│  □ Limit file upload sizes and types                           │
│                                                                │
│  Employee Input Validation:                                    │
│  □ Validate employee ID format                                 │
│  □ Validate department assignment                              │
│  □ Check action authorization                                  │
│  □ Validate reason/notes fields                                │
│                                                                │
│  Query Parameters:                                             │
│  □ Limit pagination (max 100 per page)                         │
│  □ Validate search queries (prevent injection)                 │
│  □ Validate filter values                                      │
│  □ Validate date ranges                                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Rate Limiting for Government Services

```
┌────────────────────────────────────────────────────────────────┐
│                 RATE LIMITING - GOV SERVICES                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Endpoint                      │ Limit         │ Window        │
│  ═══════════════════════════════════════════════════════════   │
│  POST /auth/citizen/login      │ 5 requests    │ per minute    │
│  POST /auth/citizen/register   │ 3 requests    │ per hour/IP   │
│  POST /auth/employee/login     │ 5 requests    │ per minute    │
│  POST /auth/forgot-password    │ 3 requests    │ per hour      │
│  POST /auth/verify-otp         │ 3 attempts    │ per OTP       │
│  POST /auth/token-verify       │ 1000 requests │ per minute    │
│  GET  /citizens (employee)     │ 60 requests   │ per minute    │
│  GET  /my/* (citizen)          │ 100 requests  │ per minute    │
│  POST /applications            │ 10 requests   │ per hour      │
│  * (default citizen)           │ 60 requests   │ per minute    │
│  * (default employee)          │ 120 requests  │ per minute    │
│                                                                │
│  Rate Limit Keys:                                              │
│  • Citizen: per user_id + IP                                   │
│  • Employee: per user_id                                       │
│  • Unauthenticated: per IP                                     │
│  • Service: per service_name                                   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Security Headers

```python
# Required security headers for government services
SECURITY_HEADERS = {
    # Prevent MIME type sniffing
    "X-Content-Type-Options": "nosniff",
    
    # Prevent clickjacking
    "X-Frame-Options": "DENY",
    
    # Enable XSS filter
    "X-XSS-Protection": "1; mode=block",
    
    # Force HTTPS
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",
    
    # Content Security Policy (adjust per app)
    "Content-Security-Policy": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'",
    
    # Prevent referrer leakage
    "Referrer-Policy": "strict-origin-when-cross-origin",
    
    # Permissions policy
    "Permissions-Policy": "geolocation=(), microphone=(), camera=()",
    
    # No caching for sensitive data
    "Cache-Control": "no-store, no-cache, must-revalidate",
    "Pragma": "no-cache"
}
```

---

## Citizen Data Protection

### Data Handling Requirements

```
┌────────────────────────────────────────────────────────────────┐
│              CITIZEN DATA PROTECTION REQUIREMENTS               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  COLLECTION:                                                   │
│  • Collect only necessary data (minimization)                  │
│  • Clear purpose for each data element                         │
│  • Obtain consent where required                               │
│  • Inform citizens of data usage                               │
│                                                                │
│  STORAGE:                                                      │
│  • Encrypt all PII at rest (AES-256)                           │
│  • Separate databases for highly sensitive data                │
│  • Implement data retention policies                           │
│  • Secure backup with encryption                               │
│                                                                │
│  ACCESS:                                                       │
│  • Role-based access control                                   │
│  • Department-scoped access for employees                      │
│  • Log all access to citizen data                              │
│  • Require justification for access                            │
│                                                                │
│  TRANSMISSION:                                                 │
│  • TLS 1.2+ for all communications                             │
│  • No PII in URLs or query strings                             │
│  • Encrypted email for sensitive communications                │
│                                                                │
│  DISPOSAL:                                                     │
│  • Secure deletion when retention expires                      │
│  • Anonymize for analytics                                     │
│  • Document disposal procedures                                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Data Masking in Logs

```python
class CitizenDataMasker:
    """Mask citizen PII in logs and responses."""
    
    @staticmethod
    def mask_national_id(value: str) -> str:
        """XXX-XX-1234"""
        if not value or len(value) < 4:
            return '***'
        return f"XXX-XX-{value[-4:]}"
    
    @staticmethod
    def mask_email(email: str) -> str:
        """j***@email.com"""
        if '@' not in email:
            return '***@***'
        local, domain = email.split('@')
        if len(local) <= 1:
            return f"*@{domain}"
        return f"{local[0]}***@{domain}"
    
    @staticmethod
    def mask_phone(phone: str) -> str:
        """***-***-5678"""
        digits = ''.join(c for c in phone if c.isdigit())
        if len(digits) < 4:
            return '***'
        return f"***-***-{digits[-4:]}"
    
    @staticmethod
    def mask_address(address: str) -> str:
        """*** Residential Ave"""
        parts = address.split()
        if len(parts) <= 2:
            return '*** ' + ' '.join(parts[-1:])
        return '*** ' + ' '.join(parts[-2:])
    
    @staticmethod
    def mask_name(name: str) -> str:
        """M*** G***"""
        parts = name.split()
        return ' '.join(f"{p[0]}***" if len(p) > 1 else p for p in parts)
    
    @staticmethod
    def mask_dob(dob: str) -> str:
        """****-**-15"""
        if len(dob) >= 10:
            return f"****-**-{dob[-2:]}"
        return '***'


# Usage in logging
def log_citizen_access(employee_id: int, citizen: dict, action: str):
    """Log citizen data access with masking."""
    masked_citizen = {
        'id': citizen['id'],
        'name': CitizenDataMasker.mask_name(citizen['name']),
        'email': CitizenDataMasker.mask_email(citizen['email']),
        'phone': CitizenDataMasker.mask_phone(citizen.get('phone', '')),
    }
    
    logger.info({
        'event': 'citizen_data_access',
        'employee_id': employee_id,
        'citizen': masked_citizen,
        'action': action,
        'timestamp': datetime.utcnow().isoformat()
    })
```

### Citizen Rights Implementation

```python
class CitizenDataRights:
    """Implement citizen data rights (GDPR/CCPA style)."""
    
    @staticmethod
    async def export_citizen_data(citizen_id: int) -> dict:
        """Right to Access: Export all citizen data."""
        citizen = await get_citizen(citizen_id)
        applications = await get_citizen_applications(citizen_id)
        payments = await get_citizen_payments(citizen_id)
        audit_logs = await get_citizen_access_logs(citizen_id)
        
        return {
            'citizen_profile': citizen,
            'applications': applications,
            'payments': payments,
            'who_accessed_my_data': audit_logs,
            'exported_at': datetime.utcnow().isoformat()
        }
    
    @staticmethod
    async def request_data_deletion(citizen_id: int) -> dict:
        """Right to Erasure: Request data deletion."""
        # Note: Government may have retention requirements
        citizen = await get_citizen(citizen_id)
        
        # Check if deletion is allowed
        retention_check = await check_retention_requirements(citizen_id)
        
        if retention_check['can_delete']:
            # Schedule deletion
            await schedule_citizen_deletion(citizen_id)
            return {
                'status': 'scheduled',
                'deletion_date': retention_check['earliest_deletion'],
                'reason': 'Data retention period completed'
            }
        else:
            return {
                'status': 'cannot_delete',
                'reason': retention_check['retention_reason'],
                'retained_until': retention_check['retention_until']
            }
    
    @staticmethod
    async def correct_citizen_data(citizen_id: int, corrections: dict):
        """Right to Rectification: Correct inaccurate data."""
        # Log the correction request
        await log_audit({
            'action': 'data_correction_request',
            'citizen_id': citizen_id,
            'corrections_requested': list(corrections.keys())
        })
        
        # Apply corrections (may require verification)
        await update_citizen(citizen_id, corrections)
        
        return {'status': 'corrected', 'fields': list(corrections.keys())}
```

---

## Attack Prevention

### SQL Injection Prevention

```python
# ❌ VULNERABLE - Never do this
query = f"SELECT * FROM citizens WHERE email = '{email}'"

# ❌ VULNERABLE - String formatting
query = "SELECT * FROM citizens WHERE email = '%s'" % email

# ✅ SAFE - Parameterized query (psycopg2)
query = "SELECT * FROM citizens WHERE email = %s"
cursor.execute(query, (email,))

# ✅ SAFE - Django ORM
citizen = Citizen.objects.filter(email=email).first()

# ✅ SAFE - SQLAlchemy ORM
citizen = session.query(Citizen).filter(Citizen.email == email).first()
```

### XSS Prevention

```python
from markupsafe import escape
import bleach

# ❌ VULNERABLE
response = f"<p>Hello, {citizen_name}</p>"

# ✅ SAFE - Escape output
response = f"<p>Hello, {escape(citizen_name)}</p>"

# ✅ SAFE - For user-provided HTML (limited)
allowed_tags = ['b', 'i', 'u', 'br']
clean_html = bleach.clean(user_html, tags=allowed_tags)

# ✅ SAFE - Use templating with auto-escape (Django, Jinja2)
# Templates auto-escape by default
```

### Brute Force Prevention

```python
from collections import defaultdict
import time
from typing import Optional

class GovBruteForceProtection:
    """Brute force protection for government login endpoints."""
    
    def __init__(self):
        self.attempts = defaultdict(list)
        self.lockout_duration = 900  # 15 minutes
        self.max_attempts = 5
        self.window = 300  # 5 minutes
        
        # More strict for employees (sensitive data access)
        self.employee_max_attempts = 3
        self.employee_lockout_duration = 1800  # 30 minutes
    
    def record_attempt(self, key: str, success: bool, user_type: str = 'citizen'):
        """Record login attempt."""
        now = time.time()
        self.attempts[key].append({
            'time': now,
            'success': success,
            'user_type': user_type
        })
        
        # Clean old attempts
        max_attempts = self.employee_max_attempts if user_type == 'employee' else self.max_attempts
        self.attempts[key] = [
            a for a in self.attempts[key]
            if now - a['time'] < self.window
        ][-max_attempts * 2:]  # Keep some history
    
    def is_locked(self, key: str, user_type: str = 'citizen') -> bool:
        """Check if account/IP is locked out."""
        now = time.time()
        max_attempts = self.employee_max_attempts if user_type == 'employee' else self.max_attempts
        
        recent_failures = [
            a for a in self.attempts.get(key, [])
            if not a['success'] and now - a['time'] < self.window
        ]
        
        return len(recent_failures) >= max_attempts
    
    def get_lockout_remaining(self, key: str, user_type: str = 'citizen') -> int:
        """Get remaining lockout time in seconds."""
        if not self.is_locked(key, user_type):
            return 0
        
        lockout_duration = self.employee_lockout_duration if user_type == 'employee' else self.lockout_duration
        failures = [a for a in self.attempts.get(key, []) if not a['success']]
        
        if failures:
            oldest = min(a['time'] for a in failures)
            remaining = lockout_duration - (time.time() - oldest)
            return max(0, int(remaining))
        return 0


# Usage in login endpoint
brute_force = GovBruteForceProtection()

async def citizen_login(request: LoginRequest, client_ip: str):
    # Check IP lockout
    if brute_force.is_locked(f"ip:{client_ip}", 'citizen'):
        remaining = brute_force.get_lockout_remaining(f"ip:{client_ip}", 'citizen')
        raise HTTPException(
            status_code=429,
            detail=f"Too many login attempts. Try again in {remaining} seconds."
        )
    
    # Check email lockout
    if brute_force.is_locked(f"email:{request.email}", 'citizen'):
        remaining = brute_force.get_lockout_remaining(f"email:{request.email}", 'citizen')
        raise HTTPException(
            status_code=429,
            detail=f"Account temporarily locked. Try again in {remaining} seconds."
        )
    
    # Attempt login
    citizen = await verify_citizen_credentials(request.email, request.password)
    
    if citizen:
        brute_force.record_attempt(f"ip:{client_ip}", success=True, user_type='citizen')
        brute_force.record_attempt(f"email:{request.email}", success=True, user_type='citizen')
        return generate_citizen_token(citizen)
    else:
        brute_force.record_attempt(f"ip:{client_ip}", success=False, user_type='citizen')
        brute_force.record_attempt(f"email:{request.email}", success=False, user_type='citizen')
        raise HTTPException(status_code=401, detail="Invalid credentials")
```

---

## Audit & Compliance

### Government Audit Log Requirements

```
┌────────────────────────────────────────────────────────────────┐
│               GOVERNMENT AUDIT LOG SCHEMA                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  {                                                             │
│    "id": "uuid",                                               │
│    "timestamp": "2024-01-25T14:30:00.123Z",                    │
│    "event_type": "citizen.profile.viewed",                     │
│                                                                │
│    "actor": {                                                  │
│      "id": 50,                                                 │
│      "type": "employee",                                       │
│      "email": "s***@gov.springfield.us",                       │
│      "employee_id": "EMP-2024-0150",                           │
│      "department": "permits"                                   │
│    },                                                          │
│                                                                │
│    "subject": {                                                │
│      "id": 123,                                                │
│      "type": "citizen",                                        │
│      "name": "M*** G***"                                       │
│    },                                                          │
│                                                                │
│    "action": {                                                 │
│      "type": "view",                                           │
│      "resource": "citizen_profile",                            │
│      "reason": "Permit application review"                     │
│    },                                                          │
│                                                                │
│    "context": {                                                │
│      "ip_address": "10.0.1.50",                                │
│      "user_agent": "Mozilla/5.0...",                           │
│      "request_id": "req-abc123",                               │
│      "service": "permits-service",                             │
│      "department_id": "permits"                                │
│    },                                                          │
│                                                                │
│    "result": "success",                                        │
│    "severity": "info"                                          │
│  }                                                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Events That MUST Be Audited

| Category | Events | Retention |
|----------|--------|-----------|
| **Authentication** | login.success, login.failure, logout, 2fa.verify | 3 years |
| **Citizen Data Access** | profile.view, profile.update, records.view | 7 years |
| **Employee Actions** | application.process, application.approve, application.deny | 7 years |
| **Administrative** | employee.create, role.assign, permission.change | 7 years |
| **Security Events** | password.change, token.revoke, account.lock | 7 years |
| **Data Export/Delete** | data.export, data.delete.request | 10 years |

### Audit Log Implementation

```python
from datetime import datetime
from typing import Optional
import json

class GovAuditLogger:
    """Government-compliant audit logging."""
    
    SEVERITY_LEVELS = {
        'info': 0,
        'warning': 1,
        'critical': 2
    }
    
    @classmethod
    async def log_citizen_data_access(
        cls,
        employee_id: int,
        citizen_id: int,
        action: str,
        resource: str,
        reason: Optional[str] = None,
        request_context: dict = None
    ):
        """Log when employee accesses citizen data."""
        
        employee = await get_employee(employee_id)
        citizen = await get_citizen(citizen_id)
        
        audit_entry = {
            'id': str(uuid.uuid4()),
            'timestamp': datetime.utcnow().isoformat(),
            'event_type': f'citizen.{resource}.{action}',
            
            'actor': {
                'id': employee_id,
                'type': 'employee',
                'email': CitizenDataMasker.mask_email(employee['email']),
                'employee_id': employee['employee_id'],
                'department': employee['department_primary']
            },
            
            'subject': {
                'id': citizen_id,
                'type': 'citizen',
                'name': CitizenDataMasker.mask_name(citizen['name'])
            },
            
            'action': {
                'type': action,
                'resource': resource,
                'reason': reason
            },
            
            'context': request_context or {},
            'result': 'success',
            'severity': 'info'
        }
        
        # Write to audit log (append-only storage)
        await cls._write_audit_log(audit_entry)
        
        # Check for suspicious patterns
        await cls._check_suspicious_access(employee_id, citizen_id)
    
    @classmethod
    async def log_security_event(
        cls,
        event_type: str,
        actor_id: int,
        actor_type: str,
        severity: str = 'warning',
        details: dict = None
    ):
        """Log security-related events."""
        
        audit_entry = {
            'id': str(uuid.uuid4()),
            'timestamp': datetime.utcnow().isoformat(),
            'event_type': f'security.{event_type}',
            'actor': {
                'id': actor_id,
                'type': actor_type
            },
            'details': details or {},
            'severity': severity
        }
        
        await cls._write_audit_log(audit_entry)
        
        # Alert on critical events
        if severity == 'critical':
            await cls._send_security_alert(audit_entry)
    
    @classmethod
    async def _write_audit_log(cls, entry: dict):
        """Write to append-only audit storage."""
        # Write to database (append-only table)
        await db.execute("""
            INSERT INTO audit_logs (id, timestamp, event_type, actor, subject, 
                                    action, context, result, severity, data)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, (
            entry['id'],
            entry['timestamp'],
            entry['event_type'],
            json.dumps(entry.get('actor', {})),
            json.dumps(entry.get('subject', {})),
            json.dumps(entry.get('action', {})),
            json.dumps(entry.get('context', {})),
            entry.get('result', 'unknown'),
            entry.get('severity', 'info'),
            json.dumps(entry)
        ))
        
        # Also write to SIEM/log aggregator
        logger.info(json.dumps(entry))
    
    @classmethod
    async def _check_suspicious_access(cls, employee_id: int, citizen_id: int):
        """Detect suspicious access patterns."""
        # Check if employee is accessing unusual number of citizens
        recent_accesses = await db.fetch("""
            SELECT COUNT(DISTINCT subject->>'id') as citizen_count
            FROM audit_logs
            WHERE actor->>'id' = %s
              AND actor->>'type' = 'employee'
              AND event_type LIKE 'citizen.%'
              AND timestamp > NOW() - INTERVAL '1 hour'
        """, (str(employee_id),))
        
        if recent_accesses[0]['citizen_count'] > 50:
            await cls.log_security_event(
                event_type='unusual_access_pattern',
                actor_id=employee_id,
                actor_type='employee',
                severity='warning',
                details={
                    'citizen_count': recent_accesses[0]['citizen_count'],
                    'period': '1 hour',
                    'threshold': 50
                }
            )
```

---

## Incident Response

### Government Security Incident Playbook

```
┌────────────────────────────────────────────────────────────────┐
│          GOVERNMENT INCIDENT RESPONSE PROCEDURE                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. DETECTION & CLASSIFICATION                                 │
│  ├── Automated alerts from monitoring                          │
│  ├── User report received                                      │
│  ├── Classify severity (Low/Medium/High/Critical)              │
│  └── Notify incident response team                             │
│                                                                │
│  2. CONTAINMENT (Immediate)                                    │
│  ├── Isolate affected systems                                  │
│  ├── Revoke compromised tokens/credentials                     │
│  ├── Block suspicious IPs                                      │
│  ├── Disable affected accounts                                 │
│  └── Preserve evidence (logs, snapshots)                       │
│                                                                │
│  3. ASSESSMENT                                                 │
│  ├── Determine scope of breach                                 │
│  ├── Identify affected citizens (if any)                       │
│  ├── Determine data exposure                                   │
│  └── Document timeline                                         │
│                                                                │
│  4. NOTIFICATION (Government Requirement)                      │
│  ├── Notify IT Security Officer                                │
│  ├── Notify Legal/Compliance                                   │
│  ├── Notify affected department heads                          │
│  ├── Notify affected citizens (if required)                    │
│  └── Notify regulatory bodies (if required)                    │
│                                                                │
│  5. ERADICATION                                                │
│  ├── Identify root cause                                       │
│  ├── Remove malicious access                                   │
│  ├── Patch vulnerabilities                                     │
│  ├── Reset all potentially compromised credentials             │
│  └── Rotate service tokens                                     │
│                                                                │
│  6. RECOVERY                                                   │
│  ├── Restore from clean backups if needed                      │
│  ├── Re-enable services gradually                              │
│  ├── Monitor for recurrence                                    │
│  └── Verify system integrity                                   │
│                                                                │
│  7. POST-INCIDENT                                              │
│  ├── Complete incident report                                  │
│  ├── Root cause analysis                                       │
│  ├── Update procedures                                         │
│  ├── Conduct lessons learned meeting                           │
│  └── Update security controls                                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Emergency Response Commands

```bash
# Revoke all tokens for a compromised citizen account
php artisan auth:revoke-citizen-tokens --citizen=123

# Revoke all tokens for a compromised employee
php artisan auth:revoke-employee-tokens --employee=50 --reason="Security incident"

# Disable a service client immediately
php artisan service:disable permits-service --reason="Security incident"

# Force password reset for affected users
php artisan auth:force-password-reset --users=123,456,789 --notify

# Block IP range
php artisan security:block-ip-range 192.168.1.0/24 --duration=24h

# Export audit logs for incident period
php artisan audit:export --from="2024-01-25 00:00" --to="2024-01-25 23:59" --format=json

# Rotate all service tokens (emergency)
php artisan service:rotate-all-tokens --force --reason="Security incident"
```

### Citizen Breach Notification Template

```
Subject: Important Security Notice from [Government Entity]

Dear [Citizen Name],

We are writing to inform you of a security incident that may have affected 
your account with [Government Entity].

WHAT HAPPENED:
[Brief description of incident]

WHAT INFORMATION WAS INVOLVED:
[List specific data types that may have been exposed]

WHAT WE ARE DOING:
- Investigating the incident with cybersecurity experts
- Strengthening our security measures
- Notifying relevant authorities

WHAT YOU CAN DO:
1. Reset your password at [link]
2. Monitor your accounts for suspicious activity
3. Be cautious of phishing emails claiming to be from us

If you have questions, contact us at:
- Phone: [Number]
- Email: [Email]
- Online: [Portal link]

We apologize for any inconvenience and are committed to protecting your information.

Sincerely,
[Government Entity Security Team]
```

---

## Security Checklist Summary

### Pre-Launch Security Checklist

**Authentication & Authorization:**
- [ ] Argon2id password hashing implemented
- [ ] Employee 2FA mandatory and working
- [ ] Token generation uses CSPRNG
- [ ] Token expiration enforced
- [ ] Department-scoped access verified
- [ ] Role hierarchy working correctly

**Data Protection:**
- [ ] Citizen PII encrypted at rest
- [ ] TLS 1.2+ enforced for all connections
- [ ] Data masking in logs verified
- [ ] Sensitive data not in URLs
- [ ] Database-level encryption enabled

**API Security:**
- [ ] Input validation on all endpoints
- [ ] Rate limiting configured
- [ ] Security headers implemented
- [ ] CORS properly configured
- [ ] SQL injection protection verified
- [ ] XSS protection verified

**Audit & Compliance:**
- [ ] Audit logging for all citizen data access
- [ ] Audit logs append-only
- [ ] Retention policies configured
- [ ] FOIA readiness verified
- [ ] Data export functionality working

**Incident Response:**
- [ ] Incident response plan documented
- [ ] Emergency contact list updated
- [ ] Token revocation procedures tested
- [ ] Backup and recovery tested
- [ ] Breach notification templates ready

**Testing:**
- [ ] Penetration testing completed
- [ ] Vulnerability scan passed
- [ ] Security review completed
- [ ] Compliance audit passed
