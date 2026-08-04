# Database Selection

> "Choosing a database is not about popularity. It's about matching the database characteristics to the workload."

---

# Why This Chapter Matters

One of the most common interview questions is:

> Why did you choose MySQL?

Most candidates answer:

> Because it's relational.

Principal Engineers answer with workload analysis.

Interviewers want to know **how you think**, not what database you know.

---

# Database Selection Framework

Never start with a database name.

Start with questions.

```
Data Model
      │
      ▼
Access Pattern
      │
      ▼
Consistency
      │
      ▼
Latency
      │
      ▼
Scale
      │
      ▼
Availability
      │
      ▼
Cost
      │
      ▼
Choose Database
```

---

# Step 1 — Understand the Data

Ask yourself:

- Is the data relational?
- Is the schema fixed?
- Does it change frequently?
- Are joins important?
- Are transactions required?

Example

Bank Account

```
Customer

Account

Transaction
```

Clearly relational.

SQL is a strong candidate.

---

# Step 2 — Understand Access Patterns

Ask

```
Read Heavy?

Write Heavy?

Read + Write?

Analytical?

Full Text Search?
```

The access pattern often determines the database more than the data model.

---

# SQL vs NoSQL

| SQL | NoSQL |
|------|--------|
| Fixed Schema | Flexible Schema |
| ACID | BASE / Tunable |
| Joins | Denormalization |
| Strong Consistency | Eventual Consistency |
| Vertical + Horizontal Scaling | Horizontal Scaling |
| Complex Queries | Simple Queries |

Neither is "better."

They solve different problems.

---

# MySQL

## Strengths

- ACID Transactions
- Strong Consistency
- Mature Ecosystem
- Excellent Joins
- Reliable Indexing

Best For

- Payments
- Banking
- Orders
- User Accounts
- Inventory

---

## Weaknesses

- Sharding complexity
- Limited write scalability
- Cross-shard joins become difficult

---

# PostgreSQL

Everything MySQL does plus

- JSON support
- Window Functions
- CTEs
- GIS
- Better query planner

Preferred for

- Enterprise applications
- Mixed relational + JSON workloads
- Reporting

---

# Redis

Purpose

Distributed in-memory cache.

Best For

- Session Store
- Rate Limiter
- Leaderboards
- Cache
- Pub/Sub

Advantages

- Extremely fast
- Rich data structures

Disadvantages

- Memory expensive
- Cache invalidation
- Limited durability (depending on configuration)

---

# Cassandra

Purpose

Massive write scalability.

Characteristics

- Wide-column database
- Peer-to-peer architecture
- No master
- Linear horizontal scaling

Best For

- IoT
- Time-series
- Messaging
- Logging

Trade-off

Eventual consistency.

---

# DynamoDB

Fully managed NoSQL.

Advantages

- Automatic scaling
- High availability
- Low operational overhead

Trade-off

- Vendor lock-in
- Access patterns must be designed carefully.

---

# MongoDB

Document database.

Best For

- Product Catalog
- CMS
- User Profiles
- Flexible schemas

Avoid when

Complex joins dominate.

---

# Elasticsearch

Purpose

Search.

Not primary storage.

Best For

- Full-text search
- Log analytics
- Product search
- Autocomplete

Never use Elasticsearch as the source of truth.

---

# ClickHouse

Purpose

Analytical queries.

Excellent for

- Dashboards
- BI
- Metrics
- Observability

Not suitable for OLTP.

---

# Neo4j

Graph database.

Best For

- Social Graph
- Fraud Detection
- Recommendation Engine
- Network Analysis

---

# Vector Database

Examples

- Pinecone
- Weaviate
- Milvus
- pgvector

Best For

- Semantic Search
- RAG
- AI Applications
- Embeddings

---

# Database Selection Cheat Sheet

| Use Case | Database |
|----------|----------|
| Banking | PostgreSQL / MySQL |
| Payments | PostgreSQL |
| Orders | PostgreSQL |
| Inventory | MySQL |
| User Profiles | MongoDB |
| Session Store | Redis |
| Cache | Redis |
| Messaging | Cassandra |
| IoT | Cassandra |
| Search | Elasticsearch |
| Analytics | ClickHouse |
| AI Search | Vector Database |
| Social Graph | Neo4j |

---

# Read vs Write Heavy

Read Heavy

Examples

- Netflix
- Instagram
- Twitter
- YouTube

Need

- Cache
- Read Replicas
- CDN

---

Write Heavy

Examples

- Logging
- IoT
- Payments
- Sensor Data

Need

- Kafka
- Cassandra
- Partitioning

---

# CAP Considerations

| Requirement | Better Choice |
|-------------|---------------|
| Strong Consistency | MySQL / PostgreSQL |
| High Availability | Cassandra |
| Low Latency Cache | Redis |
| Full Text Search | Elasticsearch |
| Analytics | ClickHouse |

---

# Common Interview Mistakes

❌ Choosing MySQL for everything

❌ Using Redis as the primary database

❌ Using Elasticsearch for transactions

❌ Ignoring access patterns

❌ Ignoring operational complexity

---

# Principal Engineer Communication

Instead of saying

> We'll use MongoDB because it's NoSQL.

Say

> The workload consists primarily of user profile documents with evolving schemas and minimal relational queries. MongoDB's document model aligns well with this access pattern, while avoiding expensive schema migrations. For transactional data such as payments, I would keep a relational database because strong consistency is more important than schema flexibility.

---

# Decision Checklist

Before selecting a database, answer:

- What is the data model?
- What are the access patterns?
- Do we need transactions?
- Is strong consistency required?
- What is the expected scale?
- How will data grow?
- What are the latency requirements?
- How much operational complexity can we tolerate?

---

# Key Takeaways

The best database is not the one with the most features.

It is the one whose characteristics best match the workload.

That is the mindset interviewers expect from a Principal Engineer.
