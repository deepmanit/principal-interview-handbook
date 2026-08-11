# MVCC and Distributed Transactions

> **The central problem is not how to store multiple versions of a row. The problem is how to allow concurrent transactions to make progress while still providing a well-defined view of history.**

---

# Part 1 — Why MVCC Exists

A database transaction operates on a world that is changing concurrently.

Suppose:

```text
Transaction T1

reads Account A
```

while simultaneously:

```text
Transaction T2

updates Account A
```

The database must answer a fundamental question:

> **Which version of Account A should T1 observe?**

A naïve database could simply lock the row.

But locking every read creates contention and reduces concurrency.

MVCC takes a different approach.

Instead of forcing every transaction to observe the latest physical row,

the database maintains multiple logical versions and determines which version is visible to a transaction.

The key idea is:

> **Readers can observe a consistent historical snapshot while writers create newer versions.**

---

# The Core Invariant

The fundamental MVCC invariant is:

> **A transaction must observe a database state that is consistent with its assigned visibility point.**

This is stronger than:

```text
Return the latest row.
```

The latest row may have been written after the transaction's logical snapshot.

MVCC therefore separates:

```text
Physical Storage State

from

Transaction Visibility
```

This separation is one of the foundations of modern relational databases.

---

# Why "Latest Value" Is Not Enough

Consider:

```text
Account balance = ₹10,000
```

Transaction T1 begins.

Then T2 executes:

```text
UPDATE balance
SET balance = 8,000
```

and commits.

If T1 reads the account after T2 commits, should it see:

```text
₹8,000
```

or:

```text
₹10,000
```

The answer depends on the isolation level and T1's snapshot.

MVCC allows the database to answer this deterministically.

---

# Versioned Rows

Conceptually, instead of storing:

```text
Account A = ₹10,000
```

the database may maintain versions:

```text
A @ 100 = ₹10,000

A @ 120 = ₹8,000

A @ 150 = ₹6,000
```

A transaction with visibility timestamp:

```text
125
```

can see:

```text
A @ 120
```

but not:

```text
A @ 150
```

Therefore:

```text
Snapshot = 125

Visible Version = newest version <= 125
```

This is the basic MVCC model.

Actual database implementations vary significantly in how these versions are physically represented.

---

# MVCC Is Not Simply "Keep Old Rows"

This is an important distinction.

MVCC is a **visibility protocol**, not merely a storage technique.

The database needs to answer:

```text
Which versions are visible?

Which versions are obsolete?

Which transactions may overwrite each other?

Which snapshots remain active?

When can old versions be garbage-collected?
```

Therefore MVCC interacts directly with:

- transaction timestamps
- isolation levels
- garbage collection
- locking
- indexes
- transaction retries
- replication

---

# Visibility Rule

A simplified MVCC rule is:

```text
Transaction snapshot = S
```

A version written at timestamp:

```text
V
```

is visible when:

```text
V <= S
```

subject to the database's transaction state and isolation rules.

The transaction chooses the newest visible version.

Conceptually:

```text
Versions:

V1 = 100
V2 = 120
V3 = 150

Snapshot = 130

Visible:

V1
V2

Chosen:

V2
```

This gives the transaction a stable historical view.

---

# Why This Improves Concurrency

Consider:

```text
Reader T1
      ↓
    Row A
```

and:

```text
Writer T2
      ↓
    Row A
```

Without MVCC:

```text
T1 locks Row A
      ↓
T2 waits
```

With MVCC:

```text
T1 reads historical version

T2 creates newer version
```

The reader does not necessarily block the writer.

This is one of the primary performance advantages of MVCC.

---

# But MVCC Does Not Eliminate Conflicts

A common interview mistake is:

> "MVCC means readers never block writers, so there are no locking problems."

Incorrect.

MVCC primarily changes how reads observe versions.

Writes can still conflict.

For example:

```text
T1 reads A

T2 updates A

T1 later attempts to update A
```

Depending on the isolation model, T1 may:

- block,
- abort,
- retry,
- or detect a serialization conflict.

MVCC improves concurrency.

It does not make concurrency control disappear.

---

# MVCC and Isolation Levels

MVCC is not an isolation level.

It is an implementation strategy that can support different isolation semantics.

Possible isolation levels include:

```text
Read Uncommitted

Read Committed

Repeatable Read

Snapshot Isolation

Serializable
```

Different databases implement these semantics differently.

Therefore:

> **"This database uses MVCC" does not tell you its isolation guarantees.**

A Principal Engineer should always ask:

> What anomalies does the implementation permit?

---

# The Anomalies Matter More Than the Names

Instead of memorizing:

```text
Repeatable Read
Snapshot Isolation
Serializable
```

reason in terms of anomalies:

```text
Dirty Read

Non-repeatable Read

Lost Update

Write Skew

Phantom / Predicate Anomaly
```

The actual interview question is usually:

> "Can this execution happen?"

not:

> "Which isolation-level name is this?"

---

# Example — Non-Repeatable Read

Suppose T1 executes:

```text
SELECT balance
```

and sees:

```text
₹10,000
```

T2 updates and commits:

```text
₹8,000
```

T1 reads again.

If the second read returns:

```text
₹8,000
```

the transaction observed two different committed versions.

That is a non-repeatable read.

A stable snapshot can prevent this by ensuring both reads use the same visibility point.

---

# Example — Write Skew

Write skew is much more subtle and extremely important for Principal interviews.

Suppose two doctors are on call.

Invariant:

```text
At least one doctor must remain on call.
```

Initially:

```text
Alice = ON

Bob = ON
```

Transaction T1:

```text
Read Alice = ON
Read Bob = ON

Set Alice = OFF
```

Transaction T2:

```text
Read Alice = ON
Read Bob = ON

Set Bob = OFF
```

Both transactions independently conclude:

```text
At least one doctor remains ON
```

Both commit.

Final state:

```text
Alice = OFF

Bob = OFF
```

Invariant violated.

Neither transaction necessarily updated the same row.

This is why simple row-level conflict detection is not enough for serializable correctness.

---

# Principal Engineer Insight

The most important MVCC interview concept is:

> **Snapshot consistency does not automatically imply serializability.**

A transaction can observe a perfectly consistent snapshot and still participate in an execution that violates a cross-row business invariant.

This is the root of write skew.

---

# Why MVCC Needs Timestamps

If transactions need snapshots, the database needs a way to identify versions.

That is where the previous chapter connects directly.

We can use:

```text
Logical timestamps

HLC timestamps

Database transaction IDs

Commit sequence numbers
```

depending on the database.

This creates a direct relationship:

```text
Clock / Timestamp Model

↓

MVCC Visibility

↓

Transaction Isolation
```

This is why distributed database clocks are not merely an infrastructure detail.

They participate in transaction correctness.

---

# Local MVCC vs Distributed MVCC

A single-node database can often use a local transaction ID or monotonically increasing sequence.

Distributed databases face a harder problem.

Suppose:

```text
Shard A

Transaction T
```

and:

```text
Shard B

Transaction T
```

need to participate in the same distributed transaction.

Both shards need a coherent transaction timestamp.

Otherwise:

```text
Shard A sees T at timestamp 100

Shard B sees T at timestamp 90
```

and the system can produce inconsistent visibility.

Distributed MVCC therefore requires a distributed timestamp model.

This is one reason HLC becomes important in systems such as CockroachDB.

---

# Transaction Timestamp

A distributed transaction may conceptually receive:

```text
Transaction T

Timestamp = 100
```

All participating shards reason about the transaction using this logical timestamp.

For example:

```text
Shard A

Read X @ <= 100
```

and:

```text
Shard B

Read Y @ <= 100
```

The transaction therefore operates against a coherent logical snapshot.

The exact protocol differs by database.

The invariant is:

> All participants must agree on the transaction's visibility semantics.

---

# MVCC and Garbage Collection

Maintaining every historical version forever is impossible.

Suppose:

```text
K@100
K@105
K@110
...
K@10,000,000
```

Old versions eventually become unnecessary.

But they cannot be deleted arbitrarily.

Suppose a long-running transaction still has:

```text
Snapshot = 105
```

Deleting:

```text
K@105
```

could make its historical read impossible.

Therefore MVCC garbage collection depends on the oldest active snapshot.

Conceptually:

```text
Oldest active transaction

↓

GC safe point

↓

Versions older than safe point
may be reclaimed
```

This creates another production concern:

> **Long-running transactions can prevent garbage collection and cause storage amplification.**

---

# Storage Amplification

Suppose a hot row changes:

```text
1,000,000 times
```

while old snapshots remain active.

MVCC may need to preserve many versions.

The result can be:

```text
More versions

↓

More storage

↓

More compaction / GC work

↓

Higher IO

↓

Higher latency
```

Therefore transaction duration is not merely an application concern.

It can directly affect database health.

---

# Long-Running Transaction Failure Mode

Consider:

```text
Transaction T1

started 2 hours ago
```

while the application continuously updates the same table.

Because T1 may still need historical versions, the database cannot reclaim all old versions.

Eventually:

```text
MVCC history grows

↓

Storage pressure

↓

Compaction pressure

↓

Performance degradation
```

A production system therefore monitors:

- transaction age
- MVCC garbage
- history retention
- snapshot age
- compaction backlog

---

# MVCC and Indexes

MVCC is not limited to table rows.

Indexes must also obey transaction visibility rules.

Suppose:

```text
Row R

old value:
status = ACTIVE

new value:
status = DELETED
```

An index lookup must not return an entry that is invisible to the transaction's snapshot.

This becomes increasingly complex with:

- secondary indexes
- range scans
- unique constraints
- predicate reads
- phantom protection

This is one reason serializable distributed databases are significantly more complex than simply storing row versions.

---

# The Connection to Phantom Reads

Suppose:

```text
T1:

SELECT *
FROM orders
WHERE amount > 1000;
```

returns:

```text
10 rows
```

T2 then inserts:

```text
amount = 5000
```

T1 repeats the query.

A system with weaker isolation may now observe:

```text
11 rows
```

The new row is a phantom relative to the original predicate.

MVCC snapshots can prevent some forms of this by giving T1 a stable view.

But serializability may require stronger protection against concurrent predicate-based modifications.

This is where techniques such as:

```text
Predicate locking

Range locking

Serializable Snapshot Isolation

```

become relevant.

---

# MVCC Does Not Automatically Mean Snapshot Isolation

A database can use MVCC while providing:

```text
Read Committed
```

rather than:

```text
Snapshot Isolation
```

For example, each statement may receive a new snapshot.

Therefore:

```text
Statement 1

Snapshot S1
```

and:

```text
Statement 2

Snapshot S2
```

can observe different committed states.

At the transaction level, that is different from a single stable snapshot.

This distinction is frequently tested in interviews.

---

# Statement Snapshot vs Transaction Snapshot

Consider:

```text
Transaction T1

SELECT A

↓

UPDATE B

↓

SELECT A again
```

Under statement-level snapshots:

```text
SELECT A

uses S1

SELECT A

uses S2
```

Under transaction-level snapshot semantics:

```text
Both reads

use S1
```

Therefore:

```text
MVCC implementation
```

does not by itself tell you:

```text
snapshot lifetime
```

You must understand the database's isolation semantics.

---

# Principal Engineer Question

Suppose an interviewer asks:

> "Why does MVCC improve read concurrency?"

A strong answer:

> MVCC decouples logical visibility from the latest physical version. Readers can often access a committed historical version corresponding to their snapshot while writers create newer versions, reducing reader-writer blocking. However, write-write conflicts and serializability constraints still require concurrency control.

That is much stronger than:

> "Readers don't lock rows."

---

# Principal Engineer Question

> "Does MVCC guarantee serializability?"

No.

MVCC is a version-management and visibility mechanism.

Snapshot Isolation implemented with MVCC can still permit anomalies such as write skew.

Serializable systems need an additional mechanism to detect or prevent serialization anomalies.

---

# Principal Engineer Question

> "Why does a distributed database need a global timestamp?"

The answer depends on the consistency model.

If transactions span multiple replicas or shards and require a coherent snapshot, participants need a compatible notion of transaction ordering and visibility.

A distributed timestamp model such as HLC can provide a logical ordering while remaining approximately aligned with physical time.

The timestamp alone is not sufficient for atomic commit; it is one component of the transaction protocol.

---

# Principal Engineer Mental Model

Think of MVCC as three separate questions:

```text
1. Which versions exist?

2. Which versions are visible to this transaction?

3. What concurrent executions are legal?
```

MVCC primarily addresses:

```text
1 + 2
```

Isolation and serializability mechanisms address:

```text
3
```

This distinction prevents a large number of design mistakes.

---

# Production Architecture

A modern distributed SQL database can conceptually look like:

```text
Client
  |
  v
SQL Layer
  |
  v
Transaction Layer
  |
  +----------------------+
  |                      |
  v                      v
MVCC / Timestamp      Concurrency
Management             Control
  |                      |
  +----------+-----------+
             |
             v
       Replicated Storage
             |
             v
        Raft / Consensus
```

In a cross-shard transaction:

```text
Transaction Coordinator
          |
          v
         2PC
      /       \
     v         v
Shard A      Shard B
  |            |
 Raft         Raft
```

Each component has a different responsibility.

---

# Component Responsibility

| Component | Responsibility |
|---|---|
| MVCC | Maintain multiple versions |
| Timestamp model | Establish transaction/version ordering |
| Locking / OCC / SSI | Control concurrent transactions |
| 2PC | Atomic commit across participants |
| Raft / Paxos | Replicated agreement and durability |
| GC | Reclaim obsolete versions |
| Application | Define business invariants |

This separation is essential when reasoning about distributed transactions.

---

# The Critical Distinction

Do not say:

```text
MVCC = Snapshot Isolation
```

Instead:

```text
MVCC

is a mechanism
```

while:

```text
Snapshot Isolation

is a concurrency/isolation guarantee
```

and:

```text
Serializable

is a stronger correctness guarantee
```

Different databases combine these mechanisms differently.

---

# Part 1 Interview Summary

The important progression is:

```text
Concurrent Transactions
        ↓
Need consistent visibility
        ↓
Multiple Versions
        ↓
MVCC
        ↓
Snapshots
        ↓
Isolation Guarantees
        ↓
Serializable Execution
```

The next question is:

> **What exactly does a snapshot guarantee, and why can Snapshot Isolation still allow write skew?**

That is the focus of Part 2.
---

# Part 2 — Snapshot Isolation

The central idea of Snapshot Isolation is:

> **A transaction reads from a consistent snapshot of committed data rather than observing every concurrent change as it happens.**

This dramatically improves read concurrency.

But there is a critical Principal-level caveat:

> **A consistent snapshot does not guarantee serializable execution.**

That distinction is the foundation of this section.

---

# What Snapshot Isolation Actually Guarantees

Conceptually, when a transaction starts:

```text
T1 starts

        ↓

Snapshot = database state at S
```

All reads performed by T1 are evaluated against that logical snapshot.

If another transaction commits later:

```text
T1 snapshot = S

T2 commits at S + 10
```

T1 does not automatically start seeing T2's changes.

Instead:

```text
T1

reads from snapshot S
```

while:

```text
T2

continues producing newer versions
```

This provides a stable view of the database.

---

# Why This Is Powerful

Without snapshot isolation, a long-running transaction can observe a changing database:

```text
Read A

↓

Another transaction commits

↓

Read B
```

The transaction may now observe a combination of states that never existed simultaneously.

Snapshot isolation instead gives:

```text
Snapshot S

+-----------------------------+
| A as of S                   |
| B as of S                   |
| C as of S                   |
+-----------------------------+
```

The transaction reasons against one coherent historical view.

---

# Snapshot Isolation Is a Logical Snapshot

The word "snapshot" should not be interpreted as:

```text
Copy the entire database.
```

That would be prohibitively expensive.

The database stores versions and uses transaction metadata to determine visibility.

Conceptually:

```text
Row A

A@100
A@120
A@150
```

Transaction snapshot:

```text
S = 125
```

The transaction sees:

```text
A@120
```

It does not need a physical copy of the entire database.

This is one of the core benefits of MVCC.

---

# Snapshot Selection

Different databases determine the snapshot timestamp differently.

Conceptually:

```text
Transaction begins

↓

Choose snapshot timestamp S

↓

All reads use versions visible at S
```

In a distributed database, `S` may be generated from a distributed timestamp mechanism such as:

```text
HLC
```

or another database-specific timestamp service.

The exact implementation is database-specific.

The semantic requirement is:

> All reads belonging to the snapshot must obey a consistent visibility boundary.

---

# Example

Suppose the database contains:

```text
Account A = ₹10,000
Account B = ₹20,000
```

Transaction T1 starts with:

```text
Snapshot = 100
```

T2 executes:

```text
A = ₹8,000
B = ₹22,000
```

and commits at:

```text
120
```

T1 continues reading.

It sees:

```text
A = ₹10,000
B = ₹20,000
```

because:

```text
120 > 100
```

T2's versions are outside T1's snapshot.

---

# Snapshot Isolation and Repeatable Reads

One useful consequence is repeatable reads.

Suppose:

```text
T1

SELECT balance
```

returns:

```text
₹10,000
```

T2 changes the balance:

```text
₹8,000
```

and commits.

T1 repeats:

```text
SELECT balance
```

Under transaction-level snapshot semantics, T1 continues seeing:

```text
₹10,000
```

because both reads use the same snapshot.

This prevents a classic non-repeatable read.

---

# Snapshot Isolation and Read-Only Transactions

Snapshot isolation is especially attractive for read-heavy workloads.

Consider:

```text
Analytics Query
```

running for several seconds while OLTP traffic continues.

Without MVCC:

```text
Analytics Query

blocks writers
```

or:

```text
writers

block analytics query
```

With MVCC:

```text
Analytics Query

reads historical snapshot

while

OLTP writes continue
```

This is one reason MVCC is widely used in modern databases.

---

# But Snapshot Isolation Has a Critical Weakness

Snapshot Isolation can allow:

```text
Write Skew
```

This is the anomaly that separates:

```text
Snapshot Isolation
```

from:

```text
Serializable Isolation
```

for many interview discussions.

---

# Write Skew

Consider the invariant:

```text
At least one doctor must remain on call.
```

Initial state:

```text
Alice = ON

Bob = ON
```

Two transactions begin against the same snapshot.

```text
T1 snapshot:

Alice = ON
Bob = ON
```

```text
T2 snapshot:

Alice = ON
Bob = ON
```

T1 decides:

```text
Alice can go OFF
```

because Bob is still ON.

T2 decides:

```text
Bob can go OFF
```

because Alice is still ON.

Then:

```text
T1 writes Alice = OFF

T2 writes Bob = OFF
```

The writes target different rows.

Therefore a simple write-write conflict detector may allow both transactions to commit.

Final state:

```text
Alice = OFF

Bob = OFF
```

The business invariant is broken.

---

# Why Snapshot Isolation Allowed This

The problem is not that either transaction saw an inconsistent snapshot.

Both saw:

```text
Alice = ON
Bob = ON
```

which was perfectly valid.

The problem is that both transactions made decisions based on a shared predicate:

```text
At least one doctor is ON
```

but updated different rows.

The database must reason about the dependency between:

```text
Read Set

and

Write Set
```

rather than only checking:

```text
Write Set ∩ Write Set
```

---

# The Core Difference

Under a simplified conflict model:

```text
T1:

Read A
Write B
```

and:

```text
T2:

Read B
Write A
```

There may be:

```text
T1 → T2

T2 → T1
```

dependencies.

Neither transaction writes the same row as the other.

Therefore a simple write-write conflict test misses the cycle.

This is the essence of write skew.

---

# Serialization Graph

A powerful Principal-level way to reason about transaction anomalies is the **serialization graph**.

Each transaction is a node:

```text
T1
T2
T3
```

Edges represent dependencies.

For example:

```text
T1 → T2
```

means T1 must precede T2 in any equivalent serial execution.

A serializable execution requires the dependency graph to be acyclic.

If we obtain:

```text
T1 → T2
↑       ↓
+-------+
```

then there is a cycle.

A serial ordering cannot represent that execution.

Therefore:

```text
Cycle

↓

Not Serializable
```

This graph-based reasoning is much more powerful than memorizing isolation-level names.

---

# Write Skew as a Dependency Cycle

Consider:

```text
T1

Read Bob = ON
Write Alice = OFF
```

and:

```text
T2

Read Alice = ON
Write Bob = OFF
```

Dependencies:

```text
T1 reads Bob
T2 writes Bob

T2 reads Alice
T1 writes Alice
```

Therefore:

```text
T1 → T2

and

T2 → T1
```

which creates:

```text
T1 ↔ T2
```

This cycle means the execution cannot be serialized while preserving all observed dependencies.

That is the deeper reason write skew violates serializability.

---

# Snapshot Isolation vs Serializability

Snapshot Isolation provides:

```text
Consistent Snapshot
+
Certain Write Conflict Detection
```

Serializable isolation requires:

```text
Execution equivalent to some serial execution
```

Therefore:

```text
Snapshot Isolation

⊂

Serializable Semantics
```

in terms of guarantees.

The exact relationship and implementation details vary by database, but conceptually serializability is the stronger requirement.

---

# Important Interview Trap

Interviewer:

> "If every transaction reads from a consistent snapshot, how can the result be non-serializable?"

Strong answer:

> A transaction-level snapshot can be internally consistent while concurrent transactions make mutually incompatible decisions based on that snapshot. Snapshot isolation typically detects conflicting writes but does not necessarily detect all read-write dependency cycles. Write skew is the classic example.

That answer demonstrates real understanding.

---

# Read-Write Conflicts Matter

Suppose:

```text
T1:

Read A
Write B
```

while:

```text
T2:

Write A
Read B
```

There may be a serialization dependency even though:

```text
WriteSet(T1) ∩ WriteSet(T2) = ∅
```

This is why serializable systems need to reason beyond write-write conflicts.

Possible techniques include:

```text
Two-Phase Locking

Serializable Snapshot Isolation

Predicate / Range Locking

Optimistic Conflict Detection
```

---

# Snapshot Isolation and Lost Updates

Consider:

```text
Balance = 100
```

T1 reads:

```text
100
```

T2 reads:

```text
100
```

T1 calculates:

```text
100 + 20 = 120
```

T2 calculates:

```text
100 + 50 = 150
```

If both blindly write:

```text
T1 → 120

T2 → 150
```

the final state may lose T1's update.

A proper implementation of Snapshot Isolation generally detects a conflicting concurrent update to the same item and aborts one transaction rather than allowing both writes to overwrite each other.

This is an important distinction:

```text
Write-write conflict

≠

Write skew
```

Snapshot Isolation is much better at the former than the latter.

---

# First-Committer-Wins

A common implementation strategy is:

```text
Two transactions write the same logical item

↓

One commits first

↓

The other detects conflict

↓

Abort / retry
```

This prevents straightforward lost-update scenarios.

Conceptually:

```text
T1 writes A

T2 writes A

↓

T1 commits

↓

T2 cannot commit its conflicting version
```

The exact behavior depends on the database.

The important principle is:

> Concurrent modifications to the same logical version must not silently overwrite one another.

---

# Why Write Skew Is Harder

With write skew:

```text
T1 writes A

T2 writes B
```

There is no direct write-write conflict.

Yet the transactions may violate an invariant involving both A and B.

This is why simply implementing:

```text
First Committer Wins
```

does not automatically produce serializable isolation.

---

# Predicate-Based Invariants

Write skew often appears when the business invariant is expressed as a predicate.

Examples:

```text
At least one doctor is ON
```

```text
At least one server remains healthy
```

```text
Total allocated credit <= credit limit
```

```text
At most 10 seats can be allocated
```

```text
There must always be an active primary
```

The transaction reads a set of rows satisfying a predicate and updates one or more rows.

A concurrent transaction can modify another row so that the predicate becomes false.

This is fundamentally harder than detecting a single-row conflict.

---

# Example — Credit Limit

Suppose:

```text
Credit Limit = ₹100,000
```

Current allocations:

```text
Customer A = ₹40,000
Customer B = ₹40,000
```

Two transactions simultaneously check:

```text
Current total = ₹80,000
```

T1 adds:

```text
₹20,000
```

T2 adds:

```text
₹20,000
```

Each transaction sees:

```text
80,000 + 20,000 <= 100,000
```

Both may commit under Snapshot Isolation.

Final total:

```text
₹120,000
```

The invariant is violated.

Again:

```text
Different rows

+

Same business predicate
```

creates the anomaly.

---

# How Serializable Isolation Prevents This

A serializable system must ensure that the concurrent execution is equivalent to some serial ordering.

For the credit-limit example, the system must prevent both transactions from simultaneously making decisions based on the same stale predicate.

Possible strategies include:

```text
Predicate / Range Locks

or

Serializable Conflict Detection

or

Optimistic Validation

or

Explicit Application Locking
```

The implementation varies.

The invariant is the same:

> The system must prevent a dependency cycle that cannot correspond to a serial execution.

---

# Predicate Locks

One approach is to lock not only existing rows but the logical range represented by the query.

For example:

```text
SELECT *
FROM accounts
WHERE balance > 100000
```

The database may need to protect the predicate/range from concurrent modifications.

This becomes complex because:

```text
The row may not exist yet.
```

A future insert can change the query result.

Therefore databases may use:

```text
Range Locks

Gap Locks

Predicate Locks
```

depending on their concurrency-control architecture.

---

# Why Predicate Locking Is Expensive

Suppose the query is:

```text
WHERE amount BETWEEN 1000 AND 5000
```

The database must reason about:

```text
Existing rows

+

Potential future inserts

+

Updates that move rows into the range
```

This can create substantial contention.

Serializable systems therefore often use more sophisticated techniques rather than naïvely locking every predicate.

---

# Serializable Snapshot Isolation

SSI provides another approach.

Instead of blocking every potentially conflicting operation, the system tracks dangerous dependency patterns.

The basic idea is:

```text
Allow concurrent reads/writes

↓

Track serialization dependencies

↓

Detect dangerous structures

↓

Abort a transaction if necessary
```

This can preserve much of the concurrency benefit of snapshot-based execution while preventing non-serializable outcomes.

---

# The Dangerous Structure

A simplified SSI concept is:

```text
T1  →  T2
↑       ↓
+-------+
```

where dependencies form a structure that could create a serialization cycle.

The system detects sufficient evidence of such a dangerous pattern and aborts one transaction.

The exact SSI algorithm is more nuanced than this simplified diagram.

For interview purposes, the key idea is:

> **Serializable Snapshot Isolation preserves snapshot-style reads while adding conflict tracking strong enough to prevent serialization anomalies.**

---

# Why Abort Is Often Better Than Blocking

Consider two conflicting transactions.

A serializable optimistic system can:

```text
T1 executes

T2 executes concurrently

Conflict detected

↓

Abort T2

↓

Retry T2
```

instead of:

```text
T1 executes

T2 blocks

T1 commits

T2 continues
```

This can provide better concurrency when conflicts are relatively rare.

The trade-off is:

```text
Retries

+

Wasted work

+

Tail latency
```

This is a recurring distributed-systems pattern:

> Detect conflicts optimistically and retry when contention is low.

---

# Transaction Retry Is Part of the Architecture

At Principal level, do not treat retries as an implementation detail.

If the database provides serializable optimistic concurrency:

```text
Transaction may abort

↓

Application must retry
```

Therefore the application architecture must account for:

- retry safety
- idempotency
- transaction boundaries
- maximum retry count
- exponential backoff
- observability
- user-visible latency

This is particularly important for distributed SQL systems.

---

# Retry and Side Effects

Suppose a transaction performs:

```text
Database Update

↓

Send Email

↓

Transaction Aborts
```

If the transaction is retried:

```text
Database Update

↓

Send Email again
```

the user may receive duplicate emails.

Therefore external side effects should not be performed directly inside a transaction that may be retried.

A safer pattern is:

```text
Transaction

↓

Transactional Outbox

↓

Commit

↓

Async Event

↓

Email Service
```

This connects transaction isolation directly to event-driven architecture.

---

# Snapshot Isolation and External Side Effects

The transaction can be internally atomic while external side effects are not.

For example:

```text
T1:

Debit Account
Create Order
Send Email
```

If the database transaction aborts after the email is sent:

```text
Email exists

Database changes do not
```

The system has an externally visible inconsistency.

Therefore transaction retry design must distinguish:

```text
Transactional State

from

External Side Effects
```

---

# Read-Only Transactions

Snapshot isolation is especially useful for read-only workloads.

A read-only transaction can often execute against a stable snapshot without acquiring write locks.

For example:

```text
Generate monthly financial report
```

while:

```text
OLTP transactions continue
```

The report can see one logical database state.

This is one of the strongest practical reasons to use MVCC.

---

# Long-Running Snapshot Problem

There is a trade-off.

A long-running snapshot may prevent old versions from being garbage-collected.

Consider:

```text
T1 starts

Snapshot = 100
```

Then the system performs:

```text
100,000 updates
```

The database may need to retain versions that are still potentially visible to T1.

Therefore:

```text
Long-running snapshot

↓

Old versions retained

↓

MVCC storage grows

↓

GC delayed
```

This is a major production failure mode.

---

# Principal-Level Production Question

> "Our database storage usage suddenly grows even though application write traffic has not increased. What would you investigate?"

A strong answer should include:

```text
Long-running transactions

Old snapshots

MVCC garbage backlog

Replication lag

Compaction backlog

History retention

Failed / abandoned transactions
```

The important insight is:

> Storage growth can be caused by slow readers, not just fast writers.

---

# Snapshot Isolation and Replication Lag

In a distributed database, snapshots can interact with replication.

Suppose:

```text
Primary

timestamp = 100
```

while a replica has only applied:

```text
timestamp = 90
```

A transaction requiring snapshot:

```text
100
```

cannot safely read from that replica unless the replica can guarantee visibility up to the required timestamp.

Therefore distributed read routing often needs concepts such as:

```text
Read timestamp

Replica applied timestamp

Safe timestamp

Read lease
```

The exact terminology varies by database.

The underlying invariant is:

> A replica must not serve a snapshot that it has not safely materialized.

---

# Safe Time

A distributed replica may know:

```text
Applied through timestamp = 100
```

but it may not know whether there are unresolved writes from earlier logical time.

A stronger concept is a **safe timestamp**:

```text
Safe(T)
```

meaning:

> The replica can safely serve reads at or before T without violating the database's consistency model.

This becomes important in geo-distributed databases.

---

# Stale Reads

Once the system understands snapshot timestamps, it can intentionally support:

```text
Read at T
```

where T is slightly behind the latest state.

This can improve:

- latency
- locality
- availability
- read scalability

For example:

```text
User requests data

↓

Nearest replica

↓

Read from timestamp T - 5 seconds
```

If the business allows slightly stale data, this can avoid expensive cross-region coordination.

Again:

> Consistency is a product decision, not merely a database configuration.

---

# Principal Engineer Trade-off

Suppose a global application has:

```text
US users

EU users

India users
```

and a read request can tolerate:

```text
2 seconds of staleness
```

There may be little value in forcing every read through the globally latest timestamp.

Instead:

```text
Local replica

+

Bounded staleness
```

may provide dramatically lower latency.

The Principal-level question is:

> What freshness does the business actually require?

---

# Snapshot Isolation — Interview Mental Model

Think of Snapshot Isolation as:

```text
Transaction starts

↓

Choose snapshot

↓

Reads see that snapshot

↓

Writes create new versions

↓

Conflicting writes may abort

↓

But cross-row dependency cycles may still exist
```

The last line is the important caveat.

---

# Isolation Comparison

| Property | Read Committed | Snapshot Isolation | Serializable |
|---|---|---|---|
| Dirty reads prevented | Usually | Yes | Yes |
| Stable transaction snapshot | Not necessarily | Yes | Yes |
| Non-repeatable reads | Possible | Prevented | Prevented |
| Lost updates | Depends | Typically detected | Prevented |
| Write skew | Possible | Possible | Prevented |
| Predicate anomalies | Possible | Possible | Prevented |
| Serial execution equivalence | No | No | Yes |

The exact behavior varies by database implementation.

Always verify the actual database semantics rather than relying solely on the SQL-standard isolation names.

---

# Principal Interview Trap

Interviewer:

> "Snapshot Isolation means every transaction sees a consistent snapshot, therefore it is serializable. Correct?"

Strong answer:

> No. Snapshot consistency is not equivalent to serializability. Two transactions can read the same valid snapshot and make mutually incompatible updates to different rows. Write skew is the classic example. Serializable execution requires preventing dependency cycles, not merely providing a consistent read snapshot.

---

# Principal Interview Trap

Interviewer:

> "If two transactions write different rows, can they always commit concurrently?"

No.

The answer depends on the business invariant.

If the transactions are independent:

```text
T1 writes A

T2 writes B
```

concurrent commit may be perfectly safe.

But if:

```text
T1 reads B and writes A

T2 reads A and writes B
```

the operations can form a serialization cycle.

Therefore row-level write conflicts are insufficient to reason about serializability.

---

# Principal Interview Trap

Interviewer:

> "Why can't we simply lock every row read by a transaction?"

You can, but that moves the design toward pessimistic concurrency control.

It can:

- increase contention
- reduce concurrency
- increase deadlock probability
- increase lock-management overhead
- complicate distributed transactions

MVCC + optimistic validation can provide better throughput when conflicts are relatively rare.

The correct choice depends on workload characteristics.

---

# Production Checklist

When evaluating Snapshot Isolation in production, monitor:

```text
Transaction abort rate

Retry rate

Long-running transactions

Oldest snapshot age

MVCC version count

GC / compaction backlog

Lock contention

Serialization conflicts

Read latency

Write latency

Cross-region transaction latency
```

The most dangerous production problem is often not a single failed transaction.

It is a workload pattern that causes:

```text
More retries

↓

More work

↓

Higher latency

↓

Longer transactions

↓

More MVCC retention

↓

More storage / GC pressure

↓

Even higher latency
```

This is a positive-feedback failure loop.

---

# Principal Engineer Insight — Retry Storm

Suppose a hot key causes:

```text
10,000 concurrent transactions
```

to update the same logical entity.

If the database uses optimistic conflict detection:

```text
Most transactions begin

↓

One commits

↓

Many abort

↓

Many retry
```

Now retries create more load.

That load causes:

```text
Higher contention
```

which causes:

```text
More retries
```

This becomes a retry storm.

A Principal Engineer should therefore consider:

- contention-aware backoff
- request coalescing
- queueing
- single-writer ownership
- sharding
- reducing transaction scope
- application-level serialization

Concurrency is not always something to maximize.

Sometimes the correct design is to deliberately serialize a hot workload.

---

# Hot Key Example

Suppose:

```text
Inventory count = 1
```

and:

```text
1 million users
```

attempt to decrement it simultaneously.

MVCC does not magically solve the contention.

A naïve optimistic design may produce:

```text
1 winner

999,999 aborts

999,999 retries
```

This is catastrophic.

A better architecture may introduce:

```text
Admission Control

+

Queue

+

Single-writer / partition ownership
```

The database remains correct, but the application avoids generating an enormous conflict storm.

---

# The Core Principle

MVCC provides concurrency.

It does not mean:

```text
Unlimited concurrency
```

The system still has finite resources.

The real objective is:

> **Maximize useful concurrency while keeping conflict probability and coordination cost under control.**

That is a much more Principal-level way to discuss database concurrency.

---

# Part 2 Summary

Snapshot Isolation gives transactions a stable logical view of committed data.

Its strengths are:

```text
Consistent snapshots

+

High read concurrency

+

Reduced reader-writer blocking

+

Historical version visibility
```

But it does not automatically provide:

```text
Serializable execution
```

because:

```text
Write Skew

+

Read-Write Dependency Cycles

+

Predicate Conflicts
```

can still produce non-serializable outcomes.

Therefore:

```text
MVCC

↓

Snapshot Isolation

↓

Serializable Conflict Control
```

are separate concepts.

The next step is to understand exactly how systems strengthen Snapshot Isolation into **Serializable execution**.

---

# Next — Part 3

The next section focuses on:

```text
Serializable Isolation

↓

Conflict Graphs

↓

Two-Phase Locking

↓

Serializable Snapshot Isolation

↓

Predicate / Range Conflicts

↓

Dangerous Structures

↓

Abort vs Block

↓

Why serializability is expensive
```

The key Principal-level question will be:

> **How can a system allow high concurrency while still guaranteeing that every committed execution is equivalent to some serial order?**
