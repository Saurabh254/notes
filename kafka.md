# Kafka: A Plain Language Guide for System Design

Everything you need to reason about Kafka in a design discussion. Simple words, real numbers, clear tradeoffs.

## 1. What is Kafka?

Kafka is a distributed event streaming platform.

The best mental model: **a huge, durable, append only log that many programs can write to and read from, each at their own pace.**

It is not a normal queue. In a queue, a message disappears after someone consumes it. In Kafka, messages stay for a set time (or forever). Readers just remember their position in the log. That single difference gives you:

* **Replay.** Rewind and reprocess old data.
* **Many independent readers.** Billing, analytics, and search can all read the same events.
* **Huge throughput.** Sequential disk writes are cheap.
* **Decoupling.** Producers do not know or care who reads.

Kafka started at LinkedIn and is now an Apache project.

### What it is great at

* Moving large volumes of events between services
* Event sourcing and audit trails
* Change data capture (CDC) from databases
* Log and metrics pipelines
* Stream processing
* Buffering traffic spikes between systems

### What it is not great at

* Per message routing, priorities, or delayed delivery (use RabbitMQ or SQS)
* Request and response style calls (use HTTP or gRPC)
* Ad hoc queries on data (use a database)
* Tiny setups where a simple queue is enough
* Microsecond latency (Kafka gives low milliseconds)

## 2. Core Building Blocks

| Term | Plain meaning |
|---|---|
| **Record (event)** | One message. Has a key, value, timestamp, and optional headers. |
| **Topic** | A named category of records. Like a table or a folder. Example: `orders`. |
| **Partition** | A topic is split into partitions. Each partition is one ordered log. |
| **Offset** | The position number of a record inside a partition. 0, 1, 2, 3... |
| **Broker** | A Kafka server. Stores partitions and serves clients. |
| **Cluster** | A group of brokers working together. |
| **Producer** | A client that writes records to topics. |
| **Consumer** | A client that reads records from topics. |
| **Consumer group** | A team of consumers that share the work of reading a topic. |
| **Controller** | The broker (or quorum) that manages cluster metadata and leader elections. |

A picture of one topic with three partitions:

```
Topic: orders

P0: [0][1][2][3][4]       → new writes go to the end
P1: [0][1][2][3]
P2: [0][1][2][3][4][5]
```

Each partition is its own log with its own offsets. Offset 3 in P0 and offset 3 in P1 are unrelated records.

## 3. How a Write and a Read Actually Flow

**Write path**

1. Producer builds a record and picks a partition (usually by hashing the key).
2. Producer asks the cluster who the **leader** broker is for that partition.
3. Producer sends the record (in a batch) to the leader.
4. Leader appends it to the end of its log on disk.
5. Followers copy it from the leader.
6. Once enough replicas have it, the leader sends an ack back to the producer.

**Read path**

1. Consumer joins a group and gets assigned partitions.
2. Consumer asks the leader of each partition: "give me records from offset N."
3. Broker returns a batch.
4. Consumer processes it and **commits** the new offset, so it knows where to resume.

Consumers **pull**. Brokers never push. This lets each consumer control its own speed and avoids overwhelming slow readers.

## 4. Why Kafka Is So Fast

Kafka looks slow on paper (it uses disk) but is very fast in practice. Reasons:

* **Sequential I/O.** Appending to the end of a file is fast, even on spinning disks. Random writes are what hurt.
* **OS page cache.** Kafka leans on the operating system cache instead of its own heap. Recent data is usually served from memory.
* **Zero copy.** Data goes from disk cache straight to the network socket without being copied through the application.
* **Batching.** Producers and consumers move many records per request.
* **Compression.** Whole batches are compressed (lz4, zstd, snappy, gzip), saving network and disk.
* **Partition parallelism.** Different partitions live on different brokers and disks, so work spreads out.

## 5. Partitions: The Heart of Kafka

If you understand partitions, you understand Kafka design. A partition is three things at once:

1. **The unit of ordering.** Order is guaranteed only inside one partition.
2. **The unit of parallelism.** One partition is read by at most one consumer per group.
3. **The unit of storage and replication.** A partition lives on a broker, and is copied as a whole.

### How a record picks its partition

* **With a key:** `partition = hash(key) % number_of_partitions`. Same key always goes to the same partition. This is how you get per entity ordering (all events for `user_42` stay in order).
* **Without a key:** records are spread across partitions in batches for even load.
* **Custom partitioner:** you can write your own logic.

### Choosing a good key

A good key has:

* **High cardinality.** Many distinct values, so load spreads evenly.
* **The ordering boundary you need.** If events for one order must be in order, use `order_id`.

Bad key example: `country`. If 60% of traffic is one country, one partition gets 60% of the load. That is a **hot partition**, and it caps your throughput and creates lag.

### How many partitions?

More partitions give more parallelism and throughput. But they also cost:

* More open files and memory on brokers
* Longer leader election and recovery after a broker failure
* More replication traffic
* More work during consumer rebalances

Practical advice:

* Estimate from throughput needs (see the capacity section).
* Plan for 2x growth, not 100x.
* Keep partitions per broker in the low thousands.

### The big trap: changing partition count

You can **add** partitions but never remove them. And when you add them, `hash(key) % N` changes. Old keys may now map to new partitions. Ordering for existing keys breaks across the change. So pick the count carefully at the start, or create a new topic and migrate.

## 6. Replication and Durability

Each partition is copied to several brokers.

* **Replication factor (RF):** how many copies exist. Production standard is **3**.
* **Leader:** one replica handles all reads and writes for the partition.
* **Followers:** the other replicas. They pull data from the leader.
* **ISR (in sync replicas):** the set of replicas that are fully caught up. Includes the leader.

A layout with RF=3 across three brokers:

```
          Broker 1      Broker 2      Broker 3
P0        Leader        Follower      Follower
P1        Follower      Leader        Follower
P2        Follower      Follower      Leader
```

Leaders are spread out so no single broker takes all the traffic.

### Acknowledgements (`acks`)

| Setting | Meaning | Risk |
|---|---|---|
| `acks=0` | Do not wait for any ack | Can lose data silently |
| `acks=1` | Leader wrote it | Lost if leader dies before followers copy |
| `acks=all` | All ISR members have it | Safest. Slightly higher latency |

### The safe production recipe

```
replication.factor = 3
min.insync.replicas = 2
acks = all
unclean.leader.election.enable = false
```

What this gives you:

* A write succeeds only if at least 2 replicas have it.
* You can lose **one** broker with no data loss and no downtime for writes.
* If 2 brokers die, writes are rejected rather than risking data loss. That is a deliberate choice: consistency over availability.

### High watermark

Consumers can only read records that every ISR replica has copied. This point is the **high watermark**. It guarantees you never read data that could vanish after a leader failure.

### Leader failure

If a leader dies, the controller picks a new leader from the ISR. Clients refresh metadata and continue. This usually takes seconds.

### ZooKeeper is gone

Old Kafka needed ZooKeeper for metadata. Modern Kafka (4.0 and later) uses **KRaft**, a built in Raft based quorum. Fewer moving parts, faster recovery, and support for far more partitions.

## 7. Producers in Detail

Key settings that matter:

| Setting | What it does |
|---|---|
| `acks` | Durability level (see above) |
| `enable.idempotence` | Prevents duplicates from retries. On by default in modern clients. |
| `linger.ms` | How long to wait to fill a batch. Higher means better throughput, more latency. |
| `batch.size` | Max bytes per batch per partition |
| `compression.type` | lz4 or zstd are good defaults |
| `retries` and `delivery.timeout.ms` | How long to keep trying before failing |
| `max.in.flight.requests.per.connection` | Parallel requests. Up to 5 is safe with idempotence. |

**Idempotent producer.** Each producer gets an ID and each record a sequence number. If a retry sends the same record twice, the broker spots it and drops the duplicate. No reordering, no duplicates, per partition.

A small example in Python:

```python
from confluent_kafka import Producer

p = Producer({
    "bootstrap.servers": "broker1:9092",
    "enable.idempotence": True,
    "acks": "all",
    "compression.type": "lz4",
    "linger.ms": 10,
})

p.produce("orders", key="user_42", value=b'{"order_id": 1}')
p.flush()
```

## 8. Consumers and Consumer Groups

### The one rule to memorize

> Within a consumer group, each partition is read by **at most one** consumer.

So:

* 4 partitions and 2 consumers: each reads 2 partitions.
* 4 partitions and 4 consumers: each reads 1 partition. This is the max useful parallelism.
* 4 partitions and 6 consumers: 2 consumers sit **idle**.

```
Group "billing" on topic with 4 partitions

2 consumers:   C1 → P0, P1      C2 → P2, P3
4 consumers:   C1 → P0   C2 → P1   C3 → P2   C4 → P3
6 consumers:   same as 4, and C5, C6 do nothing
```

**Different groups are independent.** Group `billing` and group `analytics` each get a full copy of every record, with their own offsets. This is how Kafka does pub/sub.

### Offsets and commits

Consumers track progress by committing offsets. Kafka stores them in an internal topic called `__consumer_offsets`.

* **Auto commit:** simple, but can lose or repeat work on crashes.
* **Manual commit after processing:** safer. Gives at least once delivery.

```python
from confluent_kafka import Consumer

c = Consumer({
    "bootstrap.servers": "broker1:9092",
    "group.id": "billing_service",
    "enable.auto.commit": False,
    "auto.offset.reset": "earliest",
})
c.subscribe(["orders"])

while True:
    msg = c.poll(1.0)
    if msg is None or msg.error():
        continue
    handle(msg)      # do the work first
    c.commit(msg)    # then record progress
```

### Rebalancing

When a consumer joins, leaves, or crashes, partitions get reassigned. This is a **rebalance**. Old style rebalances pause everyone ("stop the world"). Tips:

* Use the **cooperative sticky** assignor so only moved partitions pause.
* Use **static membership** (`group.instance.id`) so quick restarts do not trigger rebalances.
* Keep processing time per poll under `max.poll.interval.ms`, or the broker thinks you died.
* Newer Kafka versions ship an improved group protocol (KIP 848) that moves coordination to the broker and cuts pauses further.

### Consumer lag

**Lag = latest offset minus committed offset.** It is the number one health metric. Growing lag means consumers cannot keep up. Alert on it.

## 9. Delivery Guarantees

| Guarantee | How it happens | Result |
|---|---|---|
| **At most once** | Commit offset first, then process | Crash means lost records |
| **At least once** | Process first, then commit | Crash means duplicates. **Most common.** |
| **Exactly once** | Idempotence plus transactions | No loss, no duplicates (inside Kafka) |

### Exactly once, honestly

Kafka transactions let a "read, process, write" loop be atomic across topics. The consumer offset commit and the output records succeed or fail together. Consumers use `isolation.level=read_committed` to skip aborted records. Kafka Streams turns this on with one setting.

The catch: exactly once covers **Kafka to Kafka**. Once you write to an external system (a database, an email API), Kafka cannot protect you.

Real world answer: **at least once delivery plus idempotent consumers.** Make processing safe to repeat:

* Use a unique event ID and a dedup table.
* Use upserts instead of inserts.
* Use conditional updates (version checks).

## 10. Storage, Retention, and Compaction

Each partition is stored as a series of **segment files** on disk. New records go to the active segment. When it fills (default 1 GB) or ages out, a new one starts. Old segments get deleted or compacted as a whole file, which is cheap.

### Retention (delete policy)

Records are deleted after:

* **Time:** default 7 days (`retention.ms`)
* **Size:** optional cap per partition (`retention.bytes`)
* Or set to forever for event sourcing use cases.

Reading old data does not "use it up." It stays until retention removes it.

### Log compaction

Instead of deleting by age, keep **only the latest record per key**.

* Great for "current state" topics: user profiles, config, inventory.
* A record with a key and a **null value** is a **tombstone**, meaning delete this key.
* New consumers can rebuild full state by reading the compacted topic from the start.

You can combine both: compact and also delete after a long time.

### Tiered storage

Newer Kafka can offload old segments to cheap object storage (like S3) while hot data stays on broker disks. This lets you keep long history without huge broker disks.

## 11. Scaling Playbook

Scaling Kafka means scaling four different things. Name which one is hurting.

### A. Scaling producers

Easy. Add more producer instances. Brokers handle many clients. Tune batching and compression first.

### B. Scaling consumers

Add consumers to the group, **up to the number of partitions**. Beyond that, nothing improves. If you need more parallelism than partitions allow, you must add partitions (with the key mapping caveat), or fan out inside the consumer.

A common trick: one consumer thread per partition, handing work to a worker pool. Careful: if you process out of order inside the pool, you lose per key ordering and offset commits get tricky.

### C. Scaling brokers (the cluster)

1. Add new brokers.
2. **Existing partitions do not move on their own.** New brokers start empty.
3. Reassign partitions to spread load. Tools like **Cruise Control** automate this.
4. Reassignment copies lots of data, so throttle it to avoid hurting live traffic.

Spread replicas across racks or availability zones (`broker.rack`) so one zone failure does not kill all replicas of a partition.

### D. Scaling partitions

More partitions mean more throughput, but with the costs listed earlier. Rough guide (always benchmark your own hardware):

* One partition handles roughly 10 MB/s of producer writes comfortably.
* Consumer throughput per partition depends on your processing time. Often this is the real limit.

### Scaling across regions

* **MirrorMaker 2** or managed cluster linking copies topics between clusters.
* Common setups: active passive for disaster recovery, active active with care over conflicts.
* Do not stretch one cluster across far apart regions. Latency hurts replication and acks.

### Scaling checklist

1. Is it the producer, broker, or consumer that is slow?
2. Is one partition hot? (Check the key.)
3. Is lag growing? (Add consumers, up to partition count.)
4. Are disks or network saturated? (Add brokers, rebalance partitions.)
5. Is retention eating disk? (Compress, shorten retention, use tiered storage.)

## 12. Capacity Estimation Example

Use this pattern in interviews.

**Requirements:** 100,000 messages per second, 1 KB each, 7 days retention, RF 3.

**Step 1: Ingress rate**
100,000 × 1 KB = **100 MB/s**

**Step 2: Daily and weekly data**
100 MB/s × 86,400 s = about 8.6 TB per day
× 7 days = about **60 TB** for one copy

**Step 3: With replication**
60 TB × 3 = about **180 TB** raw. With 3x compression, around **60 TB** on disk.

**Step 4: Brokers**
If each broker has 10 TB of disk and you plan to fill it to 60%, that is 6 TB usable.
60 TB ÷ 6 TB = about **10 brokers**.

**Step 5: Network**
Producers send 100 MB/s in. Replication adds 2 extra copies, so about 200 MB/s between brokers. Every consumer group reading the full stream adds another 100 MB/s of egress.

**Step 6: Partitions**
Producer side: 100 MB/s ÷ 10 MB/s per partition = 10 partitions minimum.
Consumer side: if each consumer handles 2 MB/s, you need 50 consumers, so 50 partitions minimum.
Take the larger, add headroom: **about 100 partitions**.

## 13. Failure Scenarios

| Problem | What happens | What to do |
|---|---|---|
| **One broker dies** | Controller elects new leaders from ISR. Clients retry. | With RF 3, nothing lost. Replace the broker. |
| **Consumer crashes** | Group rebalances. Partitions move to others. | Uncommitted work is redone. Make handlers idempotent. |
| **Slow consumer** | Lag grows. May exceed `max.poll.interval.ms` and get kicked out. | Reduce work per poll, lower `max.poll.records`, scale out. |
| **Poison message** | Consumer fails on the same record forever, blocking the partition. | Catch errors, send to a **dead letter topic**, move on. |
| **Hot partition** | One broker and one consumer overloaded. | Fix the key, add salting, or split the key space. |
| **Rebalance storm** | Constant joins and leaves, no progress. | Static membership, cooperative assignor, fix long processing. |
| **Disk full** | Broker stops accepting writes. | Retention limits, alerts, tiered storage. |
| **Network partition** | Replicas fall out of ISR. Writes may be rejected. | This is `min.insync.replicas` doing its job. |

### Retry pattern

Do not block the main topic on failures. Use:

1. `orders` (main)
2. `orders_retry_1m`, `orders_retry_10m` (delayed retries)
3. `orders_dlq` (dead letter, for humans to inspect)

## 14. Design Patterns You Will Use

**Event driven microservices.** Services publish facts ("OrderPlaced"). Others react. No direct calls, so no tight coupling.

**Event sourcing.** Store every change as an event. Current state is a replay of events. Kafka is the log.

**CDC (change data capture).** Tools like Debezium read a database's transaction log and publish each row change to Kafka. Great for syncing search indexes, caches, and data warehouses.

**Outbox pattern.** Problem: you must update the database **and** publish an event, atomically. Solution: write the event to an `outbox` table in the same DB transaction. A CDC process ships it to Kafka. No dual write bugs.

**CQRS.** Write side emits events. Read side builds optimized views (search, cache) from the stream.

**Saga.** Multi step workflows across services, coordinated by events and compensating actions.

**Log aggregation and metrics.** Many agents send logs to Kafka. Consumers feed storage and alerting.

**Buffering and load leveling.** Kafka absorbs spikes. Downstream systems consume at their own pace.

**Stream processing.** Filter, join, and aggregate events in motion (Kafka Streams, Flink).

## 15. The Ecosystem

* **Kafka Connect.** Plug and play connectors to move data in and out (databases, S3, Elasticsearch). No custom code.
* **Kafka Streams.** A Java library for stream processing inside your own app. No separate cluster.
* **Apache Flink.** Powerful standalone stream processing engine. Great for complex windows and large state.
* **ksqlDB.** SQL over streams.
* **Schema Registry.** Stores Avro, Protobuf, or JSON schemas and enforces compatibility rules so producers cannot break consumers.
* **Cruise Control.** Automated cluster rebalancing.
* **Managed options.** Confluent Cloud, AWS MSK, Aiven, Redpanda (Kafka compatible).

**Schema tip:** always use a schema format with evolution rules. Add fields as optional with defaults. Never rename or change field types casually.

## 16. Kafka vs Other Systems

| | Kafka | RabbitMQ | AWS SQS | Pulsar | Kinesis |
|---|---|---|---|---|---|
| **Model** | Distributed log | Smart broker, queues | Managed queue | Log with queues | Managed log |
| **Replay** | Yes | No (mostly) | No | Yes | Yes (limited window) |
| **Ordering** | Per partition | Per queue | FIFO queues only | Per partition | Per shard |
| **Throughput** | Very high | Moderate | High, managed | Very high | High, shard limits |
| **Routing** | Basic (topics) | Rich (exchanges) | Simple | Rich | Simple |
| **Ops burden** | High (self managed) | Medium | None | High | Low |
| **Best for** | Event streams, pipelines | Task routing, RPC style | Simple decoupling | Multi tenant streaming | AWS native streams |

Quick rule:

* Need **replay, high volume, many readers, ordering per key**? Kafka.
* Need **flexible routing, priorities, per message TTL**? RabbitMQ.
* Need **zero ops simple queue**? SQS.

## 17. When Not to Use Kafka

* Low volume and you just need a job queue.
* You need per message acks with individual retries and delays. Kafka only tracks an offset, not each message.
* You need strict global ordering across everything. That means one partition, which kills scale.
* The team cannot operate it and a managed service is not an option.
* Synchronous request and reply is the real need.

## 18. Config Cheat Sheet

**Producer (reliable defaults)**

```
acks=all
enable.idempotence=true
compression.type=lz4
linger.ms=5 to 20
batch.size=64KB to 256KB
delivery.timeout.ms=120000
```

**Consumer**

```
enable.auto.commit=false
auto.offset.reset=earliest   (or latest, decide on purpose)
max.poll.records=500
max.poll.interval.ms=300000
session.timeout.ms=45000
partition.assignment.strategy=CooperativeStickyAssignor
isolation.level=read_committed   (if using transactions)
```

**Topic and broker**

```
replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
retention.ms=604800000   (7 days)
cleanup.policy=delete or compact
```

## 19. Interview Style Questions

**Q: How does Kafka guarantee ordering?**
Only within a partition. Use a key so related events land in the same partition. Global order needs a single partition.

**Q: How do you scale consumers?**
Add consumers to the group up to the partition count. Beyond that, add partitions or parallelize inside each consumer carefully.

**Q: How does Kafka avoid losing data?**
Replication factor 3, `acks=all`, `min.insync.replicas=2`, and no unclean leader election. Producers retry with idempotence on.

**Q: Why is Kafka fast?**
Sequential disk writes, page cache, zero copy, batching, compression, and partition level parallelism.

**Q: What happens when a broker fails?**
The controller elects a new leader from the ISR for each affected partition. Clients refresh metadata and continue.

**Q: What happens when a consumer fails?**
The group rebalances and its partitions go to other consumers. Uncommitted records are processed again, so handlers must be idempotent.

**Q: How do you get exactly once?**
Inside Kafka: idempotent producer plus transactions plus `read_committed`. With external systems: at least once plus idempotent writes.

**Q: How do you handle a hot partition?**
Change to a higher cardinality key, add a salt (key plus random suffix) and merge later, or split heavy keys into their own topic.

**Q: How do you update a database and publish an event safely?**
Outbox pattern with CDC. Never do two separate writes and hope.

**Q: Kafka or RabbitMQ?**
Kafka for high volume streams, replay, and many consumers. RabbitMQ for rich routing and per message task handling.

**Q: How do you handle bad messages?**
Validate against a schema, catch exceptions, send failures to a dead letter topic with error metadata, keep the main flow moving.

**Q: How do you monitor Kafka?**
Consumer lag, under replicated partitions, ISR shrink rate, request latency, disk usage, network throughput, and broker CPU.

## 20. One Page Recap

* Kafka is a **distributed, replicated, append only log**.
* **Topics** split into **partitions**. Partitions give ordering, parallelism, and storage units.
* **Key decides partition.** Same key, same partition, same order.
* **Replication** (RF 3, `acks=all`, `min.insync.replicas=2`) protects data.
* **Consumer groups** share work. Max one consumer per partition per group.
* **Offsets** are the consumer's bookmark. Commit after processing for at least once.
* **Exactly once** works inside Kafka. Outside, build **idempotent consumers**.
* **Retention** deletes by time or size. **Compaction** keeps latest per key.
* **Scale** by adding consumers (up to partitions), brokers (then rebalance), and partitions (carefully).
* Watch for **hot partitions, lag, rebalance storms, and poison messages**.
* Pick Kafka for **volume, replay, and fan out**. Pick something simpler when you do not need them.

### Questions to ask in any Kafka design

1. What is the key, and does it spread load evenly?
2. What ordering do I truly need?
3. How many partitions, and what happens at 10x growth?
4. What delivery guarantee does each consumer need?
5. How long must data be kept, and is replay required?
6. How are bad messages handled?
7. How will schemas evolve?
8. What are the lag and failure alerts?
