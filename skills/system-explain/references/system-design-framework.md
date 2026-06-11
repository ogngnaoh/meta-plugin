# System Design Fundamentals: A First-Principles Reference Framework

## TL;DR
- **System design is the discipline of satisfying functional requirements while meeting non-functional requirements (scalability, availability, latency, durability, cost) under hard physical and economic constraints — its entire body of patterns exists to manage trade-offs forced by the speed of light, the failure rate of components, and the cost of resources.** No design is "correct"; designs are only well-matched to a specific workload and set of priorities.
- The whole field derives from a handful of first principles: latency numbers that span ~9 orders of magnitude (0.5 ns L1 cache → 150 ms cross-continent), the impossibility results (CAP, PACELC, FLP), and the economics of CPU/memory/disk/network. Master these and every pattern (caching, replication, sharding, queues) becomes a derivable consequence rather than a thing to memorize.
- Use the repeatable method in Section 7 for any problem: clarify requirements → estimate scale → define API/data model → high-level design → find the bottleneck → deep-dive trade-offs. The architect's core skill is identifying the single binding constraint of a system and spending complexity only there.

---

## Key Findings

1. **System design solves a constrained optimization problem, not a search for a "right answer."** Martin Kleppmann frames data systems around three non-functional pillars — reliability, scalability, maintainability — layered on top of functional requirements. Everything else is the trade-off space.
2. **Physics sets the floor.** The speed of light in fiber (~200,000 km/s, ~2/3 of c) makes a New York–Sydney round trip take ~160 ms no matter how much you spend. This single fact drives CDNs, edge computing, data-center placement, and the choice between strong and eventual consistency.
3. **You cannot have it all.** CAP says during a network partition you choose consistency OR availability. PACELC adds the more important everyday truth: even without partitions, you trade latency vs. consistency on every request. FLP proves consensus cannot be guaranteed to terminate in a fully asynchronous system — which is *why* Paxos/Raft exist and why they rely on timeouts.
4. **Storage engines embody the read-vs-write trade-off.** B-trees (read-optimized, in-place updates) vs. LSM-trees (write-optimized, append + compaction) is the canonical example of "no free lunch" at the data-structure level.
5. **Statelessness is the master key to horizontal scaling.** Move state into dedicated data/cache tiers and the compute tier becomes trivially replicable behind a load balancer.

---

## Details

### 1. What System Design Is, From First Principles

**The fundamental problem.** A system must do certain things (functional requirements: "users can post a tweet, view a timeline"). But it must do them *well enough* along axes that the user and business care about — fast enough, reliably enough, cheaply enough, at sufficient scale. These quality attributes are the **non-functional requirements (NFRs)**. System design is the discipline of structuring components, data, and communication so the functional requirements are met **while** the NFRs are satisfied **within** the immovable constraints of physics and economics.

**Why the discipline exists.** If a single computer were infinitely fast, infinitely reliable, infinitely large, and free, system design would not exist — you'd put everything on one box. The discipline exists precisely because:
- **Machines are finite** (CPU, RAM, disk, NIC bandwidth all have ceilings).
- **Machines fail** (disks, power, network, software bugs, human error).
- **Information cannot travel faster than light** (latency has a hard floor).
- **Resources cost money** (and the cost curve for "one more nine" of reliability is roughly 10× per nine).

So the essential tension is always: **requirements (functional + non-functional) pulling against physical/economic constraints.** Every pattern in this document is a tool for resolving a specific instance of that tension, and every tool has a cost. Kleppmann's central message: *software keeps changing, but the fundamental principles remain the same.*

> **Mental model:** Think of design as allocating a fixed budget of complexity and money to "buy down" whichever NFR is most threatened by your workload. You never buy all of them maximally — you can't afford it and the trade-offs forbid it.

---

### 2. The Physical and Economic First Principles

#### 2.1 Latency Numbers Every Engineer Should Know

These are Jeff Dean's canonical "Numbers Everyone Should Know" (originally from Peter Norvig; ~2010 figures). Memorize the *orders of magnitude and ratios*, not the exact digits — the relationships barely change even as hardware improves.

| Operation | Latency | Humanized (×1 billion) |
|---|---|---|
| L1 cache reference | 0.5 ns | One heartbeat |
| Branch mispredict | 5 ns | Yawn |
| L2 cache reference | 7 ns | Long yawn |
| Mutex lock/unlock | 25 ns | Making coffee |
| Main memory reference | 100 ns | Brushing your teeth |
| Compress 1 KB (Zippy/Snappy) | 3,000 ns (3 µs) | — |
| Send 2 KB over 1 Gbps network | 20,000 ns (20 µs) | — |
| SSD random read | 150,000 ns (150 µs) | A weekend |
| Read 1 MB sequentially from memory | 250,000 ns (250 µs) | A long weekend |
| **Round trip within same datacenter** | **500,000 ns (0.5 ms)** | A medium vacation |
| Read 1 MB sequentially from SSD | 1,000,000 ns (1 ms) | — |
| Disk seek | 10,000,000 ns (10 ms) | A semester |
| Read 1 MB sequentially from disk | 20,000,000 ns (20 ms) | Almost gestating a human |
| **Send packet CA→Netherlands→CA** | **150,000,000 ns (150 ms)** | ~4.8 years (×1 billion) |

**The takeaways that drive architecture:**
- **Memory is ~100–1000× faster than SSD, which is ~10–100× faster than spinning disk, which is far faster than cross-datacenter network.** This hierarchy is *why caching works* (see §5.2): keep hot data on the fastest tier.
- **Sequential access crushes random access**, especially on disk. This is *why* LSM-trees, log-structured storage, and Kafka's append-only log are fast.
- **A cross-continent round trip (~150 ms) is ~300,000× a main-memory reference.** Network calls — especially WAN calls — dominate latency. Minimizing the *number* of sequential network round trips is often the single highest-leverage optimization.
- **Why p99 matters:** former Amazon engineer Greg Linden, in his Stanford "Make Data Useful" talk, reported that Amazon A/B tests "tried delaying the page in increments of 100 milliseconds and found that even very small delays would result in substantial and costly drops in revenue" (~1% of sales per 100 ms). This is why mature systems engineer to *tail* latency (p99/p999), not the mean — the slowest requests disproportionately drive user behavior and revenue.

#### 2.2 The Speed of Light as a Hard Constraint

Light in a vacuum travels at 299,792,458 m/s. In fiber-optic cable, the refractive index of glass (~1.5) slows signals to about **two-thirds of c**. The standard networking reference *High Performance Browser Networking* (Ilya Grigorik) states the rule of thumb directly: "the speed of light in fiber is around 200,000,000 meters per second, which corresponds to a refractive index of ~1.5... we are already within a small constant factor of the maximum speed!" In practice this is roughly **5 microseconds of delay per kilometer** of fiber (vs. 3.33 µs/km in air).

This is a *governor* on every packet — the same reference notes the speed of light "places a hard limit, and a governor, on the propagation time of any network packet." Concretely:
- **New York ↔ London** (~5,500 km): theoretical floor ~37 ms RTT in vacuum, ~55 ms in fiber. The lowest-latency deployed transatlantic cable — Hibernia Express (the 4,600 km cable now branded EXA Express, ready-for-service September 2015) — was built specifically to follow the great-circle path. Per *High Performance Browser Networking*: "The total cost of the project is estimated to be $300M+ and the new route boasts **58.95 ms latency** between the cities... This translates to $60M+ per millisecond saved!" Real engineering is already within a small constant factor of the physical limit.
- **New York ↔ Sydney** (~16,000 km): per the same source, "it nonetheless takes **160 milliseconds** to make the round-trip (RTT) from New York to Sydney," with actual real-world RTT "in the 200–300 millisecond range."

Grace Hopper famously handed out 11.8-inch (~30 cm) wires representing the distance electricity travels in one nanosecond, to make the limit visceral.

**Architectural consequences (all forced by this one fact):**
- **CDNs and edge computing** exist to move data physically closer to users — "a CDN doesn't make data move faster; it makes the data start closer."
- **Geographic data-center placement** and multi-region architectures are latency decisions before they are availability decisions.
- **You cannot have low-latency strong consistency across continents.** Synchronous cross-region replication pays the speed-of-light tax on every write. This is the physical root of the PACELC latency-consistency trade-off (§3.2).
- High-frequency trading firms colocate servers *inside* exchange buildings precisely because every kilometer is ~5 µs they can't get back.

#### 2.3 The Resources Being Traded

Every design decision spends from five fungible-but-not-quite budgets:
- **CPU** — compute cycles (compression, serialization, hashing, business logic).
- **Memory** — fast but volatile and expensive per byte; the scarce resource for caches and in-memory indexes.
- **Disk I/O** — durable, cheap per byte, but slow; random vs. sequential matters enormously.
- **Network bandwidth** — between machines, between datacenters; the cross-AZ/cross-region bytes cost both latency and literal dollars.
- **Money** — the meta-resource. You can usually convert money into more of any other resource, up to the point where the trade-offs (not the budget) bind.

Good architects reason explicitly about which resource is the bottleneck for a given workload and trade the others to relieve it (e.g., spend CPU on compression to save network/disk; spend memory on a cache to save disk I/O and latency).

#### 2.4 Why Distributed Systems Exist At All

A single machine has two fatal limits:
1. **Vertical scaling ceiling.** You can buy a bigger box (more cores, more RAM), but there's a maximum, and the price climbs super-linearly. Eventually the workload exceeds any single machine.
2. **Single point of failure (SPOF).** One machine has one power supply, one set of disks, one location. When (not if) it fails, the whole service is down.

Distribution solves both — horizontal scale-out across commodity machines, and redundancy so no single failure takes down the service — but introduces the entire problem space this document covers: partial failure, network unreliability, coordination, consistency, and the impossibility results. **Distribution is not free; it is a trade of single-machine simplicity for scale and fault-tolerance, paid for in complexity.** Kleppmann's caution applies: *building for scale you don't need is wasted effort and may lock you into an inflexible design.*

---

### 3. The Core Trade-offs and Theorems

#### 3.1 CAP Theorem

**Statement.** Eric Brewer conjectured (2000, PODC) and Seth Gilbert & Nancy Lynch formally proved (2002) that a distributed system with shared data cannot simultaneously guarantee all three of: **Consistency** (every read sees the most recent write — formally *linearizability*/atomic consistency), **Availability** (every request to a non-failing node gets a response), and **Partition tolerance** (the system keeps working despite arbitrary message loss between nodes).

**The intuition (the proof in one paragraph).** Imagine two nodes G1 and G2. The network partitions so they can't communicate. A client writes value v1 to G1. Another client reads from G2. If the system is **available**, G2 must respond — but it hasn't heard about v1, so it returns stale data, violating **consistency**. The only way to stay consistent would be for G2 to refuse to answer until it can reach G1 — violating **availability**. Since partitions *will* happen (cables cut, packets drop), you must choose, *during the partition*, between C and A.

**The crucial nuance (Brewer 2012; Kleppmann).** "P" is not a thing you choose — partitions are a fact of nature you cannot opt out of. So the real choice is only **CP vs. AP**, and only *while partitioned*. When the network is healthy, you can have both C and A. Kleppmann argues CAP is "too simplistic and too widely misunderstood" to classify real systems — which is why PACELC matters more.

#### 3.2 PACELC — The More Useful Theorem

Daniel Abadi (2010 blog post; 2012 paper "Consistency Tradeoffs in Modern Distributed Database System Design") extended CAP to capture what happens the other 99.9% of the time:

> **If there is a Partition (P), trade off Availability vs. Consistency (A/C); Else (E), trade off Latency vs. Consistency (L/C).**

The insight: **partitions are rare, but the latency-consistency trade-off is present on every single request.** Strong consistency requires coordination between replicas, and coordination takes time (a round trip — taxed by the speed of light). As Abadi put it in a February 2024 interview: "Consistency just takes time. Consistency requires coordination. You have to have two different locations communicate with each other to be able to remain consistent with one another."

**Worked classifications:**
- **Dynamo, Cassandra, Riak: PA/EL** — give up consistency for availability under partition, and for low latency otherwise.
- **Fully ACID systems (VoltDB, traditional 2-phase-commit DBs): PC/EC** — preserve consistency in both regimes, paying availability and latency costs.
- **MongoDB: PA/EC**; **PNUTS: PC/EL** (an unusual but instructive position).

For an engineer, PACELC is the better daily tool: it forces you to ask "how much staleness can this feature tolerate, and what latency am I willing to pay to avoid it?"

#### 3.3 The Consistency Spectrum

Consistency is **not** binary — it's a spectrum of guarantees, each weaker (and cheaper/faster/more available) than the last:

| Model | Guarantee | Cost | Typical use |
|---|---|---|---|
| **Linearizable (strong)** | Operations appear instantaneous, in real-time order; behaves like a single copy | Requires coordination/consensus on every op → high latency, availability hit under partition | Distributed locks, leader election, financial ledgers |
| **Sequential** | All nodes see operations in the same order, but not necessarily real-time order | Slightly cheaper than linearizable | — |
| **Causal** | Operations that are causally related are seen in the same order by all; concurrent ops may differ | Much cheaper; available under partition | Comment threads, collaborative apps |
| **Read-your-writes / monotonic reads (session guarantees)** | Per-client guarantees (you see your own writes; you never go "backward" in time) | Cheap; per-client only | User-facing apps after a write |
| **Eventual** | Replicas converge *eventually* if writes stop; no ordering promise | Cheapest, most available, lowest latency | Likes counts, DNS, shopping carts, recommendations |

**First principle:** stronger consistency = more coordination = more latency and less availability. Choose the *weakest* model that still makes your feature correct. E-commerce uses strong consistency for inventory/payment but eventual for reviews and recommendations.

#### 3.4 Latency vs. Throughput

- **Latency** = time for one operation (how long a single request takes).
- **Throughput** = operations per unit time (how many requests/sec the system handles).

They are distinct and often in tension. **Batching** improves throughput (amortize fixed costs over many items) but *increases* latency for any individual item (it waits for the batch). Pipelining, buffering, and queueing similarly trade latency for throughput. A system tuned for max throughput (e.g., a batch analytics job) looks nothing like one tuned for min latency (e.g., an ad-serving request path). Always know which one your SLO is written against — and measure **tail latency (p99/p999)**, not the mean, because the slowest requests dominate user experience (the Amazon 100 ms ≈ 1% sales finding above is why).

#### 3.5 Why Consensus Is Hard (FLP)

The **FLP impossibility result** (Fischer, Lynch, Paterson, 1985) proves that in a fully asynchronous system (no bounded message delays, no synchronized clocks) where even **one** process may crash, **no deterministic algorithm can guarantee consensus will terminate.**

**The intuition:** without bounded timing, a node cannot distinguish a *crashed* peer from a merely *slow* one. An adversarial scheduler can always delay the one message that would break a tie, keeping the system in an undecided ("bivalent") state forever.

**Why this matters and how we escape it.** FLP doesn't say consensus is *impossible* — it says it can't be guaranteed to *always* terminate. Real protocols **relax an assumption**: they assume **partial synchrony** (a "Global Stabilization Time" after which message delays are bounded). Under that assumption:
- **Paxos** (Lamport) and **Raft** (designed for understandability) guarantee **safety** (never return a wrong answer) always, and **liveness** (make progress) only during periods of network stability.
- Raft uses **randomized election timeouts** to avoid split-vote livelock — which, per FLP, it cannot *theoretically* guarantee will terminate, but which works with overwhelming probability in practice.

This is the deep reason consensus is the expensive primitive at the heart of every strongly-consistent system (leader election, distributed locks, config management like ZooKeeper/etcd). Use it sparingly, on the smallest possible amount of critical state.

#### 3.6 The Trade-off Cheat Sheet

| Trade-off | Pole A | Pole B | The tension |
|---|---|---|---|
| Normalization | Normalized (no duplication, write-cheap, read-expensive joins) | Denormalized (duplicated, read-cheap, write-expensive, risk of anomalies) | Read vs. write cost; consistency vs. speed |
| Consistency | Strong | Eventual | Correctness/simplicity vs. latency/availability |
| Latency | Low latency | High throughput | Per-op speed vs. aggregate volume |
| Storage engine | Read-optimized (B-tree) | Write-optimized (LSM) | Read amplification vs. write amplification |
| Flexibility | Simple/rigid | Flexible/complex | YAGNI vs. future-proofing |
| Coupling | Decoupled (async, resilient, eventually consistent) | Coupled (sync, simple, immediately consistent) | Resilience/scale vs. simplicity |

---

### 4. The Key Non-Functional Requirements (System Qualities)

For each: definition, why it matters, and the **levers** you pull.

- **Scalability** — the ability to handle growth in load (users, data, traffic) with proportional (not explosive) cost. *Levers:* horizontal scale-out, statelessness, sharding, caching, async processing, load balancing. First describe load *quantitatively* (e.g., requests/sec, read:write ratio) before claiming something "scales."
- **Availability** — the fraction of time the system is able to respond. Measured in "nines":

| Availability | Downtime/year | Downtime/month | Typical use |
|---|---|---|---|
| 99% (two nines) | 3.65 days | 7.2 hrs | Internal tools |
| 99.9% (three nines) | 8.76 hrs | 43.8 min | Most SaaS |
| 99.99% (four nines) | 52.6 min | 4.38 min | Critical infra, payments |
| 99.999% (five nines) | 5.26 min | 26 sec | Telecom, life-safety |

  *Key facts:* each nine costs ~10× more effort. **Composite availability of dependent services multiplies** — three services each at 99.9% in series ≈ 99.7%. *Levers:* redundancy, failover, multi-AZ/region, removing SPOFs, health checks, graceful degradation.
- **Reliability** — "continues to work *correctly* even when things go wrong." Kleppmann's distinction: a **fault** is one component deviating from spec; a **failure** is the system as a whole stopping service. The goal is to be **fault-tolerant** — prevent faults from becoming failures. *Levers:* redundancy, retries with idempotency, circuit breakers, bulkheads, chaos testing.
- **Consistency** — see §3.3. *Levers:* the consistency model you choose, quorum settings (W+R>N), synchronous vs. async replication.
- **Durability** — once acknowledged, data is never lost. *Levers:* replication factor (≥3 is common), write-ahead logs, fsync policies, geographic backup, erasure coding.
- **Maintainability** — "making life better for the engineering teams who work with the system." Three facets (Kleppmann): **operability** (easy to run/monitor), **simplicity** (easy to understand — manage complexity with good abstractions), **evolvability** (easy to change). *Levers:* modularity, good abstractions, tests, documentation, observability, automation.
- **Latency/Performance** — see §3.4. *Levers:* caching, CDNs, geographic placement, indexing, async, fewer network hops, connection pooling.
- **Cost** — the budget constraint that bounds everything. *Levers:* right-sizing, spot/reserved instances, tiered storage (hot/warm/cold), caching to cut compute, picking the cheapest sufficient consistency/availability target.

---

### 5. The Fundamental Building Blocks

For each pattern: **what it is**, **the first-principles reason it exists**, **the cost**, and **when to use it.**

#### 5.1 Load Balancing
- **What:** distributes incoming requests across multiple backend servers.
- **Why it exists:** a single server has finite capacity and is a SPOF. To scale horizontally you need many servers; something must spread traffic and route around dead ones.
- **L4 vs. L7:** **Layer 4** balances on TCP/IP (IP/port), doesn't inspect content — fast, low overhead, protocol-agnostic. **Layer 7** terminates the connection and inspects the application protocol (HTTP) — enables content-based routing (`/api/*` → API servers), SSL termination, header manipulation, per-user rate limiting, at higher CPU cost. (A Layer 7 load balancer is effectively a specialized reverse proxy.)
- **Algorithms:** round-robin, least-connections, IP-hash (sticky sessions), weighted.
- **Cost:** another component to operate and make HA itself; L7 adds latency/CPU; sticky sessions undermine statelessness.
- **When:** essentially always, the moment you have more than one server.

#### 5.2 Caching
- **What:** store the result of an expensive operation in a faster/closer tier so future requests skip the work.
- **Why it exists — fundamentally about locality.** The latency hierarchy (§2.1) means memory is 100–1000× faster than disk and far faster than recomputation or a DB query. Caching exploits **temporal locality** (recently accessed data is likely accessed again) and **spatial/geographic locality** (data near the user is faster to serve).
- **Strategies:**
  - **Cache-aside (lazy loading):** app checks cache, on miss reads DB and populates cache. Most common; simple; resilient to cache failure; but first request is always a miss and data can go stale.
  - **Read-through:** cache itself loads from DB on miss (transparent to app).
  - **Write-through:** writes go to cache and DB synchronously — strong freshness, higher write latency.
  - **Write-back (write-behind):** write to cache, flush to DB asynchronously — fast writes, risk of data loss if cache dies.
- **Cache invalidation** — "one of the two hard problems." Approaches: **TTL** (simplest, time-based expiry; tolerates bounded staleness), explicit invalidation on write, versioning. Selection rule: if predictability/simplicity matter most, start with cache-aside + TTL; if strict read-after-write is needed, write-through; if the write path is the bottleneck and deferred consistency is OK, write-back.
- **Cost:** **staleness** (the core price — cache and source can diverge), added complexity, cache-coherence problems across nodes/regions, cold-start misses, and a new failure mode (thundering herd on cache miss).
- **When:** read-heavy workloads with reuse; expensive computations/queries; reducing DB load. *Not* for write-heavy data rarely re-read.

#### 5.3 Replication
- **What:** keep copies of the same data on multiple nodes.
- **Why it exists:** (1) **availability/durability** — survive node loss; (2) **read throughput** — serve reads from many replicas; (3) **latency** — place a replica near the user.
- **Three models (Kleppmann):**
  - **Single-leader (leader-follower):** all writes go to the leader, which streams a replication log to followers; reads can hit any replica. Simple, no write conflicts. Used by PostgreSQL, MySQL, MongoDB, Kafka. *Cost:* leader is a write bottleneck and failover is tricky; **async replication** risks losing recent writes if the leader dies, **sync** adds latency.
  - **Multi-leader:** several leaders accept writes (e.g., one per region). Better write availability/latency across regions. *Cost:* **write conflicts** must be resolved (last-write-wins, CRDTs, app logic) — genuinely hard.
  - **Leaderless (Dynamo-style):** any replica takes writes; clients read/write to many nodes using **quorums**. Used by Cassandra, Riak, DynamoDB. *Cost:* eventual consistency, read-repair/anti-entropy needed, application must sometimes handle conflicts.
- **Replication lag** creates the anomalies that session guarantees (read-your-writes, monotonic reads, consistent-prefix) are designed to paper over.
- **When:** virtually all production systems replicate for durability and availability; the *model* depends on your write pattern and geography.

#### 5.4 Partitioning / Sharding
- **What:** split a dataset into pieces (shards/partitions) spread across nodes, each holding a subset.
- **Why it exists:** when data or write volume exceeds a single machine, replication alone doesn't help (every node still holds everything). Sharding is the *only* way to scale **writes** and **storage** horizontally.
- **Strategies:**
  - **Range partitioning:** contiguous key ranges per shard. Great for range scans; **but creates hot spots** (e.g., all "today's" data on one node; sequential keys overload one shard).
  - **Hash partitioning:** hash the key to assign a shard. Distributes load evenly, kills hot spots; **but destroys range-query efficiency** and naive `hash(key) mod N` reshuffles almost everything when N changes.
  - **Consistent hashing:** arranges the hash space as a ring so adding/removing a node only moves a small fraction of keys. The standard solution for elastic, dynamic clusters (Cassandra, DynamoDB use it, often with virtual nodes).
- **Cost:** cross-shard queries and transactions become expensive/complex; rebalancing is operationally hard; **hot keys** (a celebrity user, a viral item) can overload one shard regardless of strategy; you must choose a good partition key.
- **When:** dataset or write throughput exceeds one machine's capacity. (Combine with replication: each shard is itself replicated.)

#### 5.5 Database Choices and Underlying Data Structures
- **SQL (relational):** strong schema, ACID transactions, joins, mature. Best when relationships and strong consistency matter (payments, inventory, core business data). Scales vertically easily, horizontally with more effort.
- **NoSQL:** umbrella for key-value, document, column-family, graph. Trades joins/strong-schema/ACID for horizontal scalability, flexible schema, and (often) higher write throughput. Best for huge scale, simple access patterns, or flexible/evolving data.
- **The 2026 framing:** the real question isn't "SQL vs. NoSQL" but "**which specific engine and why for this workload**" — driven by access patterns, consistency needs, and scale.
- **The data-structure level — B-tree vs. LSM-tree (the read/write trade-off made physical):**
  - **B-trees** (InnoDB/MySQL, PostgreSQL): update data **in place** in fixed-size pages; a key lives in exactly one place. → **Predictable, low read latency** (3–4 page reads to any key); **read-optimized**. *Cost:* random writes, write amplification from page splits + write-ahead log, fragmentation.
  - **LSM-trees** (RocksDB, Cassandra, LevelDB, ClickHouse): buffer writes in an in-memory memtable, flush as **immutable sorted files (SSTables)** via **sequential** writes, merge via background **compaction**. → **High write throughput** (sequential I/O); **write-optimized**. *Cost:* **read amplification** (a key may be in several SSTables; mitigated by Bloom filters), and unpredictable latency spikes during compaction.
  - **Rule of thumb:** read-heavy → B-tree; write-heavy → LSM. (Modern hardware with transparent compression narrows the gap; USENIX FAST'22 research showed a tuned B-tree can match or beat RocksDB's write amplification on compressing storage.)
- **Indexing:** an index trades **write speed and storage** for **read speed** — every index must be updated on write. This is the same read-vs-write trade-off again. Index the queries you actually run; don't over-index write-heavy tables.

#### 5.6 Message Queues / Event Streaming / Async Processing
- **What:** a broker (RabbitMQ/SQS for queues, Kafka for log-based streaming) sits between producers and consumers.
- **Why it exists — decoupling.** Synchronous request/response couples services in **time** (both must be up simultaneously), **load** (a spike hits the downstream directly), and **failure** (downstream failure propagates upstream). A queue breaks all three:
  - **Temporal decoupling:** producer doesn't wait for consumer; both can be down independently.
  - **Load leveling / buffering:** absorb traffic spikes; consumers process at their own pace.
  - **Resilience:** retries, dead-letter queues, guaranteed delivery for critical operations (e.g., payments processed exactly/at-least once).
  - **Fan-out:** one event, many independent consumers (Kafka's log lets each consumer track its own offset and **replay**).
- **Backpressure:** the mechanism that slows producers when consumers can't keep up. Kafka handles it via **consumer lag** (the offset gap) — producers never slow, but lag can grow silently until retention expires and data is lost. Bounded buffers/queues provide explicit backpressure.
- **Cost:** eventual consistency, harder debugging (no linear call stack), ordering guarantees only within a partition, operational complexity, the need for idempotent consumers. Use intentionally, not because it's trendy.
- **When:** work that can be done asynchronously (emails, notifications, video transcoding, analytics), smoothing spikes, decoupling services, event-driven architectures.

#### 5.7 Rate Limiting
- **What:** cap how many requests a client may make in a time window.
- **Why it exists:** finite capacity must be allocated fairly; protect against abuse, DoS, brute-force, and a single client overwhelming infrastructure; control cost.
- **Algorithms:**
  - **Token bucket:** tokens refill at a fixed rate up to a capacity; each request spends one. **Allows controlled bursts** while enforcing a long-term average. The strongest general-purpose default for APIs.
  - **Leaky bucket:** requests enter a queue draining at a constant rate; enforces a **smooth** output rate, no bursts. Good for shaping downstream traffic.
  - **Fixed window:** count per discrete window — simplest, but allows **2× bursts at window boundaries**.
  - **Sliding window (log/counter):** moving window — most accurate, smoother; sliding-window-counter is the practical compromise at scale.
- **Cost:** state to track (often in Redis); distributed rate limiting adds coordination; risk of throttling legitimate users.
- **When:** any public API, login endpoints, expensive operations, multi-tenant fairness.

#### 5.8 CDNs (Content Delivery Networks)
- **What:** geographically distributed edge servers that cache content close to users.
- **Why it exists:** the **speed-of-light constraint** (§2.2). You cannot make a packet cross an ocean faster; you *can* serve it from an edge node 50 km away instead of an origin 8,000 km away. "A CDN doesn't make data move faster; it makes the data start closer."
- **Cost:** invalidation/staleness for dynamic content, cost, complexity; best for static/cacheable assets.
- **When:** static content (images, video, CSS, JS), globally distributed users, offloading origin. (Netflix Open Connect, etc.)

#### 5.9 Proxies (Forward and Reverse)
- **Forward proxy:** sits between *clients* and the internet; acts on the client's behalf (filtering, caching, anonymity, corporate egress control).
- **Reverse proxy:** sits in front of *servers*; acts on the servers' behalf. Provides SSL termination, caching of static assets, compression, URL rewriting, security (hides backend topology), and is the basis of L7 load balancing.
- **Why they exist:** a level of indirection that centralizes cross-cutting concerns (security, caching, routing) at a chokepoint instead of scattering them across every server.
- **Cost:** an extra hop/component; must be made HA.

#### 5.10 Consistency Mechanisms: Quorums, 2PC, Sagas
- **Quorums (W + R > N):** in leaderless replication, require writes to W nodes and reads from R nodes such that W+R > N guarantees read/write sets overlap → a read sees the latest write. Tunable: raise W for write-consistency, R for read-consistency. *Cost:* higher latency; "sloppy quorums" trade consistency for availability. (Note: quorum reads still don't *guarantee* latest value in all edge cases — eventual-consistency stores monitor replication lag.)
- **Two-Phase Commit (2PC):** a coordinator asks all participants to **prepare**, then **commit** only if all agree. Provides **atomic, strongly-consistent** distributed transactions. *Cost:* **blocking** — participants hold locks until the coordinator decides; if the coordinator crashes, locks can be held indefinitely; reduced availability (any participant down blocks everything); poor fit for microservices. Use only within a single trust domain needing strict atomicity.
- **Sagas:** break one distributed transaction into a **sequence of local transactions**, each with a **compensating transaction** that undoes it if a later step fails. No global locks. Two styles: **choreography** (services react to each other's events — simple to start, hard to debug past ~5 steps) and **orchestration** (a central coordinator drives steps — easier to debug/monitor, but the orchestrator is a SPOF to make HA). First described by Garcia-Molina & Salem (1987); revived by microservices. *Cost:* **eventual consistency** (a window where state is partially applied), no isolation (concurrent sagas can see intermediate states), and you must design + test every compensating path. *When:* multi-step business workflows across services that own their own databases, where you can tolerate eventual consistency (most modern fintech/e-commerce checkout flows). The trade is **strong consistency for availability and scalability.**

---

### 6. Key Scaling Concepts

- **Vertical scaling (scale up):** bigger machine. *Pro:* trivial, no code changes (often "turn a knob" in the cloud), no distribution complexity. *Con:* hard ceiling, super-linear cost, still a SPOF.
- **Horizontal scaling (scale out):** more machines. *Pro:* near-unlimited, commodity hardware, redundancy built in. *Con:* requires the software to support distribution; introduces all the consistency/coordination problems.
- **Stateless vs. stateful services — the master key to horizontal scaling.** A **stateless** service keeps no client/session state between requests (any instance can serve any request), so you can add/remove instances freely behind a load balancer — trivially scalable. A **stateful** service (holds session, in-memory data) is far harder to scale and move. **The dominant pattern:** push state out of the compute tier into dedicated stateful stores (databases, Redis, object storage) and keep application servers stateless. Then scaling the app tier is just "add more identical boxes."
- **Back-of-the-envelope estimation / capacity planning.** Per Jeff Dean: "back-of-the-envelope calculations are estimates you create using a combination of thought experiments and common performance numbers to get a good feel for which designs will meet your requirements." The method:
  1. **Start from users → actions → QPS.** e.g., 500M daily active users × 10 posts = 5B posts/day. 5B ÷ 86,400 s ≈ **~58K write QPS**.
  2. **Apply a read:write ratio** (often 10:1 to 100:1 for social) → read QPS.
  3. **Peak factor:** multiply average by 2–10× for peak; apply a safety factor (1.5–2× for critical systems).
  4. **Storage:** records/day × bytes/record × retention × replication factor. (e.g., 5B × 300 bytes text ≈ 1.5 TB/day text; images dominate.)
  5. **Bandwidth:** QPS × payload size.
  6. **Servers/cache:** total QPS ÷ per-server QPS; cache the hot 20% (80/20 rule).
  - **Anchors to sanity-check against:** ~86,400 seconds/day; round-trip in-DC ~0.5 ms; cross-continent ~150 ms; a commodity server handles thousands of QPS. Always ask "does this number even make sense?" — a single server cannot serve Instagram's traffic.

---

### 7. A Repeatable Method for Any System Design Problem

This is the durable mental model — the same 6–7 steps strong architects (and interviewers) use. Treat it as a loop, not a strict line; revisit earlier steps as you learn.

**Step 1 — Clarify requirements (the "what," not the "how").** Treat the system as a black box. Separate **functional** requirements (core features, use cases — the *nouns and verbs*: users, posts, "follow," "view timeline") from **non-functional** (scale, latency SLOs, consistency needs, availability target). Explicitly scope **in vs. out**. *The #1 failure mode is designing for the wrong problem* — never skip this. Identify **access patterns** early; they dictate the data model more than anything else.

**Step 2 — Estimate scale (back-of-the-envelope).** QPS (read & write), storage, bandwidth, peak vs. average, growth. This is where senior thinking shows: you anchor every later decision in numbers ("we need ~3M read QPS, so a single DB won't do — we need caching + sharding"). The numbers don't need to be exact; orders of magnitude drive architecture.

**Step 3 — Define the API and data model.** Specify a few **core** endpoints (inputs/outputs) — this nails the contract and often surfaces missed requirements. Identify **core entities** (the nouns from Step 1 → tables/collections). Decide the data model and storage type *driven by access patterns and scale estimates*, not habit.

**Step 4 — High-level design.** Sketch the major components and data flow: clients → load balancer → app servers → caches → databases → queues → object storage → CDN. Trace each core use case end-to-end (e.g., follow a write from client to storage and a read back out). Confirm it satisfies the functional requirements. Don't deep-dive yet.

**Step 5 — Identify the bottleneck / critical constraint.** *This is the heart of architectural skill.* Every system has **one** binding constraint that dominates — find it before optimizing anything else:
- Read-heavy with reuse? → the **database read path** is the bottleneck → cache, replicas.
- Write-heavy / huge data? → **write throughput / storage** → shard, LSM store, async.
- Globally distributed users? → **latency / speed of light** → CDN, edge, regional replicas.
- Strong consistency across regions? → **coordination cost** → reconsider consistency model or accept the latency.
- Spiky traffic? → **load absorption** → queues, autoscaling, rate limiting.

Ask: "What breaks first at 10×? At 100×? Where is the single point of failure?" Spend your complexity budget *only* on the binding constraint.

**Step 6 — Deep-dive and resolve trade-offs.** For the 1–2 critical components, go deep: partitioning scheme, replication model, consistency level, cache strategy, failure/recovery behavior. **Always state the trade-off explicitly** — every choice (Cassandra vs. Postgres, eventual vs. strong, sync vs. async) has costs; naming them is what separates senior from junior reasoning. Address SPOFs, failure modes ("how do we know at 3 AM something's broken?" → observability), and cost.

**Step 7 — Recap against requirements.** Tie the design back to Step 1: "this supports the required X QPS, provides HA via multi-AZ with no SPOF, and accepts eventual consistency on Y because the product tolerates it."

**The unifying meta-principle:** *let scale and requirements drive the design, never the reverse.* Don't add Kafka, microservices, or global sharding because they're impressive — add them because a quantified constraint demands them. Matching complexity to actual requirements is the mark of judgment.

---

### 8. How to Ingest and Learn System Design Effectively

The highest-leverage approach, synthesized from how strong practitioners actually build durable understanding:

1. **Learn the primitives, not the patterns.** If you understand the latency hierarchy, the speed-of-light floor, the CAP/PACELC/FLP impossibilities, and the read-vs-write trade-off, you can *derive* caching, CDNs, sharding, and consensus from first principles instead of memorizing them. Patterns are consequences; primitives are causes. This is the single highest-leverage investment.
2. **Read *Designing Data-Intensive Applications* (Kleppmann) cover to cover.** It is the canonical reference precisely because it explains the *why* at the primitive level (storage engines, replication, partitioning, transactions, consistency/consensus). Re-read Part II (distributed data) annually.
3. **Study real architectures and post-mortems.** Engineering blogs (High Scalability, the AWS Architecture blog, company eng blogs from Netflix, Uber, Discord, etc.) and incident post-mortems teach how principles play out under real load and real failure. The Twitter timeline fan-out problem (write-time vs. read-time, then a hybrid) is a perfect compact case study from DDIA.
4. **Read the cloud well-architected frameworks** (AWS/GCP/Azure) for the operational dimensions — reliability, cost, operational excellence — that books underweight.
5. **Deliberate practice with feedback.** Design systems end-to-end (even on paper), then compare against how the real system did it and *why it differs*. Force yourself to state trade-offs aloud. Do mock designs; the feedback loop (you'll overrun time/scope at first, then tighten) is where skill compounds.
6. **Build and break things.** Run a sharded database, set up a Kafka cluster, induce a partition, watch replication lag. Visceral experience of failure modes beats any reading.
7. **Internalize the numbers.** Memorize the latency table and the availability-nines table. Estimation fluency turns vague hand-waving into grounded decisions.

---

### 9. Principles of Good Architecture (Broadly)

These transcend distributed systems and apply to any non-trivial software:

- **Low coupling, high cohesion.** *Coupling* = interdependence between modules (minimize it). *Cohesion* = how focused a single module's responsibilities are (maximize it). Loose coupling means a change or failure in one module doesn't cascade; high cohesion means each module does one thing well. The shared-database-as-integration-hub anti-pattern is the classic coupling trap — integrate via APIs/events instead. (SRP, ISP, DIP all serve this.)
- **Abstraction & separation of concerns.** Each layer hides the complexity below it behind a clean interface (Kleppmann: data models layered on data models). Lets teams work independently and manages complexity — the real enemy of maintainability.
- **Design for failure — "everything fails, all the time."** Assume every component *will* crash. Build in redundancy, graceful degradation (serve a partial/stale result rather than nothing), retries, timeouts, circuit breakers, and bulkheads. The goal is to prevent **faults** from becoming **failures**.
- **Idempotency.** An operation that can be applied many times with the same effect as once. Essential because in distributed systems messages get retried and duplicated — idempotency (via idempotency keys, dedup) is what makes "at-least-once delivery" safe. Without it, retries corrupt data.
- **Design for observability.** You cannot operate what you cannot see. Build in **logs, metrics, and traces** from the start (not after an incident). "How would the team know something is broken at 3 AM?" should have a concrete answer. Add observability *before* refactoring.
- **Evolutionary architecture.** Requirements and scale change; design for change rather than a fixed end-state. Good abstractions and loose coupling make the system *evolvable*. Avoid decisions that lock you into an inflexible design.
- **Avoid premature optimization & YAGNI ("You Aren't Gonna Need It").** "Building for scale that you don't need is wasted effort and may lock you into an inflexible design" (Kleppmann). Optimize the *measured* bottleneck, not the imagined one. Match complexity to current, quantified requirements.
- **Simplicity (KISS).** Complexity is the primary enemy of maintainability and reliability. The best design is the simplest one that meets the requirements. Every added component (queue, cache, shard, service) is operational burden and a new failure mode — justify each one against a real constraint.

---

## Recommendations

**For the engineer becoming an architect, staged next steps:**

1. **Now (weeks 1–4): Build the foundation.** Memorize the two reference tables (latency numbers §2.1, availability nines §4). Read DDIA Part I + the start of Part II. Internalize CAP→PACELC→FLP and the read-vs-write trade-off until you can explain each from first principles to a peer. *Benchmark to advance:* you can derive *why* a CDN, a cache, and a shard exist without looking them up.
2. **Next (weeks 5–12): Practice the method.** Apply the 7-step method (§7) to 8–10 classic designs (URL shortener, news feed, chat, rate limiter, ride-share). For each, do the back-of-envelope math first and force yourself to name the **one** binding constraint and the explicit trade-offs. Cross-check against real engineering blogs. *Benchmark:* you instinctively start with requirements + numbers, not boxes.
3. **On the job (ongoing): Apply and observe.** On your real systems, identify the binding constraint before proposing changes. Add observability before optimizing. Run a post-mortem mindset on incidents. Push state out of compute to make services stateless. *Benchmark:* your design reviews lead with "here's the constraint and the trade-off," and you can defend why you did *not* add complexity.

**Thresholds that should change your decisions:**
- **Reach for sharding only when** data or write QPS provably exceeds a single (replicated) machine — not before. Until then, vertical scaling + read replicas + caching is simpler and usually sufficient.
- **Reach for strong consistency only when** correctness genuinely requires it (money, inventory, locks). Default to the weakest consistency that keeps the feature correct.
- **Reach for microservices / queues only when** team scaling, independent deployment, or load decoupling demands it — the operational cost is real.
- **Move from 99.9% → 99.99%** only when the business value of that ~8 hours/year of recovered downtime justifies the ~10× cost (multi-region, automated failover).

---

## Caveats

- **The classic latency numbers (§2.1) are ~2010-era** (Jeff Dean/Norvig). Their *absolute* values have improved (notably SSD/NVMe and 10/100 Gbps networking), but the **ratios and orders of magnitude** — which is what you reason with — remain accurate. Treat them as a relative ladder, not precise measurements.
- **Speed-of-light figures depend on assumptions** (exact distance, fiber refractive index 1.4–1.6). Figures like "NY–London ~55 ms fiber floor / 58.95 ms best real cable" are internally consistent given stated inputs but vary by route. The principle (a hard, unbeatable floor within a small constant of c) is what matters.
- **CAP is widely oversimplified.** "Pick two of three" is misleading: partition tolerance isn't optional, the trade-off is only CP-vs-AP and only *during* a partition, and "consistency" in CAP specifically means linearizability. Prefer PACELC for real reasoning, as Kleppmann and Abadi both argue.
- **Storage-engine comparisons are workload-dependent.** "B-tree = read-optimized, LSM = write-optimized" is a strong heuristic, but actual write/read/space amplification depends heavily on configuration, hardware (modern compression narrows the gap), and access pattern. Benchmark with *your* data before committing.
- **Many secondary sources here are practitioner blogs and interview-prep sites.** They faithfully transmit the canonical concepts (CAP, PACELC, DDIA, Dean's numbers), but for authoritative depth go to the primary sources: Kleppmann's *Designing Data-Intensive Applications*, the Gilbert–Lynch CAP proof (2002), Abadi's PACELC paper (2012), the FLP paper (1985), the Raft paper (Ongaro & Ousterhout), Grigorik's *High Performance Browser Networking*, and the cloud well-architected frameworks.
- **This is a conceptual framework, not a cookbook.** It deliberately teaches *why* over *how*. Specific technology choices (which database, which queue, which cloud) change yearly and must be re-evaluated against your concrete, quantified requirements — which is exactly the discipline this framework is meant to instill.
