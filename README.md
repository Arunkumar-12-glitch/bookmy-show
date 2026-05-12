# ShowTime - BookMyShow Architecture Design (Part A)

## Executive Summary

This repository contains the architectural design for ShowTime, a BookMyShow competitor built to handle 5 lakh concurrent users purchasing tickets for exclusive Coldplay India shows with zero tolerance for double-bookings and a strict $2,000/month AWS budget.

**Design Principle:** Every decision prioritizes correctness (zero double-bookings) first, then speed (sub-500ms response times), then cost efficiency (under $2,000/month).

---

## Part 1: Understanding the Constraints 🎯

### Constraint 1: 5 Lakh Concurrent Users at 12:00:00 Noon

**The Problem:**
- 500,000 users with the app open, finger on "Book Now" button
- All tap simultaneously at 12:00:00 noon
- Every user wants seat A-12 (worst case)

**Peak RPS Calculation:**
- Assumption: Users hit the booking endpoint in a concentrated burst over 60 seconds
- 500,000 requests / 60 seconds = **~8,333 sustained RPS** at baseline
- Reality: Spiky distribution - likely 15,000-20,000 RPS in the first 10 seconds, tapering after
- Per-seat concurrency: At 500 seats available × 5L users wanting them = intense contention

**What Hits the Bottleneck First:**
1. **API server capacity** - Need enough instances to handle 20K RPS
   - 1 Node.js instance handles ~5K RPS (single-threaded)
   - Need 4-6 instances minimum
2. **Database connection pool** - See Constraint 2 analysis
3. **Redis connection capacity** - If using distributed locks

**Our Allocation:** 6 × t3.xlarge EC2 instances = ~30K RPS total capacity ✅

---

### Constraint 2: Zero Acceptable Double-Bookings

**Technical Definition:**
A double-booking occurs when two bookings reference the same seat_id in the booking_seats table simultaneously - violating the uniqueness constraint.

**The Mechanism That Prevents It:**
We must atomically:
1. Check if seat is available
2. Mark seat as held/booked
3. Create booking record

All three must happen without interruption. Two options:

#### Option A: PostgreSQL SELECT FOR UPDATE
- **How it works:** Row-level lock at the database
- **SQL:**
  ```sql
  BEGIN;
  SELECT id FROM seats WHERE id = $1 AND status = 'available' FOR UPDATE;
  UPDATE seats SET status = 'held', held_by = $2 WHERE id = $1;
  INSERT INTO bookings (...) VALUES (...);
  COMMIT;
  ```
- **Guarantee:** Lock is held until commit; only one transaction can hold the lock
- **Hard limit:** PostgreSQL max_connections (typically 500 with PgBouncer)

#### Option B: Redis SETNX Distributed Lock
- **How it works:** Atomic "set if not exists" in Redis
- **Approach:**
  ```javascript
  const lockAcquired = await redis.set(
    `seat_lock:${seatId}`,
    userId,
    'NX',           // Only set if doesn't exist
    'EX', 50        // Expire after 50ms
  );
  ```
- **Guarantee:** Only one client can acquire the lock (atomic operation)
- **Hard limit:** Redis throughput (~100K ops/sec) and node failures

**Pool Exhaustion Math (Option A):**

With max_connections = 500 (PgBouncer):
- Assumption: 80% of requests are fast seat checks (20ms), 20% are payments (800ms)
- Connection held = (0.8 × RPS × 0.02) + (0.2 × RPS × 0.8)
- At RPS = 3,000: Connections held = (48) + (480) = 528 connections
- **Result: Pool exhausted at 3,000 RPS** ❌ (we need 8,333 RPS)

**With Option B (Redis):**
- Connection hold = (0.8 × RPS × 0.02) + (0.2 × RPS × 0.05)
  - Payment calls happen asynchronously in a worker, not in API
  - API returns immediately after enqueueing (5ms)
- At RPS = 8,333: Connections held = (133) + (8) = 141 connections
- **Result: Pool supports 8,333+ RPS** ✅

**Deadlock Risk (Multi-Seat Bookings):**
When booking 3 seats simultaneously:
- **Option A:** Two transactions each lock 3 seats. If both lock seats 1&2 but then wait for 3, deadlock occurs
  - Mitigation: Always lock seats in order (by seat ID), or use explicit deadlock retries
- **Option B:** Acquire locks sequentially; if any fails, release all - simpler, no deadlock

**Our Choice:** Hybrid approach (see CONCURRENCY.md)

---

### Constraint 3: $2,000/Month AWS Budget

**What $2,000 Actually Buys (Monthly):**

| Component | Size | Cost |
|-----------|------|------|
| EC2 (API servers) | 6 × t3.xlarge | $719 |
| RDS (Primary DB) | r6g.xlarge | $262 |
| RDS (Read replicas) | 2 × r6g.large | $262 |
| ElastiCache (Redis) | 3 × r6g.large | $359 |
| ECS Fargate workers | 0.25vCPU × 10 tasks | $73 |
| ALB + SQS + CloudFront | Networking | $165 |
| **Total** | | **~$1,840** ✅ |

**Budget Implications on Design Choices:**
1. **Cannot afford:** Expensive distributed consensus (Etcd, Consul), multi-region setup, premium support
2. **Must optimize:** Cache hit ratios (avoid expensive DB queries), connection pooling (connection creation is expensive)
3. **Must use:** Managed services (RDS, ElastiCache, SQS) over self-managed (cheaper but ops-heavy)

**If Budget Were $500/Month:**
- Drop to 2 EC2 instances, 1 RDS, single Redis node
- Would fail at 5L concurrent users
- Would need to pre-allocate all 50K popular seats via a queue system

---

## Design Documents

This architecture is documented across four core design documents:

1. **[SCHEMA.md](docs/SCHEMA.md)** - Complete PostgreSQL schema with all tables, constraints, indexes, and commentary
2. **[CONCURRENCY.md](docs/CONCURRENCY.md)** - Chosen concurrency strategy with RPS math and tradeoff analysis  
3. **[CACHE.md](docs/CACHE.md)** - Cache layer design: what to cache, TTL values, and precise invalidation triggers
4. **[QUEUE.md](docs/QUEUE.md)** - Async order processing flow: message format, worker logic, failure handling

Additional design documents for Part B:

5. **[ARCHITECTURE.md](docs/ARCHITECTURE.md)** - Full system diagram (drawn in Part B)
6. **[DESIGN-DECISIONS.md](docs/DESIGN-DECISIONS.md)** - Summary of all decisions and justifications (Part B)
7. **[DESIGN-UPDATES.md](docs/DESIGN-UPDATES.md)** - Lessons learned from panel roast (Part B)

---

## Key Design Decisions at a Glance

| Decision | Choice | Justification |
|----------|--------|---------------|
| Primary Key Type | UUID v4 | Prevents enumeration attacks; can be generated client-side for idempotency |
| Concurrency Strategy | Hybrid: Redis SETNX + PostgreSQL optimistic | Redis for hot path (seat holds), DB for final ACID guarantee |
| Cache TTL (Availability) | 30 seconds | Balance between freshness and cache hit ratio |
| Cache Invalidation | Event-driven + TTL | Targeted deletes ensure users see accurate counts |
| Order Flow | Async SQS queue | API returns in 50ms instead of waiting 2s for payment |
| Lock TTL (Seats) | 50ms (Redis), 10 min (DB hold) | Balances lost sales vs. lock contention |
| Read Replicas | 2 read replicas | Enables read scaling; one for seat map, one for user history |

---

## Development Checklist

- [x] Constraint analysis and RPS math
- [ ] Complete database schema in SCHEMA.md
- [ ] Concurrency strategy choice and math in CONCURRENCY.md
- [ ] Cache layer design in CACHE.md
- [ ] Async queue design in QUEUE.md
- [ ] Draw system architecture diagram
- [ ] Record defense video (3-5 minutes)
- [ ] Submit GitHub PR and video link

---

## How to Read These Documents

Start with **CONCURRENCY.md** - it contains the core decision that drives everything else.

Then read **SCHEMA.md** - understand why each table and column exists.

Then **CACHE.md** - understand what can be stale vs. what must be live.

Finally **QUEUE.md** - understand how async processing keeps the system responsive.

The four documents together form a complete, defensible architecture for 5L concurrent users with zero double-bookings and a $2,000/month budget.

---

**Status:** Part A (Design) - In Progress  
**Next:** Part B (Live Panel Roast Defense)