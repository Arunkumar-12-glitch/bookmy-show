# Design Updates & Lessons Learned - ShowTime (Part B)

## Status: To be completed in Part B

This document will be updated after the live panel roast defense with:

1. **Feedback from Panel:** What questions were asked, what assumptions were challenged
2. **Lessons Learned:** What we got right, what we got wrong
3. **Design Revisions:** Any changes made based on feedback
4. **Future Improvements:** What would you change with unlimited budget/resources

**Updated in Part B after live defense.**

---

### Placeholder: Panel Roast Questions to Expect

Based on the design, we anticipate these questions:

1. **On Concurrency:**
   - "Why Redis SETNX over Redlock?"
   - "What happens if the 50ms lock expires mid-operation?"
   - "How do you test this at scale?"

2. **On Cache:**
   - "Why 30 seconds for availability cache? Why not 5 or 60?"
   - "Describe a scenario where cache invalidation fails and seats appear available when they're not"
   - "How do you monitor cache hit ratio in production?"

3. **On Budget:**
   - "This is $1,840/month. What's the first thing you'd cut if Coldplay added 5 more shows?"
   - "How does this design change if budget drops to $1,000?"
   - "Which component is the bottleneck first at 10L concurrent users?"

4. **On Async Queue:**
   - "Walk through a scenario where payment times out. What does the user see?"
   - "If SQS goes down for 10 minutes, what's the impact?"
   - "DLQ has 100 messages. How do you recover?"

5. **On Correctness:**
   - "Prove that this design prevents double-bookings even if every component fails randomly"
   - "What's the weakest link in your correctness story?"

---

See the four core design documents for comprehensive answers:
- [SCHEMA.md](SCHEMA.md)
- [CONCURRENCY.md](CONCURRENCY.md)
- [CACHE.md](CACHE.md)
- [QUEUE.md](QUEUE.md)
