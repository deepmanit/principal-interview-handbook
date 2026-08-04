# Consistency Models

> "Consistency is not a binary property. It is a spectrum of guarantees that determines what a client can observe after data is written."

---

# Why This Chapter Matters

One of the most common Principal Engineer interview questions is:

> Why did you choose eventual consistency?

Most candidates answer

> Because it's faster.

That's incomplete.

Principal Engineers understand:

- Business requirements
- User expectations
- Failure scenarios
- Network partitions
- Latency trade-offs
- Operational complexity

---

# What is Consistency?

Consistency defines **what a client is guaranteed to observe after a write.**

Example

```
User A

writes

Balance = 100
```

Immediately afterward

```
User B

reads
```

Will User B see

```
100 ?

or

old value?
```

That depends on the consistency model.

---

# Strong Consistency

Definition

After a successful write,

every future read returns the latest value.

```
Write

↓

Success

↓

Every Read

↓

Latest Value
```

Examples

- PostgreSQL
- MySQL
- Google Spanner
- CockroachDB

---

## Advantages

- Easy reasoning
- No stale reads
- Suitable for financial systems

---

## Disadvantages

- Higher latency
- Reduced availability during partitions
- Cross-region coordination

---

# Eventual Consistency

Definition

Writes propagate asynchronously.

Eventually,

all replicas converge.

```
Write

↓

Replica 1

↓

Replica 2

↓

Replica 3

↓

Eventually Same
```

---

Examples

- Cassandra
- DynamoDB (default)
- Riak

---

Advantages

- High availability
- Excellent scalability
- Low latency

---

Disadvantages

- Stale reads
- Conflict resolution
- Temporary inconsistency

---

# Real Example

Instagram Like

User A

likes photo.

User B

refreshes immediately.

Like count

```
99

instead of

100
```

Acceptable?

Yes.

---

Bank Transfer

Balance

```
₹1000

↓

Transfer ₹500

↓

Read

₹1000
```

Acceptable?

No.

Need strong consistency.

---

# Read After Write Consistency

Guarantee

A client always sees its own writes.

Example

```
Upload Profile Picture

↓

Refresh

↓

New Picture Visible
```

Expected by users.

---

# Monotonic Read Consistency

Reads never go backward.

Example

```
Version 10

↓

Version 12

↓

Version 8
```

Impossible.

---

# Monotonic Write Consistency

Writes from the same client

must be applied

in order.

```
Version 1

↓

Version 2

↓

Version 3
```

Never

```
2

↓

1

↓

3
```

---

# Causal Consistency

Cause must be visible before effect.

Example

```
Comment

↓

Reply
```

User should never see

Reply

before

Comment.

---

# Session Consistency

Guarantees

Within one session

the user observes

consistent data.

Useful for

Shopping carts

Profile updates

User preferences

---

# Linearizability

One of the most important interview topics.

Definition

Every operation appears to happen

atomically

at one instant.

Think

```
Single Global Order
```

Examples

- ZooKeeper
- etcd
- Google Spanner

---

# Serializability

Guarantees

Concurrent transactions

produce

the same result

as if executed

one after another.

Concerned with

Transactions

Not

Replica visibility.

---

# Linearizability vs Serializability

| Linearizability | Serializability |
|-----------------|-----------------|
| Single operation guarantee | Transaction guarantee |
| Real-time ordering | Equivalent execution ordering |
| Distributed systems | Databases |

Interviewers love this comparison.

---

# CAP Theorem

During a network partition

choose

```
Consistency

or

Availability
```

Never both.

```
      Partition

      /      \

Consistency  Availability
```

---

Examples

CP

- ZooKeeper
- etcd

AP

- Cassandra
- DynamoDB

---

# PACELC

CAP only discusses partitions.

PACELC says

Even without partitions,

systems choose between

```
Latency

or

Consistency
```

Interviewers expect Principal Engineers to know this.

---

# Quorum

Suppose

```
N = 3 replicas
```

Write

```
W = 2
```

Read

```
R = 2
```

Rule

```
R + W > N
```

Ensures

latest value

is always visible.

---

# Conflict Resolution

Eventual consistency

requires

conflict resolution.

Techniques

- Last Write Wins
- Vector Clocks
- CRDT
- Application Merge

---

# Production Examples

## Banking

Need

Strong Consistency

---

## Social Media

Need

Eventual Consistency

---

## Shopping Cart

Need

Session Consistency

---

## Chat Application

Need

Read-after-write

---

## DNS

Need

Eventual Consistency

---

# Common Interview Mistakes

❌ Strong consistency everywhere

❌ Eventual consistency everywhere

❌ Ignoring business requirements

❌ Confusing CAP with PACELC

❌ Confusing Linearizability with Serializability

---

# Principal Engineer Communication

Instead of saying

> Cassandra is eventually consistent.

Say

> The business requirement prioritizes availability and low latency over immediate consistency. Temporary stale reads are acceptable because the application can tolerate eventual convergence. Cassandra's tunable consistency also allows adjusting read and write quorum levels based on workload requirements.

---

# Decision Framework

Before selecting a consistency model

ask

- Can users tolerate stale data?
- Is ordering important?
- Are transactions required?
- What happens during partitions?
- What latency can we accept?
- How expensive is coordination?

---

# Interview Checklist

Before finalizing a design

verify

- Consistency model chosen
- CAP implications discussed
- PACELC considered
- Quorum explained
- Failure scenarios covered
- Conflict resolution strategy defined
- Trade-offs justified

---

# Key Takeaways

There is no universally "best" consistency model.

The correct choice depends on:

- Business requirements
- User expectations
- Failure tolerance
- Latency goals
- Operational complexity

Principal Engineers choose consistency models based on business outcomes—not personal preference.
