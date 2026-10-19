# Distributed Systems: The CAP Theorem

In any distributed data store, network delays and hardware faults are inevitable. Formulated by Eric Brewer in 1998 and formally proven by Seth Gilbert and Nancy Lynch in 2002, the **CAP Theorem** states that a distributed data system can simultaneously provide at most **two out of the three** core guarantees:

- **C**onsistency
- **A**vailability
- **P**artition Tolerance

---

## <image src="https://media.licdn.com/dms/image/v2/C5112AQFbYSaM_qyt8A/article-cover_image-shrink_423_752/article-cover_image-shrink_423_752/0/1540791647609?e=1793836800&v=beta&t=YoSzfAq0s1ss0PhHUUufNKqZzyKzYQBVmeiV1q7ZXfI">

## 1. The Three Pillars Defined

```
                       Consistency (C)
                             /\
                            /  \
                           /    \
                          /  CA  \   <- (Theoretical / Single-node only)
                         /        \
                        /__________\
      Availability (A)  \    /\    /  Partition Tolerance (P)
                         \  /  \  /
                          \/____\/
                        AP        CP
```

### 1.1 Consistency (C)

- **Definition:** Every read request receives the most up-to-date write or returns an error.
- **Mechanism:** All nodes in the cluster agree on a unified, atomic state at any single point in time.
- **Formal term:** **Linearizability** (strong consistency, single-copy serializability).
- **Failure condition:** If a write completes successfully on one node, no client can read an older value from any other node.

### 1.2 Availability (A)

- **Definition:** Every non-failing node must return a non-error response for every incoming request—without guaranteeing that the response contains the latest write.
- **Mechanism:** Prioritizes continuous system uptime, fast response times, and uninterrupted service over strict synchronization.
- **Caveat:** Responses with empty, stale, or out-of-order data satisfy availability as long as they are non-error responses.

### 1.3 Partition Tolerance (P)

- **Definition:** The cluster continues to function despite an arbitrary number of messages being delayed, dropped, or corrupted by the network between nodes.
- **Reality check:** Network links between data centers, racks, and servers inevitably drop, disconnect, or experience extreme latency. Therefore, in real-world distributed architectures, **Partition Tolerance is mandatory**.

---

## 2. The Core Trade-off: Why You Can Only Pick Two

Because physical network hardware is inherently fallible, network partitions cannot be prevented:

$$\text{Distributed System} \implies P = \text{True}$$

When a network partition occurs between two nodes, the system must choose between data accuracy and operational uptime:

```
[Client 1: Write X = 10]
          │
          ▼
      [Node A]  <─────── ✗ Network Partition ───────>  [Node B]
                                                          ▲
                                                          │
                                                [Client 2: Read X]
```

- **Option 1: Choose Consistency (CP)**
  Node B knows its view of the data might be stale because it cannot communicate with Node A. It refuses to serve Client 2 and returns a timeout or error.
  - Outcome: Data remains correct across the system, but availability is sacrificed.

- **Option 2: Choose Availability (AP)**
  Node B serves its current local copy ($X = \text{stale value}$) to Client 2 without waiting for Node A.
  - Outcome: The system remains fully responsive, but consistency is violated.

---

## 3. System Classifications & Real-World Examples

| Classification | Behavior During Partition                                                 | Key Trade-off                                                                       | Representative Technologies                                                                           |
| :------------- | :------------------------------------------------------------------------ | :---------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------- |
| **CP**         | Rejects writes/reads or waits for consensus until quorum is restored.     | Low latency and uptime sacrificed for zero data corruption.                         | **ZooKeeper, etcd, Consul, Google Cloud Spanner, CockroachDB, MongoDB** (with majority write concern) |
| **AP**         | Nodes accept read/write operations independently; reconciles state later. | Serves stale data; eventual consistency across replicas.                            | **Apache Cassandra, Amazon DynamoDB, CouchDB, Riak**                                                  |
| **CA**         | Assumes communication between all nodes never fails.                      | **Does not exist across a distributed network.** Valid only for single-node setups. | **PostgreSQL, MySQL, SQLite** (running standalone on a single instance without replication)           |

---

## 4. Modern Evolution: The PACELC Theorem

Formulated by Daniel Abadi, the **PACELC Theorem** extends CAP by addressing how a distributed system behaves during normal operating conditions (when no network partition exists):

$$\text{If } \mathbf{P} \text{ (Partition)} \rightarrow \text{Choose between } \mathbf{A} \text{ and } \mathbf{C}; \quad \mathbf{E} \text{ (Else)} \rightarrow \text{Choose between } \mathbf{L} \text{ (Latency) and } \mathbf{C} \text{ (Consistency)}$$

### Common Classifications:

- **PC / EC (e.g., Google Spanner, CockroachDB):**
  - Under partition: Guarantees **Consistency**.
  - Normal operation: Prioritizes **Consistency** (pays round-trip network latency to sync replicas).
- **PA / EL (e.g., Cassandra, DynamoDB):**
  - Under partition: Guarantees **Availability**.
  - Normal operation: Prioritizes low **Latency** (reads from the nearest local replica; converges eventually).
- **PA / EC (e.g., MongoDB default):**
  - Under partition: Guarantees **Availability**.
  - Normal operation: Maintains **Consistency** on the primary replica.

---

## 5. Architectural Checklist for System Design

When selecting or designing a database architecture:

1. **Audit Financial / Ledger Integrity:** If the domain is payment processing, balance tracking, or stock trading, select a **CP** system.
2. **Evaluate Read-Heavy & Social Feeds:** If the domain is social timelines, product reviews, or activity logging, select an **AP** system with eventual consistency.
3. **Tune Read/Write Quorums:** Modern databases like DynamoDB and Cassandra allow tunable consistency:
   $$R + W > N$$
   _(Where $N$ is the replication factor, $R$ is read quorum, and $W$ is write quorum to achieve strong consistency)._
