# Capacity Estimation

> "A Principal Engineer doesn't design systems in a vacuum. Every architectural decision starts with understanding scale."

---

# Why Capacity Estimation Matters

Capacity estimation transforms vague requirements into engineering decisions.

Without understanding traffic, storage, and latency requirements, you cannot justify:

- Database choice
- Cache size
- Number of servers
- Kafka partitions
- Redis cluster size
- Load balancer strategy
- Sharding approach

Interviewers don't expect perfect numbers.

They expect **reasonable assumptions** and **correct reasoning**.

---

# Estimation Framework

Always estimate in this order.

```
Users

↓

Requests

↓

Read / Write Ratio

↓

Queries Per Second

↓

Storage

↓

Bandwidth

↓

Memory

↓

Servers
```

---

# Step 1 – Estimate Users

Suppose we're designing Twitter.

Assume

```
100 Million Daily Active Users
```

Ask the interviewer if they have a preferred scale.

If not, clearly state your assumption.

---

# Step 2 – User Activity

Suppose

```
Each user

creates 5 tweets/day

reads 200 tweets/day
```

Daily Writes

```
100M × 5

=

500 Million Writes / Day
```

Daily Reads

```
100M × 200

=

20 Billion Reads / Day
```

Immediately recognize

```
Read Heavy System
```

---

# Step 3 – Convert to QPS

Formula

```
QPS

=

Requests Per Day

/

86400
```

---

Example

```
500 Million Writes / Day

÷

86400

≈

5800 Writes/sec
```

---

Reads

```
20 Billion

÷

86400

≈

231,000 Reads/sec
```

Peak traffic is not average traffic.

Use

```
Peak QPS

≈

Average × 3
```

Peak Reads

```
231K

×

3

≈

700K QPS
```

---

# Step 4 – Storage Estimation

Suppose

Each tweet

```
300 Bytes
```

Daily Storage

```
500 Million

×

300

=

150 GB / Day
```

Yearly Storage

```
150 × 365

≈

55 TB
```

This immediately tells you

A single MySQL server is insufficient.

---

# Step 5 – Image Storage

Suppose

Average Image

```
2 MB
```

20 Million Images

```
20M × 2 MB

≈

40 TB
```

Use object storage

Example

- Amazon S3
- Google Cloud Storage
- Azure Blob Storage

Not MySQL.

---

# Step 6 – Bandwidth

Formula

```
Bandwidth

=

QPS

×

Response Size
```

Example

Feed API

Average Response

```
200 KB
```

Peak

```
700K QPS
```

Bandwidth

```
700K

×

200 KB

≈

140 GB/sec
```

This immediately suggests

- CDN
- Compression
- Caching

---

# Step 7 – Memory

Suppose

Hot tweets

```
10 Million
```

Average

```
1 KB
```

Need

```
10 GB RAM
```

Redis Cluster

Easy decision.

---

# Step 8 – Number of Servers

Suppose

One server handles

```
2000 QPS
```

Need

```
700K Peak QPS
```

Servers

```
700000

/

2000

=

350 Servers
```

Add

30%

headroom

```
≈450 Servers
```

---

# Capacity Estimation Cheat Sheet

## Seconds

```
1 Day = 86,400 sec
```

---

## Storage

```
1 KB = 1024 Bytes

1 MB = 1024 KB

1 GB = 1024 MB

1 TB = 1024 GB
```

---

## Traffic

```
Peak QPS

≈

3 × Average
```

---

## Memory

```
Cached Objects

×

Average Size
```

---

## Read Heavy Systems

Examples

- Twitter
- Instagram
- Netflix
- YouTube
- Facebook

Need

- Redis
- CDN
- Replicas

---

## Write Heavy Systems

Examples

- Banking
- Payments
- IoT
- Logging

Need

- Kafka
- Partitioning
- Durable Storage

---

# Example

Design URL Shortener

Assume

```
50 Million URLs / Day
```

Short URL

```
100 Bytes
```

Storage

```
50M × 100

=

5 GB / Day
```

One Year

```
≈1.8 TB
```

Reasonable for sharded relational storage.

---

# Common Interview Mistakes

❌ Skipping assumptions

❌ Forgetting peak traffic

❌ Ignoring read/write ratio

❌ Choosing a database before estimating scale

❌ Using unrealistic numbers

---

# Principal Engineer Communication

Instead of saying

> "I'll use Redis."

Say

> "The estimated peak read traffic is approximately 700,000 requests per second, while the working set is around 10 GB. Since this fits comfortably in memory, Redis is an appropriate cache to absorb the majority of reads and reduce pressure on the primary database."

---

# Capacity Estimation Checklist

Before designing the architecture, estimate:

- Daily Active Users
- Requests per User
- Read/Write Ratio
- Peak QPS
- Storage Growth
- Bandwidth
- Memory Requirements
- Number of Servers

---

# Interview Tip

Your numbers don't have to be perfect.

Your reasoning does.

Interviewers are evaluating whether your estimates are internally consistent and whether they lead to sensible architectural decisions.
