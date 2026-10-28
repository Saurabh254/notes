# Redis

A deep, practical walkthrough of Redis. How it works, what it offers, every major data structure, Streams in depth, persistence, scaling, and the patterns you will actually use in production.

## Table of Contents

1. [What Redis Is](#1-what-redis-is)
2. [How Redis Works Internally](#2-how-redis-works-internally)
3. [Why Redis Is Fast](#3-why-redis-is-fast)
4. [Getting Started](#4-getting-started)
5. [Keys, Expiry and Namespacing](#5-keys-expiry-and-namespacing)
6. [Data Structures](#6-data-structures)
7. [Redis Streams (Deep Dive)](#7-redis-streams-deep-dive)
8. [Pub/Sub](#8-pubsub)
9. [Transactions](#9-transactions)
10. [Lua Scripting and Functions](#10-lua-scripting-and-functions)
11. [Persistence](#11-persistence)
12. [Replication](#12-replication)
13. [High Availability with Sentinel](#13-high-availability-with-sentinel)
14. [Redis Cluster](#14-redis-cluster)
15. [Memory Management and Eviction](#15-memory-management-and-eviction)
16. [Pipelining and Client Side Caching](#16-pipelining-and-client-side-caching)
17. [Security](#17-security)
18. [Beyond Core Redis: Search, JSON, Time Series, Vectors](#18-beyond-core-redis)
19. [Common Patterns](#19-common-patterns)
20. [Operations and Observability](#20-operations-and-observability)
21. [Pitfalls and Anti Patterns](#21-pitfalls-and-anti-patterns)
22. [Redis vs Alternatives](#22-redis-vs-alternatives)
23. [Licensing and Versions](#23-licensing-and-versions)
24. [Cheat Sheet](#24-cheat-sheet)

## 1. What Redis Is

**Redis** stands for **RE**mote **DI**ctionary **S**erver. It is an in memory data structure server. You store data under keys, and the values are not just blobs. They are rich types like lists, sets, sorted sets, hashes and streams, each with its own atomic operations.

Created by Salvatore Sanfilippo in 2009. Default port is **6379**.

What people use it for:

* **Cache** in front of a slower database or API
* **Session store** for web apps
* **Rate limiting** and counters
* **Leaderboards** and rankings
* **Queues** and background job systems
* **Event streaming** with Redis Streams
* **Real time analytics** (unique counts, bitmaps)
* **Distributed locks** and coordination
* **Primary database** for workloads that fit in memory (with persistence on)
* **Vector search** for AI and RAG workloads

The mental model: Redis is a **shared, networked, concurrent data structure library**. Instead of each app server holding its own hash map, they all talk to one Redis that holds it for them, safely.

### Key properties

| Property | Meaning |
| --- | --- |
| In memory | All data lives in RAM. Disk is for durability only. |
| Single threaded execution | Commands run one at a time on a main thread. No locks needed. |
| Atomic commands | Every command is atomic by nature. |
| Rich types | Strings, Lists, Sets, Sorted Sets, Hashes, Streams, and more. |
| Optional persistence | RDB snapshots, AOF log, or both. |
| Replication | Async primary to replica copies. |
| Horizontal scale | Redis Cluster shards data across nodes. |
| Sub millisecond latency | Typical operations finish in microseconds on the server. |

## 2. How Redis Works Internally

### 2.1 Client server model

A client opens a TCP connection and sends commands using **RESP** (REdis Serialization Protocol). RESP2 is the classic text protocol. **RESP3** (enabled with `HELLO 3`) adds richer types like maps, sets, doubles, and out of band push messages, which power client side caching.

A RESP request for `SET name saurabh` looks like this on the wire:

```
*3\r\n
$3\r\nSET\r\n
$4\r\nname\r\n
$7\r\nsaurabh\r\n
```

The protocol is trivial to parse, which is part of why Redis clients exist for every language.

### 2.2 The event loop

Redis uses its own tiny event library (`ae`) on top of `epoll` (Linux), `kqueue` (BSD and macOS) or `select`. The main loop does this, over and over:

1. Wait for sockets that are readable or writable.
2. Read incoming bytes, parse commands.
3. **Execute commands one by one** on the main thread.
4. Write replies back.
5. Run time events (expiry cycles, replication, persistence housekeeping).

Because commands run one at a time, there are no data races on the keyspace. That is why `INCR` is safe without locks, and why one slow command blocks everyone.

### 2.3 Threads in modern Redis

"Redis is single threaded" is only half true today.

* **Command execution** is still on one main thread.
* **I/O threads** (since Redis 6) handle socket reads, writes and parsing. Enabled with `io-threads`. Newer 8.x releases improved this significantly.
* **Background threads** handle slow jobs like closing files, `UNLINK` lazy frees and AOF fsync.
* **Forked child processes** handle RDB snapshots and AOF rewrites.

### 2.4 The keyspace

Internally Redis keeps a big hash table per logical database mapping key to value object. Every value is a `redisObject` that stores the type, the **encoding** (how it is physically stored), an LRU/LFU field and a reference count.

The same logical type can use different encodings depending on size. Small collections use compact encodings that save memory, large ones switch to structures with better big O.

```
OBJECT ENCODING mykey
```

tells you what you actually got.

### 2.5 Logical databases

A single instance has 16 numbered databases by default (`SELECT 0` through `SELECT 15`). They share the same process and memory. In **Redis Cluster only database 0** exists. Most teams ignore numbered databases and use key prefixes plus separate instances instead.

## 3. Why Redis Is Fast

* **Memory access** is orders of magnitude faster than disk.
* **No locks.** Single threaded execution means no contention.
* **Efficient data structures** written in C and tuned for cache locality (listpack, intset, skiplist).
* **Simple protocol** with cheap parsing.
* **Event driven I/O** means thousands of connections without thousands of threads.
* **Pipelining** lets clients batch many commands per round trip.

Typical numbers: hundreds of thousands of ops per second on one core, over a million with pipelining and I/O threads on good hardware. The bottleneck is almost always **network round trip time**, not Redis itself.

The flip side: because everything runs on one thread, an O(N) command on a huge collection stalls the entire server. Respect big O.

## 4. Getting Started

### 4.1 Run it

```bash
# Docker
docker run -d --name redis -p 6379:6379 redis:8

# Connect
docker exec -it redis redis-cli
```

```bash
# Linux package manager (Debian and Ubuntu style)
sudo apt install redis-server
redis-server --port 6379
redis-cli ping
# PONG
```

### 4.2 First commands

```
127.0.0.1:6379> SET greeting "hello"
OK
127.0.0.1:6379> GET greeting
"hello"
127.0.0.1:6379> INCR visits
(integer) 1
127.0.0.1:6379> EXPIRE greeting 60
(integer) 1
127.0.0.1:6379> TTL greeting
(integer) 58
127.0.0.1:6379> TYPE visits
string
127.0.0.1:6379> DEL greeting
(integer) 1
```

### 4.3 From code

Python with `redis-py`:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

r.set("user:1:name", "Saurabh", ex=3600)
print(r.get("user:1:name"))

r.hset("user:1", mapping={"name": "Saurabh", "city": "Gurugram"})
print(r.hgetall("user:1"))

pipe = r.pipeline()
pipe.incr("hits")
pipe.expire("hits", 60)
pipe.execute()
```

Go with `go-redis`:

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

    rdb.Set(ctx, "user:1:name", "Saurabh", time.Hour)
    val, _ := rdb.Get(ctx, "user:1:name").Result()
    fmt.Println(val)

    rdb.HSet(ctx, "user:1", map[string]any{"name": "Saurabh", "city": "Gurugram"})
    all, _ := rdb.HGetAll(ctx, "user:1").Result()
    fmt.Println(all)
}
```

Always use a **connection pool** (both clients do this by default) and reuse one client per process.

## 5. Keys, Expiry and Namespacing

### 5.1 Key design

Keys are binary safe strings up to 512 MB, but keep them short and readable. Use a consistent scheme with colons:

```
user:1001:profile
order:2026:10:5521
cache:product:42:v3
rate:ip:203.0.113.9:minute
```

Rules of thumb:

* Prefix by domain, then entity, then id.
* Avoid very long keys. They cost memory and hashing time.
* Include a version in cache keys so you can invalidate by bumping it.
* In Redis Cluster, use **hash tags** `{...}` when keys must live on the same shard (see section 14).

### 5.2 Expiry (TTL)

```
SET session:abc "data" EX 3600      # seconds
SET session:abc "data" PX 1500      # milliseconds
SET session:abc "data" EXAT 1893456000   # absolute unix time
EXPIRE key 60
PEXPIRE key 1500
EXPIREAT key 1893456000
TTL key         # seconds left, -1 no expiry, -2 key missing
PTTL key
PERSIST key     # remove the expiry
GETEX key EX 60 # read and refresh TTL
```

Note: a plain `SET` on an existing key **clears its TTL** unless you pass `KEEPTTL`.

**How expiry actually works.** Redis does not scan every key. It uses two mechanisms together:

1. **Lazy expiry.** When you touch a key, Redis checks if it has expired and deletes it then.
2. **Active expiry.** A periodic job samples random keys with TTLs. If more than 25 percent of a sample was expired, it repeats immediately. This keeps memory from filling with dead keys.

Since Redis 7.4 you can also expire **individual hash fields** (`HEXPIRE`, `HPEXPIRE`, `HTTL`).

### 5.3 Finding keys safely

Never run `KEYS *` in production. It is O(N) and blocks the server. Use cursor based iteration:

```
SCAN 0 MATCH user:* COUNT 100
```

`SCAN` returns a cursor and a batch. Keep calling until the cursor returns to `0`. It may return duplicates, so make your handling idempotent. Matching commands exist for collections: `SSCAN`, `HSCAN`, `ZSCAN`.

### 5.4 Useful generic commands

| Command | Purpose |
| --- | --- |
| `EXISTS key` | Does it exist |
| `DEL key` | Delete, may block on huge values |
| `UNLINK key` | Delete, memory freed in background thread |
| `TYPE key` | Value type |
| `RENAME a b` | Rename |
| `OBJECT ENCODING key` | Internal encoding |
| `OBJECT FREQ key` | LFU counter |
| `MEMORY USAGE key` | Bytes used |
| `COPY a b` | Copy value |
| `DUMP` / `RESTORE` | Serialize and restore a value |
| `RANDOMKEY` | A random key |
| `DBSIZE` | Number of keys |
| `FLUSHDB` / `FLUSHALL` | Wipe (be careful) |

## 6. Data Structures

Quick map before the details:

| Type | Think of it as | Typical use |
| --- | --- | --- |
| String | Bytes, number, counter | Cache, counters, locks |
| List | Deque of strings | Queues, recent items |
| Hash | Map of field to value | Objects, sessions |
| Set | Unordered unique members | Tags, unique visitors, relations |
| Sorted Set | Unique members with a score | Leaderboards, priority queues, time indexes |
| Stream | Append only log with consumer groups | Event streaming, durable queues |
| Bitmap | Bit array on a string | Flags, daily active users |
| HyperLogLog | Probabilistic counter | Approximate unique counts |
| Geospatial | Lat/long in a sorted set | Nearby search |
| JSON | Native JSON documents | Document storage |
| Vector Set | Vectors with similarity search | Embeddings, semantic search |
| Probabilistic | Bloom, Cuckoo, Count Min, Top K, t digest | Membership, frequency, quantiles |
| Time Series | Timestamped samples | Metrics, IoT |

### 6.1 Strings

The most basic type. A string holds text, a number or raw bytes, up to **512 MB**.

```
SET user:1:name "Saurabh"
GET user:1:name
MSET a 1 b 2 c 3
MGET a b c
SETNX lock:job1 "worker-7"      # set only if absent
SET lock:job1 "w7" NX PX 30000   # modern form with expiry
GETSET / GETDEL / GETEX          # read plus side effect
APPEND log "line"
STRLEN log
GETRANGE log 0 9
SETRANGE log 5 "xx"
```

Atomic counters:

```
INCR page:views
INCRBY page:views 10
DECR stock:42
INCRBYFLOAT balance 2.5
```

**Encodings:**

* `int` for numbers that fit in 64 bits
* `embstr` for short strings (44 bytes or fewer), one allocation
* `raw` for longer strings, backed by **SDS** (Simple Dynamic String) with O(1) length and binary safety

**Use cases:** caching serialized objects, counters, feature flags, locks, rate limit buckets.

### 6.2 Lists

An ordered sequence of strings. Push and pop at either end is O(1). Access by index in the middle is O(N).

```
LPUSH queue "job1" "job2"      # push left
RPUSH queue "job3"             # push right
LPOP queue
RPOP queue
LRANGE queue 0 -1              # all items
LLEN queue
LINDEX queue 0
LINSERT queue BEFORE "job3" "jobX"
LTRIM recent 0 99              # keep only first 100
LMOVE src dst LEFT RIGHT       # atomic move between lists
```

Blocking variants let consumers wait without polling:

```
BLPOP queue 30           # wait up to 30s for an item
BRPOP queue 30
BLMOVE src dst LEFT RIGHT 30
LMPOP / BLMPOP           # pop from the first non empty of several lists
```

**Encoding:** `quicklist`, a linked list of compact `listpack` nodes. Small lists are a single listpack.

**Use cases:** simple job queues, activity feeds ("last 100 events" with `LPUSH` + `LTRIM`), reliable queue via `LMOVE` into a processing list.

**Limit of lists as queues:** if a worker crashes after `BLPOP`, the job is gone. Use `LMOVE`/`BLMOVE` to a processing list, or better, use **Streams** (section 7).

### 6.3 Hashes

A map of field to value inside one key. Ideal for objects.

```
HSET user:1 name "Saurabh" city "Gurugram" age 25
HGET user:1 name
HMGET user:1 name city
HGETALL user:1
HDEL user:1 age
HINCRBY user:1 logins 1
HEXISTS user:1 city
HLEN user:1
HSCAN user:1 0
HEXPIRE user:1 3600 FIELDS 1 name     # per field TTL (7.4+)
```

**Encoding:** `listpack` for small hashes (default under 128 fields, values under 64 bytes), `hashtable` beyond that. Small hashes are far more memory efficient than the same data as separate string keys.

**Use cases:** user profiles, sessions, product records, any "row like" object. You can update one field without rewriting the whole thing.

### 6.4 Sets

An unordered collection of **unique** strings.

```
SADD tags:post1 "redis" "database" "cache"
SISMEMBER tags:post1 "redis"
SMEMBERS tags:post1
SCARD tags:post1
SREM tags:post1 "cache"
SPOP tags:post1                 # remove and return random
SRANDMEMBER tags:post1 2
```

Set algebra is the superpower:

```
SINTER  followers:a followers:b     # mutual followers
SUNION  tags:post1 tags:post2
SDIFF   all_users banned_users
SINTERSTORE dest k1 k2
SINTERCARD 2 k1 k2 LIMIT 100        # count only
```

**Encodings:** `intset` (all integers, small), `listpack` (small, 7.2+), `hashtable` (large).

**Use cases:** unique visitors, tag systems, friend graphs, "who liked this", lottery draws with `SPOP`.

### 6.5 Sorted Sets (ZSets)

Unique members, each with a floating point **score**. Members are always ordered by score, then lexicographically. This is Redis's most powerful structure.

```
ZADD leaderboard 1500 "alice" 1200 "bob" 1800 "carol"
ZINCRBY leaderboard 50 "bob"
ZRANGE leaderboard 0 -1 WITHSCORES            # ascending
ZRANGE leaderboard 0 2 REV WITHSCORES         # top 3 (6.2+ syntax)
ZRANK leaderboard "alice"                     # position, ascending
ZREVRANK leaderboard "alice"                  # position, descending
ZSCORE leaderboard "alice"
ZRANGEBYSCORE leaderboard 1000 2000
ZRANGE leaderboard "(1000" "+inf" BYSCORE     # exclusive lower bound
ZREM leaderboard "bob"
ZCARD leaderboard
ZCOUNT leaderboard 1000 2000
ZPOPMIN leaderboard
BZPOPMIN leaderboard 10
ZUNIONSTORE / ZINTERSTORE
ZREMRANGEBYSCORE leaderboard -inf 900
```

**Encodings:** `listpack` for small, and for large a **skiplist plus hash table** combo. The skiplist gives O(log N) ordered operations, the hash table gives O(1) score lookup by member.

**Use cases:**

* **Leaderboards.** Update score, read top N or a user's rank, all O(log N).
* **Priority queues.** Score is priority or due time. `ZPOPMIN` takes the next job.
* **Delayed jobs and schedulers.** Score is unix timestamp, poll with `ZRANGEBYSCORE key -inf now`.
* **Sliding window rate limiting.** Score is timestamp of each request.
* **Secondary indexes** on timestamps or numeric fields.

### 6.6 Bitmaps

Not a separate type. Bit level commands on strings. A string of 512 MB holds up to about 4 billion bits.

```
SETBIT active:2026-10-26 1001 1      # user 1001 was active today
GETBIT active:2026-10-26 1001
BITCOUNT active:2026-10-26           # how many active users
BITOP AND active:week d1 d2 d3       # active on all days
BITPOS active:2026-10-26 1           # first set bit
BITFIELD counters INCRBY u8 0 1      # many small integers packed in one string
```

**Use cases:** daily active users, feature flags per user id, presence, bloom filter like structures. One bit per user means 100 million users fit in about 12 MB.

### 6.7 HyperLogLog

A probabilistic structure to count **distinct** elements using a fixed ~12 KB, with a standard error of about **0.81 percent**.

```
PFADD visitors:2026-10-26 "u1" "u2" "u3"
PFCOUNT visitors:2026-10-26
PFMERGE visitors:week visitors:d1 visitors:d2
```

You cannot list or remove items. You only get an approximate count. Perfect for unique visitor counts when exactness does not matter and sets would be too large.

### 6.8 Geospatial

Stored on top of a sorted set where the score is a 52 bit geohash.

```
GEOADD stores 77.2090 28.6139 "delhi" 77.0266 28.4595 "gurugram"
GEOPOS stores "delhi"
GEODIST stores "delhi" "gurugram" km
GEOSEARCH stores FROMLONLAT 77.1 28.5 BYRADIUS 50 km ASC WITHDIST
GEOSEARCH stores FROMMEMBER "delhi" BYBOX 100 100 km
GEOSEARCHSTORE nearby stores FROMLONLAT 77.1 28.5 BYRADIUS 50 km
```

`GEORADIUS` is deprecated, use `GEOSEARCH`. Use cases: store locators, nearby drivers, geofencing lite.

### 6.9 JSON

Native JSON documents with path based access. Included in Redis 8 (previously the RedisJSON module).

```
JSON.SET user:1 $ '{"name":"Saurabh","skills":["go","python"],"address":{"city":"Gurugram"}}'
JSON.GET user:1 $.address.city
JSON.SET user:1 $.address.city '"Delhi"'
JSON.ARRAPPEND user:1 $.skills '"redis"'
JSON.NUMINCRBY user:1 $.score 5
```

Combined with the Query Engine you can index and search JSON fields (section 18).

### 6.10 Vector Sets

Introduced in Redis 8. A type for storing vectors (embeddings) and doing similarity search with HNSW under the hood.

```
VADD docs VALUES 3 0.1 0.9 0.3 "doc:1"
VSIM docs VALUES 3 0.1 0.8 0.4 COUNT 5
```

Use for semantic search, recommendations and retrieval for LLM apps. A richer vector index is also available through the Query Engine.

### 6.11 Probabilistic structures

Part of Redis 8 (formerly RedisBloom). Trade accuracy for tiny memory.

| Structure | Answers | Commands |
| --- | --- | --- |
| Bloom filter | "Is X possibly in the set?" No false negatives | `BF.ADD`, `BF.EXISTS` |
| Cuckoo filter | Same, but supports deletion | `CF.ADD`, `CF.DEL` |
| Count Min Sketch | "About how often did X occur?" | `CMS.INCRBY`, `CMS.QUERY` |
| Top K | "Which items are the heaviest?" | `TOPK.ADD`, `TOPK.LIST` |
| t digest | Percentiles from a stream of numbers | `TDIGEST.ADD`, `TDIGEST.QUANTILE` |

### 6.12 Time Series

A native type for timestamped numeric samples with retention, downsampling and aggregation. Included in Redis 8.

```
TS.CREATE cpu:host1 RETENTION 86400000 LABELS host host1 metric cpu
TS.ADD cpu:host1 * 73.5
TS.RANGE cpu:host1 - + AGGREGATION avg 60000
```

### 6.13 Encodings cheat sheet

| Logical type | Small encoding | Large encoding |
| --- | --- | --- |
| String | `int`, `embstr` | `raw` (SDS) |
| List | `listpack` | `quicklist` |
| Hash | `listpack` | `hashtable` |
| Set | `intset`, `listpack` | `hashtable` |
| Sorted Set | `listpack` | `skiplist` + `hashtable` |
| Stream | radix tree of `listpack` nodes | same |

Thresholds are tunable (for example `hash-max-listpack-entries`, `zset-max-listpack-entries`). Staying under them saves a lot of memory.

## 7. Redis Streams (Deep Dive)

A **Stream** is an append only log of entries, stored under one key. Think of a lightweight, in memory cousin of Kafka's topic, or a log file with a built in consumer coordination layer. Added in Redis 5.

Each **entry** has:

* A unique, monotonically increasing **ID**
* One or more **field value pairs** (a small map)

### 7.1 Entry IDs

An ID looks like `1761436800123-0`, which is `<milliseconds>-<sequence>`.

* Milliseconds are the server clock at insert time (never goes backwards, even if the clock does).
* Sequence breaks ties for entries in the same millisecond.
* Pass `*` to let Redis generate it. You can also pass your own ID if it is greater than the last one.
* Special IDs: `-` smallest possible, `+` largest possible, `$` "only entries after the current last one", `>` "only entries never delivered to a consumer in this group", `0` or `0-0` "from the very beginning".

Because the ID embeds time, you can query by time range without any extra index.

### 7.2 Writing

```
XADD orders * user_id 1001 amount 499 status "created"
# "1761436800123-0"

XLEN orders

# Cap the stream while adding
XADD orders MAXLEN ~ 100000 * user_id 1002 amount 120 status "created"
XADD orders MINID ~ 1761350400000 * user_id 1003 amount 80 status "created"
```

The `~` means **approximate trimming**. Redis trims whole internal nodes, which is much cheaper than trimming to an exact length. Use `~` in almost all cases.

### 7.3 Reading without groups (simple fan out)

```
# Range queries by ID (inclusive)
XRANGE orders - + COUNT 10
XRANGE orders 1761436800000 1761436900000
XREVRANGE orders + - COUNT 5          # newest first

# Tail the stream
XREAD COUNT 10 STREAMS orders 0                  # from the start
XREAD COUNT 10 BLOCK 5000 STREAMS orders $       # block up to 5s for new entries
XREAD BLOCK 0 STREAMS orders 1761436800123-0     # block forever, after this ID
```

With plain `XREAD` every reader sees every entry. The reader remembers its own last ID. Good for broadcast and for replay.

### 7.4 Consumer groups

A **consumer group** lets many consumers **share the work** of one stream. Each entry is delivered to **exactly one consumer in the group**. Multiple different groups on the same stream each get the full stream independently.

```
# Create a group. $ = only new entries, 0 = from the beginning
XGROUP CREATE orders billing $ MKSTREAM
XGROUP CREATE orders analytics 0

# Consume. ">" means "new entries nobody in this group has received yet"
XREADGROUP GROUP billing worker-1 COUNT 10 BLOCK 5000 STREAMS orders >

# Acknowledge after processing
XACK orders billing 1761436800123-0
```

What Redis tracks for each group:

* **last-delivered-id**: the cursor for `>` reads.
* **Consumers**: created automatically on first read.
* **PEL (Pending Entries List)**: entries delivered but not yet `XACK`ed, with owner, last delivery time and delivery count.

This gives you **at least once delivery**. If a worker dies before acking, the entry stays pending and can be recovered.

### 7.5 Handling failure: pending entries and claiming

Inspect what is stuck:

```
XPENDING orders billing                                  # summary
XPENDING orders billing - + 10                           # detail for 10 entries
XPENDING orders billing IDLE 60000 - + 10                # idle longer than 60s
```

Take over entries from a dead or slow consumer:

```
# Manual: claim specific IDs idle for 60s+
XCLAIM orders billing worker-2 60000 1761436800123-0

# Automatic scan and claim (6.2+), the one you usually want
XAUTOCLAIM orders billing worker-2 60000 0-0 COUNT 25
```

`XAUTOCLAIM` returns a cursor, the claimed entries and any IDs that were deleted from the stream but still sat in the PEL.

Reading your own backlog after a restart:

```
# Use an ID instead of ">" to get this consumer's pending (unacked) entries
XREADGROUP GROUP billing worker-1 COUNT 10 STREAMS orders 0
```

**Poison messages.** An entry that always crashes workers will be redelivered forever. Redis has no built in dead letter queue. The standard pattern: check the **delivery count** from `XPENDING` (or the claim result). If it exceeds N, `XADD` the entry to a `orders:dead` stream and `XACK` the original.

### 7.6 Managing the stream

```
XTRIM orders MAXLEN ~ 100000
XTRIM orders MINID ~ 1761350400000
XDEL orders 1761436800123-0          # delete one entry (leaves a tombstone)

XINFO STREAM orders
XINFO STREAM orders FULL             # groups, consumers, PEL details
XINFO GROUPS orders
XINFO CONSUMERS orders billing

XGROUP CREATECONSUMER orders billing worker-3
XGROUP DELCONSUMER orders billing worker-3
XGROUP SETID orders billing 0        # rewind a group to replay
XGROUP DESTROY orders billing
```

Newer 8.x releases also added commands such as `XACKDEL` and `XDELEX`, and options that control what happens to consumer group references when entries are deleted or trimmed. Check the docs for the exact version you run.

### 7.7 Internals

* A stream is stored as a **radix tree** where each node holds a **listpack** of entries.
* Entries inside a node are delta encoded against a master entry, so fields that repeat (same field names) cost almost nothing extra.
* Reading by ID is O(log N) to find the start, then linear in the number of entries returned.
* `XADD` is O(1) amortized.

The result is a very memory efficient log. A stream with a stable schema stores each entry in a small number of bytes beyond the raw values.

### 7.8 Producer and consumer example (Python)

```python
import time
import redis

r = redis.Redis(decode_responses=True)
STREAM, GROUP, CONSUMER = "orders", "billing", "worker-1"

# Setup (idempotent)
try:
    r.xgroup_create(STREAM, GROUP, id="$", mkstream=True)
except redis.ResponseError as e:
    if "BUSYGROUP" not in str(e):
        raise


# Producer
def publish(order):
    return r.xadd(STREAM, order, maxlen=100_000, approximate=True)


# Consumer loop
def process(fields):
    print("processing", fields)


def consume():
    while True:
        resp = r.xreadgroup(GROUP, CONSUMER, {STREAM: ">"}, count=10, block=5000)
        if not resp:
            continue
        for _stream, entries in resp:
            for entry_id, fields in entries:
                try:
                    process(fields)
                    r.xack(STREAM, GROUP, entry_id)
                except Exception:
                    # leave it pending, a reclaimer will pick it up
                    pass


# Reclaimer: run periodically, rescues stuck entries
def reclaim():
    cursor = "0-0"
    while True:
        cursor, entries, _deleted = r.xautoclaim(
            STREAM, GROUP, CONSUMER, min_idle_time=60_000, start_id=cursor, count=25
        )
        for entry_id, fields in entries:
            try:
                process(fields)
                r.xack(STREAM, GROUP, entry_id)
            except Exception:
                pass
        if cursor == "0-0":
            break
```

### 7.9 Consumer example (Go)

```go
func consume(ctx context.Context, rdb *redis.Client) {
    for {
        res, err := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
            Group:    "billing",
            Consumer: "worker-1",
            Streams:  []string{"orders", ">"},
            Count:    10,
            Block:    5 * time.Second,
        }).Result()
        if err == redis.Nil {
            continue
        }
        if err != nil {
            time.Sleep(time.Second)
            continue
        }
        for _, stream := range res {
            for _, msg := range stream.Messages {
                if handle(msg.Values) == nil {
                    rdb.XAck(ctx, "orders", "billing", msg.ID)
                }
            }
        }
    }
}
```

### 7.10 Guarantees and caveats

| Topic | Reality |
| --- | --- |
| Delivery | At least once within a group. Make consumers **idempotent**. |
| Ordering | Total order in the stream. Within a group, entries are split across consumers, so global processing order is not guaranteed. Per consumer order holds. |
| Durability | Depends on persistence settings. AOF with `everysec` can lose about 1 second. |
| Replication | Async. A failover can lose the latest entries or acks. `WAIT` helps but is not full consensus. |
| Memory | The whole stream is in RAM. You **must** trim by length or by age. |
| Scaling | One stream key lives on one node. To scale out, shard by entity id across several stream keys such as `orders:{0}` to `orders:{15}`. |
| Replay | Easy. Read from any ID or rewind a group with `XGROUP SETID`. Limited by what you retained. |

### 7.11 Streams vs Pub/Sub vs Lists vs Kafka

| Feature | Pub/Sub | List | Stream | Kafka |
| --- | --- | --- | --- | --- |
| Persistence of messages | No | Yes | Yes | Yes (disk) |
| Replay history | No | No | Yes | Yes |
| Consumer groups | No | No (manual) | Yes | Yes |
| Ack and redelivery | No | No (manual) | Yes | Offsets |
| Fan out to many apps | Yes | No | Yes (groups) | Yes |
| Retention | None | Until popped | By length or age, in RAM | Days or forever, on disk |
| Throughput ceiling | Very high | High | High | Very high, partitioned |
| Operational weight | Tiny | Tiny | Small | Heavy |

**Rule of thumb.** Use Streams for moderate volume event pipelines, task queues with acknowledgements, and audit trails that fit in memory. Reach for Kafka when you need huge retention, many partitions, strict durability or an ecosystem (Connect, Streams, schema registry).

## 8. Pub/Sub

Fire and forget messaging. Publishers send to a **channel**, all currently connected subscribers receive it. Nothing is stored.

```
# Terminal A
SUBSCRIBE news:sports
PSUBSCRIBE news:*

# Terminal B
PUBLISH news:sports "India wins"
# (integer) 2   number of receivers
```

Characteristics:

* **At most once.** If a subscriber is disconnected or slow, it misses the message.
* No history, no acks.
* Subscribers in RESP2 are in a special mode and can only run subscribe commands on that connection.
* A slow subscriber can bloat its output buffer and get disconnected (see `client-output-buffer-limit pubsub`).
* **Sharded Pub/Sub** (`SPUBLISH`, `SSUBSCRIBE`, Redis 7.0) routes channels by slot so cluster traffic scales better.
* `PUBSUB CHANNELS`, `PUBSUB NUMSUB` for introspection.

**Keyspace notifications.** Redis can publish events when keys change or expire.

```
CONFIG SET notify-keyspace-events KEA
PSUBSCRIBE __keyevent@0__:expired
```

Handy but also at most once. Do not build critical logic on expiry events alone.

Use Pub/Sub for cache invalidation broadcasts, live notifications, chat presence and anything where a lost message is fine. Use Streams when it is not.

## 9. Transactions

```
MULTI
INCR balance:a
DECR balance:b
EXEC
```

* `MULTI` starts queueing. Commands reply `QUEUED`.
* `EXEC` runs the whole queue **atomically and in isolation**. No other client's commands interleave.
* `DISCARD` throws the queue away.

**Important differences from SQL transactions:**

* **No rollback.** If one command fails at runtime (say, `INCR` on a non numeric string), the others still run.
* A **syntax error** while queueing aborts the whole transaction at `EXEC`.
* You cannot read a value mid transaction and branch on it.

For read then write logic, use **optimistic locking** with `WATCH`:

```
WATCH balance:a
val = GET balance:a
MULTI
SET balance:a (val - 10)
EXEC           # returns nil if balance:a changed since WATCH, so retry
```

In Python:

```python
with r.pipeline() as pipe:
    while True:
        try:
            pipe.watch("balance:a")
            bal = int(pipe.get("balance:a"))
            pipe.multi()
            pipe.set("balance:a", bal - 10)
            pipe.execute()
            break
        except redis.WatchError:
            continue
```

In Redis Cluster all keys in a transaction must hash to the same slot. When logic gets complex, a Lua script is usually simpler than `WATCH`.

## 10. Lua Scripting and Functions

Scripts run **atomically** on the server. Great for multi step logic that must not interleave.

```
EVAL "return redis.call('GET', KEYS[1])" 1 mykey
```

Format: `EVAL script numkeys key1 ... arg1 ...`. Always pass key names in `KEYS` and other values in `ARGV` so Cluster can route correctly.

**Safe lock release** (delete only if I still own it):

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

```
EVAL "<script above>" 1 lock:job1 my-unique-token
```

Cache the script by SHA to save bandwidth:

```
SCRIPT LOAD "<script>"      # returns sha1
EVALSHA <sha1> 1 lock:job1 my-unique-token
```

**Redis Functions** (7.0+) are the modern evolution. Libraries are loaded once, stored with the dataset (persisted and replicated), and called by name.

```
FUNCTION LOAD "#!lua name=mylib
redis.register_function('release_lock', function(keys, args)
  if redis.call('GET', keys[1]) == args[1] then
    return redis.call('DEL', keys[1])
  end
  return 0
end)"

FCALL release_lock 1 lock:job1 my-unique-token
```

Rules: keep scripts **short and fast**. A long script blocks the whole server just like any slow command.

## 11. Persistence

Redis is in memory, but it can survive restarts. Two mechanisms, usable alone or together.

### 11.1 RDB snapshots

A **point in time** binary dump of the whole dataset.

How it works: Redis calls `fork()`. The child process writes the dataset to `dump.rdb` while the parent keeps serving. **Copy on write** memory pages keep the child's view consistent.

```
# redis.conf
save 3600 1 300 100 60 10000     # after 3600s if 1 change, 300s if 100, 60s if 10000
dbfilename dump.rdb
```

```
BGSAVE          # snapshot in background
SAVE            # snapshot in foreground (blocks, avoid)
LASTSAVE        # unix time of last success
```

Pros:

* Compact single file, great for backups and disaster recovery
* Very fast restart for large datasets
* Minimal impact on the main thread

Cons:

* You can lose everything since the last snapshot (minutes)
* `fork()` on a very large dataset can cause a latency spike

### 11.2 AOF (Append Only File)

Logs **every write command**. On restart Redis replays the log.

```
appendonly yes
appendfsync everysec     # always | everysec | no
```

| `appendfsync` | Durability | Speed |
| --- | --- | --- |
| `always` | Fsync after every write. Safest. | Slowest |
| `everysec` | At most about 1 second of loss. The default and the usual choice. | Fast |
| `no` | OS decides when to flush | Fastest, weakest |

The log grows, so Redis **rewrites** it in the background (`BGREWRITEAOF`, or automatically with `auto-aof-rewrite-percentage` and `auto-aof-rewrite-min-size`), producing the smallest set of commands that rebuild the current state.

From Redis 7 the AOF is **multi part**: a base file (often an RDB preamble), incremental files, and a manifest, all in `appenddirname`. With `aof-use-rdb-preamble yes` (default) the base is an RDB, so loading is fast and the tail is a short command log.

Repair tools: `redis-check-aof`, `redis-check-rdb`.

### 11.3 Choosing

| Need | Setup |
| --- | --- |
| Pure cache, data can be rebuilt | No persistence |
| Some loss acceptable, simple backups | RDB only |
| Strong durability | AOF `everysec` (plus RDB for backups) |
| Maximum safety | AOF `always` and replicas, accept the cost |

Operational notes: set `vm.overcommit_memory=1`, disable Transparent Huge Pages, and keep spare RAM because the fork can transiently use extra memory under heavy writes.

## 12. Replication

One **primary** accepts writes. One or more **replicas** copy it asynchronously.

```
# On the replica
REPLICAOF 10.0.0.1 6379
# or in redis.conf: replicaof 10.0.0.1 6379

INFO replication
```

How it syncs:

1. **Full resync.** Replica connects, primary produces an RDB (to disk or straight to the socket with `repl-diskless-sync`), sends it, then streams buffered writes.
2. **Partial resync.** After a short disconnect the replica sends its replication ID and offset (`PSYNC`). If the primary's **replication backlog** (`repl-backlog-size`) still holds the gap, it only sends the missing part.
3. **Continuous stream.** Every write is forwarded as it happens.

Key facts:

* **Asynchronous.** A primary can acknowledge a write and crash before any replica gets it. Data loss is possible on failover.
* Replicas are **read only** by default. Reads from replicas may be slightly **stale**.
* `WAIT numreplicas timeout` blocks until that many replicas acknowledge, narrowing (not eliminating) the loss window.
* `min-replicas-to-write` and `min-replicas-max-lag` make the primary refuse writes if too few healthy replicas exist.
* Replicas can have their own replicas (chained replication).
* Replication offloads reads and gives a hot standby. It does **not** shard data.

## 13. High Availability with Sentinel

**Redis Sentinel** is a set of separate processes that watch a primary and its replicas, and perform **automatic failover**.

What it does:

* **Monitoring** of primary and replica health
* **Notification** hooks
* **Failover**: promote a replica when the primary is dead
* **Service discovery**: clients ask Sentinel "who is primary now?"

```
# sentinel.conf
sentinel monitor mymaster 10.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

Failure detection in two steps:

1. **SDOWN** (subjectively down): one Sentinel cannot reach the primary.
2. **ODOWN** (objectively down): a **quorum** of Sentinels agree. A leader Sentinel is elected (by majority vote), picks the best replica (priority, replication offset, run ID) and promotes it.

Rules:

* Run **at least 3 Sentinels** on independent machines or zones.
* The quorum triggers detection, but a **majority** of all Sentinels is needed to authorize the failover.
* Clients must be **Sentinel aware** (most official clients support it).

```
SENTINEL get-master-addr-by-name mymaster
```

Limit: Sentinel gives HA, not sharding. Your dataset must fit on one node.

## 14. Redis Cluster

Redis Cluster provides **sharding and HA together**.

### 14.1 Hash slots

The keyspace is split into **16384 hash slots**.

```
slot = CRC16(key) mod 16384
```

Each primary owns a range of slots. With three primaries it might look like:

| Node | Slots |
| --- | --- |
| A | 0 to 5460 |
| B | 5461 to 10922 |
| C | 10923 to 16383 |

Each primary typically has one or more replicas. If a primary fails, its replica is elected and takes over its slots.

### 14.2 Setting up

```
# redis.conf on every node
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000

# Create (3 primaries, 1 replica each = 6 nodes)
redis-cli --cluster create \
  10.0.0.1:6379 10.0.0.2:6379 10.0.0.3:6379 \
  10.0.0.4:6379 10.0.0.5:6379 10.0.0.6:6379 \
  --cluster-replicas 1

redis-cli -c -h 10.0.0.1      # -c makes the CLI follow redirects
CLUSTER INFO
CLUSTER NODES
CLUSTER SLOTS
```

Nodes talk to each other over a **gossip bus** on port + 10000 (so 16379 for the default).

### 14.3 Redirection

* Cluster aware clients fetch the slot map and send each command straight to the right node.
* **MOVED slot ip:port**: the slot lives elsewhere permanently. Client updates its map.
* **ASK slot ip:port**: slot is mid migration. Retry once on the target with `ASKING`.

### 14.4 Multi key rules and hash tags

Commands touching several keys (`MGET`, `SINTER`, transactions, Lua) require all keys in the **same slot**, otherwise you get `CROSSSLOT`.

**Hash tags** force co location. Only the text inside `{}` is hashed:

```
user:{1001}:profile
user:{1001}:orders
user:{1001}:cart
```

All three land in the same slot, so you can use them in one transaction or script. Do not overuse the same tag. It creates hot spots.

### 14.5 Resharding and operations

```
redis-cli --cluster reshard 10.0.0.1:6379
redis-cli --cluster rebalance 10.0.0.1:6379
redis-cli --cluster add-node new:6379 existing:6379
redis-cli --cluster check 10.0.0.1:6379
```

Slots migrate live, key by key, with no downtime.

### 14.6 Trade offs

* Only **database 0**.
* Replication is async, so writes acknowledged just before a failover can be lost.
* A minority partition stops serving. With `cluster-require-full-coverage yes` (default) the whole cluster rejects queries if any slot is uncovered.
* Needs a **minimum of 3 primaries** for a healthy majority.

| | Sentinel | Cluster |
| --- | --- | --- |
| Failover | Yes | Yes |
| Sharding | No | Yes |
| Dataset limit | One node's RAM | Sum of primaries |
| Multi key ops | Any | Same slot only |
| Client complexity | Medium | Higher |

## 15. Memory Management and Eviction

### 15.1 maxmemory

```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

When memory hits the limit, Redis applies the policy on each new write.

| Policy | Behavior |
| --- | --- |
| `noeviction` | Reject writes with an OOM error. Reads still work. |
| `allkeys-lru` | Evict least recently used from all keys. **Best default for caches.** |
| `allkeys-lfu` | Evict least frequently used from all keys |
| `allkeys-random` | Evict random keys |
| `volatile-lru` | LRU, but only keys that have a TTL |
| `volatile-lfu` | LFU among keys with a TTL |
| `volatile-random` | Random among keys with a TTL |
| `volatile-ttl` | Evict keys closest to expiring |

LRU and LFU are **approximated**. Redis samples a few keys (`maxmemory-samples`, default 5) and evicts the best candidate. It saves memory and CPU, and larger samples approach exact behavior.

If you store data you cannot lose (queues, primary data) use `noeviction`. Use a `volatile-*` policy only when you really mix cache keys and permanent keys in one instance.

### 15.2 Reading memory stats

```
INFO memory
MEMORY STATS
MEMORY DOCTOR
MEMORY USAGE mykey SAMPLES 0
```

Watch:

* `used_memory`: bytes allocated by Redis
* `used_memory_rss`: bytes the OS gave the process
* `mem_fragmentation_ratio`: RSS divided by used. Around 1.0 to 1.5 is healthy. Far above 1.5 means fragmentation. Below 1 means swapping, which is a disaster.

Enable `activedefrag yes` to defragment online.

### 15.3 Saving memory

* Prefer **hashes** for many small related fields instead of many tiny string keys.
* Keep collections under the **listpack** thresholds.
* Use short but readable keys.
* Store integers as integers.
* Compress big values on the client (LZ4, zstd, gzip).
* Set TTLs on everything that can expire.
* Trim streams and capped lists.
* Use `UNLINK` and the `lazyfree-*` options so freeing huge values does not block.

## 16. Pipelining and Client Side Caching

### 16.1 Pipelining

Each command normally costs one **network round trip**. Pipelining sends many commands without waiting for each reply.

```python
pipe = r.pipeline(transaction=False)
for i in range(1000):
    pipe.set(f"key:{i}", i)
pipe.execute()
```

One round trip instead of 1000. Often a 5x to 10x throughput gain. Note a pipeline is **not atomic** (unlike `MULTI`). Do not make a single pipeline gigantic. Batches of a few hundred to a few thousand are typical.

### 16.2 Client side caching

Redis can tell clients when keys they cached locally have changed, so reads hit the app's own memory with zero network cost.

```
HELLO 3
CLIENT TRACKING ON
GET product:42          # server remembers this client cares about the key
# later, if someone changes product:42:
# client receives a push message: invalidate ["product:42"]
```

Modes: default tracking, `BCAST` by key prefix, `OPTIN`/`OPTOUT`. Several official clients ship a ready made implementation.

## 17. Security

Redis was designed for trusted networks. Treat it that way.

* **Never expose it to the public internet.** Bind to private interfaces, use VPC rules and firewalls.
* Keep `protected-mode yes`.
* Use **ACLs** (Redis 6+) instead of one shared password.

```
ACL SETUSER app on >s3cretPassw0rd ~app:* +@read +@write -@dangerous
ACL SETUSER readonly on >pass ~* +@read
ACL LIST
ACL WHOAMI
ACL LOG
AUTH app s3cretPassw0rd
```

An ACL rule defines: which **commands or categories** (`+@read`, `-@dangerous`), which **keys** (`~app:*`), and which **pub/sub channels** (`&chan:*`).

* Enable **TLS** for traffic between clients, replicas and cluster nodes:

```
tls-port 6380
port 0
tls-cert-file /etc/redis/redis.crt
tls-key-file /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
```

* Lock down dangerous commands with ACLs: `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG`, `KEYS`, `SHUTDOWN`.
* Run as a non root user. Keep Redis patched, security releases are frequent.

## 18. Beyond Core Redis

Since **Redis 8**, capabilities that used to live in separate modules ship in the box: the **Query Engine** (search and secondary indexing), **JSON**, **Time Series**, **probabilistic structures** and **Vector Sets**.

### 18.1 Query Engine (search)

Index hashes or JSON documents and query them with a rich language.

```
FT.CREATE idx:users ON HASH PREFIX 1 user: SCHEMA \
  name TEXT SORTABLE \
  city TAG \
  age NUMERIC SORTABLE

HSET user:1 name "Saurabh" city "Gurugram" age 25

FT.SEARCH idx:users "@city:{Gurugram} @age:[20 30]"
FT.AGGREGATE idx:users "*" GROUPBY 1 @city REDUCE COUNT 0 AS total
```

Supports full text search, tags, numeric ranges, geo filters, aggregations and **vector similarity** (FLAT and HNSW indexes) for AI retrieval.

```
FT.CREATE idx:docs ON HASH PREFIX 1 doc: SCHEMA \
  embedding VECTOR HNSW 6 TYPE FLOAT32 DIM 768 DISTANCE_METRIC COSINE
```

### 18.2 Where each fits

| Need | Use |
| --- | --- |
| Store a nested document | JSON |
| Query by field, text or range | Query Engine |
| Semantic or similarity search | Vector Sets or Query Engine vector index |
| Metrics with downsampling | Time Series |
| Membership or frequency at tiny memory | Probabilistic types |

## 19. Common Patterns

### 19.1 Cache aside

The app checks the cache first, falls back to the database, then fills the cache.

```python
def get_user(uid):
    key = f"cache:user:{uid}:v1"
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    user = db.fetch_user(uid)
    r.set(key, json.dumps(user), ex=300)
    return user
```

Other strategies:

* **Write through.** Write to cache and database together. Reads always hit.
* **Write behind.** Write to cache, flush to the database later. Fast but risky.
* **Read through.** The cache layer itself loads on miss.

Cache problems to plan for:

| Problem | What happens | Fix |
| --- | --- | --- |
| **Stampede** | Hot key expires, many requests hit the database at once | Lock around rebuild, jittered TTLs, serve stale while refreshing |
| **Penetration** | Requests for keys that never exist bypass cache | Cache negative results briefly, Bloom filter |
| **Avalanche** | Many keys expire together | Random TTL jitter |
| **Stale data** | Cache differs from source | Short TTLs, explicit invalidation, versioned keys |

### 19.2 Session store

```
HSET session:abc user_id 1001 role "admin" csrf "xyz"
EXPIRE session:abc 1800
```

Refresh the TTL on activity with `EXPIRE` (sliding sessions).

### 19.3 Counters and rate limiting

**Fixed window** (simplest):

```python
def allow(ip, limit=100, window=60):
    key = f"rate:{ip}:{int(time.time()) // window}"
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    count, _ = pipe.execute()
    return count <= limit
```

Weakness: bursts at the window edge can double the allowed rate.

**Sliding window log** with a sorted set:

```python
def allow(user, limit=100, window=60):
    now = time.time()
    key = f"rate:{user}"
    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)
    pipe.zadd(key, {f"{now}:{uuid4()}": now})
    pipe.zcard(key)
    pipe.expire(key, window)
    _, _, count, _ = pipe.execute()
    return count <= limit
```

Accurate, but costs memory per request.

**Token bucket** in Lua (atomic, smooth, cheap). Store `tokens` and `last_refill` in a hash. The script refills based on elapsed time, takes a token if available and returns allow or deny. Doing it in Lua avoids race conditions between read and write.

```lua
-- KEYS[1] bucket key; ARGV: capacity, refill_per_sec, now_ms, cost
local b = redis.call("HMGET", KEYS[1], "tokens", "ts")
local tokens = tonumber(b[1]) or tonumber(ARGV[1])
local ts = tonumber(b[2]) or tonumber(ARGV[3])
local elapsed = math.max(0, tonumber(ARGV[3]) - ts) / 1000
tokens = math.min(tonumber(ARGV[1]), tokens + elapsed * tonumber(ARGV[2]))
local allowed = 0
if tokens >= tonumber(ARGV[4]) then
  tokens = tokens - tonumber(ARGV[4])
  allowed = 1
end
redis.call("HSET", KEYS[1], "tokens", tokens, "ts", ARGV[3])
redis.call("PEXPIRE", KEYS[1], 60000)
return allowed
```

### 19.4 Distributed lock

```
SET lock:invoice:42 <random-token> NX PX 30000
```

* `NX` only if absent, `PX` auto expiry so a crashed holder cannot deadlock the system.
* The value is a **unique token** so that only the owner can release it.
* Release with the compare and delete Lua script from section 10.
* If work may outlast the TTL, **renew** the lock while holding it.

**Honest caveat.** A single node lock is fine for efficiency (avoid duplicate work). For strict **correctness** (never two holders), clock pauses, failover and network delays can break any Redis lock, including **Redlock**, which is debated in the community. If correctness is critical, add **fencing tokens** (a monotonically increasing number the protected resource checks) or use a consensus system like etcd or ZooKeeper.

### 19.5 Leaderboard

```
ZINCRBY lb:global 25 "player:7"
ZREVRANGE lb:global 0 9 WITHSCORES      # top 10
ZREVRANK lb:global "player:7"           # my rank
ZCOUNT lb:global 1000 +inf
```

Per period boards: `lb:2026-10`, `lb:week:43`, then `ZUNIONSTORE` to combine.

### 19.6 Job queue options

| Approach | Pros | Cons |
| --- | --- | --- |
| List with `BRPOP` | Dead simple | Lost job if worker crashes |
| List with `BLMOVE` to a processing list | Recoverable | You write the recovery logic |
| Sorted set by due time | Delayed and scheduled jobs | Needs a poller and atomic claim (Lua) |
| **Stream with consumer group** | Acks, retries, replay, many consumers | Slightly more code |

Popular libraries built on Redis: Celery, RQ, Sidekiq, BullMQ, Asynq.

### 19.7 Unique counts and presence

* Exact and small: `SADD` and `SCARD`
* Huge and approximate: `PFADD` and `PFCOUNT`
* Per user per day flags: bitmaps with `SETBIT` and `BITCOUNT`

### 19.8 Idempotency keys

```
SET idem:payment:abc123 "processing" NX EX 86400
```

If it returns OK you are first, proceed. If nil, you have seen this request already. Store the final result under the same key to return it on retries.

### 19.9 Real time fan out

Combine **Streams** (durable log) with **Pub/Sub** (instant wake up), or use `XREAD BLOCK` directly. For websocket gateways, each gateway instance subscribes to a channel and pushes to its connected users.

## 20. Operations and Observability

### 20.1 Health and stats

```
INFO                       # everything
INFO server
INFO clients
INFO memory
INFO stats
INFO replication
INFO persistence
INFO commandstats          # per command call counts and time
INFO keyspace
```

Numbers worth alerting on:

* `used_memory` against `maxmemory`
* `connected_clients` and `blocked_clients`
* `evicted_keys` and `expired_keys`
* `keyspace_hits` vs `keyspace_misses` (hit ratio)
* `instantaneous_ops_per_sec`
* `rdb_last_bgsave_status` and `aof_last_write_status`
* `master_link_status` and replica lag
* `latest_fork_usec`

### 20.2 Finding slow things

```
SLOWLOG GET 10            # commands slower than slowlog-log-slower-than (microseconds)
SLOWLOG RESET
LATENCY DOCTOR
LATENCY LATEST
redis-cli --latency
redis-cli --latency-history
redis-cli --bigkeys       # largest key per type
redis-cli --memkeys
CLIENT LIST
```

`MONITOR` streams every command live. It is great for debugging and **expensive**, so never leave it running in production.

### 20.3 Benchmarking

```bash
redis-benchmark -h 127.0.0.1 -p 6379 -c 50 -n 200000 -P 16 -q
```

Benchmark your own commands and payload sizes from a realistic network location.

### 20.4 Configuration

```
CONFIG GET maxmemory
CONFIG SET maxmemory 2gb
CONFIG REWRITE            # persist runtime changes back to redis.conf
```

### 20.5 Backups and recovery

* Copy `dump.rdb` (or the AOF directory) off the machine regularly.
* Test restores. An untested backup is a hope.
* For managed services, use their snapshot features and cross region copies.

### 20.6 Production checklist

* Persistence chosen on purpose, not by default
* `maxmemory` and a matching eviction policy set
* Replication or Sentinel or Cluster for HA
* ACLs, TLS, private network only
* Monitoring and alerts on the numbers above
* THP disabled, `overcommit_memory=1`, enough RAM headroom for forks
* Client timeouts, retries with backoff, connection pooling
* Key naming and TTL conventions documented

## 21. Pitfalls and Anti Patterns

| Mistake | Why it hurts | Better |
| --- | --- | --- |
| `KEYS *` in production | O(N), blocks everyone | `SCAN` |
| Huge keys (million element lists, giant JSON) | Slow deletes, replication spikes, uneven shards | Split into buckets, use `UNLINK` |
| Hot key | One key gets all traffic, one shard melts | Local cache, key sharding, replicas |
| No TTLs on cache data | Memory fills forever | Always set expiry |
| Treating Redis as a durable system of record without AOF and replicas | Data loss | Configure persistence or keep a real database |
| Long Lua scripts or `SMEMBERS` on huge sets | Blocks the single thread | Keep it small, iterate with `SSCAN` |
| One connection per request | Connection storms | Pooling |
| Blocking commands on a shared pooled connection | Starves the pool | Dedicated connection |
| Assuming `MULTI` rolls back | It does not | Validate first, or use Lua |
| Multi key commands in Cluster without hash tags | `CROSSSLOT` | Hash tags |
| Same TTL for everything | Mass expiry avalanche | Jitter |
| Relying on Pub/Sub for critical messages | Lost messages | Streams |
| Running with swap enabled | Latency collapse | Size RAM properly |
| Unbounded streams | OOM | `MAXLEN ~` or `MINID ~` |
| Exposing port 6379 to the world | Compromise | Private network, ACLs, TLS |

## 22. Redis vs Alternatives

| | Redis | Memcached | Valkey | Dragonfly | Kafka |
| --- | --- | --- | --- | --- | --- |
| Model | Data structure server | Simple key value cache | Fork of Redis 7.2 | Redis compatible, multi threaded | Distributed log |
| Data types | Rich | Strings only | Rich | Rich | Topics |
| Persistence | RDB and AOF | No | RDB and AOF | Snapshots | Disk log |
| Replication and HA | Yes | Client side only | Yes | Yes | Yes |
| Threading | Single main plus I/O threads | Multi threaded | Single main plus I/O threads | Shared nothing multi threaded | Multi |
| Best at | Versatile low latency data layer | Plain caching at scale | Open source Redis lineage | Vertical scale on big boxes | Durable high volume event logs |

**Redis vs a relational database.** Redis is not a replacement for joins, ad hoc queries and strong transactional guarantees across entities. It is the fast layer next to or in front of them, or a primary store when your access patterns are key oriented.

## 23. Licensing and Versions

A short history, because it affects what you can run:

* Through Redis 7.2, Redis was **BSD 3 Clause** licensed.
* From **7.4** (2024), Redis moved to a dual source available model (RSALv2 and SSPLv1). Cloud providers and the Linux Foundation responded with **Valkey**, a BSD licensed fork of Redis 7.2.
* From **Redis 8**, the project added **AGPLv3**, an OSI approved open source license, as an option alongside RSALv2 and SSPLv1.
* Redis 8 also folded the former modules (JSON, Query Engine, Time Series, probabilistic types) into the core distribution and introduced Vector Sets.

Versions move quickly (the 8.x line is on regular minor and security releases), so pin versions in production and read release notes for security fixes. If licensing matters to your organization, read the current terms at redis.io and compare with Valkey and the managed services you use.

## 24. Cheat Sheet

### Strings and keys

```
SET k v [EX s | PX ms] [NX | XX] [KEEPTTL]    GET k    MGET    MSET
INCR  DECR  INCRBY  INCRBYFLOAT   APPEND   STRLEN
EXPIRE  TTL  PERSIST  EXISTS  DEL  UNLINK  TYPE  SCAN
```

### Lists

```
LPUSH RPUSH LPOP RPOP LRANGE LLEN LTRIM LMOVE BLPOP BRPOP BLMOVE
```

### Hashes

```
HSET HGET HMGET HGETALL HDEL HINCRBY HEXISTS HLEN HSCAN HEXPIRE
```

### Sets

```
SADD SREM SISMEMBER SMEMBERS SCARD SPOP SINTER SUNION SDIFF SSCAN
```

### Sorted sets

```
ZADD ZINCRBY ZRANGE (BYSCORE|REV|WITHSCORES) ZRANK ZREVRANK ZSCORE
ZREM ZCARD ZCOUNT ZPOPMIN ZPOPMAX BZPOPMIN ZUNIONSTORE ZINTERSTORE
```

### Streams

```
XADD  XLEN  XRANGE  XREVRANGE  XREAD
XGROUP CREATE|SETID|DESTROY|CREATECONSUMER|DELCONSUMER
XREADGROUP  XACK  XPENDING  XCLAIM  XAUTOCLAIM
XTRIM  XDEL  XINFO STREAM|GROUPS|CONSUMERS
```

### Other types

```
SETBIT GETBIT BITCOUNT BITOP BITFIELD
PFADD PFCOUNT PFMERGE
GEOADD GEOSEARCH GEODIST
JSON.SET JSON.GET     VADD VSIM     TS.ADD TS.RANGE
BF.ADD BF.EXISTS      CMS.INCRBY    TOPK.ADD
```

### Messaging and scripting

```
PUBLISH SUBSCRIBE PSUBSCRIBE SPUBLISH SSUBSCRIBE
MULTI EXEC DISCARD WATCH
EVAL EVALSHA SCRIPT LOAD FUNCTION LOAD FCALL
```

### Server and ops

```
INFO  CONFIG GET|SET|REWRITE  SLOWLOG  LATENCY  MEMORY USAGE|STATS|DOCTOR
BGSAVE  BGREWRITEAOF  REPLICAOF  WAIT  CLUSTER INFO|NODES  ACL SETUSER|LIST
CLIENT LIST|TRACKING  MONITOR (careful)  FLUSHALL (careful)
```

### Decision guide

| I need to... | Use |
| --- | --- |
| Cache an object | String or Hash with TTL |
| Count things atomically | `INCR` |
| Rank users | Sorted set |
| Dedupe or track uniques | Set, HyperLogLog or Bloom filter |
| Daily yes or no flags for millions | Bitmap |
| Find nearby places | Geo |
| Queue work with retries | Stream and consumer group |
| Broadcast ephemeral events | Pub/Sub |
| Schedule delayed work | Sorted set by timestamp |
| Atomic multi step logic | Lua or Function |
| Scale beyond one node's RAM | Cluster |
| Survive primary failure | Replicas with Sentinel or Cluster |

Happy building.
