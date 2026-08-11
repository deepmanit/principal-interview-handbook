# Kafka Mental Model

Kafka is often described as a messaging system, but for Principal/Staff-level system design it is more useful to think of Kafka as a:

> **Distributed, durable, append-only event log where producers append records to partitions and consumers independently track their position using offsets.**

## 1. Core Mental Model

```text
Producer
   |
   v
Topic
   |
   +-------------------+
   |         |         |
   v         v         v
  P0        P1        P2
   |         |         |
   +---------+---------+
             |
       Consumer Group
```

A topic is a logical stream. A topic is divided into partitions.

Each partition is an ordered, append-only sequence:

```text
Partition 0

+-----+-----+-----+-----+-----+
| 100 | 101 | 102 | 103 | 104 |
+-----+-----+-----+-----+-----+
```

A record's position is identified by:

```text
(partition, offset)
```

An offset is meaningful only within its partition.

---

## 2. Why Partitions?

Partitions provide Kafka's primary mechanism for horizontal scalability and parallelism.

```text
             Topic
               |
       +-------+-------+
       |       |       |
       v       v       v
      P0      P1      P2
       |       |       |
      C0      C1      C2
```

Partitions are also important for:

- Ordering
- Consumer assignment
- Replication
- Throughput
- Fault isolation

> **Partitions are the fundamental unit of Kafka parallelism.**

---

## 3. Ordering

Kafka guarantees ordering **within a partition**, not across a topic.

```text
Partition 0: A B C
Partition 1: D E F
```

Kafka does not guarantee a global order such as:

```text
A B C D E F
```

### Interview answer

> Kafka provides ordering at the partition level, not at the topic level.

---

## 4. Partition Key

Suppose all events for a customer must be processed in order:

```text
Customer 123

OrderCreated
OrderPaid
OrderShipped
```

Use a stable key such as:

```text
key = customerId
```

The producer's partitioner routes the key to a partition:

```text
customer-123
      |
      v
Partition 5

OrderCreated
OrderPaid
OrderShipped
```

### Principal-level insight

Do not blindly choose `customerId`.

First ask:

> **What exactly needs to be ordered?**

If ordering is required only for an order, `orderId` may be a better key.

The partition key is therefore a **domain decision**, not just a Kafka configuration decision.

---

## 5. Hot Partitions

A poor key distribution can create a hot partition:

```text
P0 -> 5%
P1 -> 5%
P2 -> 5%
P3 -> 5%
P4 -> 5%
P5 -> 5%
P6 -> 5%
P7 -> 45%   <- HOT
P8 -> 10%
P9 -> 5%
```

Adding consumers may not solve the problem because the hot partition is still one partition.

This creates an important trade-off:

> **Strict ordering and maximum parallelism can conflict.**

Possible approaches:

1. Accept the serialization point when strict ordering is mandatory.
2. Choose a more appropriate partition key.
3. Relax ordering when the business allows it.
4. Split a hot logical entity into independently ordered sub-entities when the domain permits it.

---

## 6. Consumer Groups

A consumer group is a set of consumers cooperating to consume a topic.

For example:

```text
Topic: orders

P0
P1
P2
P3
```

```text
Consumer Group: payment-service

C1
C2
C3
C4
```

Kafka can assign:

```text
P0 -> C1
P1 -> C2
P2 -> C3
P3 -> C4
```

Within a consumer group:

> **A partition is assigned to at most one consumer at a time.**

---

## 7. More Consumers Than Partitions

If:

```text
Partitions = 3
Consumers = 5
```

then some consumers remain idle:

```text
P0 -> C1
P1 -> C2
P2 -> C3

C4 -> idle
C5 -> idle
```

Conceptually:

```text
useful parallelism ~= min(partitions, consumers)
```

Actual throughput also depends on processing time, CPU, I/O and downstream capacity.

---

## 8. Fewer Consumers Than Partitions

If:

```text
Partitions = 6
Consumers = 2
```

a consumer can own multiple partitions:

```text
C1 -> P0 P2 P4
C2 -> P1 P3 P5
```

This is normal.

---

## 9. Multiple Consumer Groups

Different consumer groups can independently consume the same topic:

```text
orders
 |
 +--> payment-group
 |
 +--> fraud-group
 |
 +--> analytics-group
```

Each group maintains its own offsets.

Therefore:

> **Offsets are tracked independently per consumer group and partition.**

This allows the same event stream to drive multiple independent business capabilities.

---

## 10. Kafka vs Traditional Queue

Traditional work queue:

```text
Message
   |
   v
Consumer
   |
  ACK
```

Kafka:

```text
                    Topic
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
   Payment Group  Fraud Group  Analytics Group
```

Each group has an independent position.

This is one reason Kafka is powerful for event-driven architectures.

---

## 11. Retention

A common misconception is:

> Kafka deletes a record after a consumer processes it.

Normally, it does not.

Kafka retains records according to topic retention policies such as:

- Time-based retention
- Size-based retention

Therefore a consumer can fall behind and later catch up, as long as the records are still retained.

This also enables replay and creation of new consumer groups that consume historical records.

---

## 12. Consumer Down for Several Hours

### Interview question

> A consumer is down for six hours. What happens when it comes back?

A strong answer:

> If the records are still within Kafka's retention period and the consumer group still has its committed offset, the consumer can resume from its previous position and catch up.

Consider:

- Topic retention
- Committed offsets
- Consumer group identity
- Offset reset policy
- Partition assignment
- Consumer lag

If records have expired, they cannot be replayed from Kafka.

---

## 13. Consumer Lag

Suppose the producer has reached offset 9 and the consumer has committed through offset 4:

```text
Producer position -> 9
Consumer position -> 4

Lag ~= 9 - 4
```

Lag tells us how far behind a consumer is.

But:

> **Lag alone does not prove that the consumer is unhealthy.**

For example:

```text
Producer = 100K msg/sec
Consumer = 90K msg/sec
```

lag will grow.

Always examine:

- Lag
- Lag growth rate
- Processing latency
- Throughput
- Partition-level distribution

---

## 14. Principal-Level Troubleshooting

### Scenario

> Consumer lag suddenly starts increasing. What do you investigate?

Do not immediately increase consumers.

Use an evidence-driven approach:

```text
Lag increasing
      |
      v
Partition-level lag?
      |
      +--> One partition -> Hot partition / skew
      |
      v
Is consumer polling?
      |
      v
Processing latency increasing?
      |
      v
DB / external API slow?
      |
      v
Connection pool saturated?
      |
      v
CPU / GC / thread contention?
      |
      v
Consumer rebalances?
      |
      v
Offset commit problems?
      |
      v
Poison / repeatedly failing records?
```

Useful signals include:

- Consumer lag
- Records consumed rate
- Processing latency
- Rebalance count
- Consumer errors
- DB latency
- Downstream API latency
- CPU
- GC pauses
- Thread-pool saturation

### Principal-level principle

> **Lag is a symptom. Find the bottleneck that prevents the consumer from making progress.**

---

## 15. Consumer Lifecycle

A simplified consumer loop:

```java
while (running) {
    ConsumerRecords<String, Event> records = consumer.poll();

    for (ConsumerRecord<String, Event> record : records) {
        process(record);
    }

    consumer.commitSync();
}
```

Conceptually:

```text
poll()
  |
  v
records returned
  |
  v
process records
  |
  v
commit offset
  |
  v
poll again
```

If processing becomes very slow, consumer liveness and rebalance-related configuration becomes important.

Key settings to understand:

- `max.poll.records`
- `max.poll.interval.ms`
- `session.timeout.ms`
- Heartbeats
- Offset commit configuration

These are covered in the Consumer and Rebalancing chapters.

---

## 16. Key Relationships

```text
                    Kafka
                      |
                 +----+----+
                 |         |
              Topic     Consumer
                 |         |
             Partitions  Consumer Group
                 |         |
             Ordering    Offset
                 |
              Key
                 |
          Partition Choice
```

Operationally:

```text
Partitions
    |
    +--> Parallelism
    +--> Ordering
    +--> Consumer assignment
    +--> Replication
    +--> Throughput
```

---

## 17. Common Interview Traps

### Trap 1

> Kafka guarantees message ordering.

**Incorrect / incomplete.**

Kafka guarantees ordering **within a partition**.

### Trap 2

> Add more consumers to increase throughput.

**Incomplete.**

More consumers help only when there are enough partitions and the consumers/downstream systems have capacity.

### Trap 3

> Kafka deletes a message after acknowledgment.

**Incorrect mental model.**

Kafka retains records according to retention policies.

### Trap 4

> Offset is globally unique.

**Incorrect.**

An offset is meaningful within a partition.

### Trap 5

> Always use customerId as the partition key.

**Incomplete.**

First determine which business entity actually requires ordering.

### Trap 6

> Consumer lag means Kafka is slow.

**Incorrect.**

Lag is a symptom. Investigate processing latency, partition skew, downstream dependencies, rebalances and resource saturation.

---

## 18. Principal-Level Interview Questions

### Fundamentals

1. What is Kafka?
2. Why does Kafka use partitions?
3. What is an offset?
4. Is an offset globally unique?
5. What is the relationship between topic and partition?
6. What does the partition key do?

### Ordering

7. Does Kafka guarantee global ordering?
8. How would you preserve ordering for a customer?
9. What is a hot partition?
10. How would you solve a hot partition?
11. What trade-off exists between ordering and scalability?

### Consumer Groups

12. What is a consumer group?
13. What happens if consumers > partitions?
14. What happens if partitions > consumers?
15. Can two consumers in the same group consume the same partition simultaneously?
16. Can two different consumer groups consume the same partition?
17. Why does Kafka support multiple consumer groups?

### Production

18. Consumer lag is increasing. How do you investigate?
19. Consumer is down for six hours. What happens when it returns?
20. What happens if records have expired before the consumer returns?
21. Why can adding consumers fail to improve throughput?
22. How can a single key become a bottleneck?

---

## 19. One-Minute Principal Interview Answer

> "I think of Kafka primarily as a distributed, durable append-only log rather than just a message queue. A topic is divided into partitions, and partitions provide the fundamental unit of scalability, parallelism and ordering. Kafka guarantees ordering within a partition, so if I need ordering for a business entity I'd choose a stable partition key that routes that entity's events to the same partition.
>
> Consumers operate in groups. Within a group, a partition is assigned to at most one consumer at a time, so useful consumer parallelism is bounded by the number of partitions. Different consumer groups maintain independent offsets and can therefore consume the same events independently.
>
> Kafka retains records according to retention policies rather than deleting them simply because a consumer processed them. This enables replay and independent consumers.
>
> In production, I'd pay particular attention to partition skew, consumer lag, processing latency, rebalances, downstream bottlenecks and resource saturation rather than assuming Kafka itself is the bottleneck."

---

## 20. Next

The next chapter should be:

**`02-Topics-Partitions-Offsets.md`**

It will go deeper into:

```text
Topic
  ↓
Partition
  ↓
Segment
  ↓
Record
  ↓
Offset
  ↓
Append
  ↓
Retention
  ↓
Segment rolling
  ↓
Partition scalability
```

