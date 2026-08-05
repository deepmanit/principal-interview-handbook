
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

Distributed transactions attempt

to preserve

business consistency

across

multiple independent systems.

The goal is

not

database consistency.

The goal is

business correctness.

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

Question

Can

five different services

commit

at exactly

the same moment?

Very difficult.

Network delays

crashes

timeouts

retries

partitions

all complicate

coordination.

---

# Why Not One Database?

Interviewers often ask

```
Why not

put everything

into

one database?
```

Answer

Because

- Independent scaling
- Independent deployments
- Team autonomy
- Different storage technologies
- Failure isolation

Microservices intentionally

avoid

shared databases.

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

# What Makes This Hard?

Suppose

Payment succeeds.

Immediately afterwards

Inventory crashes.

Question

Who tells

Payment

to undo

the charge?

This requires

coordination.

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
