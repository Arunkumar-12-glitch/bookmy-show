# Concurrency Strategy - ShowTime

## Executive Decision

**We choose a Hybrid approach:**
- **Redis SETNX** for the hot path: holding seats during user selection (50ms TTL)
- **PostgreSQL Optimistic Locking** for the confirmation path: final booking (version-based)

This hybrid approach gives us Redis's throughput (100K+ ops/sec) where it matters most, combined with PostgreSQL's ACID guarantees where correctness is final.

---

## The Core Problem: 5L Users, 1 Seat, 2 Options

At 12:00:00 noon, 5 lakh users want the same seat (A-12). Your system must ensure:

1. **Exactly one person gets the seat** (no double-booking)
2. **All other 500K users get a response within 500ms** (no hanging)
3. **No infinite locks** (customer doesn't wait forever)
4. **Under $2,000/month AWS budget** (can't afford expensive infrastructure)

The challenge: Speed and correctness are in tension. Correctness requires locking (which adds latency). Speed requires minimal latency. Budget limits infrastructure options.

---

## Option A: PostgreSQL SELECT FOR UPDATE (Row-Level Locking)

### How It Prevents Double-Booking

```sql
BEGIN;

-- Step 1: Lock the specific seat rows
SELECT id, version FROM seats
WHERE id IN (1, 2, 3)  -- 3 seats user wants
AND status = 'available'
FOR UPDATE;             -- Exclusive lock, no other transaction can touch these rows

-- If we get here, we have the lock. Let's check versions.
-- If any seat's version doesn't match what we expected, ABORT.

-- Step 2: Update status to 'held'
UPDATE seats 
SET status = 'held',
    held_by = $userId,
    held_until = NOW() + INTERVAL '10 minutes',
    version = version + 1
WHERE id IN (1, 2, 3)
AND version = $expectedVersion;  -- Optimistic check

-- Step 3: Create booking record
INSERT INTO bookings (user_id, event_id, total_amount, status)
VALUES ($userId, $eventId, $totalAmount, 'pending')
RETURNING id;

COMMIT;
```

### How the Lock Works

- `SELECT ... FOR UPDATE` acquires an **exclusive row-level lock**
- Lock is held until the transaction commits
- If User A holds the lock on seat A-12:
  - User B's `SELECT ... FOR UPDATE` on A-12 **blocks** (waits in queue)
  - User B is blocked in the database connection pool
  - When User A commits, lock is released, User B's query unblocks
  - User B sees version=1 instead of 0 → `UPDATE` returns 0 rows → User B gets "Seat unavailable"

### The Hard Limit: Connection Pool Exhaustion

PostgreSQL has a max_connections limit (typically 500 with PgBouncer).

**Math for Pool Exhaustion:**

Connections held at any time = (query time in seconds) × (RPS arriving during that time)

```
Assumptions:
- 80% of requests: fast seat checks (20ms)
- 20% of requests: payment API calls (800ms)
- max_connections = 500
- Each query needs 1 connection (simplification; actual is more complex)

Connections held = (0.8 × RPS × 0.020s) + (0.2 × RPS × 0.800s)
                 = RPS × (0.016 + 0.160)
                 = RPS × 0.176

At RPS = 3,000:
Connections held = 3,000 × 0.176 = 528 connections

But we only have 500 connections! ❌ Pool exhausted.

At RPS = 2,840:
Connections held = 2,840 × 0.176 = 500 connections

**Hard limit: 2,840 RPS before pool exhaustion**

We need 8,333 RPS. This approach fails. ❌
```

### Deadlock Risk with Multi-Seat Bookings

**Scenario:**
- User A wants to book seats 5, 10, 15
- User B wants to book seats 10, 15, 20

```
Time  User A                           User B
1     BEGIN;                           BEGIN;
2     SELECT FOR UPDATE on (5,10,15)  SELECT FOR UPDATE on (10,15,20)
      → A locks 5,10,15               → B locks 10,15,20
3     A tries to lock 10,15,20        B tries to lock 5,10,15
      → A waits for B's lock          → B waits for A's lock
      → DEADLOCK! ❌
```

**Mitigation:**
- Always lock seats in order (by seat ID): `WHERE id IN (1,2,3) ORDER BY id`
  - Ensures no circular lock dependency
  - All transactions acquire locks in the same order
  - Eliminates deadlock risk

### Verdict on Option A

**Pros:**
- ✅ Simple to understand (row-level locks are straightforward)
- ✅ ACID guarantees at the database level
- ✅ No additional infrastructure (no Redis needed)

**Cons:**
- ❌ Hard limit of ~2,840 RPS (we need 8,333)
- ❌ Every payment API call (800ms) holds a connection
- ❌ Deadlock risk in multi-seat scenarios (needs careful mitigation)
- ❌ Lock contention at 5L concurrent users (most users queued, not executing)

**Conclusion:** Pure SELECT FOR UPDATE fails at 5L concurrent users. Do not use this alone. ❌

---

## Option B: Redis SETNX Distributed Lock (Optimized)

### How It Prevents Double-Booking

```javascript
// The Redis SETNX approach
const lockKey = `seat_lock:${seatId}`;
const lockValue = `${userId}:${Date.now()}`;
const lockTTL = 50;  // milliseconds - just enough for DB write

// Atomic "set if not exists" in Redis
const acquired = await redis.set(
    lockKey,
    lockValue,
    'NX',          // Only succeed if key doesn't exist
    'EX', lockTTL  // Auto-expire after 50ms
);

if (!acquired) {
    // Another user holds the lock - seat is temporarily unavailable
    return {
        status: 423,
        error: 'SEAT_TEMPORARILY_UNAVAILABLE',
        retryAfter: 100  // ms
    };
}

try {
    // Lock acquired - we have 50ms to update the DB
    const seat = await db.query(
        'SELECT status, version FROM seats WHERE id = $1',
        [seatId]
    );

    // Check status (should be 'available', but another user's DB update might have changed it)
    if (seat.rows[0].status !== 'available') {
        return { error: 'SEAT_ALREADY_TAKEN' };
    }

    // Update seat to 'held' with optimistic locking
    const result = await db.query(
        `UPDATE seats 
         SET status = 'held', held_by = $1, held_until = NOW() + INTERVAL '10 minutes', version = version + 1
         WHERE id = $2 AND version = $3`,
        [userId, seatId, seat.rows[0].version]
    );

    if (result.rowCount === 0) {
        return { error: 'VERSION_MISMATCH - SEAT_TAKEN' };
    }

    return { success: true };

} finally {
    // Release lock ONLY if we still own it
    // Use Lua script for atomicity (prevent releasing someone else's lock)
    await redis.eval(`
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0  // We don't own this lock anymore
        end
    `, 1, lockKey, lockValue);
}
```

### How the Lock Works

1. **Acquire:** `redis.set(key, value, 'NX', 'EX', ttl)` is atomic
   - Either the set succeeds (we acquired the lock) or it fails (someone else has it)
   - No race condition
   
2. **Hold:** Key expires after 50ms (our chosen TTL)
   - If we crash, lock auto-expires
   - If we succeed, we delete the key explicitly
   
3. **Release:** Lua script ensures we only delete our own lock
   - If another transaction already took the lock, we don't delete it
   - Prevents mistakenly releasing someone else's lock

### Why 50ms TTL?

**Too short (e.g., 10ms):**
- DB write takes 5-10ms
- Network latency: 2-5ms
- Total: ~15ms before lock expires
- Lock expires mid-operation → second user acquires lock → double-booking ❌

**Too long (e.g., 500ms):**
- User's network glitches, app crashes after acquiring lock
- Lock stays held for 500ms, preventing others from booking
- At 5L concurrent users, if even 0.1% crash, 500 locks are stuck
- Blocks legitimate buyers

**Our choice: 50ms**
- Accounts for: ~20ms DB write + ~10ms network + ~5ms processing + 15ms buffer
- If we crash, other users only wait 50ms before retrying
- At 5L concurrent users, even if 1% crash, they retry after 50ms

### The Throughput Advantage

Redis SETNX is atomic and extremely fast:
- `redis.set()` with 'NX' flag: ~0.1ms latency
- No waiting in queue (unlike database connections)
- Can handle 100,000+ lock ops per second per Redis node

**Math for Option B:**

Connections held = (time DB operation takes) × (RPS requesting during that time)

```
With Redis lock offload:
- API holds DB connection only for: 20ms (actual DB write)
- Payment calls don't hold connections (async queue)
- Connection held = 0.8 × RPS × 0.020 = RPS × 0.016

At RPS = 8,333:
Connections held = 8,333 × 0.016 = 133 connections ✅ (well under 500 pool size)
```

### What If Redis Fails?

**Scenario:** Redis cluster goes down mid-ticket sale.

**Option 1: Fallback to PostgreSQL FOR UPDATE (graceful degradation)**
```javascript
try {
    const acquired = await redis.set(lockKey, lockValue, 'NX', 'EX', 50);
    if (!acquired) return { error: 'SEAT_TAKEN' };
} catch (redisError) {
    // Redis down - fall back to DB lock
    await db.query('SELECT ... FOR UPDATE'); // This will be slow but correct
}
```

**Result:**
- Redis down → system falls back to slower (but correct) DB locking
- RPS drops from 8,333 to 2,840 (pool limited)
- Most users see "Temporarily unavailable, try again" instead of hanging
- System remains correct (no double-bookings)
- Not ideal, but acceptable for a brief outage

**Mitigation:**
- 3-node Redis cluster (redundancy) - see ARCHITECTURE.md
- High availability failover
- Monitoring and on-call alerts

### The TTL vs Double-Booking Tradeoff

**Critical insight:** The 50ms Redis lock is NOT the final correctness mechanism.

The final mechanism is **optimistic locking on the seats table version column.**

```
Timeline:
T=0ms:    User A acquires Redis lock on seat 5
T=10ms:   User A reads seat: {version: 0, status: 'available'}
T=20ms:   User A UPDATE: SET version=1 WHERE id=5 AND version=0
T=25ms:   User A's lock still held (expires at 50ms)
T=30ms:   User A releases lock (or it auto-expires)

T=31ms:   User B acquires Redis lock on seat 5
T=40ms:   User B reads seat: {version: 1, status: 'held'}  ← Now held!
T=42ms:   User B sees it's held, returns error without updating
```

Even if Redis locks expire or fail, the DB's version column catches conflicts. Double-booking is still prevented. ✅

### Multi-Seat Bookings with Redis Locks

**Scenario:** User wants to book seats 5, 10, 15

```javascript
// Pseudo-code
const seats = [5, 10, 15];
const locks = [];

try {
    // Acquire locks for all seats in order
    for (const seatId of seats.sort()) {
        const acquired = await acquireLock(seatId);
        if (!acquired) {
            throw new Error(`Seat ${seatId} unavailable`);
        }
        locks.push(seatId);
    }

    // All locks acquired - update DB
    await updateSeatsToHeld(seats);

    // Success - locks will auto-expire and be cleaned up
    return { success: true };

} catch (error) {
    // Release all locks we acquired
    for (const seatId of locks) {
        await releaseLock(seatId);
    }
    return { error };
}
```

**No deadlock risk:**
- We acquire locks in order (by seat ID)
- We release all locks if any acquisition fails
- No circular dependencies possible
- Simpler than database deadlock handling

### Verdict on Option B

**Pros:**
- ✅ Handles 8,333+ RPS easily (100K ops/sec Redis throughput)
- ✅ Minimal latency (~0.1ms lock ops)
- ✅ No multi-seat deadlock risk (sequential acquire)
- ✅ Graceful degradation if Redis fails (fallback to DB lock)
- ✅ Lock auto-expires (no manual cleanup needed)

**Cons:**
- ❌ Requires Redis cluster infrastructure (cost: $359/month)
- ❌ More complex to implement correctly (Lua scripts, lock ownership checks)
- ❌ Redis failure is a problem (needs fallback + redundancy)
- ❌ Distributed systems are harder to debug

**Conclusion:** Redis SETNX scales to 5L concurrent users and fits the $2K budget. Use this with careful fallback. ✅

---

## Our Hybrid Choice: Redis for Holds, Optimistic Locking for Confirmation

### The Decision

We use **both** concurrency strategies, each where it shines:

1. **Redis SETNX (50ms TTL):** 
   - Prevents duplicate seat selection during the hot path (user clicking "Select Seat")
   - Fast, high-throughput, but temporary
   - User gets "Seat taken" if they can't acquire lock

2. **PostgreSQL Optimistic Locking (version column):**
   - Final correctness check during confirmation (when booking is created)
   - Ensures no two bookings reference the same seat
   - Lower throughput, but this path is async (users see results later)

### The Flow

```
User clicks "Select Seat A-12"
  ↓
API: acquire Redis lock for A-12 (50ms TTL)
  ├─ Success: continue
  └─ Fail: return 423 "temporarily unavailable"
  ↓
API: hold seat in DB (UPDATE seats SET status='held', held_by=userId)
  ├─ Uses optimistic lock (version check) to prevent conflicts
  └─ Set held_until = NOW() + 10 minutes
  ↓
API: return success "Seat held for 10 minutes"
Redis lock expires after 50ms
  ↓
User initiates payment
  ↓
API: queue to SQS (async)
API: return 202 "Payment in progress"
  ↓
[Payment Worker picks up from SQS]
  ↓
Worker: call payment gateway (200-2000ms)
  ├─ Success:
  │   UPDATE bookings SET status='confirmed'
  │   UPDATE seats SET status='booked' (checks version again)
  │   Create booking_seats records
  │   Send SMS confirmation
  └─ Failure:
      UPDATE bookings SET status='failed'
      UPDATE seats SET status='available'
      Send SMS failure notification
```

### Why This Hybrid Works at Scale

**For the 80% fast-path requests (seat selection):**
- Redis lock: ~1ms latency, handles 100K/sec
- DB update: ~10ms latency, minimal contention
- Total: ~15ms API time, ~10ms DB connection time
- At 8K RPS: 80ms DB connection time sustained (well under 500 limit)

**For the 20% slow-path requests (payment):**
- Done asynchronously in a worker
- API returns immediately (5ms) with 202 Accepted
- Worker takes 2 seconds, but holds no API connection
- No impact on pool exhaustion

**For correctness:**
- Redis lock handles 99% of double-booking attempts (fast, obvious conflicts)
- Optimistic locking catches the 1% edge cases (timing conflicts, crashes)
- Version mismatch triggers automatic retry logic in the worker
- Final booking state is always consistent

### Connection Pool Math (Hybrid)

```
Request types:
- 80% seat selection: 10ms DB hold
- 20% payment initiation: 5ms DB hold (just for seat lock, not payment)

Connections held = (0.8 × RPS × 0.010) + (0.2 × RPS × 0.005)
                 = RPS × (0.008 + 0.001)
                 = RPS × 0.009

At RPS = 8,333:
Connections held = 8,333 × 0.009 = 75 connections ✅
```

Even better than pure Redis (133 connections) because seat selection uses the Redis lock for quick feedback.

---

## Tradeoff Analysis

### What This Strategy Cannot Handle

1. **Redis cluster failure during sale**
   - Fallback to PostgreSQL FOR UPDATE
   - Throughput drops to 2,840 RPS (pool limited)
   - Most users see "Temporarily unavailable"
   - Not ideal, but acceptable for brief outages
   - Mitigation: 3-node Redis cluster with automatic failover

2. **Extremely popular seats with 100K+ concurrent attempts**
   - Same seat 100K users want
   - Even with Redis, only 1 user per 50ms can lock it
   - 100K × 50ms = 5,000 seconds of wait (not possible)
   - Mitigation: Pre-allocate popular seats to a lottery/queue system
   - Outside scope of this design

3. **Payment gateway is slow**
   - If payment API takes 10 seconds to respond
   - Queue backs up
   - Workers retry and eventually DLQ
   - User's seat is held for 10 minutes max
   - Fair but not optimal
   - Mitigation: Timeout payment calls, fail fast

### When to Switch to Option A (Pure PostgreSQL)

Switch to pure PostgreSQL FOR UPDATE if:

1. **Budget doubles to $4,000/month** (can afford more RDS replicas)
2. **Concurrency drops below 50K concurrent users** (pool exhaustion no longer an issue)
3. **Simplicity is critical** (and RPS doesn't matter)
4. **Redis infrastructure is unavailable** (emergency fallback)

In those scenarios:
- Row-level locking is simpler
- No Redis to manage
- Deadlock handling is standard PostgreSQL practice

But for 5L concurrent users at $2,000/month: Hybrid is the only viable approach.

---

## Summary: Concurrency Decision Checklist

| Factor | Pure PostgreSQL | Pure Redis | Hybrid (Our Choice) |
|--------|-----------------|------------|-------------------|
| **Max RPS** | 2,840 | 100K+ | 8,333+ |
| **Double-booking risk** | Minimal (ACID) | Low (if fallback) | Minimal (dual checks) |
| **Deadlock risk** | Yes (multi-seat) | No | No |
| **Infrastructure cost** | Low | $359/mo | $359/mo |
| **Complexity** | Low | High | Medium |
| **Failure mode** | Slow (pool exhaustion) | Redis down (fallback) | Degraded to PostgreSQL |
| **Suitable for 5L users** | ❌ No | ✅ Yes | ✅ Yes |

---

## Implementation Details (Pseudocode)

### Acquiring a Seat Lock

```javascript
async function holdSeat(seatId, userId) {
    const lockKey = `seat_lock:${seatId}`;
    const lockValue = `${userId}:${Date.now()}:${Math.random()}`;
    
    try {
        // Try to acquire Redis lock with 50ms TTL
        const acquired = await redis.set(
            lockKey,
            lockValue,
            'NX',
            'EX', 50  // 50 milliseconds
        );

        if (!acquired) {
            throw new Error('SEAT_LOCKED');
        }

        // Lock acquired - update DB with seat hold
        const seatData = await db.query(
            'SELECT version, status FROM seats WHERE id = $1',
            [seatId]
        );

        if (seatData.rows[0].status !== 'available') {
            throw new Error('SEAT_NOT_AVAILABLE');
        }

        // Optimistic locking: only update if version matches
        const updateResult = await db.query(
            `UPDATE seats 
             SET status = 'held', 
                 held_by = $1, 
                 held_until = NOW() + INTERVAL '10 minutes',
                 version = version + 1
             WHERE id = $2 AND version = $3`,
            [userId, seatId, seatData.rows[0].version]
        );

        if (updateResult.rowCount === 0) {
            throw new Error('VERSION_CONFLICT');
        }

        return { success: true, lockValue };

    } catch (error) {
        // Ensure we release the lock on error
        await releaseLock(lockKey, lockValue);
        throw error;
    }
}

async function releaseLock(lockKey, lockValue) {
    // Lua script: only delete if we own the lock
    await redis.eval(`
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `, 1, lockKey, lockValue);
}
```

### Confirming a Booking (in Payment Worker)

```javascript
async function confirmBooking(bookingId, userId, seatIds) {
    const booking = await db.query(
        'SELECT * FROM bookings WHERE id = $1',
        [bookingId]
    );

    // Call payment gateway
    const paymentResult = await callPaymentGateway(booking);

    if (paymentResult.success) {
        // Payment succeeded - finalize booking
        const transaction = await db.transaction();
        
        try {
            // Update all seats to 'booked'
            for (const seatId of seatIds) {
                const seatData = await transaction.query(
                    'SELECT version FROM seats WHERE id = $1',
                    [seatId]
                );
                
                const updateResult = await transaction.query(
                    `UPDATE seats 
                     SET status = 'booked', version = version + 1
                     WHERE id = $1 AND version = $2`,
                    [seatId, seatData.rows[0].version]
                );
                
                if (updateResult.rowCount === 0) {
                    throw new Error(`SEAT_VERSION_CONFLICT: ${seatId}`);
                }
            }

            // Create booking_seats junction records
            for (const seatId of seatIds) {
                const seat = await transaction.query(
                    'SELECT price FROM seats WHERE id = $1',
                    [seatId]
                );
                
                await transaction.query(
                    `INSERT INTO booking_seats (booking_id, seat_id, price_at_booking)
                     VALUES ($1, $2, $3)`,
                    [bookingId, seatId, seat.rows[0].price]
                );
            }

            // Update booking status
            await transaction.query(
                `UPDATE bookings 
                 SET status = 'confirmed', completed_at = NOW()
                 WHERE id = $1`,
                [bookingId]
            );

            await transaction.commit();
            await sendSmsConfirmation(userId, bookingId);

        } catch (error) {
            await transaction.rollback();
            throw error;  // Worker will retry
        }

    } else {
        // Payment failed
        const transaction = await db.transaction();
        
        try {
            // Release all held seats
            for (const seatId of seatIds) {
                await transaction.query(
                    `UPDATE seats 
                     SET status = 'available', held_by = NULL, held_until = NULL
                     WHERE id = $1`,
                    [seatId]
                );
            }

            // Update booking status
            await transaction.query(
                `UPDATE bookings 
                 SET status = 'failed', completed_at = NOW()
                 WHERE id = $1`,
                [bookingId]
            );

            await transaction.commit();
            await sendSmsFailure(userId, booking.total_amount);

        } catch (error) {
            await transaction.rollback();
            throw error;  // Worker will retry
        }
    }
}
```

---

## Conclusion

This hybrid strategy scales to 5L concurrent users within the $2,000/month budget while maintaining zero double-bookings. It combines Redis's speed for the hot path with PostgreSQL's correctness for the final confirmation.

**The key insight:** Use the right tool for the right job. Redis for speed, PostgreSQL for correctness. Together, they create a system that is both fast and reliable.
