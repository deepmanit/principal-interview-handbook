# Functional vs Non-Functional Requirements

> "A Principal Engineer doesn't start by proposing solutions. They first ensure everyone agrees on the problem."

---

# Why This Chapter Matters

Most candidates fail a System Design interview before they draw the first architecture diagram.

Why?

Because they skip requirement gathering.

The best designs solve the **right problem**, not just an interesting problem.

---

# What are Functional Requirements?

Functional requirements describe **what the system should do**.

They define the business capabilities.

Examples

Twitter

- User Registration
- Login
- Post Tweet
- Delete Tweet
- Like Tweet
- Follow User
- View Timeline
- Search Tweets

Uber

- Book Ride
- Cancel Ride
- Driver Matching
- Payment
- Ride History
- Live Location

Netflix

- Browse Movies
- Search Movies
- Stream Videos
- Continue Watching
- Recommendations

---

# What are Non-Functional Requirements?

Non-functional requirements describe **how well the system should perform.**

Examples

- Scalability
- Availability
- Reliability
- Durability
- Latency
- Throughput
- Consistency
- Security
- Cost
- Observability

Interviewers care about these more than the functional requirements.

---

# Requirement Gathering Framework

Whenever you're given a design problem, don't jump into architecture.

Instead, follow this sequence.

```
Problem

↓

Users

↓

Features

↓

Scale

↓

Performance

↓

Availability

↓

Consistency

↓

Security

↓

Architecture
```

---

# Example

Interviewer

> Design WhatsApp.

Poor Answer

```
I'll use Kafka,
Redis,
MySQL...
```

Excellent Answer

```
Before discussing architecture,
I'd like to clarify the requirements.
```

---

# Questions Every Principal Engineer Should Ask

## Users

- Who are the users?
- Internal or external?
- Mobile?
- Web?
- API?

---

## Features

What should users be able to do?

Example

- Send Message
- Receive Message
- Group Chat
- Voice Message
- Video
- Read Receipts

---

## Scale

How many?

- Daily Active Users?
- Peak Users?
- Peak QPS?
- Storage?

Never guess.

Ask.

---

## Latency

Example

Messaging

Should delivery happen within

```
50 ms

100 ms

1 second?
```

Different latency requirements produce different architectures.

---

## Availability

Ask

```
99%

99.9%

99.99%

99.999%
```

Five nines changes everything.

---

## Consistency

Ask

Can users tolerate eventual consistency?

Example

Instagram Likes

Yes.

Bank Transfer

No.

---

## Durability

Should messages survive failures?

Can users lose messages?

---

## Security

Questions

- Authentication?
- Authorization?
- Encryption?
- GDPR?
- PII?

---

# Requirement Prioritization

Not every feature is equally important.

Use

```
Must Have

Should Have

Could Have

Won't Have
```

Example

Design Twitter

Must

- Login
- Tweet
- Timeline

Should

- Search

Could

- Spaces

Won't

- Video Streaming

---

# Out of Scope

Principal Engineers always define boundaries.

Example

```
For today's discussion,

I'll focus on

Tweet creation,

Timeline,

Feed generation.

I'll leave advertisements and analytics out of scope.
```

---

# Interview Conversation Example

Interviewer

Design Instagram.

Candidate

Before proposing an architecture,
I'd like to clarify a few requirements.

Are we focusing on

- Photo sharing only?
- Videos?
- Stories?
- Messaging?
- Live Streaming?

Understanding the scope helps optimize the architecture for the primary use case.

Notice:

No architecture yet.

---

# Common Interview Mistakes

❌ Assuming requirements

❌ Ignoring scale

❌ Forgetting latency

❌ Forgetting availability

❌ Not defining scope

❌ Discussing databases before requirements

---

# Principal Engineer Communication Style

Instead of

> I'll use Cassandra.

Say

> Before choosing a storage technology, I'd like to understand the expected workload, consistency requirements, and access patterns. Those factors determine whether a relational database, document database, or wide-column store is the best fit.

---

# Requirement Checklist

Before drawing architecture, verify that you've discussed:

- Users
- Functional Requirements
- Non-Functional Requirements
- Scale
- Latency
- Availability
- Consistency
- Durability
- Security
- Constraints
- Out of Scope

---

# Interview Tips

Always spend the first **5–10 minutes** gathering requirements.

Many candidates worry this is "wasting time."

It isn't.

It demonstrates structured thinking, reduces ambiguity, and leads to a design that actually addresses the interviewer's problem.

Strong candidates design systems.

Principal Engineers design the **right** systems.
