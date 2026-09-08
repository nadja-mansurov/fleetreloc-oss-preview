# Database Schema & Order State Machine (Zero Trust / Ephemeral Architecture)

## PostgreSQL Schema (Supabase DDL)

```sql
-- 1. BULK BATCHES (Manifest Imports)
CREATE TABLE bulk_batches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at TIMESTAMPTZ DEFAULT now(),
    dispatcher_id UUID REFERENCES auth.users(id),
    file_name TEXT NOT NULL,
    total_vehicles INT NOT NULL,
    status TEXT CHECK (status IN ('processing', 'parsed', 'partially_assigned', 'completed', 'failed')) DEFAULT 'processing'
);

-- 2. VEHICLES / ORDERS (Ephemeral / Zero-Data-Retention State)
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id UUID REFERENCES bulk_batches(id) ON DELETE CASCADE,
    vin TEXT UNIQUE NOT NULL,
    make_model TEXT NOT NULL,
    license_plate TEXT,

    -- Origin / Destination
    pickup_address TEXT NOT NULL,
    pickup_lat DOUBLE PRECISION,
    pickup_lng DOUBLE PRECISION,
    pickup_window_start TIMESTAMPTZ NOT NULL,
    pickup_window_end TIMESTAMPTZ NOT NULL,

    dropoff_address TEXT NOT NULL,
    dropoff_lat DOUBLE PRECISION,
    dropoff_deadline TIMESTAMPTZ NOT NULL,

    -- State & Hashed/Anonymized Assignment (Zero Trust: no raw phone/name stored)
    status TEXT CHECK (status IN ('unassigned', 'locked', 'assigned', 'in_transit', 'delivered', 'cancelled')) DEFAULT 'unassigned',
    assigned_driver_hash TEXT, -- HMAC-SHA256 hashed WhatsApp ID / Session Token (Zero PII retention)

    locked_until TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Performance Indexes
CREATE INDEX idx_vehicles_status ON vehicles(status);
CREATE INDEX idx_vehicles_vin ON vehicles(vin);
CREATE INDEX idx_vehicles_driver_hash ON vehicles(assigned_driver_hash);
```
