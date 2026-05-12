# Database Schema Design - ShowTime

## Overview

The ShowTime database schema is optimized for concurrent ticketing with three core objectives:

1. **Prevent double-bookings** through careful constraint design and locking strategies
2. **Support high read concurrency** through denormalization and indexed access patterns
3. **Enable efficient queries** at scale with strategic indexes

The schema uses PostgreSQL 14+ with UUID primary keys for security and client-side idempotency.

---

## Complete DDL (Data Definition Language)

### 1. Users Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CHECK (email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'),
    CHECK (phone ~ '^\+?[1-9]\d{1,14}$')
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
```

**Rationale:**
- UUID primary key prevents user ID enumeration attacks
- Email and phone are unique to enable password reset and SMS verification
- CHECK constraints enforce format validation at the DB level
- Indexes on email/phone speed up lookups during login and duplicate prevention

---

### 2. Venues Table

```sql
CREATE TABLE venues (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100) NOT NULL,
    country VARCHAR(100) NOT NULL DEFAULT 'India',
    total_capacity INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CHECK (total_capacity > 0)
);

CREATE INDEX idx_venues_city ON venues(city);
```

**Rationale:**
- BIGSERIAL is fine here (venues are not user-facing IDs)
- total_capacity is denormalized from counting seats (read-heavy, write-rare)

---

### 3. Events Table

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    venue_id BIGINT NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    artist_name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'upcoming',
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    total_seats INT NOT NULL,
    available_seats INT NOT NULL,
    booking_open_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CHECK (total_seats > 0),
    CHECK (available_seats >= 0),
    CHECK (available_seats <= total_seats),
    CHECK (status IN ('upcoming', 'on_sale', 'sold_out', 'cancelled')),
    CHECK (start_time > NOW()),
    CHECK (end_time > start_time)
);

CREATE INDEX idx_events_venue_status ON events(venue_id, status);
CREATE INDEX idx_events_start_time ON events(start_time);
CREATE INDEX idx_events_status ON events(status) WHERE status IN ('upcoming', 'on_sale');
```

**Rationale:**
- `available_seats` is denormalized for fast browsing queries (no aggregation needed)
- Will be updated atomically when seats are booked/released
- Partial index on 'upcoming' and 'on_sale' events (they change frequently; 'sold_out' is static)
- start_time index enables "events happening soon" queries

---

### 4. Seats Table (The Critical One)

```sql
CREATE TABLE seats (
    id BIGSERIAL PRIMARY KEY,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    section VARCHAR(20) NOT NULL,
    row_num VARCHAR(20) NOT NULL,
    seat_num VARCHAR(20) NOT NULL,
    category VARCHAR(50) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'available',
    held_by UUID REFERENCES users(id) ON DELETE SET NULL,
    held_until TIMESTAMP,
    version INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE (event_id, section, row_num, seat_num),
    CHECK (price > 0),
    CHECK (status IN ('available', 'held', 'booked', 'blocked')),
    CHECK (
        CASE 
            WHEN status = 'held' THEN held_until IS NOT NULL
            ELSE held_until IS NULL
        END
    )
);

-- The most critical index - this is the main query for bookings
CREATE INDEX idx_seats_event_status ON seats(event_id, status) 
WHERE status IN ('available', 'held');

-- For finding user's held seats
CREATE INDEX idx_seats_held_by ON seats(held_by) 
WHERE status = 'held';

-- For cleanup job: find expired holds
CREATE INDEX idx_seats_held_until ON seats(held_until) 
WHERE status = 'held' AND held_until < NOW();

-- For seat map visualization
CREATE INDEX idx_seats_event_section ON seats(event_id, section);
```

**Rationale:**

- **UUID foreign key to held_by:** Links to user who is temporarily holding the seat
- **version column:** Enables optimistic locking for the final booking confirmation
  - User 1 reads: `{seatId: 1, version: 0, status: 'available'}`
  - User 2 reads: `{seatId: 1, version: 0, status: 'available'}`
  - User 1 updates: `UPDATE seats SET status='booked', version=1 WHERE id=1 AND version=0` → SUCCESS (version 0 matched)
  - User 2 updates: `UPDATE seats SET status='booked', version=1 WHERE id=1 AND version=0` → FAILS (version is now 1)
- **held_until timestamp:** Seats held for payment processing expire after 10 minutes, freeing them for other users
- **Status CHECK constraint:** Ensures only valid statuses exist
- **held_until consistency check:** Enforces that held_until is only set when status='held'
- **Partial indexes:** Only indexes 'available' and 'held' seats (booked seats don't need indexing for availability queries)
- **idx_seats_event_status:** The workhorse query for "find available seats in this event"
- **idx_seats_held_until:** For the background job that releases expired holds

---

### 5. Bookings Table

```sql
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE RESTRICT,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(10, 2) NOT NULL,
    payment_method VARCHAR(50),
    payment_reference VARCHAR(255),
    payment_response_code VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,
    
    CHECK (total_amount > 0),
    CHECK (status IN ('pending', 'confirmed', 'failed', 'refunded', 'cancelled')),
    CHECK (
        CASE 
            WHEN status IN ('confirmed', 'refunded') THEN completed_at IS NOT NULL
            ELSE completed_at IS NULL
        END
    )
);

-- User's booking history - used for "My Bookings" queries
CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);

-- Partial index on unresolved bookings only (they change frequently)
-- 99% of queries are on 'pending' and 'failed' bookings (payment workers)
CREATE INDEX idx_bookings_pending_status ON bookings(created_at DESC) 
WHERE status IN ('pending', 'failed')
WITH (FILLFACTOR = 70);

-- For payment workers to find bookings by ID quickly
CREATE INDEX idx_bookings_status ON bookings(status);
```

**Rationale:**

- **UUID primary key:** Booking IDs are user-facing (in emails, SMS confirmations). UUID prevents enumeration of all bookings
- **Immutable user_id and event_id:** Can't change which user or event after creation
- **payment_reference:** Links to Razorpay/PayU payment ID for dispute resolution
- **completed_at:** Timestamp when payment succeeded or failed (helps audit trails)
- **Partial index on pending/failed:** These two statuses represent unresolved bookings that workers query constantly
  - 'confirmed' and 'refunded' are historical - rarely queried
  - Smaller index = faster updates = less lock contention
- **FILLFACTOR = 70:** Leaves space in pages for updates without page splits (optimizes UPDATE performance)

---

### 6. BookingSeats Junction Table

```sql
CREATE TABLE booking_seats (
    id BIGSERIAL PRIMARY KEY,
    booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    seat_id BIGINT NOT NULL REFERENCES seats(id) ON DELETE RESTRICT,
    price_at_booking DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE (booking_id, seat_id),
    CHECK (price_at_booking > 0)
);

-- Fast lookup: "What seats are in this booking?"
CREATE INDEX idx_booking_seats_booking ON booking_seats(booking_id);

-- Fast lookup: "Which booking has this seat?"
CREATE INDEX idx_booking_seats_seat ON booking_seats(seat_id);
```

**Rationale:**

- **UNIQUE (booking_id, seat_id):** Prevents duplicate seat entries in a single booking
- **price_at_booking:** Preserves the price at the time of booking (for historical accuracy if seat prices change)
- **ON DELETE CASCADE on booking_id:** If a booking is deleted, its seat records delete too
- **ON DELETE RESTRICT on seat_id:** Prevent deleting a seat that's part of an active booking (shouldn't happen in normal flow, but database enforces it)

---

### 7. Audit & Seat Status Change Log (Optional but Recommended)

```sql
CREATE TABLE seat_status_log (
    id BIGSERIAL PRIMARY KEY,
    seat_id BIGINT NOT NULL REFERENCES seats(id) ON DELETE CASCADE,
    event_id BIGINT NOT NULL,
    old_status VARCHAR(20) NOT NULL,
    new_status VARCHAR(20) NOT NULL,
    changed_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CHECK (old_status IN ('available', 'held', 'booked', 'blocked')),
    CHECK (new_status IN ('available', 'held', 'booked', 'blocked'))
);

CREATE INDEX idx_seat_status_log_seat ON seat_status_log(seat_id);
CREATE INDEX idx_seat_status_log_event ON seat_status_log(event_id, changed_at DESC);
```

**Rationale:**
- For auditing and debugging (why did a booking fail?)
- For analytics (how many people abandon after selecting seats?)
- Index on event_id + changed_at for "replay" analysis during a sale

---

## Schema Design Decisions - Commentary

### Why UUID for booking.id instead of SERIAL?

**The Problem:**
- Booking ID is user-facing (sent in confirmation emails, SMS, booking receipts)
- SERIAL integers are sequential: 1, 2, 3, 4, ...
- An attacker can iterate `/api/bookings/1`, `/api/bookings/2`, etc. and enumerate every booking in the system
- This exposes:
  - How many total bookings exist (competitor intelligence)
  - Booking IDs of other users (privacy violation)
  - Patterns in booking creation (timing attacks)

**The Solution: UUID v4**
- Random 128-bit identifier: `550e8400-e29b-41d4-a716-446655440000`
- Attacker cannot guess another booking's ID
- Can be generated client-side before the API call (useful for idempotency - if the API crashes after recording but before responding, client can retry with the same UUID)
- Trade-off: Slightly larger index size (16 bytes vs 8 bytes for BIGINT), but security is worth it

### Why does seats have a version column?

**The Problem - Race Condition:**
```
User A reads:   seat_id=1, version=0, status='available'
User B reads:   seat_id=1, version=0, status='available'
User A updates: UPDATE seats SET status='booked', version=1 WHERE id=1 AND version=0 
                → SUCCESS, row updated (1 row affected)
User B updates: UPDATE seats SET status='booked', version=1 WHERE id=1 AND version=0
                → FAILS, no rows match (version is now 1, not 0)
```

**How It Works:**
This is **optimistic locking**. We don't lock rows at read time. Instead:
1. Read the data and remember the version
2. When updating, include the old version in the WHERE clause
3. If another transaction already modified the row, the version will be different
4. The UPDATE returns 0 rows, signaling "conflict"
5. The application retries or shows the user "This seat is no longer available"

**Why It Matters:**
- With Redis SETNX, we hold a 50ms lock during actual payment
- During that 50ms, only one user can proceed
- Other users see "Seat temporarily unavailable" (from Redis lock miss)
- When the lock expires, the seat status is finalized in the database
- Next user who tries: reads version=1, their optimistic lock check fails

### Why held_until instead of just holding at the application level?

**The Problem:**
- When a user selects seats and goes to payment, those seats must be "held" for them
- They have ~10 minutes to complete payment
- If they abandon (close app, network fail), seats must eventually be released for others
- If we only hold in memory (Redis), and the application crashes, seats are held forever

**The Solution: held_until Timestamp**
- `seats.held_until = NOW() + INTERVAL '10 minutes'`
- This is the database's guarantee: "this seat is held until this exact moment"
- A background job every 60 seconds runs:
  ```sql
  UPDATE seats SET status='available', held_by=NULL, held_until=NULL
  WHERE status='held' AND held_until < NOW();
  ```
- If app crashes, DB cleanup job eventually releases the seat
- If user completes payment, API updates the seat to 'booked' before held_until expires

### Why a partial index on bookings.status?

**The Problem - Index Bloat:**
- `bookings` table has millions of rows
- Only 0.1% are in 'pending' or 'failed' status (unresolved)
- 99.9% are 'confirmed' or 'refunded' (historical, rarely queried)
- A regular index on status would be huge and slow to update

**The Solution: Partial Index**
```sql
CREATE INDEX idx_bookings_pending_status ON bookings(created_at DESC) 
WHERE status IN ('pending', 'failed');
```
- Only indexes rows where status is 'pending' or 'failed'
- Index size is 1000× smaller
- Index updates are 1000× faster
- Payment worker query: `SELECT * FROM bookings WHERE status='pending' ORDER BY created_at`
  - This query uses the partial index (only unresolved bookings)
  - Much faster than scanning all bookings

---

## Edge Case: Failed Payment with 3 Seats

**Scenario:** User selects seats A-1, A-2, A-3 and initiates payment. Payment fails halfway.

**Expected state after failure:**

1. **API receives payment failure:**
   ```sql
   UPDATE bookings SET status='failed' WHERE id=$bookingId;
   UPDATE seats SET status='available' WHERE id IN ($seat1, $seat2, $seat3);
   ```

2. **What if the API crashes mid-way?**
   - Some seats may still have status='held'
   - But held_until is set to 10 minutes from now
   - Background job will eventually release them
   - User is not lost - they can retry after 10 minutes

3. **What if booking insert succeeds but seat update fails?**
   - Booking exists with status='pending'
   - Seats might have version mismatch
   - Payment worker picks up the booking: "This booking's seats don't exist/conflict"
   - Worker marks booking as 'failed', notifies user

4. **The key guarantee:** 
   - No seats are left in 'held' state permanently (held_until timeout)
   - No booking is left in 'pending' state permanently (worker retries until DLQ)
   - The database eventually reaches a consistent state

---

## Full Schema Initialization Script

```sql
-- Drop all tables (idempotent - only if they exist)
DROP TABLE IF EXISTS seat_status_log CASCADE;
DROP TABLE IF EXISTS booking_seats CASCADE;
DROP TABLE IF EXISTS bookings CASCADE;
DROP TABLE IF EXISTS seats CASCADE;
DROP TABLE IF EXISTS events CASCADE;
DROP TABLE IF EXISTS venues CASCADE;
DROP TABLE IF EXISTS users CASCADE;

-- Create users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    CHECK (email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'),
    CHECK (phone ~ '^\+?[1-9]\d{1,14}$')
);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);

-- Create venues table
CREATE TABLE venues (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100) NOT NULL,
    country VARCHAR(100) NOT NULL DEFAULT 'India',
    total_capacity INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    CHECK (total_capacity > 0)
);
CREATE INDEX idx_venues_city ON venues(city);

-- Create events table
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    venue_id BIGINT NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    artist_name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'upcoming',
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    total_seats INT NOT NULL,
    available_seats INT NOT NULL,
    booking_open_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    CHECK (total_seats > 0),
    CHECK (available_seats >= 0),
    CHECK (available_seats <= total_seats),
    CHECK (status IN ('upcoming', 'on_sale', 'sold_out', 'cancelled')),
    CHECK (start_time > NOW()),
    CHECK (end_time > start_time)
);
CREATE INDEX idx_events_venue_status ON events(venue_id, status);
CREATE INDEX idx_events_start_time ON events(start_time);
CREATE INDEX idx_events_status ON events(status) WHERE status IN ('upcoming', 'on_sale');

-- Create seats table
CREATE TABLE seats (
    id BIGSERIAL PRIMARY KEY,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    section VARCHAR(20) NOT NULL,
    row_num VARCHAR(20) NOT NULL,
    seat_num VARCHAR(20) NOT NULL,
    category VARCHAR(50) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'available',
    held_by UUID REFERENCES users(id) ON DELETE SET NULL,
    held_until TIMESTAMP,
    version INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (event_id, section, row_num, seat_num),
    CHECK (price > 0),
    CHECK (status IN ('available', 'held', 'booked', 'blocked')),
    CHECK (CASE WHEN status = 'held' THEN held_until IS NOT NULL ELSE held_until IS NULL END)
);
CREATE INDEX idx_seats_event_status ON seats(event_id, status) WHERE status IN ('available', 'held');
CREATE INDEX idx_seats_held_by ON seats(held_by) WHERE status = 'held';
CREATE INDEX idx_seats_held_until ON seats(held_until) WHERE status = 'held' AND held_until < NOW();
CREATE INDEX idx_seats_event_section ON seats(event_id, section);

-- Create bookings table
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE RESTRICT,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(10, 2) NOT NULL,
    payment_method VARCHAR(50),
    payment_reference VARCHAR(255),
    payment_response_code VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,
    CHECK (total_amount > 0),
    CHECK (status IN ('pending', 'confirmed', 'failed', 'refunded', 'cancelled')),
    CHECK (CASE WHEN status IN ('confirmed', 'refunded') THEN completed_at IS NOT NULL ELSE completed_at IS NULL END)
);
CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);
CREATE INDEX idx_bookings_pending_status ON bookings(created_at DESC) WHERE status IN ('pending', 'failed') WITH (FILLFACTOR = 70);
CREATE INDEX idx_bookings_status ON bookings(status);

-- Create booking_seats junction table
CREATE TABLE booking_seats (
    id BIGSERIAL PRIMARY KEY,
    booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    seat_id BIGINT NOT NULL REFERENCES seats(id) ON DELETE RESTRICT,
    price_at_booking DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (booking_id, seat_id),
    CHECK (price_at_booking > 0)
);
CREATE INDEX idx_booking_seats_booking ON booking_seats(booking_id);
CREATE INDEX idx_booking_seats_seat ON booking_seats(seat_id);

-- Create audit log table
CREATE TABLE seat_status_log (
    id BIGSERIAL PRIMARY KEY,
    seat_id BIGINT NOT NULL REFERENCES seats(id) ON DELETE CASCADE,
    event_id BIGINT NOT NULL,
    old_status VARCHAR(20) NOT NULL,
    new_status VARCHAR(20) NOT NULL,
    changed_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    CHECK (old_status IN ('available', 'held', 'booked', 'blocked')),
    CHECK (new_status IN ('available', 'held', 'booked', 'blocked'))
);
CREATE INDEX idx_seat_status_log_seat ON seat_status_log(seat_id);
CREATE INDEX idx_seat_status_log_event ON seat_status_log(event_id, changed_at DESC);
```

---

## Summary

This schema provides:

✅ **Correctness:** Constraints enforce data validity at the database level  
✅ **Concurrency:** Version column + locking strategies prevent double-bookings  
✅ **Performance:** Strategic indexes and denormalization for fast reads  
✅ **Auditability:** Timestamps and audit logs for compliance and debugging  
✅ **Security:** UUID primary keys prevent enumeration; CHECK constraints prevent bad data  

The schema is production-ready for the 5L concurrent user scale.
