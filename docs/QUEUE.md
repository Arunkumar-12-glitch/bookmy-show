# Async Order Processing Queue Design - ShowTime

## Overview

The payment gateway takes 200–2,000ms to respond. If the API waits for payment, that time × 500K concurrent users = system collapse. The async queue pattern decouples payment processing from the API response, keeping the API fast and the DB responsive.

---

## Part 1: Why Async Is Non-Negotiable

### The Sync Anti-Pattern (Don't Do This)

```
User clicks "Pay Now"
  ↓
API Request arrives
  ↓
API: Hold seats in DB (20ms)
API: Call Razorpay API (200-2000ms) ← DATABASE CONNECTION HELD ENTIRE TIME
API: Update booking status (10ms)
API: Respond to user

Total time: 230-2010ms
DB connection held: 230-2010ms
```

### Connection Pool Exhaustion Math

```
Assumptions:
- 80% of requests = fast operations (20ms)
- 20% of requests = payment calls (800ms average)
- DB connection pool = 500 (max_connections via PgBouncer)

Connections held = (0.8 × RPS × 0.020) + (0.2 × RPS × 0.800)
                 = RPS × (0.016 + 0.160)
                 = RPS × 0.176

At RPS = 3,000:
Connections held = 3,000 × 0.176 = 528 connections

Pool size = 500 connections
Pool exhausted at 3,000 RPS ❌

But we need 8,333 RPS! System collapses. ❌
```

### The Async Solution

```
User clicks "Pay Now"
  ↓
API Request arrives
  ↓
API: Hold seats in DB (20ms) ← Quick, releases connection
API: Create booking record (10ms)
API: Publish to SQS queue (5ms)
API: Return 202 "Payment in progress"

Total time: 35ms ✅
DB connection held: 30ms ✅

[Meanwhile, in the background...]

Payment Worker: Reads message from SQS
Payment Worker: Calls Razorpay API (200-2000ms) ← No API connection held
Payment Worker: Updates booking status in DB (10ms)
Payment Worker: Sends SMS confirmation (100ms)
Payment Worker: Deletes message from SQS
```

### Connection Pool with Async

```
Connections held = (0.8 × RPS × 0.020) + (0.2 × RPS × 0.005)
                 = RPS × (0.016 + 0.001)
                 = RPS × 0.017

At RPS = 8,333:
Connections held = 8,333 × 0.017 = 142 connections ✅ (well under 500)
```

**Async saves us 400+ connection slots and enables 2.8× higher throughput.** 🚀

---

## Part 2: SQS Queue Architecture

### Why SQS? (Not Kafka, Not RabbitMQ)

| Feature | SQS | Kafka | RabbitMQ |
|---------|-----|-------|----------|
| **Setup Time** | 5 minutes | Days (self-managed) | Hours |
| **Operations** | Fully managed | Must manage cluster | Must manage |
| **Cost at Scale** | $0.40/million requests | $1/GB ingress | $0.00 (self-managed) |
| **Durability** | 99.9999999% | 99.9% | 99.9% |
| **Delivery Guarantees** | At-least-once | At-least-once | At-least-once |
| **Dead Letter Queue** | Built-in | Custom | Custom |
| **Monitoring** | CloudWatch native | Custom | Custom |

**For ShowTime:** SQS wins on ops burden and cost. We're already using AWS.

### Queue Configuration

**Queue Name:** `showtime-payment-queue`

**Visibility Timeout:** 60 seconds
- When worker reads message, it becomes "invisible" for 60 seconds
- If worker crashes, message reappears after 60s for another worker to retry
- Why 60s? Payment calls take 200-2000ms, we add buffer: 2 × 2000ms = 4s + 56s buffer = 60s
- If worker takes > 60s, message reappears and might be processed twice (idempotency required)

**Max Receive Count:** 3
- After 3 failed processing attempts, message goes to DLQ
- Why 3? Transient network errors usually resolve within 3 retries

**Message Retention Period:** 4 days
- Messages stay in queue for 4 days if not processed
- Protects against worker crashes lasting < 4 days

**DLQ (Dead Letter Queue):** `showtime-payment-queue-dlq`
- Messages that failed 3 times go here
- Monitored by CloudWatch alarm
- An engineer manually investigates

---

## Part 3: Queue Message Format

### Message Structure

```json
{
    "bookingId": "550e8400-e29b-41d4-a716-446655440000",
    "userId": "660e8400-e29b-41d4-a716-446655440001",
    "eventId": 123,
    "totalAmount": 4500.00,
    "currency": "INR",
    "seatIds": [1001, 1002, 1003],
    "paymentToken": "tokens_abc123xyz789...",
    "paymentMethod": "card",
    "cardLast4": "4242",
    "idempotencyKey": "550e8400-e29b-41d4-a716-446655440000",
    "enqueuedAt": "2024-12-15T18:15:45.123Z",
    "expiresAt": "2024-12-15T18:25:45.123Z"
}
```

### Field Explanations

| Field | Purpose | Why It's Here |
|-------|---------|--------------|
| **bookingId** | Unique booking identifier | Links payment to booking record |
| **userId** | User who made the booking | Send SMS notification to right user |
| **eventId** | Event being booked | For audit and analytics |
| **totalAmount** | Price in paisa (smallest unit) | Worker verifies this matches DB |
| **currency** | INR (assumed) | For display in notifications |
| **seatIds** | List of seat IDs in this booking | Worker updates these seats to 'booked' |
| **paymentToken** | Token from frontend (from Razorpay) | Worker uses this to complete payment |
| **paymentMethod** | How user is paying (card/upi/wallet) | For analytics and retry logic |
| **cardLast4** | Last 4 digits of card | For SMS confirmation (secure) |
| **idempotencyKey** | Same as bookingId | If message is processed twice, payment gateway deduplicates |
| **enqueuedAt** | When message was created | For latency tracking |
| **expiresAt** | When to stop retrying | After this, mark as abandoned |

### Why This Message is Self-Contained

The worker **does not query the DB** for most information. It has:
- ✅ Booking ID (for updating DB)
- ✅ Seat IDs (for marking seats as booked)
- ✅ Payment amount (for verification)
- ✅ User ID (for notification)

This minimizes DB queries and prevents the worker from making wrong decisions due to stale DB state.

---

## Part 4: Worker Logic - Success and Failure Paths

### Complete Worker Flow

```javascript
// Pseudo-code for payment worker

async function processPaymentMessage(message) {
    const {
        bookingId,
        userId,
        seatIds,
        totalAmount,
        paymentToken,
        idempotencyKey,
        expiresAt
    } = JSON.parse(message.Body);

    console.log(`[Worker] Processing booking: ${bookingId}`);

    // Step 1: Validate message expiry
    if (new Date() > new Date(expiresAt)) {
        console.log(`[Worker] Booking expired: ${bookingId}`);
        await markBookingExpired(bookingId);
        await deleteMessageFromQueue(message);
        return;
    }

    // Step 2: Get current booking state from DB
    const booking = await db.query(
        'SELECT * FROM bookings WHERE id = $1',
        [bookingId]
    );

    if (!booking) {
        console.error(`[Worker] Booking not found: ${bookingId}`);
        await deleteMessageFromQueue(message);  // Don't retry, nothing to process
        return;
    }

    // Step 3: Check if already processed
    if (booking.status === 'confirmed') {
        console.log(`[Worker] Booking already confirmed: ${bookingId}`);
        await deleteMessageFromQueue(message);  // Already done
        return;
    }

    // Step 4: Call payment gateway
    let paymentResult;
    try {
        paymentResult = await callPaymentGateway({
            paymentToken,
            amount: totalAmount,
            idempotencyKey,  // Prevents duplicate charges
            timeout: 30000   // 30s timeout, fail fast
        });
    } catch (error) {
        console.error(`[Worker] Payment gateway error: ${error.message}`);
        
        // Determine if error is retryable
        if (isRetryableError(error)) {
            // Let message visibility timeout expire (it will be retried by SQS)
            console.log(`[Worker] Retryable error, releasing message for retry`);
            return;  // Don't delete message
        } else {
            // Non-retryable error (invalid token, expired, etc.)
            console.log(`[Worker] Non-retryable error: ${error.message}`);
            await markBookingFailed(bookingId, error.message);
            await notifyUserPaymentFailed(userId, bookingId, totalAmount);
            await deleteMessageFromQueue(message);
            return;
        }
    }

    // Step 5: Process payment result
    if (paymentResult.status === 'success') {
        console.log(`[Worker] Payment succeeded: ${paymentResult.paymentId}`);
        
        try {
            // TRANSACTION: Commit all updates atomically
            const transaction = await db.transaction();

            try {
                // Step 5a: Update booking to confirmed
                await transaction.query(
                    `UPDATE bookings 
                     SET status = 'confirmed', 
                         payment_reference = $1,
                         payment_response_code = $2,
                         completed_at = NOW()
                     WHERE id = $3`,
                    [paymentResult.paymentId, paymentResult.code, bookingId]
                );

                // Step 5b: Mark all seats as booked
                for (const seatId of seatIds) {
                    const result = await transaction.query(
                        `UPDATE seats 
                         SET status = 'booked', version = version + 1
                         WHERE id = $1 AND status = 'held'`,
                        [seatId]
                    );

                    if (result.rowCount === 0) {
                        throw new Error(`Seat ${seatId} not in held state`);
                    }
                }

                // Step 5c: Create booking_seats junction records
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

                // Step 5d: Invalidate caches
                await redis.del(`user_bookings:${userId}`);

                await transaction.commit();
                console.log(`[Worker] Transaction committed for booking: ${bookingId}`);

            } catch (txError) {
                await transaction.rollback();
                console.error(`[Worker] Transaction failed: ${txError.message}`);
                // Let message retry (leave it in queue, visibility timeout will expire)
                return;
            }

            // Step 5e: Send confirmation to user
            try {
                await sendSmsConfirmation(userId, bookingId);
                await sendEmailConfirmation(userId, bookingId);
            } catch (notificationError) {
                // Notification failure is non-critical
                console.error(`[Worker] Failed to send notification: ${notificationError}`);
                // But we still delete message (booking is confirmed)
            }

            // Step 5f: Delete message from queue (success!)
            await deleteMessageFromQueue(message);
            console.log(`[Worker] Booking completed: ${bookingId}`);

        } catch (error) {
            console.error(`[Worker] Unexpected error during success path: ${error.message}`);
            // Retry the message (let visibility timeout expire)
            return;
        }

    } else {
        // Payment failed
        console.log(`[Worker] Payment failed: ${paymentResult.failureReason}`);

        try {
            // TRANSACTION: Release seats and mark booking as failed
            const transaction = await db.transaction();

            try {
                // Step 6a: Update booking to failed
                await transaction.query(
                    `UPDATE bookings 
                     SET status = 'failed', 
                         payment_response_code = $1,
                         completed_at = NOW()
                     WHERE id = $2`,
                    [paymentResult.code, bookingId]
                );

                // Step 6b: Release all held seats
                for (const seatId of seatIds) {
                    await transaction.query(
                        `UPDATE seats 
                         SET status = 'available', 
                             held_by = NULL, 
                             held_until = NULL,
                             version = version + 1
                         WHERE id = $1`,
                        [seatId]
                    );
                }

                // Step 6c: Invalidate caches
                await redis.del(`user_bookings:${userId}`);

                await transaction.commit();
                console.log(`[Worker] Failed booking cleaned up: ${bookingId}`);

            } catch (txError) {
                await transaction.rollback();
                console.error(`[Worker] Failed to clean up after payment failure: ${txError.message}`);
                return;  // Retry
            }

            // Step 6d: Notify user
            try {
                await sendSmsFailure(userId, totalAmount, paymentResult.failureReason);
            } catch (notificationError) {
                console.error(`[Worker] Failed to notify user of failure: ${notificationError}`);
            }

            // Step 6e: Delete message from queue
            await deleteMessageFromQueue(message);
            console.log(`[Worker] Payment failure handled: ${bookingId}`);

        } catch (error) {
            console.error(`[Worker] Error in failure path: ${error.message}`);
            return;  // Retry
        }
    }
}

// Helper: Determine if error is retryable
function isRetryableError(error) {
    if (error.code === 'TIMEOUT') return true;
    if (error.code === 'ECONNREFUSED') return true;
    if (error.code === 'ECONNRESET') return true;
    if (error.status === 502 || error.status === 503) return true;  // Gateway errors

    if (error.code === 'INVALID_TOKEN') return false;
    if (error.code === 'EXPIRED_TOKEN') return false;
    if (error.status === 400 || error.status === 401) return false;  // Client errors

    return false;  // Default: non-retryable
}
```

---

## Part 5: Edge Cases

### Edge Case 1: API Crashes After SQS Publish

**Scenario:**
- API successfully publishes message to SQS
- API crashes before sending 202 response to user
- User never knows if payment was queued

**What Happens:**

```
T=0s:      API receives payment request
T=5ms:     API: Hold seats in DB ✅
T=10ms:    API: Create booking (status='pending') ✅
T=15ms:    API: Publish to SQS ✅ (SQS acknowledges receipt)
T=20ms:    API: Trying to send 202 response
           → API process crashes 💥
           → HTTP connection closes without response
           → Browser shows error "Connection closed" or timeout

[User experience:]
- User sees error message
- User doesn't know if booking went through
- User might click "Try again" creating duplicate booking attempt

[What actually happened:]
- Message in SQS queue waiting to be processed
- Booking exists in DB with status='pending'
- Seats are held in DB (held_until = NOW() + 10 minutes)
- Payment worker will eventually pick up the message from SQS

[Recovery:]
- User can retry the payment (browser "Pay Now" button again)
- Payment worker will see booking already exists with status='pending'
- Worker calls payment gateway with same idempotencyKey
- Payment gateway deduplicates: "This payment already in progress"
- Payment worker retries until success or error
- Eventually, user sees confirmation SMS or receives refund
```

**Mitigation:**
1. **Idempotency Key:** Same bookingId ensures payment gateway doesn't double-charge
2. **Polling:** User's app can poll `/api/bookings/{bookingId}/status` to check if booking eventually confirmed
3. **Email Confirmation:** Even if user doesn't see SMS, confirmation email arrives with booking details
4. **User Support:** Worst case, customer service can look up booking by email and confirm status

**Conclusion:** User might not get immediate confirmation, but system remains consistent. Booking eventually succeeds or fails. ✅

---

### Edge Case 2: Payment Gateway Returns Timeout (Neither Success Nor Failure)

**Scenario:**
- Worker calls Razorpay
- Razorpay processes the payment
- Network dies before response reaches worker
- Worker never gets success/failure response

**How Worker Decides:**

```javascript
const paymentResult = await callPaymentGateway({
    paymentToken,
    amount: totalAmount,
    idempotencyKey,
    timeout: 30000  // 30s timeout
});
// Network timeout before response arrives
// Throws error with code 'TIMEOUT'

// In the error handler:
if (error.code === 'TIMEOUT') {
    console.log(`[Worker] Timeout - payment status unknown`);
    // Don't mark as success (might double-charge)
    // Don't mark as failed (might lose the payment)
    // Let message visibility timeout expire
    // Message is retried by SQS
    return;
}
```

**What Happens Next:**

```
T=0s:      Worker 1: Calls payment gateway, times out
T=5s:      Message visibility timeout runs out
T=65s:     Worker 2: Picks up same message, retries
           Calls payment gateway with same idempotencyKey
           Razorpay: "Payment already processed in 2024-12-15T18:15:45Z"
           Response: SUCCESS (deduplicated)
T=70s:     Worker 2: Receives successful duplicate payment response
           Updates booking to 'confirmed'
           Deletes message
           Sends SMS confirmation
```

**Why This Works:**
- **Idempotency Key** prevents double-charging
- **Retry** eventually gets the result
- **User gets confirmation** either way

**What If Razorpay Fails and Discards the Payment?**
```
T=0s:      Worker 1: Calls payment gateway, times out
T=65s:     Worker 2: Retries, calls Razorpay
           Razorpay: "No payment found for this idempotencyKey"
           Response: FAILURE (payment never went through)
T=70s:     Worker 2: Receives failure response
           Marks booking as 'failed'
           Releases seats
           Sends SMS: "Payment failed, please retry"
           User retries with fresh payment token
```

**Conclusion:** Timeout is handled gracefully. Retry logic + idempotency ensure the right result. ✅

---

### Edge Case 3: Worker Crashes After DB Commit But Before Message Delete

**Scenario:**
- Worker successfully charges payment
- Worker successfully updates DB (booking confirmed, seats booked)
- Worker crashes before calling `deleteMessageFromQueue`
- Message remains in queue

**What Happens:**

```
T=0s:      Worker: Process message, call payment gateway → SUCCESS
T=20ms:    Worker: Commit transaction (booking confirmed)
T=25ms:    Worker: Trying to delete message from SQS
           → Worker process crashes 💥
           → Message delete never sends to SQS

[Queue state:]
- Message is invisible for visibility_timeout (60s)
- After 60s, message reappears in queue
- Another worker picks it up

T=65s:     Worker 2: Reads same message from SQS
           Booking ID: check DB → status='confirmed' (already done!)
           Return early without re-processing
           Delete message (now succeeds)
```

**Why This Works:**
- DB commit is persistent
- Message remains in queue as a "backup notification"
- Next worker sees booking is already confirmed
- Worker idempotently skips reprocessing
- Message is eventually cleaned up

**Mitigation:**
- Idempotency check: `if (booking.status === 'confirmed') return;`
- Exactly-once semantics through deduplication

**Conclusion:** Even if worker crashes, no double-charging occurs. ✅

---

## Part 6: DLQ (Dead Letter Queue) Handling

A message goes to DLQ after 3 failed processing attempts.

**DLQ Message Example:**
```json
{
    "original_message": {
        "bookingId": "550e8400-e29b-41d4-a716-446655440000",
        ...
    },
    "receive_count": 3,
    "first_received_at": "2024-12-15T18:15:45.123Z",
    "last_received_at": "2024-12-15T18:16:45.123Z",
    "failure_reason": "ECONNREFUSED: Connection refused to payment gateway"
}
```

**DLQ Monitoring:**
```
CloudWatch Alarm:
- Alert if any message in DLQ
- Page on-call engineer
- Trigger: ApproximateNumberOfMessagesVisible > 0 on DLQ
```

**Manual Investigation:**
```
Engineer checks DLQ message:
1. Identify failure reason
2. Check payment gateway status page
3. If payment gateway is down:
   - Wait for recovery
   - Update message with recovered payment status
   - Requeue to main queue
4. If booking is expired:
   - Mark as abandoned
   - Release seats
   - Don't charge user
5. If payment went through but DB didn't update:
   - Manually confirm booking
   - Send SMS to user
```

---

## Part 7: SQS Configuration Values

| Setting | Value | Rationale |
|---------|-------|-----------|
| **Visibility Timeout** | 60 seconds | 2× max payment time (2s) + buffer |
| **Max Receive Count** | 3 | Transient errors usually resolve by 3rd retry |
| **Message Retention** | 4 days | Buffer for worker downtime |
| **Batch Size** | 10 messages/worker | Workers process multiple bookings in parallel |
| **Long Polling** | 20 seconds | Reduces API calls to SQS, saves cost |
| **Lambda/Worker Concurrency** | 10 workers | Can process 10 bookings in parallel |

---

## Part 8: Example Scenarios

### Scenario A: Happy Path - Payment Succeeds

```
T=0s:      User submits payment form
T=50ms:    API:
           - Holds seats ✅
           - Creates booking (pending) ✅
           - Publishes to SQS ✅
           - Returns 202 ✅
           Total: 50ms ✅ (under 500ms limit)

T=100ms:   User sees "Payment in progress" screen

T=5s:      Payment worker reads message from SQS
T=500ms:   Worker calls Razorpay → SUCCESS
T=520ms:   Worker updates DB:
           - booking status → confirmed ✅
           - seats status → booked ✅
T=530ms:   Worker sends SMS: "Booking confirmed, Seats: A-1, A-2, A-3"
T=535ms:   Worker deletes message from SQS ✅

T=6s:      User receives SMS confirmation ✅
           User sees "Booking confirmed" on app ✅
```

### Scenario B: Payment Fails

```
T=0s:      User submits payment form
T=50ms:    API returns 202 ✅

T=5s:      Payment worker reads message
T=500ms:   Worker calls Razorpay → FAILURE (declined card)
T=520ms:   Worker updates DB:
           - booking status → failed ✅
           - seats status → available (released) ✅
T=530ms:   Worker sends SMS: "Payment declined. Please try another card."
T=535ms:   Worker deletes message ✅

T=6s:      User receives SMS ✅
           User can retry with different card ✅
           Seats are available for others ✅
```

### Scenario C: Timeout and Retry

```
T=0s:      User submits payment form
T=50ms:    API returns 202 ✅

T=5s:      Worker 1 reads message
T=35s:     Worker 1: Calls payment gateway
T=65s:     Worker 1: Timeout (network glitch)
           Message visibility expires, returns to queue
           
T=66s:     Worker 2 reads same message
T=500ms:   Worker 2: Calls payment gateway with idempotencyKey
           Razorpay: "Payment already processed, status=CONFIRMED"
           Response: SUCCESS (deduplicated by Razorpay)
T=520ms:   Worker 2 updates DB (booking confirmed)
T=535ms:   Worker 2 deletes message ✅

T=37s:     User eventually receives SMS confirmation
           (But it might be delayed 30+ seconds due to retry)
```

---

## Complete SQS + Worker Deployment

### Infrastructure:
```
AWS SQS:
- showtime-payment-queue (main queue)
- showtime-payment-queue-dlq (dead letter queue)

Payment Workers:
- ECS Fargate service with 10 tasks
- Each task: 0.25 vCPU, 0.5 GB memory
- Auto-scale based on SQS queue depth
- Minimum 5 tasks, Maximum 20 tasks

CloudWatch:
- Alarm: DLQ message count > 0 (page engineer)
- Alarm: Queue depth > 1000 (scale up workers)
- Metric: Message processing time (p95 < 5s)
- Metric: Success rate > 99%
```

### Deployment:
```bash
# Deploy worker Docker image
aws ecs update-service \
  --cluster payment-workers \
  --service payment-worker-service \
  --force-new-deployment

# Monitor
aws sqs get-queue-attributes \
  --queue-url https://sqs.ap-south-1.amazonaws.com/xxx/showtime-payment-queue \
  --attribute-names ApproximateNumberOfMessages
```

---

## Summary: Queue Design Checklist

- [x] Async processing decouples payment from API response
- [x] Keeps API response time < 100ms (target: 50ms)
- [x] Frees up DB connections (pool exhaustion solved)
- [x] Idempotency prevents double-charging
- [x] DLQ catches persistent failures
- [x] Edge cases handled (crash, timeout, duplicate)
- [x] Fallback behavior defined for worker failures
- [x] Monitoring and alerting on critical paths
- [x] SQS configuration optimized for payment processing

This async queue design ensures 5L concurrent users can all initiate payments simultaneously without overwhelming the API or database, while maintaining correctness (no double-bookings, no lost payments).
