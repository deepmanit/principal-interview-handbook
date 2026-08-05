
# Distributed Transactions

> *"Distributed transactions ensure that a business operation spanning multiple services either completes successfully or fails safely while preserving data consistency."*

---

# Table of Contents

1. Why Distributed Transactions Exist
2. Local vs Distributed Transactions
3. The Atomicity Problem
4. Why ACID Stops at the Database Boundary
5. Distributed Transaction Patterns
6. Production Examples

---

# Learning Objectives

After completing this chapter you should be able to

- Explain distributed transactions.
- Explain why ACID does not work across microservices.
- Identify transaction boundaries.
- Understand why 2PC is rarely used in internet-scale systems.
- Explain when Saga is required.
- Answer Principal Engineer interview questions.

---

# The Problem

Suppose

Customer places an order.

Business flow

```text
Order Service

↓

Payment Service

↓

Inventory Service

↓

Shipping Service

↓

Notification Service
```

Question

Should all services succeed?

Yes.

Otherwise

business data

becomes inconsistent.

---

# Local Transaction

Inside

a single database

ACID guarantees

```
BEGIN

↓

UPDATE

↓

INSERT

↓

DELETE

↓

COMMIT
```

Either

everything succeeds

or

everything rolls back.

Simple.

---

# Example

Transfer

₹100

between accounts.

```
Debit

↓

Credit

↓

Commit
```

If

Credit fails,

Debit

also rolls back.

Database guarantees consistency.

---

# Why Doesn't This Work?

Suppose we have the following microservices, each using its own database:

* **Order Service** → PostgreSQL
* **Payment Service** → MySQL
* **Inventory Service** → MongoDB
* **Shipping Service** → Redis

### Question

If a distributed transaction fails, **who is responsible for performing the rollback?**

### Answer

**Nobody performs a global rollback.**

Each microservice **owns and manages its own database**. Therefore, one service cannot directly roll back changes made by another service.

This is why traditional database transactions such as `ACID` transactions cannot span all these services. Instead, distributed systems typically use patterns such as **Saga**, where each service performs a **compensating transaction** to undo or compensate for its own completed operation.


---

# ACID Boundary

# ACID Transaction Boundary

**Single Database**
↓
**Single Transaction**

**Multiple Databases**
↓
**No Shared Transaction Manager**
↓
**No Single ACID Transaction Across Services**

This is one of the **fundamental challenges of distributed transactions in microservices**.

Each microservice owns its own database and transaction boundary. Therefore, a single business operation that spans multiple services cannot usually be handled by one traditional ACID transaction.

---

# Example Failure

Customer places order.

Timeline

```text
Create Order

↓

Payment Success

↓

Inventory Failure

↓

Shipping Never Starts
```

# Business vs. Technical Reality

### Question

**Should the customer be charged?**

### Business Answer

**No.**

The order cannot be completed, so the customer should not be charged.

### Technical Problem

**The payment has already succeeded.**

The Payment Service has already committed the transaction, but a subsequent step failed.

This creates a **distributed transaction consistency problem**:

> The business expects the entire operation to fail, but one microservice has already successfully committed its part.

---

# Why Distributed Transactions Exist

Distributed transactions exist to maintain **business consistency across multiple independent services and databases**.

The goal is **not simply to maintain database consistency**.

The real goal is to ensure **business correctness** across the entire workflow.

### In Simple Terms

**Database consistency**
→ Is each individual database in a valid state?

**Business consistency**
→ Is the overall business operation in the correct state across all services?

> **The goal of distributed transactions is not database consistency.
> The goal is business correctness.**


---

# Business Transaction

Think

from

the customer's perspective.

Customer expects

```
Order Created

AND

Payment Successful

AND

Inventory Reserved

AND

Shipment Created
```

Not

```
Payment Charged

Inventory Failed

Order Cancelled
```

---

# Atomicity Problem

# Can Multiple Services Commit at Exactly the Same Time?

### Question

Can **five different microservices** commit their transactions **at exactly the same moment**?

### Answer

**Very difficult.**

In a distributed system, each service may have its own database and transaction manager. Coordinating a single atomic commit across all of them is extremely challenging.

Several factors can interfere with coordination:

* **Network delays** — messages may arrive late.
* **Service crashes** — a service may fail during the transaction.
* **Timeouts** — responses may not arrive within the expected time.
* **Retries** — the same operation may be executed more than once.
* **Network partitions** — services may temporarily lose communication with each other.

Therefore, achieving a **single atomic commit across multiple independent services** is fundamentally difficult.

> **Distributed systems cannot assume that multiple services will commit at exactly the same time.**


---

# Why Not Use One Shared Database?

Interviewers often ask:

> **"Why not put everything into one database?"**

### Answer

Microservices intentionally **avoid a shared database** because each service should own its data and evolve independently.

A shared database creates tight coupling, whereas separate databases provide several important benefits:

* **Independent scaling** – Each service can scale its database based on its own workload.
* **Independent deployments** – Services can be updated or released without affecting others.
* **Team autonomy** – Different teams can develop, test, and maintain their services independently.
* **Technology flexibility** – Each service can choose the database that best fits its requirements (PostgreSQL, MySQL, MongoDB, Redis, etc.).
* **Failure isolation** – Problems in one database are less likely to impact other services.

### The Trade-off

While separate databases improve scalability and maintainability, they also make **distributed transactions** much more challenging because there is **no single ACID transaction spanning all services**.

> **Microservices trade the simplicity of a shared database for scalability, autonomy, flexibility, and resilience.**

---

# Typical Distributed Transaction

```text
Order Service

↓

Payment Service

↓

Inventory Service

↓

Shipping Service

↓

Notification Service
```

Every service

owns

its own data.

No service

can directly

rollback

another service's database.

---

# What Makes Distributed Transactions Hard?

Suppose an order involves multiple services:

1. **Payment Service** processes the payment successfully.
2. Immediately afterward, **Inventory Service crashes**.
3. The overall business operation can no longer be completed.

### Question

**Who tells the Payment Service to undo the charge?**

The Inventory Service cannot do it because it has crashed, and the Payment Service has already committed its transaction.

### The Real Problem

This requires **coordination between independent services**.

The system must determine:

* What happened?
* Which operations already succeeded?
* Which operations failed?
* Should the successful operations be undone?
* How do we safely retry or compensate for them?

> **This coordination problem is at the heart of distributed transactions in microservices.**


---

# Two Families of Solutions

There are only two broad approaches.

| Approach | Philosophy |
|-----------|------------|
| Two-Phase Commit (2PC) | Coordinate all participants before committing |
| Saga | Commit locally and compensate if later steps fail |

Modern cloud-native systems

mostly prefer

Saga.

We will study

both approaches

in depth.

---

# Principal Engineer Insight

> [!IMPORTANT]
>
> Distributed transactions are **not** about databases.
>
> They are about preserving **business invariants** across independently deployed services.
>
> The most important question is not:
>
> "How do I commit?"
>
> It is:
>
> "How do I recover safely when one step fails?"

---

# Interview Conversation

**Interviewer**

Why can't a normal SQL transaction span multiple microservices?

---

**Weak Answer**

Because they use different databases.

---

**Principal Engineer Answer**

A traditional ACID transaction relies on a single transaction manager controlling one database engine. In a microservice architecture, each service owns its own datastore and commits independently. There is no shared transaction manager capable of atomically committing or rolling back changes across all participants. Coordinating those services therefore requires distributed transaction protocols such as Two-Phase Commit or Saga.

---

# Common Interview Mistakes

> [!WARNING]
> Assuming ACID automatically extends across multiple databases.

---

> [!WARNING]
> Thinking distributed transactions are only a database concern.

---

> [!WARNING]
> Ignoring business consistency.

---

# Key Takeaways

- ACID works within a single database.
- Microservices introduce multiple independent transaction boundaries.
- Distributed transactions preserve business correctness across services.
- Traditional database transactions cannot span independently owned datastores.
- Two major approaches are Two-Phase Commit and Saga.

---

# Two-Phase Commit (2PC)

Two-Phase Commit (2PC) is the classical protocol for achieving atomic commitment across multiple independent resource managers.

The objective is straightforward:

> Either **every participant commits** or **every participant aborts**.

Unlike Raft, which establishes a globally ordered log, 2PC coordinates the commit decision of a single distributed transaction.

It answers a different question:

> **Can multiple databases make the same commit/abort decision despite failures?**

---

# Why 2PC Exists

Consider an e-commerce checkout.

```
Order Service      PostgreSQL
Payment Service    MySQL
Inventory Service  MongoDB
Loyalty Service    Cassandra
```

Business invariant:

```
Order Created
AND
Payment Captured
AND
Inventory Reserved
AND
Reward Points Added
```

If any participant fails after another has committed, the business invariant is violated.

The problem is **atomic commitment**, not replication.

---

# Atomic Commitment Problem

Each participant has three possible states.

```
Working

↓

Prepared

↓

Committed
```

The difficulty is ensuring that every participant reaches the same terminal state despite:

- Process crashes
- Network partitions
- Coordinator failures
- Timeouts
- Duplicate messages
- Retries

---

# Components

A 2PC transaction contains two roles.

| Component | Responsibility |
|-----------|----------------|
| Coordinator | Drives the protocol and decides Commit/Abort |
| Participants | Execute local transaction and obey the final decision |

Unlike Raft, participants never elect a leader.

The coordinator exists only for the lifetime of the transaction.

---

# Phase 1 — Prepare

Coordinator sends

```
PREPARE
```

to every participant.

Example

```text
Coordinator

↓

Order DB

↓

Payment DB

↓

Inventory DB
```

Each participant executes the transaction **locally**, but does **not** commit.

Instead it reaches a **prepared state**.

Prepared means:

- All locks acquired
- Undo log written
- Redo log written (implementation dependent)
- Changes durable
- Able to commit later
- Unable to rollback unilaterally

This distinction is extremely important.

Prepared is **not** committed.

---

# Durable Prepare Record

One of the most common Principal interview questions is:

> Why must PREPARE be durable?

Suppose a participant replies

```
YES
```

and immediately crashes.

If the prepare record was never persisted, after restart it has forgotten:

- transaction id
- acquired locks
- modified rows
- voting decision

The coordinator may later instruct

```
COMMIT
```

but the participant no longer knows what to commit.

Therefore every participant **must persist the prepared state before replying YES**.

This is a fundamental invariant of 2PC.

---

# Voting

Each participant returns one of two responses.

```
YES
```

or

```
NO
```

Decision rule:

- Every participant votes YES → Commit phase
- At least one NO → Abort

Unlike Raft, there is **no majority**.

Consensus is unanimous.

One NO aborts the entire transaction.

---

# Phase 2 — Commit

If every participant voted YES:

```
Coordinator

↓

Persist COMMIT Decision

↓

Send COMMIT
```

Every participant:

- commits local transaction
- releases locks
- acknowledges completion

Only after receiving acknowledgements can the coordinator forget the transaction.

---

# Why Must Coordinator Persist the Decision?

Suppose

```
YES
YES
YES
```

Coordinator decides

```
COMMIT
```

but crashes before notifying participants.

After restart,

how does it know whether the decision was:

```
COMMIT
```

or

```
ABORT
```

Without a durable coordinator log,

different participants may make different decisions.

Therefore the coordinator must write the global decision to stable storage before sending any commit messages.

---

# Failure Scenario 1 — Participant Crash Before Voting

Timeline

```
PREPARE

↓

Participant crashes

↓

Timeout

↓

Coordinator aborts
```

Simple recovery.

---

# Failure Scenario 2 — Participant Crashes After YES

Timeline

```
PREPARE

↓

YES

↓

Crash

↓

Coordinator commits
```

Participant eventually restarts.

Reads durable prepare record.

Contacts coordinator.

Learns final outcome.

Completes commit.

---

# Failure Scenario 3 — Coordinator Crash Before Decision

```
YES
YES
YES

↓

Coordinator crashes
```

Participants remain prepared.

They cannot:

- commit
- rollback

because they do not know the global decision.

This is the famous **blocking problem**.

---

# Why 2PC is Blocking

Prepared participants hold locks.

Applications wait.

Other transactions block.

System throughput drops.

Until the coordinator recovers,

participants cannot safely proceed.

This blocking behavior is not an implementation bug.

It is a property of the protocol.

---

# Interview Question

**Why can't participants simply timeout and rollback?**

Because another participant may already have committed.

Unilateral rollback could violate atomicity.

Once a participant votes YES, it has transferred decision authority to the coordinator.

---

# Lock Contention

Prepared transactions continue holding:

- row locks
- page locks
- index locks

Long coordinator failures therefore reduce concurrency dramatically.

This is one of the biggest operational disadvantages of 2PC.

---

# FLP Connection

A common Principal follow-up is:

> Doesn't timeout solve coordinator failures?

No.

A timeout cannot distinguish between:

- crashed coordinator
- slow coordinator
- partitioned coordinator

This is a direct consequence of the FLP impossibility result.

In an asynchronous network,

waiting longer never proves a node has failed.

---

# XA Transactions

Most relational databases implement 2PC through the XA specification.

Typical participants include:

- Oracle
- PostgreSQL
- MySQL
- DB2
- SQL Server

The XA interface standardizes:

- prepare
- commit
- rollback
- recovery

Understanding XA is useful, but modern internet-scale systems rarely expose it directly to application developers.

---

# Why Cloud-Native Systems Avoid 2PC

Although 2PC guarantees atomicity, it introduces:

- blocking
- long-lived locks
- coordinator bottleneck
- reduced availability
- poor WAN latency

For this reason,

companies such as Uber, Netflix, Amazon, DoorDash and Stripe generally prefer **eventual consistency with Saga and compensating actions** over distributed locking.

---

# Common Misconceptions

### "2PC solves distributed consistency."

No.

It solves **atomic commit**.

Replication, ordering, and consensus are separate problems.

---

### "2PC tolerates coordinator failure."

Only after recovery.

Until then, prepared participants remain blocked.

---

### "Majority voting is enough."

No.

2PC requires unanimous agreement.

A single NO aborts the transaction.

---

# Principal Engineer Insight

> A common interview mistake is comparing Raft and 2PC as competing algorithms.
>
> They solve fundamentally different problems.
>
> Raft establishes a globally ordered replicated log under crash failures.
>
> 2PC coordinates the atomic outcome of one distributed transaction.
>
> Many production systems use both simultaneously—for example, a distributed SQL database may use Raft for replication and 2PC for transactions spanning multiple Raft groups.

---

# Interview Questions

### Q1

Why is the PREPARE record required to be durable?

---

### Q2

Why is 2PC considered a blocking protocol?

---

### Q3

Why can't a participant rollback after voting YES?

---

### Q4

Why does 2PC require unanimous agreement while Raft requires only a majority?

---

### Q5

Can Raft replace 2PC?

If not, why not?

---

# Key Takeaways

- 2PC solves atomic commitment, not consensus.
- Participants must durably persist the prepared state before voting YES.
- The coordinator must durably persist the global decision before notifying participants.
- Prepared participants cannot make unilateral decisions after voting YES.
- The protocol is fundamentally blocking under coordinator failures.
- Long-lived locks make 2PC a poor fit for many cloud-native systems.
- Modern distributed databases often combine Raft (replication) with 2PC (cross-shard atomicity).

---

# Advanced Topics in Atomic Commit

At this point, we understand the classical Two-Phase Commit protocol.

However, production systems rarely deploy textbook 2PC without modifications.

Real systems optimize around three major concerns:

- Recovery overhead
- Coordinator availability
- Wide-area network latency

Understanding these optimizations is essential for Staff and Principal-level interviews.

---

# Presumed Abort

Classical 2PC requires the coordinator to durably log every transaction outcome.

For large systems processing millions of transactions per second, this creates significant logging overhead.

Presumed Abort optimizes for the observation that **most failed transactions eventually abort**.

Instead of explicitly recording every abort decision, the coordinator assumes:

> "If no commit record exists, the transaction must have aborted."

Recovery rule:

```
No Commit Record

↓

Assume Abort
```

Advantages:

- Smaller transaction logs
- Faster recovery
- Fewer disk writes
- Better throughput

This optimization is widely implemented in enterprise transaction managers.

---

# Presumed Commit

Some workloads are commit-heavy.

Examples include:

- Banking systems
- Order processing
- Payment processing

In these environments, recording every successful commit is wasteful.

Presumed Commit reverses the optimization.

Recovery rule:

```
No Abort Record

↓

Assume Commit
```

This reduces logging for commit-heavy workloads.

Trade-off:

Recovery logic becomes slightly more complex because participants must retain more metadata.

---

# Comparison

| Feature | Presumed Abort | Presumed Commit |
|----------|----------------|-----------------|
| Optimized For | Abort-heavy systems | Commit-heavy systems |
| Default Recovery | Abort | Commit |
| Log Volume | Lower for failed transactions | Lower for successful transactions |
| Operational Complexity | Lower | Slightly Higher |

---

# Can We Eliminate Blocking?

The natural question becomes

> Why not introduce another phase?

This led to

Three-Phase Commit (3PC).

---

# Three-Phase Commit (3PC)

3PC attempts to remove the blocking problem by inserting an intermediate phase.

```
CanCommit

↓

PreCommit

↓

Commit
```

The additional phase tries to ensure that participants never remain indefinitely uncertain.

Unlike 2PC, participants can sometimes make progress independently after timeouts.

---

# Why 3PC Never Became Popular

Although theoretically attractive,

3PC assumes

- bounded network delay
- bounded process pause
- reliable timeout estimation

Modern cloud environments satisfy none of these assumptions.

Examples:

- JVM stop-the-world GC
- Hypervisor scheduling delays
- Cross-region latency spikes
- Network partitions

Consequently,

3PC provides little practical benefit in asynchronous distributed systems.

Most production systems skipped directly from 2PC to consensus-based approaches.

---

# 2PC + Consensus

A common misconception is that modern distributed databases replaced 2PC with Raft.

This is incorrect.

Most distributed SQL databases use **both**.

Example architecture

```text
Transaction Coordinator

↓

2PC

↓

Raft Group A

Raft Group B

Raft Group C
```

The responsibilities are different.

Raft guarantees

```
Replication
Ordering
Durability
```

2PC guarantees

```
Atomic Commit
```

These protocols complement each other rather than compete.

---

# Example — CockroachDB

Suppose a transaction updates

```
Customer

↓

Shard A
```

and

```
Orders

↓

Shard B
```

Each shard is independently replicated using Raft.

The transaction still spans multiple Raft groups.

CockroachDB therefore executes

```
2PC

↓

Across Raft Groups
```

Each participant is itself a replicated state machine.

This distinction is frequently tested in Principal interviews.

---

# Example — Google Spanner

Google Spanner combines

- Paxos replication
- Two-Phase Commit
- TrueTime

Workflow

```text
Shard A

↓

Paxos Commit
```

```text
Shard B

↓

Paxos Commit
```

```
Transaction Coordinator

↓

2PC

↓

Atomic Commit
```

Finally,

TrueTime provides externally consistent commit timestamps.

The key insight is:

Consensus protects each replica group.

2PC coordinates multiple replica groups.

---

# Why Consensus Cannot Replace 2PC

Interview Question

Suppose every shard already uses Raft.

Why do we still need 2PC?

Example

```
Account Service

Raft Group A

Inventory

Raft Group B
```

Each group independently reaches consensus.

However,

nothing guarantees

both groups commit

the same business transaction.

Without atomic coordination,

possible outcome:

```
Payment Committed

Inventory Rolled Back
```

Consensus preserves consistency *within* a shard.

2PC preserves atomicity *across* shards.

---

# Coordinator High Availability

Textbook 2PC assumes a single coordinator.

Production systems rarely do.

Coordinator metadata is itself replicated.

Common approaches include:

- Raft
- Paxos
- ZooKeeper
- etcd

The coordinator can therefore fail without permanently losing transaction state.

Important observation:

Replicating the coordinator reduces coordinator failure risk.

It **does not eliminate the blocking property** of 2PC.

Blocking is a protocol characteristic, not an implementation detail.

---

# Coordinator Failover

Suppose

Coordinator crashes.

A replica becomes the new coordinator.

The new coordinator recovers

- transaction log
- prepare records
- global decision

and resumes transaction processing.

Recovery is transparent,

provided the decision log was replicated.

---

# Why Long-Lived Transactions Are Dangerous

Prepared transactions hold resources.

For example

```
Customer Row Lock

↓

Inventory Lock

↓

Payment Lock
```

A transaction waiting several minutes

can significantly reduce concurrency.

This is why distributed databases aggressively avoid

- user interaction
- external HTTP calls
- long-running business workflows

inside distributed transactions.

---

# Production Guidelines

Modern systems generally follow these principles:

- Keep distributed transactions short.
- Never perform remote API calls while holding transaction locks.
- Minimize the number of participating shards.
- Co-locate related data whenever possible.
- Prefer Saga when business workflows are naturally long-running.

---

# Principal Engineer Insight

> One of the strongest indicators of Principal-level understanding is recognizing that distributed systems are layered.
>
> A modern distributed SQL database often combines:
>
> - Consensus (Raft/Paxos) for replication
> - MVCC for concurrency control
> - Two-Phase Commit for atomicity
> - Timestamp ordering for serialization
>
> Each protocol solves a different problem. Replacing one does not eliminate the need for the others.

---

# Interview Questions

### Q1

Why doesn't Raft eliminate the need for Two-Phase Commit?

---

### Q2

Why does CockroachDB execute 2PC across Raft groups?

---

### Q3

Explain why Three-Phase Commit is rarely used in production.

---

### Q4

What problem do Presumed Abort and Presumed Commit solve?

---

### Q5

Does replicating the coordinator eliminate the blocking problem?

Why or why not?

---

### Q6

Why are long-running distributed transactions considered harmful?

---

# Key Takeaways

- Presumed Abort and Presumed Commit reduce transaction logging overhead.
- Three-Phase Commit attempts to reduce blocking but depends on timing assumptions that are unrealistic in modern distributed systems.
- Consensus protocols and atomic commit protocols solve different problems.
- Production databases frequently combine Raft/Paxos, MVCC, and 2PC.
- Replicating the coordinator improves availability but does not remove the fundamental blocking property of Two-Phase Commit.
- Long-lived distributed transactions reduce throughput by holding locks across multiple participants.

---

# Saga Pattern — Beyond Two-Phase Commit

Two-Phase Commit guarantees atomicity by coordinating all participants before commit.

Saga takes a fundamentally different approach.

Instead of delaying commits until everyone agrees, each service commits its own local transaction immediately. If a later step fails, previously completed steps are compensated through explicit business actions.

The key difference is philosophical.

2PC tries to prevent inconsistency.

Saga accepts temporary inconsistency and guarantees eventual business consistency.

This distinction is the foundation of almost every cloud-native architecture.

---

# Why Modern Companies Prefer Saga

Suppose an order workflow spans:

```text
Order Service

↓

Payment Service

↓

Inventory Service

↓

Shipping Service

↓

Notification Service
```

Each service:

- owns its own database
- deploys independently
- scales independently
- may be temporarily unavailable

Holding locks across all of them for several seconds—or even minutes—is operationally unacceptable.

Instead, every service commits immediately and publishes an event.

---

# Business Invariants

Principal Engineers should always identify business invariants before discussing implementation.

Example

```
Customer must never be charged
without either

Inventory Reservation

OR

Automatic Refund.
```

Notice that the invariant is **not**

```
Everything commits atomically.
```

It is

```
Business remains correct.
```

This mindset separates business correctness from database atomicity.

---

# Basic Saga Flow

```text
Create Order

↓

Authorize Payment

↓

Reserve Inventory

↓

Create Shipment

↓

Send Confirmation
```

Each step is an independent local transaction.

No distributed lock exists.

---

# Failure Example

Suppose

```
Create Order

✅
```

```
Authorize Payment

✅
```

```
Reserve Inventory

❌
```

Question

Should Order Service rollback?

Impossible.

It already committed.

Instead,

Saga executes

```
Refund Payment

↓

Cancel Order
```

---

# Compensation Is Not Rollback

This is one of the most common interview traps.

Database rollback

restores

previous physical state.

Compensation

creates

a new business transaction.

Example

Database rollback

```
UPDATE

↓

UNDO
```

Compensation

```
Charge ₹500

↓

Refund ₹500
```

Notice

the original transaction

still exists.

An additional transaction

restores

business correctness.

Accounting systems depend on this property.

---

# Compensation Must Be Idempotent

Suppose

```
Refund Payment
```

times out.

Orchestrator retries.

Question

Can customer receive

two refunds?

Absolutely not.

Every compensation

must be idempotent.

Typical implementation

```
Refund Request

↓

Idempotency Key

↓

Already Processed?

↓

Ignore Duplicate
```

Compensation failures are often more dangerous than the original failure.

---

# Compensation Order

Compensation executes

in reverse order.

Forward execution

```text
A

↓

B

↓

C

↓

D
```

Compensation

```text
Undo D

↓

Undo C

↓

Undo B

↓

Undo A
```

Exactly like stack unwinding.

---

# Interview Question

Can every operation be compensated?

No.

Examples

```
Email Sent

SMS Delivered

Push Notification

Physical Parcel Delivered
```

cannot truly be undone.

Instead,

the business defines

corrective actions.

Examples

```
Second Email

↓

Ignore Previous

```

or

```
Issue Refund

↓

Apology Email
```

Principal Engineers think in terms of business recovery, not database recovery.

---

# Saga Failure Matrix

| Failure | Recovery Strategy |
|----------|-------------------|
| Payment fails | Cancel Order |
| Inventory fails | Refund Payment |
| Shipping fails | Release Inventory + Refund |
| Notification fails | Retry asynchronously |
| Compensation fails | Retry until successful |
| Duplicate Event | Idempotency |

This table is frequently discussed during Staff interviews.

---

# Orchestration vs Choreography

There are two execution models.

---

# Choreography

Each service publishes events.

Example

```text
Order Created

↓

Payment Service

↓

Payment Completed

↓

Inventory Service

↓

Inventory Reserved

↓

Shipping Service
```

No central coordinator exists.

---

## Advantages

- Loosely coupled
- Highly scalable
- Easy to add new consumers
- Natural fit for Kafka

---

## Disadvantages

- Difficult debugging
- Long event chains
- Hidden dependencies
- Harder observability

One failed event may affect dozens of downstream services.

---

# Orchestration

A central orchestrator controls the workflow.

```text
Orchestrator

↓

Payment

↓

Inventory

↓

Shipping

↓

Notification
```

Each participant performs exactly one task.

The orchestrator decides what happens next.

---

## Advantages

- Explicit workflow
- Centralized retries
- Easier monitoring
- Easier compensation
- Better auditability

---

## Disadvantages

- Orchestrator becomes critical infrastructure
- Additional operational complexity
- Requires high availability

---

# Which Should You Choose?

Interviewers rarely want

"Orchestration is better."

The correct answer is

**It depends on workflow complexity.**

Use choreography when:

- Events are independent
- Consumers evolve independently
- Loose coupling is more important than visibility

Use orchestration when:

- Business workflow is critical
- Compensation logic is complex
- Auditing is required
- Regulatory compliance matters

Payment systems almost always prefer orchestration.

---

# Real Production Examples

| Company | Typical Approach |
|----------|------------------|
| Uber | Orchestration for trip/payment workflows |
| Stripe | Orchestrated payment workflows with idempotent APIs |
| Amazon | Mixed approach depending on domain |
| Netflix | Conductor (workflow orchestration) |
| Temporal | Durable workflow orchestration |
| Camunda | BPM/workflow orchestration |
| AWS Step Functions | Managed orchestration |

---

# Why Temporal Is Popular

Temporal persists workflow state.

Suppose

Workflow executes

```
Payment

↓

Inventory

↓

Shipping
```

Worker crashes

after

Payment.

Temporal resumes

exactly

from

Inventory.

Developers do not implement recovery logic manually.

This significantly reduces workflow complexity.

---

# Principal Engineer Insight

> The biggest mistake candidates make is treating Saga as "distributed rollback."
>
> Saga is not rollback.
>
> It is **forward recovery**.
>
> Every compensating action is a new business transaction with its own audit trail, retries, failure handling, and idempotency guarantees.
>
> Principal Engineers design compensation around business invariants rather than database state.

---

# Interview Conversation

**Interviewer**

Why do companies like Uber or Stripe prefer Saga over Two-Phase Commit?

---

**Weak Answer**

Because Saga is faster.

---

**Principal Engineer Answer**

Two-Phase Commit provides atomicity but requires long-lived coordination, blocking participants and reducing availability during failures. Modern cloud-native systems prioritize service autonomy, independent deployments, and fault isolation. Saga allows each service to commit locally while preserving business correctness through compensating transactions. Although temporary inconsistency is possible, the system remains available and eventually converges to a valid business state.

---

# Common Interview Mistakes

> [!WARNING]
> Saying Saga guarantees ACID across services.

It does not.

---

> [!WARNING]
> Treating compensation as database rollback.

Compensation creates new business actions.

---

> [!WARNING]
> Ignoring idempotency for compensation.

Retries are inevitable.

---

> [!WARNING]
> Claiming choreography is always more scalable.

Operational complexity often outweighs scalability benefits.

---

# Key Takeaways

- Saga prioritizes business consistency over atomic commit.
- Every step commits locally.
- Failures trigger compensating business transactions rather than database rollbacks.
- Compensation must be idempotent.
- Orchestration centralizes workflow management; choreography distributes it through events.
- The right choice depends on business requirements, observability, and operational complexity.

---

# Transactional Outbox, CDC and Idempotency

Distributed systems rarely fail because of consensus algorithms.

They fail because two independent systems are updated separately.

Consider a typical order service.

```
BEGIN TRANSACTION

↓

Insert Order

↓

Publish Kafka Event

↓

COMMIT
```

Looks simple.

Unfortunately,

it is fundamentally broken.

This problem is known as the **Dual Write Problem**.

---

# The Dual Write Problem

Suppose

```
INSERT Order

↓

Success
```

Immediately afterwards

```
Publish Kafka Event
```

fails because Kafka is temporarily unavailable.

Result

Database contains

```
Order Created
```

Kafka contains

```
Nothing
```

Inventory Service

Payment Service

Shipping Service

never learn that the order exists.

The system has permanently diverged.

---

# Reverse Failure

Now reverse the order.

```
Publish Kafka Event

↓

Success
```

Database commit fails.

Now consumers process

```
OrderCreated
```

for an order that never existed.

Again,

business consistency is broken.

---

# Why Distributed Transactions Don't Help

Interview Question

Why not use 2PC between PostgreSQL and Kafka?

Answer

Although technically possible with XA-capable systems,

it introduces

- blocking
- coordinator complexity
- reduced availability
- poor throughput

Modern cloud-native architectures deliberately avoid distributed XA transactions across messaging systems.

---

# Transactional Outbox

The key insight is simple.

Never update

two independent systems

inside one business operation.

Instead

update

only

the database.

---

# Outbox Schema

Instead of

```
Orders
```

only,

create another table.

```
Orders

Outbox
```

Example

| id | aggregate_id | event_type | payload | status |
|----|--------------|------------|----------|--------|
| 1001 | Order-123 | OrderCreated | JSON | NEW |

---

# Local Transaction

```sql
BEGIN;

INSERT INTO orders (...);

INSERT INTO outbox (...);

COMMIT;
```

Notice

both inserts

belong

to

the same

database transaction.

Either

both succeed

or

both rollback.

Atomicity

is restored.

---

# Event Publishing

A separate publisher

continuously scans

the Outbox table.

```
Outbox

↓

Publisher

↓

Kafka

↓

Consumers
```

Database transaction

is now

completely independent

from Kafka availability.

---

# Why This Works

Suppose Kafka crashes.

Order transaction

still commits.

Outbox row

still exists.

Publisher retries

later.

No business event

is lost.

---

# Polling Publisher

Simplest implementation

```
Every Second

↓

SELECT

status='NEW'

↓

Publish

↓

UPDATE status='SENT'
```

Advantages

- Easy
- Portable
- Works everywhere

Disadvantages

- Polling latency
- Additional database load

---

# Change Data Capture (CDC)

Polling does not scale well.

Modern systems prefer

CDC.

Instead of polling,

changes are streamed directly

from the database transaction log.

---

# Debezium

Debezium reads

the database WAL

(Write Ahead Log)

instead of tables.

Example

```
Application

↓

INSERT Outbox

↓

PostgreSQL WAL

↓

Debezium

↓

Kafka
```

No polling.

Almost real-time.

---

# Why WAL?

Every committed transaction

already exists

inside

the WAL.

Reading

the WAL

avoids

- repeated SELECTs
- table scans
- polling delays

This dramatically improves scalability.

---

# Event Flow

```
Business Transaction

↓

Commit

↓

WAL

↓

Debezium

↓

Kafka

↓

Consumers
```

The application

never publishes events directly.

---

# Exactly-Once Myth

One of the biggest interview traps.

Interviewer asks

> "How do you guarantee exactly-once delivery?"

Correct answer

You usually don't.

You guarantee

**exactly-once effects**.

Those are different.

---

# Why Exactly-Once Delivery Is Impossible

Suppose

Producer sends event.

Broker stores event.

Producer crashes

before receiving ACK.

Question

Did Kafka receive it?

Nobody knows.

Producer retries.

Duplicate event

appears.

Network uncertainty

makes perfect delivery impossible.

---

# Exactly-Once Effects

Instead of preventing duplicates,

systems tolerate them.

Consumer becomes

idempotent.

Processing

the same event

100 times

produces

the same final state.

---

# Idempotency Example

Payment Event

```
PaymentCompleted

TransactionId = T123
```

Consumer table

```
ProcessedEvents
```

Before processing

```
SELECT

WHERE

TransactionId=T123
```

Already processed?

↓

Ignore.

Otherwise

↓

Process

↓

Insert TransactionId

↓

Commit.

---

# Inbox Pattern

Outbox solves

producer reliability.

Consumer

needs

its own protection.

This leads to

Inbox Pattern.

```
Kafka

↓

Inbox Table

↓

Business Logic
```

Consumer first

stores

the event

inside

its Inbox table.

Only then

processes

business logic.

If processing crashes,

event

still exists

inside Inbox.

Recovery becomes deterministic.

---

# Outbox + Inbox

Producer

```
Business Tables

↓

Outbox
```

CDC

↓

Kafka

↓

Inbox

↓

Business Logic

This architecture

is widely used

inside financial systems.

---

# Consumer Failure

Suppose

Consumer

updates Inventory

then crashes

before committing offset.

Kafka redelivers

the same event.

Without idempotency

Inventory

decrements twice.

With Inbox

or

deduplication

duplicate processing

is ignored.

---

# Kafka Exactly-Once Semantics

Kafka supports

idempotent producers

and

transactional producers.

Important limitation

Kafka guarantees

exactly-once

**within Kafka**.

It cannot guarantee

exactly-once

inside

your database.

Applications

still require

idempotency.

---

# Principal Engineer Insight

> One of the most common interview mistakes is believing Kafka's Exactly-Once Semantics eliminates application-level duplicates.
>
> Kafka prevents duplicate records within Kafka transactions.
>
> It does **not** prevent duplicate business processing after a consumer updates an external database.
>
> End-to-end correctness still requires idempotent consumers.

---

# Production Example — Stripe

Payment API

```
POST /payments
```

Every request

contains

an Idempotency-Key.

Suppose

client retries

after timeout.

Server checks

```
Idempotency-Key
```

Already processed?

↓

Return previous response.

Customer

is never charged twice.

Notice

idempotency

starts

at the API layer,

not Kafka.

---

# Production Example — Uber

Trip Completed

↓

Outbox

↓

Kafka

↓

Billing

↓

Rewards

↓

Analytics

↓

Fraud Detection

Every downstream service

must tolerate

duplicate events.

No service

assumes

exactly-once delivery.

---

# Architecture Review Questions

Suppose another team proposes

```
INSERT Order

↓

Publish Kafka

```

Questions

- What if Kafka is unavailable?
- What if publish succeeds but DB commit fails?
- Where is retry state stored?
- How is duplicate processing prevented?
- How are poison messages handled?

A Principal Engineer immediately identifies

the Dual Write Problem.

---

# Interview Conversation

**Interviewer**

Why is the Outbox Pattern preferred over directly publishing to Kafka after a database commit?

---

**Weak Answer**

Because retries become easier.

---

**Principal Engineer Answer**

Publishing directly to Kafka creates a dual-write problem because the database and Kafka are updated independently. A failure between those operations permanently diverges system state. The Transactional Outbox collapses both business data and event metadata into a single local ACID transaction. Event publication becomes an asynchronous concern driven by polling or CDC, guaranteeing that every committed business transaction eventually produces its corresponding event.

---

# Common Interview Mistakes

> [!WARNING]
> Believing Kafka Exactly-Once eliminates duplicates.

---

> [!WARNING]
> Assuming Outbox alone guarantees end-to-end correctness.

Consumers must still be idempotent.

---

> [!WARNING]
> Ignoring the Inbox Pattern.

Reliable producers do not imply reliable consumers.

---

> [!WARNING]
> Using XA transactions between databases and Kafka.

Operationally expensive and rarely justified.

---

# Key Takeaways

- Dual writes are fundamentally unsafe.
- Transactional Outbox converts two independent writes into one local ACID transaction.
- CDC streams committed events directly from the database WAL.
- Debezium eliminates polling overhead.
- Exactly-once delivery is generally unattainable in distributed systems.
- Design for exactly-once effects through idempotent consumers.
- Outbox protects producers; Inbox protects consumers.
- End-to-end reliability requires retries, deduplication, and idempotency across every layer.
