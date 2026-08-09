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
