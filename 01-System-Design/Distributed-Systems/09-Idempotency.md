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
