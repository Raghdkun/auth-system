# Database Permissions & Data Synchronization

## 📋 Table of Contents
1. [Permission Architecture Overview](#permission-architecture-overview)
2. [Service Write/Read Matrix](#service-writeread-matrix)
3. [Database User Configuration](#database-user-configuration)
4. [Change Notification System](#change-notification-system)
5. [Data Consistency Patterns](#data-consistency-patterns)
6. [Conflict Resolution](#conflict-resolution)
7. [Implementation Examples](#implementation-examples)
8. [Monitoring & Alerting](#monitoring--alerting)

---

## Permission Architecture Overview

### Core Principle: Single Source of Truth

```
┌────────────────────────────────────────────────────────────────────────────┐
│              SINGLE SOURCE OF TRUTH ARCHITECTURE                           │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  RULE: Each piece of data has exactly ONE authoritative source             │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                      │  │
│  │  ┌─────────────────┐                                                 │  │
│  │  │   AUTH SERVICE  │  ◄── OWNER of citizen identity data            │  │
│  │  │   (PostgreSQL)  │                                                 │  │
│  │  │                 │                                                 │  │
│  │  │  ✅ WRITE:      │      📤 PUBLISHES:                              │  │
│  │  │  • citizens     │      • citizen.created                         │  │
│  │  │  • employees    │      • citizen.updated                         │  │
│  │  │  • roles        │      • citizen.deleted                         │  │
│  │  │  • permissions  │      • role.assigned                           │  │
│  │  │  • sessions     │      • permission.granted                      │  │
│  │  └────────┬────────┘                                                 │  │
│  │           │                                                          │  │
│  │           │ Events                                                   │  │
│  │           ▼                                                          │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │                     EVENT BUS (Kafka)                          │ │  │
│  │  └───────────────────────────┬────────────────────────────────────┘ │  │
│  │                              │                                       │  │
│  │        ┌─────────────────────┼─────────────────────┐                │  │
│  │        │                     │                     │                 │  │
│  │        ▼                     ▼                     ▼                 │  │
│  │  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐        │  │
│  │  │ PERMITS SVC   │    │ REVENUE SVC   │    │  PARKS SVC    │        │  │
│  │  │               │    │               │    │               │        │  │
│  │  │ 📖 READ ONLY: │    │ 📖 READ ONLY: │    │ 📖 READ ONLY: │        │  │
│  │  │ • citizens    │    │ • citizens    │    │ • citizens    │        │  │
│  │  │   (cached)    │    │   (cached)    │    │   (cached)    │        │  │
│  │  │               │    │               │    │               │        │  │
│  │  │ ✅ WRITE:     │    │ ✅ WRITE:     │    │ ✅ WRITE:     │        │  │
│  │  │ • permits     │    │ • tax_records │    │ • bookings    │        │  │
│  │  │ • inspections │    │ • payments    │    │ • facilities  │        │  │
│  │  └───────────────┘    └───────────────┘    └───────────────┘        │  │
│  │                                                                      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Permission Levels

| Level | Description | Use Case |
|-------|-------------|----------|
| **OWNER** | Full CRUD + schema changes | Service that manages the data |
| **WRITER** | INSERT, UPDATE, DELETE | Event sync consumers |
| **READER** | SELECT only | Application queries |
| **NONE** | No direct access | Must use API |

---

## Service Write/Read Matrix

### Complete Permission Matrix

| Database | Table | Auth | Permits | Revenue | Parks | Works | Notification |
|----------|-------|------|---------|---------|-------|-------|--------------|
| **auth_db** | citizens | OWNER | ❌ | ❌ | ❌ | ❌ | ❌ |
| **auth_db** | employees | OWNER | ❌ | ❌ | ❌ | ❌ | ❌ |
| **auth_db** | roles | OWNER | ❌ | ❌ | ❌ | ❌ | ❌ |
| **auth_db** | permissions | OWNER | ❌ | ❌ | ❌ | ❌ | ❌ |
| **auth_db** | sessions | OWNER | ❌ | ❌ | ❌ | ❌ | ❌ |
| **permits_db** | citizen_cache | ❌ | READ | ❌ | ❌ | ❌ | ❌ |
| **permits_db** | permits | ❌ | OWNER | ❌ | ❌ | ❌ | ❌ |
| **permits_db** | applications | ❌ | OWNER | ❌ | ❌ | ❌ | ❌ |
| **permits_db** | inspections | ❌ | OWNER | ❌ | ❌ | ❌ | ❌ |
| **revenue_db** | citizen_cache | ❌ | ❌ | READ | ❌ | ❌ | ❌ |
| **revenue_db** | tax_records | ❌ | ❌ | OWNER | ❌ | ❌ | ❌ |
| **revenue_db** | payments | ❌ | ❌ | OWNER | ❌ | ❌ | ❌ |
| **parks_db** | citizen_cache | ❌ | ❌ | ❌ | READ | ❌ | ❌ |
| **parks_db** | bookings | ❌ | ❌ | ❌ | OWNER | ❌ | ❌ |
| **parks_db** | facilities | ❌ | ❌ | ❌ | OWNER | ❌ | ❌ |

### Sync User Permissions (Separate)

| Database | Sync User | Permissions | Purpose |
|----------|-----------|-------------|---------|
| permits_db | permits_sync_user | INSERT, UPDATE on citizen_cache | Event consumer |
| revenue_db | revenue_sync_user | INSERT, UPDATE on citizen_cache | Event consumer |
| parks_db | parks_sync_user | INSERT, UPDATE on citizen_cache | Event consumer |

---

## Database User Configuration

### PostgreSQL User Setup

```sql
-- ============================================================
--                    AUTH DATABASE USERS
-- ============================================================

-- 1. Service User (Application)
CREATE USER auth_service_user WITH PASSWORD 'secure_password_1';
GRANT CONNECT ON DATABASE auth_db TO auth_service_user;
GRANT USAGE ON SCHEMA public TO auth_service_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO auth_service_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO auth_service_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO auth_service_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO auth_service_user;

-- 2. Read Replica User (Reporting/Analytics)
CREATE USER auth_readonly_user WITH PASSWORD 'secure_password_2';
GRANT CONNECT ON DATABASE auth_db TO auth_readonly_user;
GRANT USAGE ON SCHEMA public TO auth_readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO auth_readonly_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO auth_readonly_user;

-- ============================================================
--                   PERMITS DATABASE USERS
-- ============================================================

-- Connect to permits_db
\c permits_db

-- Create schemas
CREATE SCHEMA IF NOT EXISTS public;  -- Owned tables
CREATE SCHEMA IF NOT EXISTS cache;   -- Cached external data

-- 1. Application User (read/write owned tables, read-only cache)
CREATE USER permits_service_user WITH PASSWORD 'secure_password_3';
GRANT CONNECT ON DATABASE permits_db TO permits_service_user;

-- Public schema: FULL access (owned tables)
GRANT USAGE ON SCHEMA public TO permits_service_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO permits_service_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO permits_service_user;

-- Cache schema: READ ONLY
GRANT USAGE ON SCHEMA cache TO permits_service_user;
GRANT SELECT ON ALL TABLES IN SCHEMA cache TO permits_service_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA cache GRANT SELECT ON TABLES TO permits_service_user;

-- 2. Sync User (writes to cache tables only)
CREATE USER permits_sync_user WITH PASSWORD 'secure_password_4';
GRANT CONNECT ON DATABASE permits_db TO permits_sync_user;
GRANT USAGE ON SCHEMA cache TO permits_sync_user;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA cache TO permits_sync_user;
-- IMPORTANT: No access to public schema
REVOKE ALL ON SCHEMA public FROM permits_sync_user;

-- 3. Read-only User (reporting)
CREATE USER permits_readonly_user WITH PASSWORD 'secure_password_5';
GRANT CONNECT ON DATABASE permits_db TO permits_readonly_user;
GRANT USAGE ON SCHEMA public, cache TO permits_readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO permits_readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA cache TO permits_readonly_user;

-- ============================================================
--                    CACHE TABLE STRUCTURE
-- ============================================================

-- Create citizen cache table
CREATE TABLE cache.citizen_refs (
    citizen_id      BIGINT PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    address         TEXT,
    city            VARCHAR(100),
    postal_code     VARCHAR(20),
    email_verified  BOOLEAN DEFAULT FALSE,
    identity_verified BOOLEAN DEFAULT FALSE,
    event_version   BIGINT NOT NULL DEFAULT 0,
    last_synced_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes for common queries
CREATE INDEX idx_citizen_refs_email ON cache.citizen_refs(email);
CREATE INDEX idx_citizen_refs_phone ON cache.citizen_refs(phone);
CREATE INDEX idx_citizen_refs_synced ON cache.citizen_refs(last_synced_at);

-- Trigger to update updated_at
CREATE OR REPLACE FUNCTION cache.update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER citizen_refs_updated_at
    BEFORE UPDATE ON cache.citizen_refs
    FOR EACH ROW
    EXECUTE FUNCTION cache.update_updated_at();
```

### Row-Level Security (Advanced)

```sql
-- Enable RLS for sensitive tables
ALTER TABLE public.permits ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their department's permits
CREATE POLICY department_isolation ON public.permits
    FOR ALL
    TO permits_service_user
    USING (department_id = current_setting('app.department_id')::BIGINT);

-- Set department context in application
-- SET app.department_id = '123';
```

---

## Change Notification System

### Notification Flow

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    CHANGE NOTIFICATION FLOW                                 │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  1. User updates citizen profile                                           │
│     ┌──────────────────────────────────────────────────────────────────┐  │
│     │  PUT /api/v1/citizens/123                                         │  │
│     │  { "phone": "+1-555-0199" }                                       │  │
│     └──────────────────────────────────────────────────────────────────┘  │
│                                         │                                  │
│                                         ▼                                  │
│  2. Auth Service processes update                                          │
│     ┌──────────────────────────────────────────────────────────────────┐  │
│     │  BEGIN TRANSACTION;                                               │  │
│     │  UPDATE citizens SET phone = '+1-555-0199' WHERE id = 123;        │  │
│     │  INSERT INTO outbox (event_type, payload, ...) VALUES (...);      │  │
│     │  COMMIT;                                                          │  │
│     └──────────────────────────────────────────────────────────────────┘  │
│                                         │                                  │
│                                         ▼                                  │
│  3. Outbox Publisher sends to Kafka                                        │
│     ┌──────────────────────────────────────────────────────────────────┐  │
│     │  Topic: gov.auth.citizen                                          │  │
│     │  Key: citizen_123                                                 │  │
│     │  {                                                                │  │
│     │    "event_type": "citizen.updated",                               │  │
│     │    "event_version": 42,                                           │  │
│     │    "data": { ... }                                                │  │
│     │  }                                                                │  │
│     └──────────────────────────────────────────────────────────────────┘  │
│                                         │                                  │
│         ┌───────────────────────────────┼───────────────────────────┐     │
│         │                               │                           │      │
│         ▼                               ▼                           ▼      │
│  4. Consumers receive and process                                          │
│                                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│  │ Permits Sync    │  │ Revenue Sync    │  │ Notification    │            │
│  │                 │  │                 │  │                 │            │
│  │ UPDATE cache.   │  │ UPDATE cache.   │  │ Check if user   │            │
│  │ citizen_refs    │  │ citizen_refs    │  │ has pending     │            │
│  │ SET phone = ... │  │ SET phone = ... │  │ applications    │            │
│  │ WHERE citizen_id│  │ WHERE citizen_id│  │ to notify       │            │
│  │ = 123           │  │ = 123           │  │                 │            │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘            │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Transactional Outbox Pattern

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    TRANSACTIONAL OUTBOX PATTERN                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Problem: How to reliably publish events after database changes?           │
│                                                                            │
│  ❌ BAD: Two separate operations                                           │
│     1. UPDATE citizens...    ← Succeeds                                    │
│     2. kafka.publish(...)    ← Fails!  (Data updated, no event)           │
│                                                                            │
│  ✅ GOOD: Transactional Outbox                                             │
│     1. BEGIN                                                               │
│     2. UPDATE citizens...                                                  │
│     3. INSERT INTO outbox...  ← Same transaction!                          │
│     4. COMMIT                                                              │
│     5. Background: Poll outbox → Publish → Delete                          │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │  OUTBOX TABLE                                                     │     │
│  ├──────────────────────────────────────────────────────────────────┤     │
│  │  id              BIGSERIAL PRIMARY KEY                            │     │
│  │  aggregate_type  VARCHAR(100)    -- 'citizen', 'employee'         │     │
│  │  aggregate_id    BIGINT          -- Entity ID                     │     │
│  │  event_type      VARCHAR(100)    -- 'created', 'updated'          │     │
│  │  payload         JSONB           -- Event data                    │     │
│  │  created_at      TIMESTAMPTZ     -- When inserted                 │     │
│  │  published_at    TIMESTAMPTZ     -- When sent (NULL = pending)    │     │
│  │  retry_count     INT DEFAULT 0   -- For error handling            │     │
│  └──────────────────────────────────────────────────────────────────┘     │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Outbox Publisher Implementation

```python
# outbox_publisher.py
import asyncio
import json
from kafka import KafkaProducer
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

class OutboxPublisher:
    def __init__(self):
        self.engine = create_engine('postgresql://auth_service_user:pass@db/auth_db')
        self.Session = sessionmaker(bind=self.engine)
        self.producer = KafkaProducer(
            bootstrap_servers=['kafka:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            acks='all',  # Wait for all replicas
            retries=3
        )
    
    async def run(self):
        """Main loop - poll and publish"""
        while True:
            try:
                await self.process_outbox_batch()
                await asyncio.sleep(0.1)  # 100ms polling interval
            except Exception as e:
                print(f"Error in outbox publisher: {e}")
                await asyncio.sleep(1)
    
    async def process_outbox_batch(self):
        """Process a batch of outbox messages"""
        session = self.Session()
        
        try:
            # Lock and fetch pending messages
            messages = session.execute(text("""
                SELECT id, aggregate_type, aggregate_id, event_type, payload
                FROM outbox
                WHERE published_at IS NULL
                ORDER BY created_at
                LIMIT 100
                FOR UPDATE SKIP LOCKED
            """)).fetchall()
            
            for msg in messages:
                topic = f"gov.auth.{msg.aggregate_type}"
                key = f"{msg.aggregate_type}_{msg.aggregate_id}"
                
                event = {
                    'event_type': f"{msg.aggregate_type}.{msg.event_type}",
                    'event_id': f"evt_{msg.id}",
                    **json.loads(msg.payload)
                }
                
                # Send to Kafka
                future = self.producer.send(topic, key=key, value=event)
                future.get(timeout=10)  # Wait for confirmation
                
                # Mark as published
                session.execute(text("""
                    UPDATE outbox 
                    SET published_at = NOW() 
                    WHERE id = :id
                """), {'id': msg.id})
            
            session.commit()
            
            # Cleanup old published messages
            session.execute(text("""
                DELETE FROM outbox 
                WHERE published_at < NOW() - INTERVAL '7 days'
            """))
            session.commit()
            
        except Exception as e:
            session.rollback()
            raise
        finally:
            session.close()

# Alternative: Use Debezium CDC for automatic outbox capture
```

---

## Data Consistency Patterns

### Eventual Consistency Model

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    EVENTUAL CONSISTENCY TIMELINE                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Time ─────────────────────────────────────────────────────────────────▶  │
│                                                                            │
│  T0: User updates phone number in Auth Service                             │
│      Auth DB: phone = "+1-555-0199" ✓                                      │
│      Permits DB: phone = "+1-555-0123" (old)                               │
│      Revenue DB: phone = "+1-555-0123" (old)                               │
│                                                                            │
│  T0 + 50ms: Event published to Kafka                                       │
│      Auth DB: phone = "+1-555-0199" ✓                                      │
│      Permits DB: phone = "+1-555-0123" (old)                               │
│      Revenue DB: phone = "+1-555-0123" (old)                               │
│                                                                            │
│  T0 + 100ms: Permits consumer receives event                               │
│      Auth DB: phone = "+1-555-0199" ✓                                      │
│      Permits DB: phone = "+1-555-0199" ✓ (updated!)                        │
│      Revenue DB: phone = "+1-555-0123" (old)                               │
│                                                                            │
│  T0 + 150ms: Revenue consumer receives event                               │
│      Auth DB: phone = "+1-555-0199" ✓                                      │
│      Permits DB: phone = "+1-555-0199" ✓                                   │
│      Revenue DB: phone = "+1-555-0199" ✓ (updated!)                        │
│                                                                            │
│  CONSISTENT! (Typical lag: 50-200ms)                                       │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Handling Stale Reads

```python
# How to handle potentially stale cached data

class CitizenService:
    def __init__(self, db, auth_client):
        self.db = db
        self.auth_client = auth_client
    
    def get_citizen_for_display(self, citizen_id: int):
        """
        For display purposes - cached data is acceptable.
        Used for: Lists, dashboards, non-critical views.
        """
        return self.db.query(CitizenRef).filter_by(citizen_id=citizen_id).first()
    
    def get_citizen_for_payment(self, citizen_id: int):
        """
        For payment/critical operations - must be fresh.
        Used for: Payment processing, document generation.
        """
        # Always fetch from source
        response = self.auth_client.get_citizen(citizen_id)
        
        if response.success:
            # Optionally update local cache
            self.update_citizen_cache(response.citizen)
            return response.citizen
        else:
            # Fallback to cache if auth service unavailable
            cached = self.get_citizen_for_display(citizen_id)
            if cached and cached.last_synced_at > datetime.now() - timedelta(minutes=5):
                return cached
            raise ServiceUnavailableError("Cannot verify citizen data")
    
    def get_citizen_with_freshness(self, citizen_id: int, max_age_seconds: int = 60):
        """
        Get cached data only if fresh enough.
        """
        cached = self.get_citizen_for_display(citizen_id)
        
        if cached:
            age = (datetime.now() - cached.last_synced_at).total_seconds()
            if age <= max_age_seconds:
                return cached
        
        # Cache is stale or missing - fetch from source
        return self.get_citizen_for_payment(citizen_id)
```

### Version-Based Conflict Prevention

```python
# Prevent out-of-order updates using version numbers

def handle_citizen_event(event: dict):
    """
    Process citizen event with version check.
    Prevents older events from overwriting newer data.
    """
    data = event['data']
    event_version = event['event_version']
    citizen_id = data['citizen_id']
    
    # Atomic upsert with version check
    result = db.execute("""
        INSERT INTO cache.citizen_refs 
            (citizen_id, name, email, phone, address, event_version, last_synced_at)
        VALUES 
            (:citizen_id, :name, :email, :phone, :address, :version, NOW())
        ON CONFLICT (citizen_id) DO UPDATE SET
            name = EXCLUDED.name,
            email = EXCLUDED.email,
            phone = EXCLUDED.phone,
            address = EXCLUDED.address,
            event_version = EXCLUDED.event_version,
            last_synced_at = NOW()
        WHERE 
            -- Only update if incoming version is newer
            citizen_refs.event_version < EXCLUDED.event_version
        RETURNING citizen_id
    """, {
        'citizen_id': citizen_id,
        'name': data['snapshot']['name'],
        'email': data['snapshot']['email'],
        'phone': data['snapshot']['phone'],
        'address': data['snapshot'].get('address'),
        'version': event_version
    })
    
    if result.rowcount == 0:
        # Event was older than current data - log but don't fail
        logger.info(f"Skipped stale event v{event_version} for citizen {citizen_id}")
    else:
        logger.info(f"Updated citizen {citizen_id} to v{event_version}")
```

---

## Conflict Resolution

### Conflict Scenarios

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    CONFLICT SCENARIOS                                       │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  SCENARIO 1: Out-of-Order Events                                           │
│  ════════════════════════════════                                          │
│                                                                            │
│  Event v2 arrives before v1:                                               │
│                                                                            │
│  Time ─────────────────────────────────────────────────────────────────▶  │
│  │                                                                         │
│  │  Kafka    ┌─────┐    ┌─────┐                                           │
│  │  Produce: │ v1  │    │ v2  │                                           │
│  │           └──┬──┘    └──┬──┘                                           │
│  │              │          │                                               │
│  │  Consumer   │          │                                               │
│  │  Receives:  │   ┌──────┘    Network delay!                             │
│  │             │   │                                                       │
│  │             │   ▼                                                       │
│  │             │  ┌─────┐                                                  │
│  │             │  │ v2  │ ◄── Processed first!                            │
│  │             │  └─────┘                                                  │
│  │             │                                                           │
│  │             ▼                                                           │
│  │           ┌─────┐                                                       │
│  │           │ v1  │ ◄── Arrives late, MUST BE REJECTED                   │
│  │           └─────┘                                                       │
│  │                                                                         │
│  SOLUTION: Version check in UPDATE query (shown above)                     │
│                                                                            │
│  ──────────────────────────────────────────────────────────────────────── │
│                                                                            │
│  SCENARIO 2: Consumer Crash During Processing                              │
│  ═══════════════════════════════════════════════                           │
│                                                                            │
│  1. Consumer reads event from Kafka                                        │
│  2. Consumer starts updating database                                      │
│  3. Consumer CRASHES                                                       │
│  4. Kafka rebalances, another consumer picks up                           │
│  5. Event is processed AGAIN                                               │
│                                                                            │
│  SOLUTION: Idempotent operations                                           │
│  - Use UPSERT (INSERT ON CONFLICT UPDATE)                                  │
│  - Track processed event IDs                                               │
│  - Design operations to be safe to repeat                                  │
│                                                                            │
│  ──────────────────────────────────────────────────────────────────────── │
│                                                                            │
│  SCENARIO 3: Inconsistent Snapshot                                         │
│  ═════════════════════════════════════                                     │
│                                                                            │
│  Permits shows: Maria Garcia, +1-555-0123 (old)                           │
│  Revenue shows: Maria Garcia, +1-555-0199 (new)                           │
│                                                                            │
│  CAUSE: Revenue consumer processed faster                                  │
│                                                                            │
│  MITIGATION:                                                               │
│  - Accept eventual consistency (milliseconds)                              │
│  - UI shows "Last updated: X seconds ago"                                  │
│  - Critical operations fetch from source                                   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Idempotency Implementation

```python
# Idempotent event processing with tracking

class IdempotentEventProcessor:
    def __init__(self, db):
        self.db = db
    
    def process_event(self, event: dict) -> bool:
        """
        Process event idempotently.
        Returns True if processed, False if duplicate.
        """
        event_id = event['event_id']
        
        # Check if already processed
        exists = self.db.execute("""
            SELECT 1 FROM processed_events 
            WHERE event_id = :event_id
        """, {'event_id': event_id}).fetchone()
        
        if exists:
            logger.info(f"Duplicate event {event_id}, skipping")
            return False
        
        try:
            # Start transaction
            with self.db.begin():
                # Process the event
                self.handle_event(event)
                
                # Mark as processed
                self.db.execute("""
                    INSERT INTO processed_events (event_id, processed_at)
                    VALUES (:event_id, NOW())
                """, {'event_id': event_id})
            
            return True
            
        except Exception as e:
            logger.error(f"Failed to process {event_id}: {e}")
            raise
    
    def cleanup_old_events(self, days: int = 7):
        """Remove old processed event records"""
        self.db.execute("""
            DELETE FROM processed_events
            WHERE processed_at < NOW() - INTERVAL ':days days'
        """, {'days': days})

# Table for tracking
"""
CREATE TABLE processed_events (
    event_id    VARCHAR(100) PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_processed_events_time ON processed_events(processed_at);
"""
```

---

## Implementation Examples

### Complete Sync Consumer

```python
# sync_consumer.py - Complete implementation

import asyncio
import json
import logging
from kafka import KafkaConsumer
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker
from contextlib import contextmanager

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class CitizenSyncConsumer:
    def __init__(self, config: dict):
        self.config = config
        
        # Database connection (sync user - limited permissions)
        self.engine = create_engine(
            config['db_url'],
            pool_size=5,
            max_overflow=10,
            pool_pre_ping=True
        )
        self.Session = sessionmaker(bind=self.engine)
        
        # Kafka consumer
        self.consumer = KafkaConsumer(
            'gov.auth.citizen',
            bootstrap_servers=config['kafka_brokers'],
            group_id=config['consumer_group'],
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            enable_auto_commit=False,  # Manual commit for reliability
            max_poll_records=100
        )
    
    @contextmanager
    def get_session(self):
        session = self.Session()
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()
    
    def run(self):
        """Main consumer loop"""
        logger.info(f"Starting consumer for {self.config['consumer_group']}")
        
        try:
            for message in self.consumer:
                try:
                    self.process_message(message)
                    # Commit offset after successful processing
                    self.consumer.commit()
                except Exception as e:
                    logger.error(f"Error processing message: {e}")
                    # Don't commit - will retry on restart
                    raise
        finally:
            self.consumer.close()
    
    def process_message(self, message):
        """Process a single Kafka message"""
        event = message.value
        event_type = event.get('event_type')
        event_id = event.get('event_id')
        
        logger.info(f"Processing {event_type} - {event_id}")
        
        # Check for duplicate
        if self.is_duplicate(event_id):
            logger.info(f"Duplicate event {event_id}, skipping")
            return
        
        # Route to handler
        if event_type == 'citizen.created':
            self.handle_citizen_created(event)
        elif event_type == 'citizen.updated':
            self.handle_citizen_updated(event)
        elif event_type == 'citizen.deleted':
            self.handle_citizen_deleted(event)
        else:
            logger.warning(f"Unknown event type: {event_type}")
        
        # Mark as processed
        self.mark_processed(event_id)
    
    def is_duplicate(self, event_id: str) -> bool:
        """Check if event was already processed"""
        with self.get_session() as session:
            result = session.execute(text("""
                SELECT 1 FROM cache.processed_events 
                WHERE event_id = :event_id
            """), {'event_id': event_id})
            return result.fetchone() is not None
    
    def mark_processed(self, event_id: str):
        """Mark event as processed"""
        with self.get_session() as session:
            session.execute(text("""
                INSERT INTO cache.processed_events (event_id, processed_at)
                VALUES (:event_id, NOW())
                ON CONFLICT (event_id) DO NOTHING
            """), {'event_id': event_id})
    
    def handle_citizen_created(self, event: dict):
        """Handle citizen.created event"""
        data = event['data']
        snapshot = data['snapshot']
        version = event.get('event_version', 1)
        
        with self.get_session() as session:
            session.execute(text("""
                INSERT INTO cache.citizen_refs 
                    (citizen_id, name, email, phone, address, city, 
                     postal_code, email_verified, identity_verified,
                     event_version, last_synced_at, created_at)
                VALUES 
                    (:citizen_id, :name, :email, :phone, :address, :city,
                     :postal_code, :email_verified, :identity_verified,
                     :version, NOW(), NOW())
                ON CONFLICT (citizen_id) DO UPDATE SET
                    name = EXCLUDED.name,
                    email = EXCLUDED.email,
                    phone = EXCLUDED.phone,
                    address = EXCLUDED.address,
                    city = EXCLUDED.city,
                    postal_code = EXCLUDED.postal_code,
                    email_verified = EXCLUDED.email_verified,
                    identity_verified = EXCLUDED.identity_verified,
                    event_version = EXCLUDED.event_version,
                    last_synced_at = NOW()
                WHERE citizen_refs.event_version < EXCLUDED.event_version
            """), {
                'citizen_id': snapshot['id'],
                'name': snapshot['name'],
                'email': snapshot['email'],
                'phone': snapshot.get('phone'),
                'address': snapshot.get('address'),
                'city': snapshot.get('city'),
                'postal_code': snapshot.get('postal_code'),
                'email_verified': snapshot.get('email_verified', False),
                'identity_verified': snapshot.get('identity_verified', False),
                'version': version
            })
        
        logger.info(f"Created/updated citizen {snapshot['id']} v{version}")
    
    def handle_citizen_updated(self, event: dict):
        """Handle citizen.updated event - same as created (upsert)"""
        self.handle_citizen_created(event)
    
    def handle_citizen_deleted(self, event: dict):
        """Handle citizen.deleted event"""
        data = event['data']
        citizen_id = data['citizen_id']
        
        with self.get_session() as session:
            # Soft delete - mark as inactive
            session.execute(text("""
                UPDATE cache.citizen_refs
                SET is_active = FALSE, 
                    deleted_at = NOW(),
                    last_synced_at = NOW()
                WHERE citizen_id = :citizen_id
            """), {'citizen_id': citizen_id})
        
        logger.info(f"Marked citizen {citizen_id} as deleted")

# Run the consumer
if __name__ == '__main__':
    config = {
        'db_url': 'postgresql://permits_sync_user:pass@db/permits_db',
        'kafka_brokers': ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092'],
        'consumer_group': 'permits-citizen-sync'
    }
    
    consumer = CitizenSyncConsumer(config)
    consumer.run()
```

---

## Monitoring & Alerting

### Key Metrics

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    SYNC MONITORING METRICS                                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  1. CONSUMER LAG                                                           │
│     ═══════════════                                                        │
│     kafka_consumer_lag{group="permits-citizen-sync"} > 1000                │
│     → Warning: Processing falling behind                                   │
│                                                                            │
│  2. SYNC LATENCY                                                           │
│     ══════════════                                                         │
│     Time from event_timestamp to last_synced_at                            │
│     → Target: < 500ms p99                                                  │
│                                                                            │
│  3. FAILED SYNCS                                                           │
│     ═════════════                                                          │
│     sync_failures_total{service="permits"}                                 │
│     → Alert if > 0 sustained                                               │
│                                                                            │
│  4. STALE RECORDS                                                          │
│     ══════════════                                                         │
│     Records where last_synced_at > 5 minutes ago                           │
│     (while auth_db has newer version)                                      │
│     → Indicates sync problems                                              │
│                                                                            │
│  5. CONSISTENCY CHECK                                                      │
│     ═══════════════════                                                    │
│     Periodic comparison: auth_db.citizens vs cache.citizen_refs            │
│     → Should match (within version tolerance)                              │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Alerting Rules

```yaml
# prometheus/sync_alerts.yml
groups:
  - name: data-sync-alerts
    rules:
      - alert: ConsumerLagHigh
        expr: kafka_consumer_group_lag{group=~".*-citizen-sync"} > 5000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Citizen sync consumer lag is high"
          description: "{{ $labels.group }} has lag of {{ $value }}"
      
      - alert: ConsumerLagCritical
        expr: kafka_consumer_group_lag{group=~".*-citizen-sync"} > 50000
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Citizen sync consumer lag is critical"
      
      - alert: SyncFailures
        expr: increase(sync_failures_total[5m]) > 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Multiple sync failures detected"
      
      - alert: StaleCache
        expr: >
          count(
            citizen_cache_age_seconds > 300
          ) > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Many stale records in citizen cache"
      
      - alert: ConsumerNotRunning
        expr: kafka_consumer_group_members{group=~".*-citizen-sync"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No active consumers for citizen sync"
```

### Consistency Check Script

```python
# consistency_check.py - Run periodically

async def check_consistency():
    """
    Compare auth_db citizens with cached copies.
    Run hourly to detect sync issues.
    """
    auth_db = get_auth_db_connection()
    permits_db = get_permits_db_connection()
    
    # Get sample of citizens from auth
    auth_citizens = auth_db.execute("""
        SELECT id, name, email, phone, updated_at
        FROM citizens
        WHERE updated_at > NOW() - INTERVAL '1 hour'
        LIMIT 1000
    """).fetchall()
    
    inconsistencies = []
    
    for citizen in auth_citizens:
        cached = permits_db.execute("""
            SELECT citizen_id, name, email, phone, last_synced_at
            FROM cache.citizen_refs
            WHERE citizen_id = :id
        """, {'id': citizen.id}).fetchone()
        
        if not cached:
            inconsistencies.append({
                'type': 'missing',
                'citizen_id': citizen.id,
                'auth_updated': citizen.updated_at
            })
        elif cached.email != citizen.email or cached.phone != citizen.phone:
            # Allow 5 minute lag
            if (citizen.updated_at - cached.last_synced_at).seconds > 300:
                inconsistencies.append({
                    'type': 'stale',
                    'citizen_id': citizen.id,
                    'auth_updated': citizen.updated_at,
                    'cache_synced': cached.last_synced_at
                })
    
    # Report metrics
    prometheus_gauge('consistency_missing_records', len([i for i in inconsistencies if i['type'] == 'missing']))
    prometheus_gauge('consistency_stale_records', len([i for i in inconsistencies if i['type'] == 'stale']))
    
    if inconsistencies:
        logger.warning(f"Found {len(inconsistencies)} inconsistencies")
        # Trigger re-sync for missing/stale records
        for issue in inconsistencies:
            trigger_resync(issue['citizen_id'])
    
    return inconsistencies
```
