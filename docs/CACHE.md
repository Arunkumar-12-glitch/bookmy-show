# Cache Layer Design - ShowTime

## Overview

Caching is not "throw stuff in Redis and hope." It is a precise contract: "Data in cache X will be at most Y seconds old, and we will invalidate it when Z happens."

This document defines every piece of data ShowTime caches, its TTL, and the exact event that triggers invalidation.

---

## The Caching Hierarchy

ShowTime has three layers of caching:

1. **Browser/App Cache** (Client-side) - 5 minutes
   - Event details, seat maps
   - User rarely sees updates during ticket sale

2. **Redis Cache** (Server-side, distributed) - Varies by data type
   - Event details, availability counts, seat maps
   - Shared across all API servers

3. **Database Cache** (PostgreSQL query results) - Implicit
   - Indexes serve as implicit caches
   - No separate caching layer needed

---

## Cache Entries Defined

### 1. Event Details Cache

**Redis Key:** `event:{eventId}`

**TTL:** 3600 seconds (1 hour)

**Data Structure:**
```json
{
    "id": 123,
    "name": "Coldplay India Tour 2024",
    "artist": "Coldplay",
    "venue": {
        "id": 1,
        "name": "Jio World Centre",
        "city": "Mumbai"
    },
    "startTime": "2024-12-15T18:00:00Z",
    "endTime": "2024-12-15T23:00:00Z",
    "status": "on_sale",
    "totalSeats": 50000,
    "ticketCategories": [
        {"category": "VIP", "price": 5000},
        {"category": "Premium", "price": 3000},
        {"category": "General", "price": 1500}
    ]
}
```

**When to Populate:**
- First request for event details (cache miss)
- When event is created/updated

**Invalidation Trigger:**
- ON UPDATE events table: DELETE cache key `event:{eventId}`
- Event status changes: DELETE immediately
- Event details change (name, venue, etc.): DELETE immediately

**Invalidation Method:** Event-driven (eager delete)
- When admin updates event details, API deletes the Redis key immediately
- Next read triggers cache repopulation
- No stale data served during ticket sale

**Why 1 Hour TTL:**
- Event details don't change during a sale
- 1 hour is long enough to avoid cache misses
- If our event update process fails, data is fresh again in 1 hour
- Serves as a backup if DB is unavailable (serve stale event info)

---

### 2. Seat Availability Count Cache

**Redis Key:** `availability:{eventId}:{category}`

**Example Keys:**
- `availability:123:vip`
- `availability:123:premium`
- `availability:123:general`

**TTL:** 30 seconds

**Data Structure:**
```json
{
    "eventId": 123,
    "category": "vip",
    "totalSeats": 5000,
    "availableSeats": 342,
    "heldSeats": 28,
    "bookedSeats": 4630,
    "lastUpdatedAt": "2024-12-15T18:15:45.123Z"
}
```

**When to Populate:**
- First request (cache miss) - query database: `SELECT COUNT(*) FROM seats WHERE event_id=$1 AND category=$2 AND status IN ('available', 'held')`
- Periodic refresh (every 30s even if not requested)

**Invalidation Trigger:**
- When ANY seat changes status for this event and category:
  - Seat held: DELETE `availability:{eventId}:{category}`
  - Seat booked: DELETE `availability:{eventId}:{category}`
  - Seat released: DELETE `availability:{eventId}:{category}`
- Background job: Re-cache every 30 seconds (TTL expiry)

**Invalidation Method:** Event-driven + TTL
- Eager deletion when seat status changes
- TTL fallback ensures data is eventually fresh

**Why 30 Seconds:**
- During a ticket sale, seat count changes constantly
- Users refresh the app expecting somewhat fresh counts
- 30-second stale count is acceptable ("42 seats left" instead of actual "38")
- Event-driven invalidation catches major changes (last seat sold) immediately
- Balance between freshness and cache hit ratio

**Example Timeline:**
```
T=0s:  Last seat sells
       API: DELETE availability:123:vip
T=1s:  User A refreshes app
       Cache miss → query DB: 0 seats available
       Populate cache with 0 seats
T=2s:  User B refreshes app
       Cache hit → serve 0 seats (same as User A)
T=31s: Cache TTL expires naturally (even if count would have stayed 0)
       Next refresh triggers repopulation
```

---

### 3. Seat Map (Layout) Cache

**Redis Key:** `seatmap:{eventId}`

**TTL:** 86400 seconds (24 hours)

**Data Structure:**
```json
{
    "eventId": 123,
    "layout": {
        "sections": [
            {
                "name": "A",
                "rows": [
                    {
                        "rowName": "1",
                        "seats": [
                            {"seatNum": "1", "category": "vip"},
                            {"seatNum": "2", "category": "vip"},
                            {"seatNum": "3", "category": "premium"}
                        ]
                    }
                ]
            }
        ]
    }
}
```

**When to Populate:**
- First request for seat map
- When event is created

**Invalidation Trigger:**
- Event is cancelled: DELETE `seatmap:{eventId}`
- Venue layout changes (rare): DELETE `seatmap:{eventId}`

**Invalidation Method:** TTL only
- Seat map structure never changes during a ticket sale
- 24-hour TTL is long enough
- Eager invalidation only needed if event is cancelled

**Why 24 Hours:**
- Seat layout is immutable once event starts
- Reduces unnecessary Redis writes
- If cache is lost, repopulation is cheap (one query to layouts)
- TTL-only simplicity

---

### 4. User's Booking History Cache (Optional)

**Redis Key:** `user_bookings:{userId}`

**TTL:** 600 seconds (10 minutes)

**Data Structure:**
```json
{
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "bookings": [
        {
            "id": "UUID1",
            "eventId": 123,
            "eventName": "Coldplay",
            "status": "confirmed",
            "totalAmount": 4500,
            "bookedAt": "2024-12-15T18:15:45Z"
        }
    ]
}
```

**When to Populate:**
- User opens "My Bookings" page (cache miss)

**Invalidation Trigger:**
- New booking created: DELETE `user_bookings:{userId}`
- Booking status changes: DELETE `user_bookings:{userId}`

**Invalidation Method:** Event-driven + TTL

**Why 10 Minutes:**
- User's bookings change infrequently (once per sale)
- Historical data, not hot path
- 10 minutes balances freshness vs DB load
- Event-driven invalidation ensures new bookings appear immediately

---

## What NOT to Cache

### ❌ Individual Seat Status

**DO NOT CACHE:** Whether seat A-12 is `available`, `held`, or `booked`

**Why Not:**
```
Problem scenario:
- User A: queries DB, sees seat A-12 = available, caches it in local storage
- Seat A-12 is actually held/booked in reality
- User B: queries DB, sees seat A-12 = available
- Both User A and User B proceed to checkout
- Double-booking! ❌

The issue: Cache window creates a race condition.
```

**The Solution:**
- Always query the actual seat status from DB (or Redis lock state)
- Cache COUNTS (X seats available) not STATES (specific seat available)
- Counts are approximate; individual seat status must be accurate

**What to cache instead:**
- `availability:eventId:category` → "123 VIP seats available" (approximate)
- Individual seat status → always query DB
- Seat map layout → cache (immutable)

---

### ❌ Payment Status

**DO NOT CACHE:** Whether a payment was successful

**Why Not:**
- Payment status determines whether a user gets the tickets
- Cached result could be outdated after 1 second
- Payment is idempotent by ID, but cache creates confusion

**What to do:**
- Store payment status in DB only
- Query DB for payment status
- Payment worker updates DB, not cache

---

### ❌ User Authentication/Authorization

**DO NOT CACHE:** Whether user is logged in, user permissions

**Why Not:**
- User could be logged out in one app instance but cache says logged in
- Permission changes must be reflected immediately
- Security breach if stale auth decision is served

**What to do:**
- Use session tokens (short-lived, 1 hour)
- Validate token with DB on each request or use JWT with short expiry
- Don't cache auth decisions

---

## Cache Invalidation Strategy

### Strategy: Targeted Event-Driven Invalidation

When a seat status changes, we use a database trigger or application logic to immediately delete affected cache keys.

**SQL Trigger (Optional - can also do in application code):**

```sql
-- On UPDATE to seats table
CREATE OR REPLACE FUNCTION invalidate_availability_cache()
RETURNS TRIGGER AS $$
BEGIN
    -- Invalidate availability count for this seat's category
    PERFORM redis_delete(
        'availability:' || NEW.event_id || ':' || NEW.category
    );
    -- Invalidate event details (includes seat count)
    PERFORM redis_delete('event:' || NEW.event_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER seats_update_trigger
AFTER UPDATE ON seats
FOR EACH ROW
EXECUTE FUNCTION invalidate_availability_cache();
```

**Application-Level Invalidation (Preferred):**

```javascript
// When seat status changes
async function updateSeatStatus(seatId, newStatus, eventId, category) {
    // 1. Update DB
    await db.query(
        `UPDATE seats SET status = $1 WHERE id = $2`,
        [newStatus, seatId]
    );

    // 2. Invalidate Redis cache
    await redis.del(`availability:${eventId}:${category}`);

    // 3. Invalidate event cache (seat counts)
    await redis.del(`event:${eventId}`);

    // 4. Log change for audit
    await logSeatChange(seatId, newStatus);
}
```

### Why Application-Level is Better:

- ✅ Explicit: see invalidation logic in code
- ✅ Flexible: can add custom logic (e.g., notify other services)
- ✅ Debuggable: easy to add logging
- ✅ No trigger complexity: PostgreSQL triggers can hide behavior
- ✅ Works across DB replicas: application layer handles consistency

---

## Cache Hit Ratio Strategy

### Target Metrics:

- **Event details:** 99% hit ratio (rarely change)
- **Availability counts:** 85% hit ratio (changes frequently, but 30s TTL helps)
- **Seat maps:** 99% hit ratio (immutable)
- **User bookings:** 70% hit ratio (changes on each booking)

### How to Achieve High Hit Ratios:

1. **Warm the cache** on startup:
   ```javascript
   // On API server startup
   async function warmCache() {
       const events = await db.query('SELECT * FROM events WHERE status IN (?, ?)');
       for (const event of events) {
           await redis.set(`event:${event.id}`, JSON.stringify(event), 'EX', 3600);
           await redis.set(`seatmap:${event.id}`, getLayout(event), 'EX', 86400);
       }
   }
   ```

2. **Pre-populate availability counts** every 30 seconds:
   ```javascript
   // Background job
   setInterval(async () => {
       const events = await db.query('SELECT DISTINCT event_id FROM seats');
       for (const event of events) {
           const categories = ['vip', 'premium', 'general'];
           for (const category of categories) {
               const counts = await db.query(
                   'SELECT COUNT(*) as available FROM seats WHERE event_id=$1 AND category=$2 AND status="available"',
                   [event.id, category]
               );
               await redis.set(`availability:${event.id}:${category}`, counts);
           }
       }
   }, 30000);  // Every 30 seconds
   ```

3. **Monitor cache hit ratio:**
   ```
   Hit Ratio = Cache Hits / (Cache Hits + Cache Misses)
   
   Ideal: > 80%
   If < 70%: increase TTL or improve warming
   ```

---

## Cache Eviction Policy

Redis memory limit: 5 GB (on cache.r6g.large instance)

**Eviction Policy:** `allkeys-lru`
- When memory limit reached, evict least recently used keys
- Prioritizes keeping frequently accessed data

**Better approach: Set explicit TTL on all keys**
```javascript
// Never set a key without TTL
await redis.set(key, value, 'EX', ttl);  // ✅ Has TTL

// ❌ Wrong - no TTL
await redis.set(key, value);
```

**Memory Management:**
- Event cache: ~100 events × 2KB = 200 KB
- Availability cache: ~100 events × 3 categories × 1KB = 300 KB
- Seat map cache: ~100 events × 10KB = 1 MB
- Locks: ~50K concurrent × 100 bytes = 5 MB (temporary, auto-expires)
- Session cache: ~100K users × 200 bytes = 20 MB

**Total:** ~30 MB (well under 5GB limit) ✅

---

## Complete Cache Invalidation Flow

### When a User Books a Seat

```
Timeline:
T=0ms:    API: User clicks "Select Seat A-12"
T=5ms:    API: Try Redis lock → Success
T=10ms:   API: Update seats table → status='held'
T=15ms:   API: DELETE availability:{eventId}:vip
T=20ms:   API: DELETE event:{eventId}
T=25ms:   Redis lock auto-expires (50ms TTL)

Result:
- Seat A-12 is now held
- Availability count is stale (deleted, will be repopulated on next read)
- Event cache is stale (deleted, will be repopulated on next read)
- Next user who browses sees "341 VIP seats left" (down from 342)
- No stale data served; cache lag is minimal

T=30s:    Background job: repopulate availability:eventId:vip
          (even if not requested)
```

### When a User's Payment Succeeds

```
Timeline:
T=0s:     Payment worker: payment succeeds
T=5ms:    Worker: UPDATE seats SET status='booked'
T=10ms:   Worker: DELETE availability:{eventId}:{category}
T=15ms:   Worker: CREATE booking_seats records
T=20ms:   Worker: UPDATE bookings SET status='confirmed'
T=25ms:   Worker: DELETE user_bookings:{userId}
T=30ms:   Worker: Send SMS confirmation

Result:
- Booking is confirmed
- Seat is marked as sold
- Availability count is invalidated
- User's booking history is invalidated
- Next user to browse sees updated count
```

---

## Cache Layer Monitoring

### Metrics to Track:

```
1. Cache Hit Ratio (%) - target > 80%
   Hit Ratio = Redis Hits / (Redis Hits + Redis Misses)

2. Cache Miss Latency (ms) - when we have to hit DB
   Should be < 50ms for availability queries

3. Invalidation Latency (ms) - how long until cache is deleted
   Should be < 10ms

4. Memory Usage (MB) - Redis memory consumption
   Should stay < 500 MB (out of 5GB)

5. TTL Accuracy (%) - are keys expiring as expected?
   Monitor via redis.info() EXPIRED_KEYS counter
```

### CloudWatch Alarms:

```
- Alert if Hit Ratio < 70% (cache effectiveness declining)
- Alert if Miss Latency > 100ms (DB is slow)
- Alert if Memory Usage > 2GB (approaching limit)
- Alert if Invalidation fails (application error)
```

---

## Summary: Cache Decision Checklist

| Data | Cache? | TTL | Invalidation | Hit Ratio Target |
|------|--------|-----|--------------|-----------------|
| Event details | ✅ Yes | 1 hour | Event-driven | 99% |
| Availability count | ✅ Yes | 30 sec | Event-driven + TTL | 85% |
| Seat map layout | ✅ Yes | 24 hours | Event-driven | 99% |
| User bookings | ✅ Yes | 10 min | Event-driven + TTL | 70% |
| Individual seat status | ❌ No | — | Never cache | — |
| Payment status | ❌ No | — | Never cache | — |
| User auth | ❌ No | — | Never cache | — |

---

## Fallback: What If Redis Is Down?

**Scenario:** Redis cluster fails during ticket sale.

**Impact:**
- All cache keys are unavailable
- Application falls back to direct DB queries

**DB Query Performance:**
- `SELECT * FROM events WHERE id = $1` - ~5ms (indexed by PK)
- `SELECT COUNT(*) FROM seats WHERE event_id=$1 AND category=$2 AND status='available'` - ~20ms (partial index hit)
- `SELECT * FROM seats WHERE event_id=$1 AND section=$2` - ~10ms (indexed)

**System Behavior:**
- API responses take 20-50ms longer (DB queries instead of Redis)
- Database connection pool is now under more load
- At 8,333 RPS, can sustain for ~10 minutes before pool exhaustion
- Capacity drops to ~5,000 RPS (pool limited) after 10 minutes

**Mitigation:**
- Redis cluster with failover (3 nodes, automatic promotion)
- Backoff and timeout on DB queries (don't let slow query hang connection)
- Alert on Redis failures; engineer on-call
- Drain slow queries from pool; prioritize booking queries

**Conclusion:** Redis failure degrades performance but doesn't break correctness. System remains functional.

---

## Implementation Checklist

- [ ] Redis cluster provisioned (3 nodes, r6g.large)
- [ ] All cache keys have explicit TTL (no unbounded growth)
- [ ] Invalidation logic in application code (not triggers)
- [ ] Monitoring and alerting on hit ratio, memory, TTL expiry
- [ ] Fallback to DB queries when Redis unavailable
- [ ] Cache warming job on API startup
- [ ] Background repopulation job every 30 seconds
- [ ] Load testing to verify cache hit ratios under 5L concurrent users

This cache design ensures data freshness while providing 99%+ availability and fast response times even under peak load.
