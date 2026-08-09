# Idempotency

> *"In distributed systems, retries are inevitable. Idempotency ensures retries never produce duplicate business effects."*

---

# Table of Contents

1. Why Idempotency Exists
2. Retry Semantics
3. API Idempotency
4. Kafka Consumer Idempotency
5. Payment Systems
6. Production Patterns

---

# Learning Objectives

After completing this chapter you should be able to:

- Explain why retries are unavoidable.
- Design idempotent APIs.
- Design idempotent consumers.
- Prevent duplicate payments.
- Explain Stripe's Idempotency-Key.
- Design production-grade retry mechanisms.

---

# The Reality of Distributed Systems

Every distributed system eventually performs retries.

Reasons include

- Network timeout
- TCP retransmission
- Process crash
- Leader failover
- Client retry
- Load balancer retry
- Reverse proxy retry
- Kafka redelivery
- Consumer restart

A Principal Engineer assumes retries are normal.

The real question becomes

> **Can the operation execute multiple times without changing the final business outcome?**

---

# What Is Idempotency?

An operation is idempotent if

```
Executing it once

↓

Produces

Result X
```

and

```
Executing it

100 times

↓

Still Produces

Result X
```

Notice

The operation itself

may execute multiple times.

Only

the business outcome

must remain unchanged.

---

# Mathematical Definition

Function

```
f(x)
```

is idempotent if

```
f(f(x))

=

f(x)
```

Distributed systems borrow exactly this concept.

---

# Why Retries Are Inevitable

Suppose

Client sends

```
Transfer ₹1000
```

Server completes

the transfer.

Immediately afterwards

the response packet

is lost.

Client waits.

Timeout occurs.

Client retries.

Question

Should another

₹1000

be transferred?

Absolutely not.

---

# Failure Timeline

```text
Client

↓

POST /payments

↓

Server Processes Payment

↓

Response Lost

↓

Timeout

↓

Retry

↓

Duplicate Request
```

Question

How does the server know

this is

the same request?

# Why Idempotency Is a Distributed Systems Requirement

The primary reason idempotency exists is not client retries—it is **uncertainty**.

In an asynchronous distributed system, the client can rarely distinguish between the following scenarios after a timeout:

1. The request never reached the server.
2. The request reached the server but was never executed.
3. The request executed successfully, but the response was lost.
4. The response was generated but lost by an intermediary (proxy, load balancer, gateway).
5. The request completed, but the server crashed before persisting the response.

From the client's perspective, these states are indistinguishable.

This uncertainty is a direct consequence of asynchronous communication and is closely related to the FLP impossibility result: timeouts indicate only that progress has not been observed, not that a particular outcome occurred.

As a result, the only safe client behavior is to retry.

The system therefore has a fundamental requirement:

> A retry must never produce an additional business effect.

Notice that this is stronger than preventing duplicate HTTP requests.

The real invariant is:

> **Every logical business operation must produce at most one externally visible effect, regardless of how many times the request is transmitted or processed.**

For a payment system, the invariant is not:

```
Only one HTTP POST is received.
```

It is:

```
The customer's account is charged exactly once.
```

Multiple request executions are acceptable.

Multiple financial effects are not.

# Failure Matrix

Consider a payment API.

The client issues

```
POST /payments
```

with a unique business operation identifier.

The server commits the payment successfully.

Before the response reaches the client, the TCP connection is terminated.

At this point, neither side has sufficient information.

The client cannot determine whether:

- the request was never executed,
- the request committed successfully,
- the request is still executing,
- or the response alone was lost.

Retrying is therefore the only correct behavior.

However, retries introduce a new invariant:

> Every retry must be recognized as the same logical operation.

The implementation mechanism—whether an Idempotency-Key, unique constraint, or deduplication table—is merely an implementation detail.

The invariant being preserved is that business state changes at most once per logical operation.
# Problem

What fundamental property are we trying to preserve?

# Why This Problem Exists

Explain the distributed systems limitation.

# Required Invariant

What must always remain true?

# Why Naive Solutions Fail

Failure analysis.

# Production Design

How large systems solve it.

# Trade-offs

What is sacrificed?

# Principal Engineer Insight

What interviewers actually expect.



---

# Failure Semantics Before Idempotency

Many engineers incorrectly view idempotency as a retry optimization.

It is not.

Idempotency is a correctness property that allows a distributed system to remain deterministic despite uncertainty.

Before discussing implementation techniques, we must understand the failure model.

---

# Distributed Systems Cannot Observe Reality

One of the most important concepts in distributed systems is that **no process has a globally correct view of reality**.

Suppose a client sends

```
POST /payments
```

After a timeout, the client knows only one fact:

> It has not yet received a response.

It does **not** know whether:

- the request never reached the server,
- the request reached the server but was discarded,
- the request committed successfully,
- the response was lost,
- the server committed and then crashed,
- or another replica already processed the request.

These states are observationally equivalent.

A timeout conveys lack of information—not failure.

This uncertainty is fundamental.

No amount of additional retry logic can eliminate it.

---

# The Core Invariant

The invariant is not

> "Every request executes exactly once."

That invariant is practically impossible in an asynchronous distributed system.

The real invariant is

> **Every logical business operation produces at most one externally visible side effect.**

Notice the distinction.

The transport layer may execute the request

- once,
- twice,
- or ten times.

The application must still produce one business outcome.

Examples:

```
Charge Customer

Maximum Visible Effect = 1
```

```
Issue Coupon

Maximum Visible Effect = 1
```

```
Reserve Seat

Maximum Visible Effect = 1
```

The implementation may retry indefinitely.

The business state must remain deterministic.

---

# Why Retries Are Mandatory

A common interview mistake is saying

> "Clients should avoid retries."

This is incorrect.

Without retries, temporary failures become permanent failures.

Imagine the following sequence.

```
Client

↓

Payment Service

↓

Database
```

The database commits successfully.

Immediately afterward,

the API server crashes before sending the HTTP response.

From the client's perspective,

the request appears to have failed.

From the server's perspective,

the payment already exists.

The client has only two choices:

1. Retry.
2. Give up forever.

Only one of these is operationally acceptable.

Therefore retries are not optional.

Distributed systems are designed assuming retries will occur.

---

# The Retry Paradox

Retries simultaneously improve reliability and threaten correctness.

Without retries:

- temporary failures become permanent failures.

With retries:

- duplicate business operations become possible.

Idempotency resolves this paradox.

It allows systems to aggressively retry while preserving business correctness.

This is one of the most important design principles in modern distributed systems.

---

# Transport Semantics vs Business Semantics

Another common misunderstanding is confusing HTTP semantics with business semantics.

Consider

```
PUT /users/123
```

HTTP defines PUT as idempotent.

However,

this says nothing about your business logic.

Suppose the implementation sends

```
Welcome Email
```

every time PUT executes.

Although the HTTP operation is technically idempotent,

the business behavior is not.

Principal Engineers evaluate idempotency at the business layer,

not the protocol layer.

---

# Business Operation Identity

The most important question is not

> "Did I receive this HTTP request before?"

The important question is

> "Have I already executed this logical business operation?"

Those are completely different questions.

Suppose a mobile application retries.

Every retry produces

- a different TCP connection,
- a different HTTP request,
- a different request identifier,
- possibly a different backend instance.

Yet every request represents

the same payment.

The system therefore needs a stable identifier representing the business operation,

not the network request.

---

# Designing the Correct Identity

Poor identifiers

```
Connection ID

HTTP Request ID

Load Balancer Request ID
```

These change on every retry.

Good identifiers

```
Payment ID

Order ID

Transfer ID

Idempotency-Key

Business Transaction ID
```

These remain constant across retries.

Notice that idempotency is fundamentally an identity problem.

---

# Principal Engineer Insight

A useful mental model is:

> Retries operate on messages.
>
> Idempotency operates on business operations.

This distinction explains why network-level exactly-once delivery is insufficient.

Even if the messaging system delivers a message only once,

multiple clients may independently submit the same logical business operation.

Conversely,

even if a message is delivered multiple times,

the business operation should still execute only once.

The invariant belongs to the application,

not the transport.

---

# Interview Discussion

**Interviewer**

Why can't we simply disable retries?

**Principal Engineer Answer**

Disabling retries converts transient failures into permanent failures, significantly reducing system availability. Distributed systems must assume message loss, node failures, and network partitions. Therefore retries are mandatory for liveness. Idempotency complements retries by ensuring repeated execution of the same logical business operation produces at most one externally visible side effect. The combination of retries and idempotency simultaneously provides availability and correctness.

---

# Common Interview Mistakes

❌ Thinking idempotency is an HTTP feature.

Idempotency is a business correctness property.

---

❌ Using request identifiers as idempotency keys.

Network requests and business operations have different identities.

---

❌ Attempting to eliminate retries.

Retries are essential for availability.

The correct solution is deterministic retry behavior.

---

# Key Takeaways

- Timeouts represent uncertainty, not failure.
- Retries are required for liveness.
- Idempotency preserves correctness under retries.
- The invariant applies to business operations, not transport messages.
- Stable business identifiers are the foundation of every idempotent design.



---

# API Idempotency — Production Design

Almost every payment platform exposes an API similar to

```http
POST /payments
```

Unlike

```
GET
PUT
DELETE
```

a payment request creates a new business operation.

Therefore,

HTTP semantics alone cannot provide idempotency.

The application must explicitly define the identity of the operation.

---

# The Fundamental Requirement

A client may retry the same logical request because of

- timeout
- connection reset
- gateway retry
- mobile reconnect
- browser refresh
- process restart

Every retry must produce one of two outcomes:

1. Return the previously computed result.
2. Continue processing the existing operation.

Creating a second payment is never acceptable.

---

# Business Identity

Stripe requires every payment request to include

```
Idempotency-Key
```

Example

```http
POST /payments

Idempotency-Key:
7f93d73d-8d7b-4e79-a548-cd84e44dbe53
```

Notice that this key identifies

the business operation,

not

the HTTP request.

Every retry carries

the same key.

---

# Required Invariant

The invariant is

> A given Idempotency-Key maps to exactly one business outcome.

Not

```
One request.
```

Not

```
One database row.
```

One

business outcome.

That outcome may be

```
Success

Failure

Validation Error
```

but it must never change.

---

# Naïve Implementation

Many systems implement

```sql
SELECT *

FROM idempotency

WHERE key=?
```

If not found

↓

Process request

↓

INSERT key

This implementation is incorrect.

---

# Race Condition

Suppose

two API servers receive

the same request simultaneously.

```
Server A

SELECT

↓

Not Found
```

```
Server B

SELECT

↓

Not Found
```

Both proceed.

Both charge the customer.

Both insert the key.

Business invariant violated.

The problem is

**check-then-act**.

---

# Correct Design

The idempotency record must be created atomically.

Typical SQL implementation

```sql
INSERT INTO idempotency_keys
(
    key,
    status,
    created_at
)
VALUES
(
    ?,
    'IN_PROGRESS',
    NOW()
)
ON CONFLICT DO NOTHING;
```

The insert itself becomes the synchronization primitive.

Exactly one server succeeds.

Every other server observes

that another request already owns the operation.

---

# Why "IN_PROGRESS" Exists

Suppose

Server A

successfully inserts

```
IN_PROGRESS
```

Immediately afterwards

the payment gateway call

takes

5 seconds.

Meanwhile

Server B

receives the same request.

Question

Should Server B

start another payment?

No.

It should detect

that another execution

already owns the business operation.

Possible strategies include:

- wait for completion,
- poll the existing record,
- return HTTP 409 (Conflict),
- return HTTP 202 (Accepted),
- or instruct the client to retry later.

The correct choice depends on API semantics and latency objectives.

---

# State Machine

A production implementation typically stores

```
IN_PROGRESS

↓

COMPLETED

↓

EXPIRED
```

Some systems also include

```
FAILED
```

Notice

```
FAILED
```

does not always mean

the request may be safely retried.

Failures must be classified.

---

# Should We Cache the Response?

Suppose

the original request returned

```json
{
    "paymentId": "pay_123",
    "status": "SUCCESS"
}
```

A retry arrives.

Should the payment service

recompute the response?

Usually,

no.

Most payment systems persist

the original response body.

Future retries simply return

the stored response.

Benefits

- deterministic client behavior
- identical HTTP response
- lower latency
- simpler retry handling

---

# What Should Be Stored?

A typical schema

| Column | Purpose |
|---------|----------|
| idempotency_key | Business identity |
| request_hash | Detect payload mismatch |
| status | IN_PROGRESS / COMPLETED |
| response_code | HTTP status |
| response_body | Cached response |
| created_at | TTL management |
| expires_at | Cleanup |

The stored response effectively becomes part of the contract.

---

# Payload Validation

Suppose

the client sends

```
Idempotency-Key = K1

Amount = ₹500
```

Later,

the client accidentally retries

using

the same key

but

```
Amount = ₹700
```

Question

Should the request execute?

Absolutely not.

The server should reject it.

Therefore

many payment systems persist

a hash

of the original request.

Future retries compare

the incoming payload

with

the original.

Same key

+

Different payload

↓

HTTP 409

or

HTTP 422

depending on API semantics.

---

# TTL Strategy

Should idempotency keys

live forever?

No.

Storage grows without bound.

Typical retention

depends on business requirements.

| Domain | Typical TTL |
|---------|-------------|
| Payment | 24–72 hours |
| Checkout | 24 hours |
| Internal APIs | Minutes |
| Batch Jobs | Job lifetime |

Choosing TTL is a business decision,

not merely a storage optimization.

The retention period must exceed the maximum retry window.

---

# Redis or SQL?

Interview Question

Where should idempotency keys be stored?

The answer depends on the required guarantee.

### Redis

Advantages

- extremely low latency
- automatic TTL
- high throughput

Disadvantages

- persistence depends on configuration
- eviction policies may remove active keys
- failover may lose recently written data

Suitable when occasional duplicate execution is acceptable.

---

### Relational Database

Advantages

- transactional
- durable
- integrates naturally with payment records
- strong consistency

Disadvantages

- higher latency
- increased write contention

Preferred for financial operations.

---

# Multi-Region Considerations

Suppose

the same payment request

reaches

two different regions.

If each region maintains

an independent idempotency table,

duplicate processing becomes possible.

Therefore

the idempotency store

must itself provide

a globally consistent view,

or requests for the same business entity must be routed consistently to a single home region.

This trade-off often appears in multi-region payment architecture interviews.

---

# Principal Engineer Insight

> Idempotency is fundamentally a concurrency control problem.
>
> The challenge is not recognizing duplicate requests after they finish.
>
> The challenge is ensuring that **only one execution acquires ownership of the business operation**, even when identical requests arrive concurrently on different application instances.
>
> Atomic insertion, conditional writes, or distributed compare-and-set operations solve this ownership problem.

---

# Interview Discussion

**Interviewer**

Two identical payment requests arrive simultaneously at two API servers.

How do you guarantee only one payment is executed?

---

**Weak Answer**

Store the Idempotency-Key in Redis.

---

**Principal Engineer Answer**

The key alone is insufficient because both servers may observe that the key is absent and proceed concurrently. The solution requires an atomic ownership acquisition step, typically implemented using a unique database constraint, Redis `SET NX`, or another compare-and-set primitive. The server that acquires ownership transitions the operation to `IN_PROGRESS`, while all other requests observe the existing state and either wait, retry, or return the previously computed result once execution completes.

---

# Common Interview Mistakes

❌ Performing a `SELECT` before the `INSERT`.

This creates a race window.

---

❌ Using request IDs as idempotency keys.

Request identity and business identity are different.

---

❌ Storing only the key.

Persist the response and request hash as well.

---

❌ Using Redis without considering persistence and eviction.

Technology selection must match business correctness requirements.

---

# Key Takeaways

- Idempotency begins with a stable business identifier.
- Ownership of a business operation must be acquired atomically.
- The idempotency store should preserve both request identity and response semantics.
- Concurrent duplicate requests are a synchronization problem, not merely a lookup problem.
- Storage technology should be selected based on the required durability and consistency guarantees, not just latency.

---

# Kafka Consumer Idempotency

Producer-side idempotency solves only half of the problem.

The more difficult challenge begins after an event is delivered to a consumer.

Consider the following workflow.

```text
Kafka

↓

Inventory Consumer

↓

Inventory Database
```

The consumer processes

```
OrderCreated
```

and decrements inventory.

At first glance, the design appears correct.

The problem is that Kafka and the database have independent failure semantics.

---

# The Consumer Atomicity Problem

Assume the consumer performs the following sequence.

```
Read Event

↓

Update Database

↓

Commit Kafka Offset
```

Question

What happens if the database transaction commits successfully but the consumer crashes before committing the Kafka offset?

After restart,

Kafka redelivers the event because the offset was never committed.

The inventory update executes again.

Inventory is decremented twice.

Kafka behaved correctly.

The application did not.

---

# Reverse Failure

Now reverse the order.

```
Read Event

↓

Commit Offset

↓

Crash

↓

Database Never Updated
```

Kafka believes the message has been processed.

The business update is permanently lost.

Neither ordering solves the problem.

This is a classic example of coordinating two independent systems without atomic commit.

---

# The Required Invariant

The invariant is not

> "Every Kafka message is delivered exactly once."

The invariant is

> **Every business event produces its externally visible effect at most once.**

Kafka offsets are an implementation detail.

Business correctness is the objective.

---

# Why Kafka EOS Is Not Enough

Kafka's transactional producer and idempotent producer guarantee:

- records are not duplicated within Kafka,
- committed transactions become visible atomically,
- aborted transactions remain invisible.

These guarantees stop at the Kafka log.

Once the consumer executes

```
UPDATE inventory
```

Kafka has no visibility into the external database.

Application-level deduplication remains mandatory.

---

# Consumer State Machine

A robust consumer typically progresses through the following states.

```text
Received

↓

Persist Ownership

↓

Execute Business Logic

↓

Commit Database

↓

Commit Offset
```

The critical observation is that ownership must be acquired before business logic executes.

---

# Inbox Pattern Revisited

The Inbox Pattern provides durable ownership of an event.

Instead of processing immediately,

the consumer first persists the event inside its own database.

```text
Kafka

↓

Inbox Table

↓

Business Transaction

↓

Offset Commit
```

The inbox becomes the source of truth for consumer progress.

---

# Atomic Consumer Transaction

A typical relational implementation performs the following steps inside one local transaction.

```sql
BEGIN;

INSERT INTO inbox(event_id, received_at)
VALUES (?, NOW());

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = ?;

COMMIT;
```

Both operations succeed together.

If the transaction aborts,

the inbox record never appears,

allowing Kafka to safely redeliver the event.

---

# Why the Inbox Table Matters

Suppose the consumer crashes after committing the database transaction but before committing the Kafka offset.

Kafka redelivers the same event.

The consumer attempts

```sql
INSERT INTO inbox(event_id)
```

The unique constraint rejects the duplicate.

Business logic is skipped.

The consumer simply commits the Kafka offset.

The duplicate event has been transformed into a harmless replay.

---

# UPSERT vs SELECT-Then-INSERT

Many implementations perform

```sql
SELECT *

FROM inbox

WHERE event_id = ?
```

followed by

```sql
INSERT
```

This introduces the same race condition discussed in API idempotency.

Two consumer instances may observe

```
Not Found
```

simultaneously.

The correct implementation relies on an atomic uniqueness guarantee.

Examples include:

PostgreSQL

```sql
INSERT ...

ON CONFLICT DO NOTHING;
```

MySQL

```sql
INSERT IGNORE
```

or

```sql
INSERT ...

ON DUPLICATE KEY UPDATE
```

DynamoDB

```
ConditionExpression
attribute_not_exists(pk)
```

Redis

```
SET key value NX
```

The synchronization primitive varies.

The invariant does not.

---

# Partition Rebalancing

Kafka consumers periodically rebalance partitions.

Suppose

Consumer A

owns

Partition 3.

During processing,

Consumer A crashes.

Kafka assigns

Partition 3

to

Consumer B.

Consumer B reprocesses the last uncommitted event.

Without idempotency,

business state diverges.

With an Inbox table,

duplicate execution becomes harmless.

---

# Poison Messages

Not every failure is transient.

Suppose an event contains malformed business data.

Retries produce the same exception forever.

Infinite retries prevent progress for every subsequent event in the partition.

This is known as a poison message.

Typical production strategies include:

- Dead Letter Queue (DLQ)
- Retry Topics
- Exponential Backoff
- Manual Replay
- Operational Alerting

Blind retries are not a recovery strategy.

---

# End-to-End Exactly-Once Effects

A reliable event-driven architecture typically combines:

```text
Producer

↓

Transactional Outbox

↓

CDC

↓

Kafka

↓

Inbox

↓

Business Transaction

↓

Offset Commit
```

Notice that no component independently guarantees exactly-once processing.

Correctness emerges from composing multiple idempotent components.

---

# Production Example — Inventory Reservation

Order Service publishes

```
OrderCreated
```

Inventory Consumer receives the event.

Business transaction:

1. Insert Event ID into Inbox.
2. Reserve inventory.
3. Commit transaction.
4. Commit Kafka offset.

If any failure occurs before step 3,

Kafka safely retries.

If failure occurs after step 3,

the Inbox prevents duplicate reservation.

---

# Principal Engineer Insight

> Kafka offsets represent delivery progress.
>
> They do **not** represent business progress.
>
> One of the most common design mistakes is assuming that committing a Kafka offset implies that the associated business operation completed successfully.
>
> Principal Engineers separate messaging correctness from business correctness and make each independently recoverable.

---

# Interview Discussion

**Interviewer**

Kafka guarantees Exactly-Once Semantics.

Why do you still need an Inbox table?

---

**Weak Answer**

For duplicate events.

---

**Principal Engineer Answer**

Kafka's Exactly-Once Semantics guarantee transactional correctness within Kafka itself. They do not extend to external systems such as relational databases. A consumer may successfully update its database and crash before committing the Kafka offset, causing Kafka to redeliver the event. The Inbox table provides durable business-level deduplication by atomically recording event ownership alongside the business transaction. This converts duplicate deliveries into idempotent replays while preserving business correctness.

---

# Common Interview Mistakes

❌ Equating Kafka offset commits with successful business processing.

Offset progression and business state are independent.

---

❌ Trusting Kafka EOS to eliminate application duplicates.

Kafka cannot coordinate your database transaction.

---

❌ Using `SELECT` before `INSERT` in deduplication tables.

This creates a race window.

---

❌ Retrying poison messages indefinitely.

Retries without classification reduce availability.

---

# Key Takeaways

- Kafka delivery guarantees and business correctness are separate concerns.
- Offset commits must never be treated as proof of successful business processing.
- Inbox tables provide durable event ownership and deduplication.
- Deduplication must rely on atomic ownership acquisition rather than check-then-act logic.
- End-to-end exactly-once effects emerge from combining Outbox, CDC, Inbox, idempotent business logic, and reliable offset management.

---

# Distributed Idempotency in Multi-Region Systems

Everything discussed so far assumes a single logical idempotency store.

That assumption breaks down in a globally distributed architecture.

Consider a payment platform deployed in three active regions.

```text
US-East

Europe

Asia
```

Each region independently receives client requests.

The challenge is no longer duplicate retries.

The challenge becomes **distributed ownership**.

---

# The Problem

Suppose a mobile client submits

```
POST /payments
```

with

```
Idempotency-Key = P123
```

The request times out.

The mobile SDK retries.

Because of DNS changes, Anycast routing, or Global Load Balancing, the retry reaches another region.

Now two independent application clusters are processing the same logical payment.

```text
Region A

Payment(P123)
```

```text
Region B

Payment(P123)
```

Both requests are valid.

Neither region knows the other exists.

Without coordination,

both may successfully charge the customer.

Notice that this failure occurs **even if every region is internally correct**.

---

# Why Local Idempotency Fails

Suppose each region maintains its own database.

```text
Region A

idempotency_keys
```

```text
Region B

idempotency_keys
```

Processing timeline

```
Region A

INSERT P123

Success
```

```
Region B

INSERT P123

Success
```

Both inserts succeed because they occur against different databases.

Each region believes it owns the business operation.

This is not a race condition.

It is a distributed ownership problem.

---

# The Required Invariant

The invariant changes from

> "One server processes one request."

to

> **Exactly one region owns a given business operation globally.**

Everything else is an implementation strategy.

---

# Strategy 1 — Home Region Routing

This is the approach used by many payment systems.

Every business entity has a deterministic home.

Example

```text
Payment ID

↓

Hash

↓

Region
```

```
Payment P123

↓

US-East
```

Regardless of where the client connects,

the request is internally forwarded to the owning region.

```text
Client

↓

Europe

↓

Forward

↓

US-East

↓

Process
```

Advantages

- Simple reasoning
- Single writer
- Strong correctness
- Local idempotency remains sufficient

Disadvantages

- Extra cross-region latency
- Temporary dependence on the home region

---

# Why Hashing Works

A deterministic hash guarantees

```
Payment P123

↓

Always Region A
```

Every retry

every client

every application instance

computes the same destination.

Ownership becomes deterministic.

No distributed coordination is required.

This is conceptually similar to consistent hashing used in distributed caches.

---

# Strategy 2 — Globally Consistent Database

Instead of routing,

every region writes into a globally consistent datastore.

Examples include

- Spanner
- CockroachDB
- YugabyteDB

All regions observe the same uniqueness constraint.

```sql
INSERT idempotency_key
```

becomes globally serialized.

Advantages

- Any region may process requests
- Strong consistency
- No application-level routing

Trade-offs

- Higher write latency
- Cross-region quorum
- Increased operational complexity

Correctness is purchased with latency.

---

# Strategy 3 — Active-Passive Processing

Some financial systems intentionally avoid active-active writes.

Only one region accepts payment creation.

Other regions remain read-only or standby.

Advantages

- Simpler correctness model
- Easier auditing
- Predictable recovery

Disadvantages

- Lower regional write availability
- Failover procedures become operationally significant

Many regulated financial institutions still prefer this architecture.

---

# Strategy 4 — Distributed Lock

Another possibility is acquiring a global lock.

Example

```
SET payment:P123 NX
```

Question

Is this sufficient?

Usually not.

Why?

The lock service itself becomes a distributed system.

Questions immediately arise:

- How is the lock replicated?
- What happens during failover?
- Can leases expire?
- What if network partitions occur?

The problem has simply moved.

Principal Engineers recognize that distributed locks require their own correctness proof.

---

# Why Redis Alone Is Not Enough

Interview Question

Can we store Idempotency Keys in Redis?

The correct answer is

"It depends."

Single-node Redis

provides atomic operations.

However,

a replicated Redis deployment introduces new failure modes.

Suppose

Primary accepts

```
SET P123
```

Before replication completes,

the primary crashes.

A replica is promoted.

The replica never received

```
P123
```

The retry executes again.

Duplicate payment.

This is a replication consistency problem,

not a Redis problem.

---

# Active-Active vs Active-Passive

| Property | Active-Active | Active-Passive |
|-----------|--------------|----------------|
| Write Latency | Lower (local) | Higher after failover |
| Availability | Higher | Lower |
| Coordination Complexity | High | Low |
| Duplicate Risk | Higher | Lower |
| Operational Complexity | High | Moderate |

Notice the trade-off.

There is no universally correct architecture.

The choice depends on business requirements.

---

# CAP Perspective

Suppose two regions become partitioned.

Both continue accepting payment requests.

Question

Can both safely create the same payment?

Only if they continue coordinating through a strongly consistent system.

Otherwise,

the application must choose between

- rejecting requests (Consistency)
- accepting requests with duplicate risk (Availability)

Idempotency therefore cannot be discussed independently of CAP.

---

# Principal Engineer Trade-off

A common interview discussion is

> "Would you prefer global uniqueness or regional availability?"

Financial systems usually prioritize

```
Consistency

>

Availability
```

Social media platforms often choose the opposite.

Business requirements determine the architecture.

---

# Production Example — Stripe

Stripe exposes an Idempotency-Key to clients.

Internally,

the system must ensure that retries arriving through different front-end servers still converge to the same business operation.

Whether this is achieved through routing, globally consistent storage, or a combination of both is an implementation detail.

The architectural invariant remains unchanged:

> One logical payment must map to one business outcome.

---

# Production Example — Uber

Suppose a rider retries payment while traveling internationally.

The retry may enter through a different regional edge.

The backend cannot assume transport affinity.

Instead,

ownership of the payment must be established independently of where the request entered the system.

---

# Architecture Review

Suppose another team proposes

```
Each region stores its own
idempotency table.
```

Questions a Principal Engineer should ask:

- How are duplicate requests across regions prevented?
- What is the source of global ownership?
- How are failovers handled?
- What consistency guarantees does the datastore provide?
- Is the system AP or CP during partitions?
- What is the expected maximum retry window?
- How are stale idempotency records cleaned up?

These questions focus on invariants rather than implementation details.

---

# Principal Engineer Insight

> Local idempotency solves concurrent execution within one failure domain.
>
> Global idempotency solves ownership across multiple failure domains.
>
> The difficult problem is not detecting duplicate requests.
>
> The difficult problem is ensuring that every region independently reaches the same conclusion about who owns a business operation.
>
> Every production design ultimately reduces to one of three approaches:
>
> - deterministic routing,
> - globally consistent ownership,
> - or accepting weaker guarantees.

---

# Interview Discussion

**Interviewer**

Two identical payment requests arrive simultaneously in different AWS regions.

How would you guarantee only one charge?

---

**Weak Answer**

Store the Idempotency-Key in Redis.

---

**Principal Engineer Answer**

A regional Redis instance cannot establish global ownership because each region may independently acquire the same key. The design must first establish a globally consistent ownership decision. Common approaches include deterministic routing to a home region, storing idempotency records in a globally consistent database with uniqueness constraints, or restricting writes to a single active region. The implementation depends on the latency and availability objectives, but the invariant is that only one region may acquire ownership of a logical payment.

---

# Key Takeaways

- Global idempotency is fundamentally a distributed ownership problem.
- Regional uniqueness does not imply global uniqueness.
- Deterministic routing eliminates coordination by assigning ownership.
- Globally consistent databases trade latency for stronger correctness.
- Distributed locks require their own correctness guarantees and do not eliminate the underlying coordination problem.
- Architecture decisions must be driven by business invariants rather than implementation convenience.

---

# Choosing the Right Correctness Mechanism

One of the most common mistakes in distributed system design is applying the same solution to fundamentally different problems.

Interview candidates often answer every concurrency question with one of the following:

- Use Redis Lock
- Use Optimistic Locking
- Use Idempotency
- Use Kafka Exactly Once
- Use Distributed Transaction

These are not interchangeable.

Each mechanism preserves a different correctness property.

Choosing the wrong one often creates unnecessary complexity while still failing to preserve the required business invariant.

---

# Step 1 — Identify the Invariant

Before selecting a solution, identify the property that must always remain true.

Examples

| Business Requirement | Required Invariant |
|----------------------|-------------------|
| Customer must never be charged twice | At most one successful charge |
| Inventory must never become negative | Stock ≥ 0 |
| Every coupon redeemed once | Single redemption |
| Every order receives unique Order ID | Global uniqueness |
| Auction highest bid wins | Serialized ordering |
| Only one scheduler executes a cron job | Single active executor |

Notice

Every invariant is different.

Therefore the solution may also be different.

---

# Idempotency

Idempotency protects

> **Duplicate execution of the same logical operation.**

Example

```
POST /payments

Idempotency-Key = P123
```

If the client retries ten times,

all requests represent

the same business operation.

Idempotency ensures

only one business effect is produced.

It does **not** coordinate concurrent updates to unrelated operations.

---

# Optimistic Locking

Optimistic locking protects

> **Lost updates caused by concurrent modification.**

Typical implementation

```text
Account

Version = 12
```

Client A

reads Version 12.

Client B

also reads Version 12.

Client A updates successfully.

Version becomes

```
13
```

Client B attempts update.

Version mismatch.

Update rejected.

Notice

both operations are legitimate.

They are not duplicates.

Optimistic locking detects stale state.

It does not identify duplicate business requests.

---

# Pessimistic Locking

Pessimistic locking assumes

contention is expected.

Resources are locked before modification.

Example

```sql
SELECT *

FROM account

WHERE id=?

FOR UPDATE;
```

Advantages

- Prevents concurrent modification
- Simple correctness model

Trade-offs

- Reduced concurrency
- Deadlock risk
- Long lock duration

Usually appropriate only for short-lived database transactions.

---

# Distributed Locks

Distributed locks protect

> **Exclusive ownership of a shared resource across multiple processes.**

Typical examples

- Leader Election
- Scheduled Jobs
- Cache Rebuild
- Distributed File Processing

Example

```
Nightly Billing Job
```

100 application instances start simultaneously.

Only one should execute the job.

Distributed locking is appropriate.

Now consider

```
Customer Payment
```

Should every payment require a distributed lock?

Usually not.

Each payment is an independent business operation.

Idempotency provides a better fit.

---

# Why Payment Systems Rarely Use Distributed Locks

Suppose every payment acquires

```
Redis Lock
```

Problems immediately appear.

- Lock expiry
- Failover
- Network partition
- Lease renewal
- Clock drift
- Lock recovery

The payment problem

is fundamentally

duplicate detection,

not mutual exclusion.

Idempotency solves the actual problem.

Distributed locking introduces unnecessary coordination.

---

# Unique Constraints

One of the most powerful synchronization mechanisms is often overlooked.

Database uniqueness.

Example

```sql
CREATE UNIQUE INDEX

ON payments(idempotency_key);
```

Now

every concurrent request

competes

inside the database.

Exactly one succeeds.

Notice

the database already provides

- atomicity
- durability
- crash recovery
- concurrency control

No distributed lock required.

---

# Comparing the Mechanisms

| Mechanism | Protects Against | Typical Use Case |
|-----------|------------------|------------------|
| Idempotency | Duplicate logical operations | Payments, APIs, Webhooks |
| Optimistic Locking | Lost updates | Account/Profile updates |
| Pessimistic Locking | Concurrent writers | Banking, inventory within one DB |
| Distributed Lock | Multiple active workers | Cron jobs, leader election |
| UNIQUE Constraint | Duplicate inserts | Order IDs, Payment IDs |
| 2PC | Atomic commit | Cross-shard transactions |
| Raft | Replicated ordering | Metadata, consensus |

Notice

Several mechanisms may appear in the same architecture,

but each protects a different invariant.

---

# Example 1 — Payment API

Requirement

```
Customer retries payment.
```

Question

Distributed Lock?

No.

Correct mechanism

```
Idempotency Key

+

Unique Constraint
```

Reason

The challenge is duplicate execution,

not concurrent modification of shared state.

---

# Example 2 — Inventory Reservation

Requirement

```
Two customers purchase

the last item.
```

Question

Idempotency?

Not sufficient.

The requests are different business operations.

Correct mechanisms include

- Optimistic Locking
- Pessimistic Locking
- Atomic conditional update

depending on contention characteristics.

---

# Example 3 — Nightly Settlement Job

Requirement

```
Exactly one instance executes settlement.
```

Idempotency cannot solve this.

Each node performs a different execution attempt.

Correct mechanism

```
Leader Election

or

Distributed Lock
```

---

# Example 4 — Coupon Redemption

Requirement

```
Coupon ABC

redeemed once.
```

Best solution

```sql
UNIQUE(user_id, coupon_id)
```

The database naturally serializes concurrent redemption attempts.

Adding Redis locks often complicates the design without improving correctness.

---

# Decision Framework

A useful interview framework is:

```
Duplicate business request?

↓

Idempotency
```

```
Concurrent modification?

↓

Optimistic / Pessimistic Locking
```

```
Need one active worker?

↓

Distributed Lock
```

```
Need global uniqueness?

↓

Unique Constraint
```

```
Need atomic updates across multiple databases?

↓

2PC / Saga
```

```
Need replicated ordering?

↓

Consensus
```

Choosing the mechanism follows directly from the invariant.

---

# Principal Engineer Insight

> One of the strongest indicators of architectural maturity is resisting the urge to introduce coordination unnecessarily.
>
> Every distributed lock, quorum protocol, and transaction coordinator increases latency, operational complexity, and failure modes.
>
> Before introducing coordination, ask:
>
> **Can the invariant be preserved with a simpler mechanism?**
>
> Many production systems rely on database uniqueness, conditional writes, or idempotency rather than distributed locking because these mechanisms provide sufficient correctness with significantly lower operational cost.

---

# Interview Discussion

**Interviewer**

A customer retries the same payment three times because of network timeouts.

Would you use a distributed lock?

---

**Weak Answer**

Yes. Acquire a Redis lock before processing.

---

**Principal Engineer Answer**

No. A distributed lock solves mutual exclusion, not duplicate business requests. The correct invariant is that the payment should be executed at most once. An idempotency key combined with an atomic ownership mechanism such as a unique database constraint or conditional write preserves that invariant without introducing lease management, failover semantics, or distributed lock recovery.

---

# Common Interview Mistakes

❌ Using distributed locks for duplicate request handling.

Locks protect ownership.

Idempotency protects business effects.

---

❌ Using optimistic locking to detect retries.

Version conflicts indicate concurrent modification, not duplicate requests.

---

❌ Assuming UNIQUE constraints are "too simple."

In many systems, they provide the strongest and simplest correctness guarantee.

---

❌ Solving every concurrency problem with Redis.

Correctness mechanisms should follow invariants, not technology preferences.

---

# Key Takeaways

- Correctness mechanisms are not interchangeable.
- Always identify the business invariant before selecting a coordination strategy.
- Idempotency protects duplicate logical operations.
- Optimistic and pessimistic locking protect concurrent modification.
- Distributed locks coordinate ownership across processes.
- Database uniqueness often provides the simplest and strongest synchronization primitive for duplicate creation problems.

---

# Production Failure Scenarios & Interview Workbook

Principal Engineers are evaluated on their ability to reason about failures.

The interviewer is rarely interested in whether you know what an Idempotency-Key is.

Instead, they want to know whether your design continues to preserve business invariants when every component behaves unexpectedly.

This section analyzes common production failures and the reasoning process expected during Principal-level interviews.

---

# Failure Scenario 1 — Response Lost After Commit

Architecture

```text
Client

↓

API

↓

Database
```

Timeline

```text
POST /payments

↓

Payment committed

↓

Response lost

↓

Client retries
```

### Question

Should the payment execute again?

### Required Invariant

```
One logical payment

↓

One financial charge
```

### Incorrect Solution

```
Process request again.
```

Duplicate payment.

Business invariant violated.

### Correct Solution

The retry must be recognized as the same logical business operation.

The server returns the previously computed response instead of executing payment logic again.

Notice that the retry succeeds without producing another business effect.

---

# Failure Scenario 2 — Concurrent Duplicate Requests

Two requests carrying the same Idempotency-Key arrive simultaneously.

```text
Server A

↓

Payment(P123)
```

```text
Server B

↓

Payment(P123)
```

Both requests are legitimate.

The challenge is ownership.

### Incorrect Design

```sql
SELECT ...

↓

INSERT
```

Both servers observe

```
Not Found
```

Both continue.

### Correct Design

Atomic ownership acquisition.

Examples

```sql
INSERT ...

ON CONFLICT DO NOTHING
```

or

```
SET key value NX
```

Only one execution becomes the owner.

---

# Failure Scenario 3 — Database Commit Succeeds, Kafka Publish Fails

Timeline

```text
Insert Order

↓

Commit

↓

Kafka Publish

↓

Failure
```

Question

What invariant is violated?

Answer

Business state and event stream diverge.

Inventory, Payment and Shipping never observe the new order.

### Correct Design

Transactional Outbox.

The event becomes part of the same local transaction.

Publication becomes asynchronous.

---

# Failure Scenario 4 — Consumer Commits Database But Crashes Before Offset Commit

Timeline

```text
Consume Event

↓

Update Inventory

↓

Commit Database

↓

Crash

↓

Kafka Replay
```

Question

Should inventory decrease again?

No.

### Correct Design

Inbox Pattern

+

Idempotent Consumer

Duplicate deliveries become harmless.

---

# Failure Scenario 5 — Redis Primary Fails

Suppose Redis is used for Idempotency.

Timeline

```text
SET payment:P123

↓

Primary crashes

↓

Replica promoted
```

Replication was asynchronous.

The promoted replica never received the write.

The retry now appears to be a new request.

Question

Is Redis incorrect?

No.

The system selected an architecture whose consistency guarantees were weaker than the business invariant.

The problem is architectural, not technological.

---

# Failure Scenario 6 — Multi-Region Retry

A mobile client retries after a timeout.

The retry reaches another AWS region.

Each region owns an independent idempotency database.

Both process the payment successfully.

Question

Where is the bug?

Not in HTTP.

Not in retries.

The system failed to establish global ownership of the business operation.

---

# Failure Scenario 7 — Compensation Failure

Saga executes

```text
Payment

↓

Inventory

↓

Shipping
```

Inventory fails.

Saga begins compensation.

Refund request times out.

Question

Has the Saga completed?

No.

Compensation itself is a business workflow.

It must be retried until completion.

Compensation requires exactly the same durability and idempotency guarantees as the forward path.

---

# Failure Scenario 8 — Duplicate Webhooks

Payment provider sends

```
PaymentCompleted
```

Webhook.

Network timeout occurs.

Provider retries.

Merchant receives

the same webhook

five times.

Question

Should five orders be created?

Never.

Webhook consumers must be idempotent.

This is why providers such as Stripe explicitly document that webhook delivery is **at least once**.

---

# Failure Scenario 9 — Unique Constraint Violation

Two checkout requests create

```
Order-123
```

simultaneously.

Database returns

```text
Unique Constraint Violation
```

Question

Is this an error?

Not necessarily.

In many systems,

this is evidence that another execution already completed successfully.

The application should determine whether this represents:

- duplicate request
- conflicting request
- genuine failure

Correct handling depends on business semantics.

---

# Failure Scenario 10 — Orchestrator Crash

Saga orchestrator crashes after

```
Payment Completed

Inventory Reserved
```

Workflow state has already been persisted.

Question

How should recovery occur?

Correct answer

Workflow execution resumes from durable state.

The system must never restart from the beginning.

This is one reason platforms such as Temporal persist workflow history.

---

# Architecture Review Exercise 1

Design

```
POST /payments
```

Requirements

- Multi-region deployment
- No duplicate charges
- Retry support
- Kafka integration
- Payment gateway timeout
- Event-driven downstream systems

Discussion points

- Business invariant
- Idempotency ownership
- Outbox
- Inbox
- Retry policy
- Global routing
- Failure recovery

---

# Architecture Review Exercise 2

Design coupon redemption.

Requirements

- One coupon
- One redemption
- Millions of concurrent users

Questions

- Is idempotency sufficient?
- Do you need optimistic locking?
- Would a UNIQUE constraint solve the problem?
- Is Redis required?
- What happens during failover?

---

# Architecture Review Exercise 3

Design inventory reservation for flash sales.

Requirements

- 10 million requests
- No overselling
- High throughput

Questions

- Should requests be serialized?
- Would optimistic locking scale?
- Is sharding required?
- Is a distributed lock appropriate?
- Which invariant is most important?

---

# Principal Engineer Discussion

One of the biggest differences between Senior and Principal engineers is the direction of reasoning.

Senior engineers often begin with technology.

```
Redis

Kafka

PostgreSQL

ZooKeeper
```

Principal engineers begin with invariants.

Example

```
Customer charged at most once.
```

↓

Determine ownership.

↓

Choose synchronization primitive.

↓

Select storage.

↓

Optimize latency.

Technology is selected only after correctness has been defined.

---

# Common Anti-Patterns

### Anti-Pattern 1

Adding Redis locks before identifying the business invariant.

---

### Anti-Pattern 2

Believing Kafka Exactly-Once Semantics eliminate duplicate business processing.

---

### Anti-Pattern 3

Using request identifiers instead of business operation identifiers.

---

### Anti-Pattern 4

Performing check-then-act instead of atomic ownership acquisition.

---

### Anti-Pattern 5

Ignoring multi-region ownership.

Local correctness does not imply global correctness.

---

### Anti-Pattern 6

Treating compensation as rollback.

Compensation creates new business state.

---

# Principal Engineer Decision Matrix

| Requirement | Primary Mechanism | Why |
|------------|-------------------|-----|
| Duplicate API retries | Idempotency | Preserve business effect |
| Duplicate Kafka delivery | Inbox + Idempotent Consumer | Durable event ownership |
| Cross-service event publication | Transactional Outbox | Eliminate dual writes |
| Concurrent update of same row | Optimistic Locking | Detect lost updates |
| Single worker execution | Distributed Lock / Leader Election | Exclusive ownership |
| Global uniqueness | UNIQUE Constraint / Conditional Write | Atomic ownership |
| Cross-shard atomic commit | 2PC | Atomicity |
| Replicated metadata | Raft | Consensus |

---

# Whiteboard Exercise

Design a globally distributed payment platform.

Requirements

- Multi-region active-active
- No duplicate charges
- Retry support
- Event-driven architecture
- Kafka integration
- Regulatory audit trail
- Disaster recovery

Before drawing boxes,

identify the invariants.

Then explain

- request identity,
- ownership,
- synchronization,
- event publication,
- recovery,
- observability,
- and operational trade-offs.

---

# Principal Engineer Takeaway

Strong candidates explain patterns.

Exceptional candidates explain **why those patterns preserve correctness**.

Whenever discussing idempotency, begin with three questions:

1. What business invariant must never be violated?
2. Who owns the operation?
3. Which synchronization mechanism preserves that ownership with the least coordination?

Everything else—Redis, SQL, Kafka, or consensus—is an implementation choice.

---

# Chapter Summary

Idempotency is not an optimization for retries.

It is a correctness property that allows distributed systems to remain deterministic despite message loss, retries, crashes, replays, and concurrent execution.

A production-grade design combines:

- Stable business identifiers
- Atomic ownership acquisition
- Durable state transitions
- Idempotent consumers
- Transactional Outbox
- Inbox Pattern
- Retry-safe compensation
- Globally consistent ownership where required

The technology stack may change.

The invariants must not.

