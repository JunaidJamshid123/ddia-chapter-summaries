# DDIA — Chapter 6: Partitioning
### Complete study guide — theory, every diagram redrawn, and measured simulations

> *Clearly, we must break away from the sequential and not limit the computers. We must state definitions and provide for priorities and descriptions of data. We must state relationships, not procedures.*
> — Grace Murray Hopper, *Management and the Computer of the Future* (1962)

---

## 0. The map of this chapter

Chapter 5 gave you **many copies of the same data**. Chapter 6 gives you **different data on different machines**. Replication buys availability; partitioning buys **scale**.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│  ① HOW TO PARTITION KEY-VALUE DATA                                         │
│        by KEY RANGE  ←─── trade-off ───→  by HASH OF KEY                   │
│        range queries ✅                     even load ✅                     │
│        hot spots ❌                         no range queries ❌              │
│                                                                            │
│  ② HOW SECONDARY INDEXES INTERACT WITH PARTITIONING                        │
│        DOCUMENT-partitioned (local)  ←→  TERM-partitioned (global)         │
│        cheap writes, scatter/gather      one-partition reads, costly writes│
│                                                                            │
│  ③ REBALANCING — adding and removing nodes                                 │
│        ❌ hash mod N   ✅ fixed partitions   ✅ dynamic splitting             │
│                                                                            │
│  ④ REQUEST ROUTING — "which IP do I connect to for key 'foo'?"             │
│        any node · routing tier · partition-aware client   (+ ZooKeeper)    │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 📖 Terminological confusion — worth memorizing

> What we call a **partition** here is called:

```
   SHARD    in MongoDB, Elasticsearch, SolrCloud
   REGION   in HBase
   TABLET   in BigTable
   VNODE    in Cassandra and Riak
   VBUCKET  in Couchbase

   "However, PARTITIONING is the most established term, so we'll stick with that."
```

> ⚠️ **And a critical footnote:** partitioning here means **intentionally breaking a large database into smaller ones.** It has **nothing to do with network partitions (netsplits)**, a *fault* in the network between nodes (Chapter 8). The same word, two completely unrelated meanings.

### Why partition at all?

> **The main reason for wanting to partition data is SCALABILITY.**

```
   Different partitions → different nodes in a SHARED-NOTHING cluster.
      • a large dataset is distributed across MANY DISKS
      • the query load is distributed across MANY PROCESSORS

   SMALL QUERIES on a single partition → each node independently executes
      queries for its own partition → THROUGHPUT SCALES BY ADDING NODES

   LARGE, COMPLEX QUERIES → can potentially be parallelized across many
      nodes, "although this gets SIGNIFICANTLY HARDER"
```

**History:** pioneered in the **1980s** by **Teradata** and **Tandem NonStop SQL**; **"more recently rediscovered by NoSQL databases and Hadoop-based data warehouses."** The fundamentals apply to both transactional and analytic workloads.

Each partition is, in effect, **"a small database of its own"** — though the database may support operations touching several at once.

---

## 1. Partitioning and replication together

> Partitioning is **usually combined with replication**, so copies of each partition are stored on multiple nodes. **Even though each record belongs to exactly one partition, it may still be stored on several different nodes for fault tolerance.**

### 🔷 Figure 6-1 — Each node is leader for some partitions, follower for others

```
   ╔═══════════════════════════════════╗   ╔═══════════════════════════════════╗
   ║  NODE 1                           ║   ║  NODE 2                           ║
   ║ ┌─────────┬─────────┬─────────┐   ║   ║ ┌─────────┬─────────┬─────────┐   ║
   ║ │Partition│Partition│Partition│   ║   ║ │Partition│Partition│Partition│   ║
   ║ │    1    │    2    │    3    │   ║   ║ │    2    │    3    │    4    │   ║
   ║ │ ★LEADER │ follower│ follower│   ║   ║ │ follower│ ★LEADER │ follower│   ║
   ║ └─────────┴─────────┴─────────┘   ║   ║ └─────────┴─────────┴─────────┘   ║
   ╚═══════════════════════════════════╝   ╚═══════════════════════════════════╝
              ▲          ▲                        ▲          ▲
              │          └──── replication ───────┘          │
              │            streams (PER PARTITION)           │
              ▼                                              ▼
   ╔═══════════════════════════════════╗   ╔═══════════════════════════════════╗
   ║  NODE 3                           ║   ║  NODE 4                           ║
   ║ ┌─────────┬─────────┬─────────┐   ║   ║ ┌─────────┬─────────┬─────────┐   ║
   ║ │Partition│Partition│Partition│   ║   ║ │Partition│Partition│Partition│   ║
   ║ │    1    │    2    │    4    │   ║   ║ │    1    │    3    │    4    │   ║
   ║ │ follower│ ★LEADER │ follower│   ║   ║ │ follower│ follower│ ★LEADER │   ║
   ║ └─────────┴─────────┴─────────┘   ║   ║ └─────────┴─────────┴─────────┘   ║
   ╚═══════════════════════════════════╝   ╚═══════════════════════════════════╝
                                                       ▲
                                              writing to partition 4
                                              goes HERE, its leader

   ⚑ EVERY NODE IS BUSY. There is no idle "standby" machine — each node
     leads some partitions and follows others, so write load spreads too.
```

> **"The choice of partitioning scheme is MOSTLY INDEPENDENT of the choice of replication scheme, so we will keep things simple and IGNORE REPLICATION in this chapter."**

That orthogonality is worth holding onto: you pick a partitioning strategy and a replication strategy separately, and they compose.

---

## 2. Partitioning of key-value data

> **Our goal is to spread the data and the query load EVENLY across nodes.** If every node takes a fair share, then in theory **ten nodes should handle ten times as much data and ten times the throughput of a single node.**

### 🔑 The two words you need

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  SKEW    — partitioning is unfair; some partitions have more data or  ║
   ║            queries than others. "This makes partitioning MUCH LESS    ║
   ║            EFFECTIVE."                                                ║
   ║                                                                       ║
   ║  HOT SPOT — a partition with DISPROPORTIONATELY HIGH LOAD.            ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   THE EXTREME CASE:
   ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌██████┐
   │idle│ │idle│ │idle│ │idle│ │idle│ │idle│ │idle│ │idle│ │idle│ │ 100% │
   └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └██████┘
   "nine out of ten nodes are idle, and your bottleneck is the single
    busy node."
```

### Why not just assign records randomly?

```
   ✅ would distribute data quite evenly
   ❌ BUT: "when you're trying to read a particular item, YOU HAVE NO WAY
        OF KNOWING WHICH NODE IT IS ON, so you would have to QUERY ALL
        NODES IN PARALLEL."

   ➜ We can do better. Assume a key-value model where you always access a
     record BY ITS PRIMARY KEY.
```

---

## 3. Partitioning by key range

### 🔷 Figure 6-2 — A print encyclopedia is partitioned by key range

```
   ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
   │ A-ak │Bayeu │Ceara │Delu- │Freon │Holder│Krasno│Menage│Otter │ Reti │Solov-│Trudeau│
   │  —   │  —   │  —   │sion  │  —   │ness  │kamsk │  —   │  —   │  —   │yov   │  —    │
   │Bayes │Cean- │Deluc │  —   │Holder│  —   │  —   │Ottawa│Rethi-│Solov-│  —   │Zywiec │
   │      │othus │      │Frens-│lin   │Krasn-│Menad-│      │mnon  │ets   │Truck │       │
   │      │      │      │sen   │      │oje   │ra    │      │      │      │      │       │
   ├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
   │  1   │  2   │  3   │  4   │  5   │  6   │  7   │  8   │  9   │  10  │  11  │  12  │
   └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘

   ⚑ NOTE THE UNEVEN LETTER SPANS.
     Volume 1  covers A and B  (two letters)
     Volume 12 covers T,U,V,X,Y,Z  (SIX letters)

   "Simply having ONE VOLUME PER TWO LETTERS of the alphabet would lead to
    some volumes being MUCH BIGGER than others. In order to distribute the
    data evenly, THE PARTITION BOUNDARIES NEED TO ADAPT TO THE DATA."
```

Boundaries may be chosen **manually by an administrator, or automatically by the database.**

**Used by:** BigTable, **HBase**, RethinkDB, and MongoDB before version 2.4.

### ✅ The advantage: sorted keys within each partition

```
   Within each partition we keep keys in SORTED ORDER (→ SSTables, Ch.3):

   ✅ RANGE SCANS ARE EASY
   ✅ you can treat the key as a CONCATENATED INDEX to fetch several
      related records in one query (→ multi-column indexes, Ch.3)

   EXAMPLE: sensor network, key = timestamp (year-month-day-hour-min-sec)
      → "range scans let you easily fetch ALL THE READINGS FROM A
         PARTICULAR MONTH."
```

### ❌ The disadvantage: certain access patterns cause hot spots

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE TIMESTAMP HOT SPOT                                               ║
   ║                                                                       ║
   ║  If the key is a timestamp, partitions = ranges of time, e.g. one     ║
   ║  partition per day.                                                   ║
   ║                                                                       ║
   ║  ⚠️ "Because we write data from the sensors AS IT HAPPENS, ALL THE     ║
   ║     WRITES END UP GOING TO THE SAME PARTITION (the one for today),    ║
   ║     so that partition can be OVERLOADED WITH WRITES WHILE OTHERS SIT  ║
   ║     IDLE."                                                            ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  THE FIX: use something OTHER than the timestamp as the FIRST         ║
   ║  ELEMENT of the key.                                                  ║
   ║                                                                       ║
   ║     key = (sensor_name, timestamp)                                    ║
   ║            └─ partition by this ─┘                                    ║
   ║                                                                       ║
   ║  "Assuming you have MANY SENSORS ACTIVE AT THE SAME TIME, the write   ║
   ║   load will end up more evenly spread."                               ║
   ║                                                                       ║
   ║  ⚠️ THE COST: "when you want to fetch the values of MULTIPLE SENSORS   ║
   ║     within a time range, you need to perform A SEPARATE RANGE QUERY   ║
   ║     FOR EACH SENSOR NAME."                                            ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 💻 I measured both the hot spot and the fix

```
KEY = timestamp only, one partition per day, all writes happen TODAY:
   2024-03-01      0
   2024-03-02      0
   ...
   2024-03-08   4000  ██████████████████████████████████
   -> 7 of 8 partitions IDLE. One partition takes 100% of writes.

KEY = (sensor_name, timestamp) -- partition by sensor name first:
   p0   485   p1   409   p2   396   p3   312
   p4   673   p5  1014   p6   188   p7   523
   -> max/min = 5.39x. Write load SPREAD across all 8.
```

Not perfectly even (only 40 sensors, so there's sampling noise), **but the difference between "one node doing everything" and "all eight doing something" is the whole point.**

---

## 4. Partitioning by hash of key

> **"A good hash function takes skewed data and makes it uniformly distributed."** Even if the input strings are very similar, **their hashes are evenly distributed.**

### 🔷 Figure 6-3 — Partitioning by hash of key

```
   These keys are nearly identical…        …but their hashes are scattered:

   "2014-04-19 17:08:10"  ──┐                        7,372
   "2014-04-19 17:08:11"  ──┤                       18,805
   "2014-04-19 17:08:12"  ──┼── hash ──►            50,537
   "2014-04-19 17:08:13"  ──┤  (first 2 bytes       31,579
   "2014-04-19 17:08:14"  ──┤   of MD5)             62,253
   "2014-04-19 17:08:15"  ──┘                       24,510

   Assign each partition A RANGE OF HASHES (not a range of keys):

   ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
   │  p0  │  p1  │  p2  │  p3  │  p4  │  p5  │  p6  │  p7  │
   └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
   0            16,383        32,767        49,151      65,535
```

> 📖 *Footnote:* it **doesn't need to be a cryptographically strong hash function.** Cassandra and MongoDB use **MD5**; Voldemort uses **Fowler–Noll–Vo**.

### 📦 Sidebar: Consistent hashing — and why the book tells you to stop saying it

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "Consistent hashing", as defined by Karger et al., is a way of       ║
   ║  evenly distributing load across an internet-wide system of CACHES,   ║
   ║  such as a CDN. It uses RANDOMLY CHOSEN partition boundaries to       ║
   ║  avoid the need for central control or distributed consensus.         ║
   ║                                                                       ║
   ║  ⚠️ "CONSISTENT here has NOTHING to do with replica consistency        ║
   ║     (Ch.5) or ACID consistency (Ch.7)" — it describes a particular    ║
   ║     approach to REBALANCING.                                          ║
   ║                                                                       ║
   ║  ⚠️ "This particular approach ACTUALLY DOESN'T WORK VERY WELL FOR      ║
   ║     DATABASES, and so IT IS RARELY USED IN PRACTICE (the              ║
   ║     documentation of some databases still refers to consistent        ║
   ║     hashing, BUT IT IS USUALLY INACCURATE)."                          ║
   ║                                                                       ║
   ║  ➜ "Because this is so confusing, IT'S BEST TO AVOID THE TERM         ║
   ║     CONSISTENT HASHING, AND JUST CALL IT HASH PARTITIONING."          ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### ❌ What hashing costs you

> By using the hash of the key, **we also lost a nice property of key-range partitioning: the ability to do efficient range queries. Keys that were once adjacent are now scattered across all the partitions, so their sort order is lost.**

| System | Range queries on the primary key |
|---|---|
| **MongoDB** (hash sharding) | Any range query must be **sent to all partitions** |
| **Riak, Couchbase, Voldemort** | **Not supported** |

### 🔑 Cassandra's compromise — the best of both

```
   A table can have a COMPOUND PRIMARY KEY of several columns.

      PRIMARY KEY ( (user_id) , update_timestamp )
                     ▲            ▲
                     │            └── used as a CONCATENATED INDEX for
                     │                sorting within Cassandra's SSTables
                     └── ONLY THIS PART IS HASHED, to pick the partition

   ❌ you cannot range-scan over the FIRST column
   ✅ but with a FIXED VALUE for the first column, you get an EFFICIENT
      RANGE SCAN over the rest

   ➜ "This allows an ELEGANT DATA MODEL FOR ONE-TO-MANY RELATIONSHIPS."
     On a social site, one user posts many updates. Key
     (user_id, update_timestamp) lets you efficiently retrieve all updates
     by one user in a time interval, sorted by time. DIFFERENT USERS live
     on DIFFERENT PARTITIONS; WITHIN each user, updates are ordered by
     timestamp ON A SINGLE PARTITION.
```

### 💻 I measured skew across all three strategies on realistically skewed data

Using ~27,000 surnames distributed by real English first-letter frequencies:

```
key range (naive A-Z split into 8 equal letter-blocks):
   p0   5760  ██████████████████████████████████
   p1   2660  ███████████████
   p2   1760  ██████████
   p3   4680  ███████████████████████████
   p4   1440  ████████
   p5   3840  ██████████████████████
   p6   1320  ███████
   p7    340  ██
   -> max/min = 16.94x          ← BADLY SKEWED

key range (ADAPTIVE boundaries, chosen from the data):
   p0..p7 all ≈ 2725
   -> max/min = 1.00x           ← PERFECT

hash of key:
   p0..p7 range 2601–2807
   -> max/min = 1.08x           ← near perfect, with NO KNOWLEDGE of the data
```

**Three lessons in one table.** Naive key ranges are catastrophically skewed on real-world data. *Adaptive* key ranges are perfect — **but only because I computed the boundaries from data I already had.** Hashing gets within 8% of perfect **while knowing nothing about the distribution in advance**, which is exactly why it's the safer default.

---

## 5. Skewed workloads and relieving hot spots

> Hashing **can't avoid hot spots entirely: in the extreme case where all reads and writes are for THE SAME KEY, you still end up with all requests routed to the same partition.**

```
   "This kind of workload is perhaps unusual, BUT NOT IMPOSSIBLE: on a
    social media site, A CELEBRITY USER WITH MILLIONS OF FOLLOWERS may
    cause a STORM OF ACTIVITY when they do something."

   → a large volume of writes to the same key (the celebrity's user ID,
     or the ID of the action people are commenting on)

   ⚠️ "HASHING THE KEY DOESN'T HELP, AS THE HASH OF TWO IDENTICAL IDs IS
      STILL THE SAME."
```

*(The book's reference here is wonderful: **"3% of Twitter's Servers Dedicated to Justin Bieber."**)*

### The application-level fix

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "Today, MOST DATA SYSTEMS ARE NOT ABLE TO AUTOMATICALLY COMPENSATE   ║
   ║   for such a highly skewed workload, so IT'S THE RESPONSIBILITY OF    ║
   ║   THE APPLICATION to reduce the skew."                                ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  THE TECHNIQUE: add a RANDOM NUMBER to the beginning or end of a      ║
   ║  known-hot key.                                                       ║
   ║                                                                       ║
   ║     celebrity_123        →  celebrity_123_00                          ║
   ║                             celebrity_123_01                          ║
   ║                             …                                         ║
   ║                             celebrity_123_99                          ║
   ║                                                                       ║
   ║  "Just a TWO-DIGIT decimal random number would split the writes       ║
   ║   evenly across 100 DIFFERENT KEYS."                                  ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  ⚠️ THE TWO COSTS                                                      ║
   ║  ① READS must now read from ALL 100 KEYS and COMBINE the results.     ║
   ║  ② BOOKKEEPING: "it only makes sense to append the random number for  ║
   ║     the SMALL NUMBER OF HOT KEYS; for the vast majority of keys this  ║
   ║     would be unnecessary overhead. Thus you also need SOME WAY OF     ║
   ║     KEEPING TRACK OF WHICH KEYS ARE BEING SPLIT."                     ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 💻 Measured: hashing fails, the suffix fixes it

```
hash partitioning, but 90% of writes hit ONE key:
   p0   101   p1   111   p2  9153 ██████████████  p3  149
   p4   147   p5    94   p6   124                 p7  121
   -> max/min = 97.4x   HASHING DIDN'T HELP

FIX: append a 2-digit random suffix to the hot key only
   p0  1237   p1   861   p2   877   p3  1337
   p4  1837   p5  1141   p6  1680   p7  1030
   -> max/min = 2.13x   writes now spread across all partitions
```

**From 97× imbalance to 2×**, with a two-character change to one key. The read cost is real, though — one logical read became 100 physical ones.

---
---

## 6. Partitioning and secondary indexes

> The schemes so far rely on a key-value model. **The situation becomes more complicated if SECONDARY INDEXES are involved.**

```
   A secondary index usually DOESN'T IDENTIFY A RECORD UNIQUELY — it's a
   way of SEARCHING FOR OCCURRENCES of a particular value:
      "find all actions by user 123"
      "find all articles containing the word hogwash"
      "find all cars whose color is red"

   • the BREAD AND BUTTER of relational databases; common in document DBs
   • key-value stores (HBase, Voldemort) AVOIDED them due to
     implementation complexity; some (Riak) started adding them because
     they are so useful for data modeling
   • they are "the RAISON D'ÊTRE of search servers such as Solr and
     Elasticsearch"

   ⚠️ "THE PROBLEM WITH SECONDARY INDEXES IS THAT THEY DON'T MAP NEATLY
      TO PARTITIONS."
```

---

### Approach A — Partitioning secondary indexes by DOCUMENT (local index)

### 🔷 Figure 6-4 — Document-partitioned secondary indexes

```
   ╔═══════════════════════════════════════╗ ╔═══════════════════════════════════════╗
   ║ PARTITION 0   (doc IDs 0–499)         ║ ║ PARTITION 1   (doc IDs 500–999)       ║
   ╠═══════════════════════════════════════╣ ╠═══════════════════════════════════════╣
   ║ PRIMARY KEY INDEX                     ║ ║ PRIMARY KEY INDEX                     ║
   ║  191 {color:red,    make:Honda}       ║ ║  515 {color:silver, make:Ford}        ║
   ║  214 {color:black,  make:Dodge}       ║ ║  768 {color:red,    make:Volvo}       ║
   ║  306 {color:red,    make:Ford}        ║ ║  893 {color:silver, make:Audi}        ║
   ╟───────────────────────────────────────╢ ╟───────────────────────────────────────╢
   ║ SECONDARY INDEXES (by document)       ║ ║ SECONDARY INDEXES (by document)       ║
   ║  color:black   [214]                  ║ ║  color:black   []                     ║
   ║  color:red     [191, 306]  ◄──┐       ║ ║  color:red     [768]        ◄──┐      ║
   ║  color:yellow  []             │       ║ ║  color:silver  [515, 893]      │      ║
   ║  make:Dodge    [214]          │       ║ ║  make:Audi     [893]           │      ║
   ║  make:Ford     [306]          │       ║ ║  make:Ford     [515]           │      ║
   ║  make:Honda    [191]          │       ║ ║  make:Volvo    [768]           │      ║
   ╚═══════════════════════════════╪═══════╝ ╚════════════════════════════════╪══════╝
                                   │                                          │
                                   └──────────────┬───────────────────────────┘
                                                  │
                                   ┌──────────────┴──────────────┐
                                   │  SCATTER/GATHER read from   │
                                   │  ALL PARTITIONS             │
                                   │  "I am looking for a red car"│
                                   └─────────────────────────────┘
```

```
   ✅ EACH PARTITION IS COMPLETELY SEPARATE. It maintains its own secondary
      indexes covering ONLY the documents in that partition. It doesn't
      care what's in other partitions.
      → a WRITE (add/remove/update) touches ONLY THE ONE PARTITION holding
        that document ID.
      → hence: a LOCAL INDEX.

   ❌ READING requires care. "Unless you have done something special with
      the document IDs, THERE IS NO REASON WHY ALL THE CARS WITH A
      PARTICULAR COLOR WOULD BE IN THE SAME PARTITION." Red cars appear in
      BOTH partitions. To search for red cars you must send the query to
      ALL partitions and COMBINE the results.
      → SCATTER/GATHER.
      ⚠️ "Even if you query the partitions in parallel, scatter/gather is
         PRONE TO TAIL LATENCY AMPLIFICATION" (→ Chapter 1).
```

**Used by:** MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB — **essentially everyone.**

> ⚠️ **"Most database vendors recommend that you structure your partitioning scheme so that secondary index queries can be served from a single partition, BUT THAT IS NOT ALWAYS POSSIBLE, especially when you're using MULTIPLE SECONDARY INDEXES IN A SINGLE QUERY"** (filtering by colour *and* make at once).

> 📖 **A footnote worth heeding:** if your database only supports key-value, you may be tempted to build a secondary index yourself as a value-to-document-ID mapping in application code. **"If you go down this route, you need to take GREAT CARE to ensure your indexes remain consistent with the underlying data. Race conditions and intermittent write failures can VERY EASILY cause the data to go out of sync."**

---

### Approach B — Partitioning secondary indexes by TERM (global index)

> We can construct a **global index** covering data in all partitions. **But we can't just store that index on one node, since it would become a bottleneck and defeat the purpose of partitioning. A global index must ALSO be partitioned — but it can be partitioned DIFFERENTLY from the primary key index.**

### 🔷 Figure 6-5 — Term-partitioned secondary indexes

```
   ╔═══════════════════════════════════════╗ ╔═══════════════════════════════════════╗
   ║ PARTITION 0                           ║ ║ PARTITION 1                           ║
   ╠═══════════════════════════════════════╣ ╠═══════════════════════════════════════╣
   ║ PRIMARY KEY INDEX                     ║ ║ PRIMARY KEY INDEX                     ║
   ║  191 {color:red,    make:Honda}       ║ ║  515 {color:silver, make:Ford}        ║
   ║  214 {color:black,  make:Dodge}       ║ ║  768 {color:red,    make:Volvo}       ║
   ║  306 {color:red,    make:Ford}        ║ ║  893 {color:silver, make:Audi}        ║
   ╟───────────────────────────────────────╢ ╟───────────────────────────────────────╢
   ║ SECONDARY INDEXES (by TERM)  a–r      ║ ║ SECONDARY INDEXES (by TERM)  s–z      ║
   ║  color:black  [214]                   ║ ║  color:silver [515, 893]              ║
   ║  color:red    [191, 306, 768] ◄────┐  ║ ║  color:yellow []                      ║
   ║  make:Audi    [893]                │  ║ ║  make:Honda   [191]                   ║
   ║  make:Dodge   [214]                │  ║ ║  make:Volvo   [768]                   ║
   ║  make:Ford    [306, 515]           │  ║ ║                                       ║
   ╚════════════════════════════════════╪══╝ ╚═══════════════════════════════════════╝
                                        │
                           ┌────────────┴────────────┐
                           │ "I am looking for a     │
                           │  red car"               │
                           │  → ONE partition. Done. │
                           └─────────────────────────┘

   ⚑ NOTE: ALL red cars from EVERY primary partition — 191, 306 AND 768 —
     are now in ONE index entry. The index is partitioned by the TERM
     (colors a–r on p0, s–z on p1), not by the document.
```

> The name **term** comes from **full-text indexes**, where the terms are all the words occurring in a document. A term here would be `color:red`.

```
   Partition the index BY THE TERM ITSELF     → useful for RANGE SCANS
                                                 (e.g. asking price)
   Partition the index BY A HASH OF THE TERM  → more EVEN distribution
```

### ⚖️ The trade-off, stated plainly

```
   ┌─────────────────────────────────┬─────────────────────────────────────┐
   │  DOCUMENT-PARTITIONED (local)   │  TERM-PARTITIONED (global)          │
   ├─────────────────────────────────┼─────────────────────────────────────┤
   │  READ   ❌ SCATTER/GATHER across │  READ   ✅ ONE REQUEST to the        │
   │            ALL partitions       │            partition holding that   │
   │            (tail amplification) │            term                     │
   ├─────────────────────────────────┼─────────────────────────────────────┤
   │  WRITE  ✅ touches exactly ONE   │  WRITE  ❌ "a write to a SINGLE      │
   │            partition            │            DOCUMENT may now affect  │
   │                                 │            MULTIPLE PARTITIONS of   │
   │                                 │            the index (every term in │
   │                                 │            the document might be on │
   │                                 │            a different partition,   │
   │                                 │            on a different node)"    │
   └─────────────────────────────────┴─────────────────────────────────────┘

   ⚠️ "In an ideal world the index would always be up to date… However, in a
      term-partitioned index, that would require A DISTRIBUTED TRANSACTION
      ACROSS ALL PARTITIONS AFFECTED BY A WRITE, WHICH IS NOT SUPPORTED IN
      ALL DATABASES."

   ➜ "IN PRACTICE, UPDATES TO GLOBAL SECONDARY INDEXES ARE OFTEN
      ASYNCHRONOUS — if you read the index shortly after a write, THE
      CHANGE YOU JUST MADE MAY NOT YET BE REFLECTED."

   Amazon DynamoDB: global secondary indexes are updated "within a fraction
   of a second in normal circumstances, but may experience LONGER
   PROPAGATION DELAYS in case of faults."
   Also: Riak's search feature; Oracle data warehouse (lets you CHOOSE
   between local and global).
```

### 💻 I built both indexes and counted what each operation touches

```
DOCUMENT-PARTITIONED (local) -- Figure 6-4
   Partition 0: color:black[214] color:red[191,306] make:Dodge[214]
                make:Ford[306] make:Honda[191]
   Partition 1: color:red[768] color:silver[515,893] make:Audi[893]
                make:Ford[515] make:Volvo[768]

   READ 'red car'  -> SCATTER/GATHER over partitions [0, 1]
   WRITE one car   -> touches exactly 1 partition

TERM-PARTITIONED (global) -- Figure 6-5
   Partition 0: color:black[214] color:red[191,306,768] make:Audi[893]
                make:Dodge[214] make:Ford[306,515] make:Honda[191]
   Partition 1: color:silver[515,893] make:Volvo[768]

   READ 'red car'      -> partition 0 ONLY. One request.
   WRITE one red Volvo -> touches index partitions [0, 1]
                          -> needs a DISTRIBUTED TRANSACTION
```

Note `color:red[191, 306, 768]` in the global index — **all three red cars in one entry**, pulled from both primary partitions. That's exactly the thing a local index cannot give you.

### 💻 And I measured the tail amplification the book warns about

```
partitions queried     p50      p99     p999    % of requests hitting a slow shard
────────────────────────────────────────────────────────────────────────────
                 1    10.0    212.2    400.8                              1.0%
                 4    12.1    339.0    417.6                              3.9%
                 8    12.9    371.0    435.4                              7.7%
                16    13.7    391.2    451.4                             14.9%
                64    16.4    429.5    476.6                             47.4%
               256   337.2    459.8    497.8                             92.4%
```

**Look at the p50 column.** At 64 partitions the median is still 16 ms — everything looks fine on your dashboard. At 256 partitions the **median itself jumps to 337 ms**, because 92% of requests now touch at least one slow shard. A 1% per-shard problem becomes a *majority* problem purely through fan-out. This is Chapter 1's Figure 1-5, arriving as a concrete consequence of a partitioning choice.

---
---

## 7. Rebalancing partitions

> Over time, things change: **query throughput increases** (add CPUs), **dataset size increases** (add disks and RAM), **a machine fails** (others take over). **"All of these call for data to be moved from one node to another. The process of moving data around between nodes in the cluster is called REBALANCING."**

### The three minimum requirements

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  ① After rebalancing, THE LOAD (data storage, read and write          ║
   ║     requests) SHOULD BE SHARED FAIRLY between nodes.                  ║
   ║                                                                       ║
   ║  ② WHILE REBALANCING IS HAPPENING, the database should CONTINUE       ║
   ║     ACCEPTING READS AND WRITES.                                       ║
   ║                                                                       ║
   ║  ③ DON'T MOVE MORE DATA THAN NECESSARY, to avoid overloading the      ║
   ║     network.                                                          ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

---

### ❌ Strategy 0 — How NOT to do it: hash mod N

> Perhaps you wondered why we don't just use `mod`. `hash(key) mod 10` returns a number 0–9; with 10 nodes that seems like an easy assignment.

```
   ⚠️ "The problem with the MOD N approach is that IF THE NUMBER OF NODES
      N CHANGES, MOST OF THE KEYS WOULD NEED TO BE MOVED."

   The book's worked example, hash(key) = 123456:

      10 nodes →  123456 mod 10 = 6   ← starts on node 6
      11 nodes →  123456 mod 11 = 3   ← must MOVE to node 3
      12 nodes →  123456 mod 12 = 0   ← must MOVE AGAIN, to node 0

   "That makes rebalancing EXCESSIVELY EXPENSIVE."
```

### 💻 I measured exactly how bad, on 100,000 keys

```
change                   keys moved (mod N)    % moved     ideal minimum
────────────────────────────────────────────────────────────────────────
  10 nodes ->  11 nodes             90,996      91.0%           9.1%
  10 nodes ->  12 nodes             83,124      83.1%          16.7%
  10 nodes ->  20 nodes             49,777      49.8%          50.0%
   4 nodes ->   5 nodes             80,086      80.1%          20.0%
 100 nodes -> 101 nodes             99,037      99.0%           1.0%
```

**Read the last row.** Adding **one** node to a hundred should move **1%** of the data. `mod N` moves **99%**. You would shift essentially your entire dataset across the network to add a single machine.

*(Note the `10 → 20` row is the exception: doubling is the one case `mod N` handles near-optimally, because every key either stays or moves to exactly one new place. That's a special case, not a reprieve.)*

---

### ✅ Strategy 1 — Fixed number of partitions

> **The solution: create MANY MORE PARTITIONS THAN THERE ARE NODES, and assign several partitions to each node.** A database on 10 nodes may be split into **1,000 partitions** from the outset, so ~100 partitions per node.

### 🔷 Figure 6-6 — Adding a node to a cluster with multiple partitions per node

```
   BEFORE REBALANCING (4 nodes)
   ╔══════════════════╗╔══════════════════╗╔══════════════════╗╔══════════════════╗
   ║ NODE 0           ║║ NODE 1           ║║ NODE 2           ║║ NODE 3           ║
   ║ p0 p4 p8 p12 p16 ║║ p1 p5 p9 p13 p17 ║║ p2 p6 p10 p14 p18║║ p3 p7 p11 p15 p19║
   ╚══════════════════╝╚══════════════════╝╚══════════════════╝╚══════════════════╝
            │                   │                   │                   │
            │  a NEW NODE STEALS A FEW PARTITIONS FROM EVERY EXISTING NODE
            ▼                   ▼                   ▼                   ▼
   AFTER REBALANCING (5 nodes)
   ╔══════════════════╗╔══════════════════╗╔══════════════════╗╔══════════════════╗╔══════════════════╗
   ║ NODE 0           ║║ NODE 1           ║║ NODE 2           ║║ NODE 3           ║║ NODE 4  (NEW)    ║
   ║ p0 p8 p12 p16    ║║ p1 p5 p13 p17    ║║ p2 p6 p10 p18    ║║ p3 p7 p11 p15    ║║ p4 p9 p14 p19    ║
   ╚══════════════════╝╚══════════════════╝╚══════════════════╝╚══════════════════╝╚══════════════════╝
        ▲                   ▲                    ▲                                      ▲
        └─ p4 left          └─ p9 left           └─ p14 left                            └─ ONE partition
                                                                                           taken from each

   🔑 THE KEY INSIGHT:
      • ONLY ENTIRE PARTITIONS MOVE between nodes
      • THE NUMBER OF PARTITIONS DOES NOT CHANGE
      • THE ASSIGNMENT OF KEYS TO PARTITIONS DOES NOT CHANGE
      • THE ONLY THING THAT CHANGES IS THE ASSIGNMENT OF PARTITIONS TO NODES
```

> **The change is not immediate** — transferring a large amount of data takes time — **so the OLD assignment is used for any reads and writes that happen while the transfer is in progress.**

> 💡 **A nice bonus:** *"In principle, you can even account for MISMATCHED HARDWARE in your cluster: by assigning MORE PARTITIONS TO MORE POWERFUL NODES, you can force those nodes to take a greater share of the load."*

**Used by:** Riak, **Cassandra since 1.2**, Elasticsearch, Couchbase, Voldemort.

### ⚠️ The one hard decision: choosing the number

```
   The number of partitions is usually FIXED WHEN THE DATABASE IS FIRST
   SET UP, and not changed afterwards. Splitting/merging is possible in
   principle, but "a fixed number of partitions is OPERATIONALLY SIMPLER,
   and so many fixed-partition databases choose not to implement it."

   ┌──────────────────────────────────────────────────────────────────────┐
   │  TOO FEW  →  "the number of partitions configured at the outset is   │
   │              THE MAXIMUM NUMBER OF NODES YOU CAN HAVE"               │
   │              → you must choose it high enough for future growth      │
   │                                                                      │
   │  TOO MANY →  "each partition also has MANAGEMENT OVERHEAD, so it's   │
   │              COUNTERPRODUCTIVE to choose too high a number"          │
   └──────────────────────────────────────────────────────────────────────┘
```

### 💻 The same rebalance, done properly — measured

I implemented the actual steal-a-few-partitions algorithm over 256 fixed partitions:

```
change                          keys moved    % moved    ideal
────────────────────────────────────────────────────────────────
  10 nodes ->  11 nodes              8,912       8.9%     9.1%
  10 nodes ->  12 nodes             16,443      16.4%    16.7%
   4 nodes ->   5 nodes             19,986      20.0%    20.0%
 100 nodes -> 101 nodes                754       0.8%     1.0%
```

**Compare with `mod N`: 91.0% → 8.9%, and 99.0% → 0.8%.** The fixed-partition scheme hits the theoretical minimum almost exactly. The whole trick is the indirection: `key → partition` is frozen forever, and only `partition → node` is allowed to change.

---

### ✅ Strategy 2 — Dynamic partitioning

> A fixed number works well with **hash** partitioning, because the hash spreads keys uniformly. **But for KEY RANGE partitioning, fixed boundaries would be very inconvenient: "if you get the boundaries wrong, you could end up with ALL OF THE DATA IN ONE PARTITION and all of the other partitions being empty."**

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  HBase and RethinkDB CREATE PARTITIONS DYNAMICALLY                    ║
   ║                                                                       ║
   ║  SPLIT:  when a partition grows beyond a configured size              ║
   ║          (HBase default: 10 GB), it SPLITS INTO TWO so that roughly   ║
   ║          HALF THE DATA ends up on each side.                          ║
   ║                                                                       ║
   ║  MERGE:  if lots of data is deleted and a partition shrinks below a   ║
   ║          threshold, it MERGES with an adjacent partition.             ║
   ║                                                                       ║
   ║  ⚑ "This is similar to what happens at the top level of a B-TREE."    ║
   ║    (Chapter 3 page splits, one layer up.)                             ║
   ║                                                                       ║
   ║  After a split, one of the two halves can be TRANSFERRED TO ANOTHER   ║
   ║  NODE to balance load. In HBase, the transfer happens through HDFS.   ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

```
   ✅ THE ADVANTAGE: "the NUMBER OF PARTITIONS ADAPTS TO THE TOTAL DATA
      VOLUME. If there is only a small amount of data, a small number of
      partitions is sufficient, so OVERHEADS ARE SMALL; if there is a huge
      amount of data, the size of each individual partition is LIMITED TO
      A CONFIGURABLE MAXIMUM."

   ❌ THE CAVEAT — THE COLD-START PROBLEM:
      "An EMPTY DATABASE STARTS OFF WITH A SINGLE PARTITION, since there
       is no a priori information about where to draw the boundaries.
       While the dataset is small — until it hits the point at which the
       first partition is split — ALL WRITES HAVE TO BE PROCESSED BY A
       SINGLE NODE WHILE THE OTHER NODES SIT IDLE."

      ➜ MITIGATION: PRE-SPLITTING. HBase and MongoDB allow an initial set
        of partitions to be configured on an empty database.
        ⚠️ For key-range partitioning, this requires that YOU ALREADY KNOW
           WHAT THE KEY DISTRIBUTION IS GOING TO LOOK LIKE.
```

> ⚑ **Dynamic partitioning is not only for key-range data** — it works equally well with hash-partitioned data. **MongoDB since 2.4 supports both key-range and hash partitioning, but splits partitions dynamically in either case.**

### 💻 I simulated splitting, and the cold start is visible in the data

```
inserted 6,000 keys, max partition size 1000
splits occurred at insert #: [1000, 1917, 2117, 3790, 3803, 4221, 4263]
ended with 8 partitions:
   [    0-330381) n= 782
   [330381-405737) n= 804
   [405737-461056) n= 794
   [461056-512403) n= 791
   [512403-558134) n= 729
   [558134-613514) n= 706
   [613514-685560) n= 714
   [685560-1000000) n= 680
```

Two things to notice. First, **the final partitions are nicely balanced (680–804 keys) even though the input was a Gaussian distribution** — boundaries adapted to the data automatically, and the ranges are correspondingly *uneven in width* (the first spans 330,381 key values, the fifth spans 45,731). That's adaptation working.

Second, **the first split happened at insert #1000.** Every one of those first thousand writes went to a single partition on a single node. **That's the cold-start problem, and it's why pre-splitting exists.**

---

### Strategy 3 — Other approaches, and why Cassandra changed

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  CASSANDRA BEFORE 1.2 used consistent hashing with PSEUDO-RANDOM      ║
   ║  partition boundaries (Karger et al.). Rather than many small         ║
   ║  partitions per node, it used ONE BIG PARTITION PER NODE covering a   ║
   ║  large continuous range of hashes.                                    ║
   ║                                                                       ║
   ║  ❌ "This approach suffered from POOR LOAD DISTRIBUTION"               ║
   ║  ❌ "and made it DIFFICULT TO ADD NODES: an existing node had to SPLIT ║
   ║     ITS RANGE to give half of its data to a new node. This EXPENSIVE  ║
   ║     OPERATION was difficult to perform in the background without      ║
   ║     impacting query performance."                                     ║
   ║                                                                       ║
   ║  ➜ Replaced with the FIXED-NUMBER-OF-PARTITIONS approach.             ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ➜ THE BOOK'S BOTTOM LINE:
     "In practice, THE MOST WIDELY-USED PARTITIONING MODELS ARE EITHER
      HASHING WITH A FIXED NUMBER OF PARTITIONS, OR DYNAMIC PARTITIONING
      BY KEY RANGE (when range queries are required)."
```

### 💻 I measured the "poor load distribution" claim directly

100,000 keys across 10 nodes, varying tokens (vnodes) per node:

```
vnodes per node      min      max   max/min   stdev%
─────────────────────────────────────────────────────
              1      403   20,085     49.84    68.0%
              4    4,666   21,601      4.63    51.0%
             16    7,309   14,515      1.99    24.0%
             64    7,322   11,681      1.60    12.8%
            256    9,427   10,993      1.17     5.7%
```

**With one token per node, one node got 403 keys and another got 20,085 — a 50× imbalance.** Random boundaries on a ring simply don't divide evenly with few samples. Adding more vnodes averages the randomness out, and **by 256 vnodes you're within 17%**. But notice what "many vnodes per node" *is*: it's the fixed-number-of-partitions approach wearing a ring-shaped hat. The two converge.

---

### Operations: automatic or manual rebalancing?

```
   FULLY AUTOMATIC ◄─────────────── a gradient ───────────────► FULLY MANUAL
   the system decides                                    an administrator
   when to move partitions,                              explicitly configures
   no administrator involved                             the assignment

                    ▲
                    │
   Couchbase, Riak and Voldemort sit in the middle: they GENERATE A
   SUGGESTED ASSIGNMENT AUTOMATICALLY, but REQUIRE AN ADMINISTRATOR TO
   COMMIT IT before it takes effect.
```

### ⚠️ The cascading-failure scenario — the best argument in the section

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "Fully automated rebalancing CAN BE UNPREDICTABLE. Rebalancing is an ║
   ║   EXPENSIVE OPERATION, because it requires RE-ROUTING REQUESTS and    ║
   ║   MOVING A LARGE AMOUNT OF DATA. If not done carefully, this can      ║
   ║   OVERLOAD THE NETWORK OR THE NODES."                                 ║
   ║                                                                       ║
   ║  DANGEROUS IN COMBINATION WITH AUTOMATIC FAILURE DETECTION:           ║
   ║                                                                       ║
   ║    one node is OVERLOADED and temporarily slow to respond             ║
   ║                        │                                              ║
   ║                        ▼                                              ║
   ║    other nodes conclude it is DEAD                                    ║
   ║                        │                                              ║
   ║                        ▼                                              ║
   ║    they AUTOMATICALLY REBALANCE to move load away from it             ║
   ║                        │                                              ║
   ║                        ▼                                              ║
   ║    this puts ADDITIONAL LOAD on the other nodes and the network       ║
   ║                        │                                              ║
   ║                        ▼                                              ║
   ║    potentially OVERLOADING MORE NODES ──► CASCADING FAILURE ──┐       ║
   ║                        ▲                                       │       ║
   ║                        └───────────────────────────────────────┘       ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  ➜ "IT CAN BE A GOOD THING TO HAVE A HUMAN IN THE LOOP for            ║
   ║     rebalancing. It's SLOWER than performing it fully automatically,  ║
   ║     but it can HELP PREVENT OPERATIONAL SURPRISES."                   ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

**This is the same conclusion as Chapter 5's failover section**, reached independently: the automation is correct in the normal case and catastrophic in the degraded case, because *slow* and *dead* are indistinguishable from the outside.

---
---

## 8. Request routing

> **"When a client wants to make a request, HOW DOES IT KNOW WHICH NODE TO CONNECT TO?"** As partitions are rebalanced, the assignment changes. Somebody must answer: **"If I want to read or write the key 'foo', which IP address and port number do I need to connect to?"**

> This is an instance of a more general problem called **service discovery**, which isn't limited to databases. **"Any piece of software accessible over a network has this problem, especially if it is aiming for high availability."**

### 🔷 Figure 6-7 — Three ways of routing a request

```
   ═══ APPROACH 1 ═══════════  ═══ APPROACH 2 ═══════════  ═══ APPROACH 3 ═══════════
       ┌────────┐                  ┌────────┐                  ┌────────┐
       │ CLIENT │                  │ CLIENT │                  │ CLIENT │░░
       └───┬────┘                  └───┬────┘                  └───┬────┘░░
    get "foo"  │                 get "foo" │                       │  ░ CLIENT IS
    choose node 0                          ▼                 get "foo"  PARTITION-
    RANDOMLY   │                  ┌─────────────┐            connect    AWARE
           ▼                      │ ROUTING TIER│░░          DIRECTLY
   ┌────┬────┬────┐               └──────┬──────┘░           to node 2
   │ n0 │ n1 │ n2 │░                     │  ░ knows the           │
   └─┬──┴────┴────┘                      │    mapping             ▼
     │       "foo" lives          "foo" lives on node 2   ┌────┬────┬────┐
     │        on node 2                  │                │ n0 │ n1 │ n2 │
     └──── forwards ──────►         ┌────┴────┬────┐      └────┴────┴─┬──┘
              ┌────┐            ┌───┤ n0 │ n1 │ n2 │                  │
              │"foo"│           │   └────┴────┴──┬─┘               ┌──▼──┐
              └────┘            │           ┌────▼─┐               │"foo"│
                                            │"foo" │               └─────┘
   Any node can be contacted;   └───────────└──────┘
   it FORWARDS if it doesn't    A partition-aware        No intermediary
   own the partition            LOAD BALANCER that       at all
                                handles no requests itself

   ░░ = the knowledge of which partition is assigned to which node
```

### 🔑 The real problem, regardless of approach

> **"In all cases, the KEY PROBLEM is: how does the component making the routing decision LEARN ABOUT CHANGES in the assignment of partitions to nodes?"**
>
> This is challenging **"because it is important that ALL PARTICIPANTS AGREE — otherwise requests would be sent to the wrong nodes. There are protocols for achieving CONSENSUS in a distributed system, but they are HARD TO IMPLEMENT CORRECTLY"** (Chapter 9).

### 🔷 Figure 6-8 — Using ZooKeeper to track partition assignment

```
      ┌────────┐
      │ CLIENT │                    ╔═══════════════════════════════════════════╗
      └───┬────┘                    ║  ZOOKEEPER — the authoritative mapping    ║
   get "Danube"                     ╠═══════════════╤═══════════╤═══════════════╣
          │                         ║ KEY RANGE     │ PARTITION │ NODE  IP      ║
          ▼          subscribes to  ║ A-ak—Bayes    │ partition0│ node0 …100    ║
   ┌─────────────┐   this info      ║ Bayeu—Ceanoth.│ partition1│ node1 …101    ║
   │ ROUTING TIER│◄────────────────►║ Ceara—Deluc   │ partition2│ node2 …102    ║
   └──────┬──────┘   and is NOTIFIED║ Delusion—Fren.│ partition3│ node0 …100    ║
          │          on any change  ║ Freon—Holderl.│ partition4│ node1 …101    ║
          │                         ║ Holderness—Kr.│ partition5│ node2 …102    ║
    ┌─────┴────┬──────┐             ║ Krasnokamsk—M.│ partition6│ node0 …100    ║
    ▼          ▼      ▼             ║ Menage—Ottawa │ partition7│ node1 …101    ║
  ┌────┐    ┌────┐  ┌────┐          ║ Otter—Rethimn.│ partition8│ node2 …102    ║
  │ n0 │    │ n1 │  │ n2 │══════════╣ Reti—Solovets │ partition9│ node0 …100    ║
  └────┘    └────┘  └────┘  each    ║ Solovyov—Truck│ partition10│node1 …101    ║
                            node    ║ Trudeau—Zywiec│ partition11│node2 …102    ║
                            REGISTERS╚══════════════╧═══════════╧═══════════════╝
                            itself
```

> **"Whenever a partition changes ownership, or a node is added or removed, ZooKeeper NOTIFIES the routing tier so that it can keep its routing information up to date."**

### Who does what

| System | Approach |
|---|---|
| **LinkedIn Espresso** | **Helix** for cluster management (which uses ZooKeeper), with a routing tier |
| **MongoDB** | Similar architecture, but its **own config server** implementation + **`mongos`** daemons as the routing tier |
| **HBase, SolrCloud, Kafka** | **ZooKeeper** to track partition assignment |
| **Cassandra, Riak** | 🔀 **A GOSSIP PROTOCOL among the nodes** to disseminate and agree on cluster state. Requests go to any node, which forwards (approach 1). **"This puts MORE COMPLEXITY IN THE DATABASE NODES, but AVOIDS THE DEPENDENCY on an external coordination service."** |
| **Couchbase** | **Does not rebalance automatically**, which **simplifies the agreement protocol**. Uses a routing tier called **`moxi`** that learns changes via a management connection |

> 💡 **A practical closing note:** when using a routing tier or contacting a random node, clients still need the **IP addresses** of machines. **"However, those addresses are NOT AS FAST-CHANGING as the assignment of partitions to nodes, so IT IS OFTEN SUFFICIENT TO USE DNS for this purpose."**

---

## 9. Parallel query execution

> So far we've focused on very simple queries reading or writing a single key, plus scatter/gather. **"This is about the level of access supported by most NoSQL distributed data stores."**

```
   MPP ("MASSIVELY PARALLEL PROCESSING") relational products, often used
   for ANALYTICS, are much more sophisticated.

   A typical data warehouse query contains SEVERAL JOINS, FILTERING,
   GROUPING and AGGREGATION.

   ┌──────────────────────────────────────────────────────────────────────┐
   │  The MPP QUERY OPTIMIZER breaks the complex query into a number of   │
   │  EXECUTION STAGES AND PARTITIONS, MANY OF WHICH CAN BE EXECUTED IN   │
   │  PARALLEL on different nodes.                                        │
   │                                                                      │
   │  "Queries that involve SCANNING OVER LARGE PARTS OF THE DATASET      │
   │   particularly benefit from such parallel execution."                │
   └──────────────────────────────────────────────────────────────────────┘

   → A specialized topic that "gets a lot of commercial interest"
     given the business importance of analytics. More in Chapter 10.
```

---

## 10. Chapter Summary

> **"The main goal of partitioning is to SPREAD THE DATA AND THE QUERY LOAD EVENLY across multiple machines, AVOIDING HOT SPOTS."** This requires choosing a scheme appropriate to your data, **and rebalancing as nodes are added or removed.**

### The two partitioning approaches

```
   ╔═══════════════════════════════════════╦═══════════════════════════════════════╗
   ║  KEY RANGE PARTITIONING               ║  HASH PARTITIONING                    ║
   ╠═══════════════════════════════════════╬═══════════════════════════════════════╣
   ║  Keys are SORTED; a partition owns    ║  A hash function is applied to each   ║
   ║  all keys from some minimum up to     ║  key; a partition owns a RANGE OF     ║
   ║  some maximum.                        ║  HASHES.                              ║
   ║                                       ║                                       ║
   ║  ✅ EFFICIENT RANGE QUERIES            ║  ❌ DESTROYS THE ORDERING of keys,     ║
   ║  ❌ RISK OF HOT SPOTS if the app often ║     making range queries inefficient  ║
   ║     accesses keys close together in   ║  ✅ MAY DISTRIBUTE LOAD MORE EVENLY    ║
   ║     sorted order                      ║                                       ║
   ║                                       ║                                       ║
   ║  REBALANCE: DYNAMICALLY, by SPLITTING ║  REBALANCE: create a FIXED NUMBER of  ║
   ║  a range into two sub-ranges when a   ║  partitions in advance, assign        ║
   ║  partition gets too big               ║  several to each node, and MOVE       ║
   ║                                       ║  ENTIRE PARTITIONS when nodes change  ║
   ╚═══════════════════════════════════════╩═══════════════════════════════════════╝

   ➜ "HYBRID APPROACHES ARE ALSO POSSIBLE, for example with a COMPOUND
     KEY: using ONE PART of the key to IDENTIFY THE PARTITION, and
     ANOTHER PART for the SORT ORDER."   (= the Cassandra model)
```

### The two secondary-index approaches

```
   DOCUMENT-PARTITIONED (local)       TERM-PARTITIONED (global)
   ───────────────────────────        ─────────────────────────
   indexes stored in the SAME         indexes partitioned SEPARATELY,
   partition as the primary key       using the INDEXED VALUES
   and value

   ✅ only a SINGLE PARTITION needs    ❌ SEVERAL PARTITIONS of the index
      to be updated on WRITE             need updating on WRITE
   ❌ a READ requires SCATTER/GATHER   ✅ a READ can be served from a
      across ALL partitions              SINGLE PARTITION
```

### And the closing hook into Chapter 7

> **"By design, every partition operates MOSTLY INDEPENDENTLY — that's what allows a partitioned database to scale to multiple machines. However, OPERATIONS THAT NEED TO WRITE TO SEVERAL PARTITIONS CAN BE DIFFICULT TO REASON ABOUT: for example, what happens if the write to one partition succeeds, but another fails? In the next chapter we will turn to the topic of TRANSACTIONS."**

---

# 11. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.6 — PARTITIONING                                                     ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  AKA: shard (MongoDB/ES/Solr) · region (HBase) · tablet (BigTable) ·          ║
║       vnode (Cassandra/Riak) · vBucket (Couchbase)                            ║
║  ⚠️ NOT the same as a NETWORK partition (netsplit) — unrelated meaning, ch.8.  ║
║  GOAL = SCALABILITY. Each partition is "a small database of its own."         ║
║  Combined with replication: each node LEADS some partitions, FOLLOWS others.  ║
║  The two choices are ORTHOGONAL — pick them independently.                    ║
║                                                                               ║
║  SKEW = unfair distribution.  HOT SPOT = partition with disproportionate load.║
║  Random assignment avoids skew but you'd have to QUERY EVERY NODE to read.    ║
║                                                                               ║
║  ── KEY RANGE ──────────────────────────────────────────────────────────────  ║
║  Contiguous ranges, like encyclopedia volumes. Boundaries must ADAPT TO THE   ║
║  DATA (vol 1 = A–B, vol 12 = T–Z). BigTable, HBase, RethinkDB, MongoDB<2.4.   ║
║  ✅ sorted within a partition → RANGE SCANS, concatenated-index tricks         ║
║  ❌ THE TIMESTAMP HOT SPOT: all of today's writes hit today's partition.       ║
║     FIX: put something else FIRST in the key, e.g. (sensor_name, timestamp).  ║
║     COST: one range query PER SENSOR.                                         ║
║  MEASURED: naive A–Z split on real surname data = 16.9x skew.                 ║
║            adaptive boundaries = 1.00x. hash = 1.08x KNOWING NOTHING.         ║
║                                                                               ║
║  ── HASH OF KEY ────────────────────────────────────────────────────────────  ║
║  Hash need NOT be cryptographic (Cassandra/MongoDB: MD5; Voldemort: FNV).     ║
║  Partition owns a RANGE OF HASHES.                                            ║
║  ❌ ORDERING DESTROYED → no efficient range queries (Riak/Couchbase/Voldemort  ║
║     don't support them; MongoDB sends them to ALL partitions).                ║
║  ⚠️ "CONSISTENT HASHING" — the book says STOP USING THE TERM. Nothing to do    ║
║     with replica or ACID consistency; rarely used in DBs; say HASH            ║
║     PARTITIONING.                                                             ║
║  CASSANDRA COMPROMISE: compound key ((user_id), timestamp) — hash only the    ║
║     FIRST part for the partition, use the rest as a sort key within it.       ║
║     → elegant one-to-many modelling.                                          ║
║                                                                               ║
║  ── HOT KEYS ───────────────────────────────────────────────────────────────  ║
║  A CELEBRITY defeats hashing: hash(same id) is always the same.               ║
║  MEASURED: 90% of writes on one key → 97x imbalance even WITH hashing.        ║
║  FIX: append a 2-digit random suffix → 100 keys → measured 2.1x. But now      ║
║       EVERY READ fans out to 100 keys, and you must TRACK WHICH KEYS ARE SPLIT║
║  "Most data systems cannot automatically compensate — it's the APPLICATION'S  ║
║   responsibility."                                                            ║
║                                                                               ║
║  ── SECONDARY INDEXES: they DON'T MAP NEATLY TO PARTITIONS ─────────────────  ║
║  DOCUMENT-PARTITIONED (LOCAL): index lives with its documents.                ║
║     ✅ WRITE touches 1 partition   ❌ READ = SCATTER/GATHER over ALL            ║
║     MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB.               ║
║     ⚠️ TAIL LATENCY AMPLIFICATION — MEASURED: 1% slow shards →                 ║
║        8 partitions = 7.7% of requests slow; 256 partitions = 92.4%, and the  ║
║        MEDIAN jumps from 10ms to 337ms.                                       ║
║  TERM-PARTITIONED (GLOBAL): index partitioned by the indexed VALUE.           ║
║     ✅ READ hits 1 partition   ❌ WRITE touches MANY index partitions           ║
║     → would need a DISTRIBUTED TRANSACTION → so in practice ASYNCHRONOUS      ║
║       (DynamoDB GSIs: "a fraction of a second… longer under faults").         ║
║     Partition by term itself → range scans; by hash of term → even load.      ║
║                                                                               ║
║  ── REBALANCING ────────────────────────────────────────────────────────────  ║
║  REQUIREMENTS: fair load after · KEEP SERVING reads+writes during · move NO   ║
║     MORE DATA THAN NECESSARY.                                                 ║
║  ❌ HASH MOD N — MEASURED: 100→101 nodes moves 99.0% of keys (ideal 1.0%).     ║
║  ✅ FIXED NUMBER OF PARTITIONS — make FAR MORE partitions than nodes (e.g.     ║
║     1000 partitions on 10 nodes). A new node STEALS A FEW FROM EACH.          ║
║     🔑 key→partition NEVER CHANGES; only partition→node does.                  ║
║     MEASURED: 100→101 nodes moves 0.8% (ideal 1.0%). 10→11: 8.9% vs 91%.      ║
║     Old assignment serves traffic DURING the transfer. More partitions to     ║
║     beefier nodes = weighted load. Riak, Cassandra≥1.2, ES, Couchbase,        ║
║     Voldemort. ⚠️ partition count is usually FIXED AT SETUP = your MAX NODE    ║
║     COUNT; but too many costs management overhead.                            ║
║  ✅ DYNAMIC PARTITIONING — SPLIT when a partition exceeds a size (HBase: 10GB),║
║     MERGE when it shrinks. "Similar to the top level of a B-tree."            ║
║     ✅ count adapts to data volume  ❌ COLD START: an empty DB has ONE          ║
║     partition, so early writes hit ONE NODE → PRE-SPLITTING (needs you to     ║
║     know the key distribution). Works for hash data too (MongoDB≥2.4).        ║
║  CASSANDRA PRE-1.2 used one big random range per node: MEASURED 50x imbalance ║
║     with 1 token/node; 1.17x at 256. That's why they switched.                ║
║  AUTO vs MANUAL: full automation + automatic failure detection = a slow node  ║
║     is declared dead → rebalance → more load → CASCADING FAILURE.             ║
║     ➜ "It can be a good thing to have a HUMAN IN THE LOOP."                   ║
║                                                                               ║
║  ── REQUEST ROUTING (= service discovery) ──────────────────────────────────  ║
║  ① any node, forwards if not the owner  ② a partition-aware ROUTING TIER      ║
║  ③ a PARTITION-AWARE CLIENT, connects directly                                ║
║  The hard part is AGREEMENT on the mapping → a CONSENSUS problem (ch.9).      ║
║  ZOOKEEPER holds the authoritative map and NOTIFIES subscribers on change:    ║
║     Espresso (via Helix), HBase, SolrCloud, Kafka. MongoDB uses its own       ║
║     config servers + mongos. CASSANDRA/RIAK use a GOSSIP PROTOCOL instead —   ║
║     more complexity in the nodes, no external dependency. Couchbase: moxi,    ║
║     and NO automatic rebalancing (which simplifies agreement).                ║
║  Node IPs change slowly → DNS is usually enough for those.                    ║
║                                                                               ║
║  MPP analytics DBs go far beyond this: the optimizer splits a complex query   ║
║  into STAGES AND PARTITIONS executed in parallel (ch.10).                     ║
║                                                                               ║
║  ➜ NEXT: partitions are independent by design, so what happens when a write   ║
║    to one partition SUCCEEDS and another FAILS? → TRANSACTIONS (ch.7).        ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 12. ✅ Test yourself

1. **Why doesn't hashing the key protect you from a celebrity hot spot?**
   → Because the hash of a given key is always the same value. Hashing spreads *different* keys evenly; it does nothing when the load concentrates on *one* key. My simulation showed 97× imbalance despite hash partitioning.

2. **You partition a sensor database by timestamp. Writes are crushing one node. What's wrong and how do you fix it?**
   → Data is written as it happens, so every write lands in the current time range. Make the sensor name the first element of the key so writes spread across sensors. You pay for it at read time: fetching a time range across sensors now needs one range query per sensor.

3. **Why does the book tell you to stop saying "consistent hashing"?**
   → Three reasons. "Consistent" has nothing to do with replica or ACID consistency. The technique was designed for CDN caches, not databases. And it works poorly for databases in practice — Cassandra abandoned it in 1.2. Say "hash partitioning."

4. **Adding one node to a 100-node cluster. How much data moves under `hash mod N`, and how much should move?**
   → 99% moves; about 1% should. My measurement gave 99,037 of 100,000 keys. The fixed-partition approach moved 754 keys — 0.8%, essentially the theoretical minimum.

5. **What exactly stays fixed in the "fixed number of partitions" scheme?**
   → The mapping from *key to partition*. Only the mapping from *partition to node* changes during a rebalance. That one layer of indirection is the entire difference between moving 91% of your data and moving 9%.

6. **When would you pick a term-partitioned index over a document-partitioned one?**
   → When reads dominate and you can tolerate asynchronous index updates. Term-partitioned reads hit a single partition instead of scattering; the cost is that a single document write touches several index partitions, which would require a distributed transaction to do synchronously.

7. **Your search feature has a fine p50 but terrible user complaints. You have 128 shards. What's likely happening?**
   → Scatter/gather tail amplification. The end-user request waits for the slowest shard, so a small per-shard slow rate becomes a large per-request one. At 256 shards my simulation had 92% of requests touching a slow shard, and the *median* itself degraded to 337 ms.

8. **Why does an empty HBase table start slow, and what's the mitigation?**
   → With dynamic partitioning there's no information about where to draw boundaries, so the database starts with a single partition. Until the first split, every write goes to one node. Pre-splitting fixes it, but requires you to know the key distribution in advance.

9. **Why can automatic rebalancing plus automatic failure detection be dangerous together?**
   → Because a slow node is indistinguishable from a dead one. An overloaded node gets declared dead, triggering an expensive rebalance that adds network and node load, which can overload more nodes and cascade. It's the same "slow vs dead" trap as automatic failover in Chapter 5.

10. **Cassandra uses gossip instead of ZooKeeper for routing metadata. What's the trade?**
    → Gossip avoids an external coordination-service dependency, which is one fewer system to run and to fail. The cost is significantly more complexity inside the database nodes themselves, since they must implement dissemination and agreement on cluster state.

---

*All quoted material, figures, examples and statistics are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 6. Diagrams have been redrawn in ASCII from the book's originals. The simulations and all measured outputs are supplementary material I added; every number quoted was produced by running the accompanying code.*
