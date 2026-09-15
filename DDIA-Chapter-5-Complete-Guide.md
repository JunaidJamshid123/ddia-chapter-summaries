# DDIA — Part II: Distributed Data
# Chapter 5: Replication
### Complete study guide — theory, every diagram redrawn, and working simulations

> *For a successful technology, reality must take precedence over public relations, for nature cannot be fooled.*
> — Richard Feynman, Rogers Commission Report (1986)

> *The major difference between a thing that might go wrong and a thing that cannot possibly go wrong is that when a thing that cannot possibly go wrong goes wrong it usually turns out to be impossible to get at or repair.*
> — Douglas Adams, *Mostly Harmless* (1992)

Those two epigraphs set the tone for the entire second half of the book. Part I assumed one machine. **From here on, machines fail, networks lie, and clocks disagree.**

---

# PART II INTRODUCTION — DISTRIBUTED DATA

## 0. Why distribute at all?

> In Part I we discussed aspects of data systems that apply when data is stored on a single machine. Now we move up a level and ask: **what happens if multiple machines are involved in storage and retrieval of data?**

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │  ① SCALABILITY                                                      │
   │     data volume, read load or write load grows bigger than a        │
   │     single machine can handle → spread the load                     │
   ├─────────────────────────────────────────────────────────────────────┤
   │  ② FAULT TOLERANCE / HIGH AVAILABILITY                              │
   │     continue working even if one machine (or several, or the        │
   │     network, or an ENTIRE DATACENTER) goes down                     │
   ├─────────────────────────────────────────────────────────────────────┤
   │  ③ LATENCY                                                          │
   │     users around the world served from a nearby datacenter, so      │
   │     packets don't travel halfway around the planet                  │
   └─────────────────────────────────────────────────────────────────────┘
```

## 1. Three architectures for scaling

```
   ╔═══════════════════════╦═══════════════════════╦═══════════════════════╗
   ║  SHARED-MEMORY        ║  SHARED-DISK          ║  SHARED-NOTHING       ║
   ║  (vertical / scale up)║                       ║  (horizontal/scale out)║
   ╠═══════════════════════╬═══════════════════════╬═══════════════════════╣
   ║  ┌─────────────────┐  ║  ┌────┐ ┌────┐ ┌────┐ ║  ┌────┐ ┌────┐ ┌────┐ ║
   ║  │ CPU CPU CPU CPU │  ║  │CPU │ │CPU │ │CPU │ ║  │CPU │ │CPU │ │CPU │ ║
   ║  │ RAM RAM RAM RAM │  ║  │RAM │ │RAM │ │RAM │ ║  │RAM │ │RAM │ │RAM │ ║
   ║  │ disk disk disk  │  ║  └──┬─┘ └──┬─┘ └──┬─┘ ║  │disk│ │disk│ │disk│ ║
   ║  │ ONE OS          │  ║     └──────┼──────┘   ║  └──┬─┘ └──┬─┘ └──┬─┘ ║
   ║  └─────────────────┘  ║      ┌─────▼─────┐    ║     └───network───┘   ║
   ║  fast interconnect;   ║      │ SHARED    │    ║                       ║
   ║  any CPU reaches any  ║      │ DISK ARRAY│    ║  each node uses its   ║
   ║  memory or disk       ║      │ (NAS/SAN) │    ║  OWN CPU, RAM, disk   ║
   ║                       ║      └───────────┘    ║  independently        ║
   ╠═══════════════════════╬═══════════════════════╬═══════════════════════╣
   ║ ❌ COST IS SUPER-      ║ ❌ CONTENTION and the ║ ✅ no special hardware ║
   ║    LINEAR: 2x the      ║    OVERHEAD OF        ║ ✅ best price/perf     ║
   ║    machine costs       ║    LOCKING limit      ║    machines           ║
   ║    MUCH more than 2x   ║    scalability        ║ ✅ multi-region → low  ║
   ║ ❌ 2x the size can't    ║                       ║    latency + survive  ║
   ║    necessarily handle  ║ used for SOME data    ║    losing a whole DC  ║
   ║    2x the load         ║ warehousing workloads ║                       ║
   ║ ❌ LIMITED to a SINGLE  ║                       ║ ❌ additional          ║
   ║    GEOGRAPHIC LOCATION ║                       ║    complexity for     ║
   ║ ⚠️ hot-swap components  ║                       ║    applications       ║
   ║    give limited fault  ║                       ║ ❌ may limit data-     ║
   ║    tolerance           ║                       ║    model expressivity ║
   ╚═══════════════════════╩═══════════════════════╩═══════════════════════╝
```

> 📖 **NUMA footnote worth knowing:** even in a large shared-memory machine, "although any CPU can access any part of memory, some banks of memory are **closer** to one CPU than others (**non-uniform memory access**)." To use it efficiently, each CPU should mostly access nearby memory — **"which means that partitioning is still required, even when ostensibly running on one machine."**

### Why the book focuses on shared-nothing

> Not because they are necessarily the best choice for every use case, **but rather because they require the most caution from you, the application developer.** If your data is distributed across multiple nodes, you need to be aware of the constraints and trade-offs — **the database cannot magically hide these from you.**

⚠️ **And a genuinely humbling caveat:** *"In some cases, a simple single-threaded program can perform significantly better than a cluster with over 100 CPU cores."* (The reference is McSherry et al., *"Scalability! But at what COST?"* — required reading before you reach for a cluster.)

## 2. Replication vs partitioning

```
   ┌───────────────────────────────────┬───────────────────────────────────┐
   │  REPLICATION (Chapter 5)          │  PARTITIONING (Chapter 6)         │
   ├───────────────────────────────────┼───────────────────────────────────┤
   │  Keeping a COPY OF THE SAME DATA  │  Splitting a big database into    │
   │  on several different nodes,      │  smaller subsets called           │
   │  potentially in different         │  PARTITIONS, so different         │
   │  locations.                       │  partitions go to different       │
   │                                   │  nodes. (Also: SHARDING.)         │
   │  → REDUNDANCY: if some nodes are  │                                   │
   │    unavailable, data is still     │  → lets the dataset exceed what   │
   │    served from the rest           │    one machine can hold           │
   │  → can also improve performance   │                                   │
   └───────────────────────────────────┴───────────────────────────────────┘

   "These are SEPARATE MECHANISMS, but they OFTEN GO HAND IN HAND."
```

### 🔷 Figure II-1 — Two partitions, two replicas each

```
   ╔═══════════════════════════════════╗   ╔═══════════════════════════════════╗
   ║  PARTITION 1, REPLICA 1           ║   ║  PARTITION 2, REPLICA 1           ║
   ║ ┌──────┬──────┬──────┐            ║   ║ ┌──────┬──────┬──────┐            ║
   ║ │ 136  │ 211  │ 377  │            ║   ║ │ 629  │ 696  │ 858  │            ║
   ║ │Four  │Johan-│Where-│            ║   ║ │Die   │Where-│We    │            ║
   ║ │score │nes   │as    │            ║   ║ │Würde │as    │hold  │            ║
   ║ └──────┴──────┴──────┘            ║   ║ └──────┴──────┴──────┘            ║
   ╚═══════════════╤═══════════════════╝   ╚═══════════════╤═══════════════════╝
                   │ copy of the same data                 │ copy of the same data
   ╔═══════════════▼═══════════════════╗   ╔═══════════════▼═══════════════════╗
   ║  PARTITION 1, REPLICA 2           ║   ║  PARTITION 2, REPLICA 2           ║
   ║ ┌──────┬──────┬──────┐            ║   ║ ┌──────┬──────┬──────┐            ║
   ║ │ 136  │ 211  │ 377  │            ║   ║ │ 629  │ 696  │ 858  │            ║
   ║ │Four  │Johan-│Where-│            ║   ║ │Die   │Where-│We    │            ║
   ║ │score │nes   │as    │            ║   ║ │Würde │as    │hold  │            ║
   ║ └──────┴──────┴──────┘            ║   ║ └──────┴──────┴──────┘            ║
   ╚═══════════════════════════════════╝   ╚═══════════════════════════════════╝

   ── horizontal axis = PARTITIONING (different data) ──►
   │
   └ vertical axis = REPLICATION (same data, copied)
```

---
---

# CHAPTER 5 — REPLICATION

## 3. The setup

**Three reasons to replicate** (same list as Part II, restated):

```
   • keep data geographically close to users        → reduce LATENCY
   • continue working even if parts have failed     → increase AVAILABILITY
   • scale out the machines serving read queries    → increase READ THROUGHPUT
```

**The assumption for this chapter:** the dataset is small enough that **each machine can hold a copy of the entire dataset.** (Chapter 6 relaxes that.)

### 🔑 The one sentence that frames everything

> **If the data that you're replicating does not change over time, then replication is easy:** you just copy the data to every node once, and you're done. **All of the difficulty in replication lies in handling CHANGES to replicated data.**

### The three algorithms

```
              ┌──────────────────────────────────────────────┐
              │  Almost ALL distributed databases use one of │
              │  these three. All have pros and cons.        │
              └──────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
   ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
   │ SINGLE-      │        │ MULTI-       │        │ LEADERLESS   │
   │ LEADER       │        │ LEADER       │        │              │
   ├──────────────┤        ├──────────────┤        ├──────────────┤
   │ one node     │        │ several      │        │ NO leader.   │
   │ accepts      │        │ nodes accept │        │ Client writes│
   │ writes;      │        │ writes; they │        │ to SEVERAL   │
   │ others       │        │ replicate to │        │ nodes and    │
   │ follow       │        │ each other   │        │ reads from   │
   │              │        │              │        │ SEVERAL      │
   │ ✅ easy to    │        │ ⚠️ conflicts  │        │ ⚠️ conflicts  │
   │   understand │        │   possible   │        │   possible   │
   │ ✅ no conflict│        │ ✅ robust to  │        │ ✅ robust to  │
   │   resolution │        │   faults     │        │   faults     │
   └──────────────┘        └──────────────┘        └──────────────┘
```

> ⚑ Replication of databases is **an old topic — the principles haven't changed much since they were studied in the 1970s, because the fundamental constraints of networks have remained the same.** But mainstream use of distributed databases is recent, so **"there has been a lot of misunderstanding around issues such as eventual consistency."**

---

## 4. Leaders and Followers

Each node storing a copy is a **replica**. The question: **how do we ensure all the data ends up on all the replicas?**

> Every write needs to be processed by **every** replica, otherwise they'd diverge. The most common solution is **leader-based replication** (also: *active/passive*, *master-slave*).

```
   1. ONE replica is designated the LEADER (master, primary).
      Clients send ALL writes to the leader, which first writes to its
      own local storage.

   2. The others are FOLLOWERS (read replicas, slaves, hot standbys).
      Whenever the leader writes locally, it ALSO sends the data change
      to every follower as part of a REPLICATION LOG or CHANGE STREAM.
      Each follower applies the writes IN THE SAME ORDER as the leader.

   3. A client can READ from the leader OR any follower.
      But WRITES ARE ONLY ACCEPTED ON THE LEADER.
```

### 🔷 Figure 5-1 — Leader-based replication

```
   User 1234 configures                                        ┌──────────────┐
   new profile picture                                    ┌───►│  FOLLOWER    │
          │                                               │    │  REPLICA     │
          │ read-write queries                            │    └──────┬───────┘
          │                                               │           │
          ▼                                               │           │ read-only
   ┌──────────────────────┐    replication stream         │           ▼
   │  update users        │    ┌──────────────────────┐   │   select * from users
   │  set picture_url =   │    │  DATA CHANGE         │   │   where user_id = 1234
   │      'me-new.jpg'    │    │  table:       users  │───┤
   │  where user_id=1234  │    │  primary key: 1234   │   │    ┌──────────────┐
   └──────────┬───────────┘    │  column:  picture_url│   └───►│  FOLLOWER    │
              │                │  old_value: me-old   │        │  REPLICA     │
              ▼                │  new_value: me-new   │        └──────┬───────┘
      ┌───────────────┐        │  transaction: 987654 │               │
      │    LEADER     │───────►└──────────────────────┘               ▼
      │    REPLICA    │                                          User 2345 views
      └───────────────┘                                          user 1234's profile
```

**Who uses it — note how universal it is:**

| Category | Systems |
|---|---|
| Relational | **PostgreSQL** (9.0+), **MySQL**, **Oracle Data Guard**, **SQL Server AlwaysOn Availability Groups** |
| Non-relational | **MongoDB**, **RethinkDB**, **Espresso** |
| Message brokers | **Kafka**, **RabbitMQ** highly available queues |
| Block devices | **DRBD**, some network file systems |

> 📖 *Footnote on terminology:* in PostgreSQL, **hot standby** accepts reads from clients, whereas a **warm standby** processes changes but serves no queries. For the book's purposes the difference doesn't matter.

---

## 5. Synchronous vs asynchronous replication

### 🔷 Figure 5-2 — One synchronous and one asynchronous follower

```
                update users set picture_url = 'me-new.jpg'
                where user_id = 1234
                       │                                            TIME ──────►
   User 1234  ─────────┼─────────────────────────────────────────────────► ok
                       │                                                   ▲
                       ▼                                                   │
   Leader      ────────●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━●───►
                       │  ╲ data change      ╱ ok      │
                       │   ╲               ╱           │
                       │    ▼            ╱             │
   Follower 1  ────────┼─────●──────────●───────────────┼──────────────────────►
                       │   SYNCHRONOUS: leader WAITS for this ok
                       │
                       │      ╲ data change
                       │        ╲                              ╲ ok
                       │          ▼                              ▼
   Follower 2  ────────┼────────────────────────────────●─────────●─────────────►
                       │   ASYNCHRONOUS: leader sends and does NOT wait
                       │
                       └─── substantial delay before follower 2 processes it
```

```
   ╔═══════════════════════════════╦═══════════════════════════════════════╗
   ║  SYNCHRONOUS                  ║  ASYNCHRONOUS                         ║
   ╠═══════════════════════════════╬═══════════════════════════════════════╣
   ║ ✅ the follower is GUARANTEED  ║ ✅ the leader can CONTINUE PROCESSING  ║
   ║   to have an up-to-date copy   ║   WRITES even if ALL followers have   ║
   ║   consistent with the leader.  ║   fallen behind.                      ║
   ║   If the leader suddenly       ║                                       ║
   ║   fails, the data is still     ║ ❌ if the leader fails and is NOT      ║
   ║   available on the follower.   ║   recoverable, any writes not yet     ║
   ║                                ║   replicated ARE LOST — so a write    ║
   ║ ❌ if the sync follower doesn't ║   IS NOT GUARANTEED DURABLE EVEN IF   ║
   ║   respond, THE WRITE CANNOT    ║   IT WAS CONFIRMED TO THE CLIENT.     ║
   ║   BE PROCESSED. The leader     ║                                       ║
   ║   must BLOCK ALL WRITES and    ║                                       ║
   ║   wait.                        ║                                       ║
   ╚═══════════════════════════════╩═══════════════════════════════════════╝
```

### 🔑 Why "all followers synchronous" is impractical, and what people actually do

> It is impractical for **all** followers to be synchronous: **any one node outage would cause the whole system to grind to a halt.**

```
   SEMI-SYNCHRONOUS — what "enabling synchronous replication" usually means:

      ┌────────┐   sync    ┌────────────┐
      │ LEADER │──────────►│ FOLLOWER 1 │  ← the ONE synchronous follower
      │        │           └────────────┘
      │        │   async   ┌────────────┐
      │        │──────────►│ FOLLOWER 2 │
      │        │   async   ├────────────┤
      │        │──────────►│ FOLLOWER 3 │
      └────────┘           └────────────┘

   If follower 1 becomes unavailable or slow, one of the ASYNC followers
   IS MADE SYNCHRONOUS.

   ➜ GUARANTEE: an up-to-date copy on AT LEAST TWO NODES —
     the leader and one synchronous follower.
```

> **"Weakening durability may sound like a bad trade-off, but asynchronous replication is almost inevitable if there are many followers, or if they are geographically distributed."**

**How long is the lag?** Normally *"most database systems apply changes to followers in less than a second."* But **there is no guarantee.** Followers may fall behind by **several minutes or more** if: a follower is recovering from a failure, the system is near maximum capacity, or there are network problems.

---

## 6. Setting up new followers

> Simply copying data files from one node to another is **typically not sufficient**: clients are constantly writing, the data is always in flux, so a standard file copy would see **different parts of the database at different points in time. The result might not make any sense.**

You *could* lock the database — **but that goes against our goal of high availability.**

```
   THE FOUR-STEP PROCESS (usually achievable with NO DOWNTIME):

   ① TAKE A CONSISTENT SNAPSHOT of the leader's database at some point
      in time — if possible, WITHOUT LOCKING the entire database.
      Most databases have this feature, since backups need it too.
      (Third-party tools sometimes needed: innobackupex for MySQL.)
                              │
                              ▼
   ② COPY THE SNAPSHOT to the new follower node.
                              │
                              ▼
   ③ THE FOLLOWER CONNECTS TO THE LEADER and requests all data changes
      SINCE THE SNAPSHOT WAS TAKEN.
      ⚑ This requires the snapshot to be associated with AN EXACT
        POSITION IN THE LEADER'S REPLICATION LOG:
            PostgreSQL calls it the LOG SEQUENCE NUMBER
            MySQL      calls it BINLOG COORDINATES
                              │
                              ▼
   ④ When the follower has processed the backlog, it has CAUGHT UP.
      It now processes changes from the leader as they happen.
```

> ⚠️ *"The practical steps vary significantly by database. In some systems the process is fully automated, whereas in others it can be a somewhat **arcane multi-step workflow** that needs to be manually performed by an administrator."*

---

## 7. Handling node outages

> Any node can go down — perhaps unexpectedly due to a fault, **but just as likely due to planned maintenance** (rebooting to install a kernel security patch). **Being able to reboot individual nodes without downtime is a big advantage for operations.**

### Follower failure: catch-up recovery — the easy case

```
   Each follower keeps a LOG ON ITS LOCAL DISK of the changes received
   from the leader.

   crash / network interruption
              │
              ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ From its log, the follower knows THE LAST TRANSACTION IT          │
   │ PROCESSED before the fault.                                       │
   │  → connect to the leader                                          │
   │  → request all changes that occurred while disconnected           │
   │  → apply them → CAUGHT UP → resume the normal stream              │
   └──────────────────────────────────────────────────────────────────┘
```

### Leader failure: failover — the hard case

```
   ① DETERMINING THAT THE LEADER HAS FAILED
      "There is NO FOOLPROOF WAY of detecting what has gone wrong, so
       most systems simply use A TIMEOUT" — nodes bounce messages back
       and forth; no response for, say, 30 seconds → assumed dead.

   ② CHOOSING A NEW LEADER
      An ELECTION (chosen by a majority of remaining replicas), or
      appointment by a previously-elected CONTROLLER NODE.
      Best candidate = the replica with the MOST UP-TO-DATE data.
      ⚑ "Getting all the nodes to agree on a new leader is a CONSENSUS
        PROBLEM" → Chapter 9.

   ③ RECONFIGURING THE SYSTEM TO USE THE NEW LEADER
      Clients must now send writes to the new leader. And if the OLD
      LEADER COMES BACK, it might still believe it is leader — the system
      must ensure it BECOMES A FOLLOWER and recognizes the new leader.
```

### ⚠️ The four ways failover goes wrong

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║ ① LOST WRITES (with asynchronous replication)                         ║
   ║    The new leader may not have received all writes from the old       ║
   ║    leader. If the former leader rejoins, what happens to those?       ║
   ║    "The most common solution is for the old leader's unreplicated     ║
   ║     writes to SIMPLY BE DISCARDED — which may VIOLATE CLIENTS'        ║
   ║     DURABILITY EXPECTATIONS."                                         ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║ ② DISCARDING WRITES IS ESPECIALLY DANGEROUS when other storage        ║
   ║    systems must stay coordinated with the database.                   ║
   ║    📌 THE GITHUB INCIDENT — worth memorizing:                          ║
   ║       • an OUT-OF-DATE MySQL follower was promoted to leader          ║
   ║       • the DB used an AUTO-INCREMENTING COUNTER for primary keys     ║
   ║       • the new leader's counter LAGGED BEHIND                        ║
   ║       • so it RE-USED primary keys already assigned by the old leader ║
   ║       • those PKs were ALSO USED IN A REDIS STORE                     ║
   ║       • → inconsistency between MySQL and Redis                       ║
   ║       • → SOME PRIVATE DATA WAS DISCLOSED TO THE WRONG USERS          ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║ ③ SPLIT BRAIN — two nodes both believe they are the leader            ║
   ║    If both accept writes and there's no conflict resolution process,  ║
   ║    "data is likely to be LOST OR CORRUPTED."                          ║
   ║    Safety catch: shut down one node if two leaders are detected —     ║
   ║    known as FENCING or STONITH (Shoot The Other Node In The Head).    ║
   ║    ⚠️ "However, if you're unlucky, you can end up with BOTH NODES      ║
   ║       BEING SHUT DOWN."                                               ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║ ④ WHAT IS THE RIGHT TIMEOUT?                                          ║
   ║                                                                       ║
   ║    LONGER timeout  → longer time to recovery when the leader fails    ║
   ║    SHORTER timeout → UNNECESSARY FAILOVERS. A temporary load spike    ║
   ║                      or a network glitch triggers one.                ║
   ║                                                                       ║
   ║    ⚠️ "If the system is ALREADY STRUGGLING with high load or network   ║
   ║       problems, an unnecessary failover is likely to MAKE THE         ║
   ║       SITUATION WORSE, NOT BETTER."                                   ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ➜ "There are NO EASY SOLUTIONS to these problems. For this reason, SOME
     OPERATIONS TEAMS PREFER TO PERFORM FAILOVER MANUALLY, even if the
     software supports automatic failover."
```

### 💻 I simulated the GitHub scenario. Here's the actual output

```
leader log       : [(1,'alice'), (2,'bob'), (3,'carol'), (4,'dave'), (5,'eve')]
follower 2 (lagging) has: {1:'alice', 2:'bob', 3:'carol'}

*** LEADER CRASHES. The OUT-OF-DATE follower 2 is promoted. ***
new leader's auto-increment counter restarts at 4
new write 'mallory' gets primary key 4
but pk 4 was ALREADY USED for 'dave' on the OLD leader

=> Two different rows now share primary key 4.
   Also silently lost: ['dave', 'eve'] -- writes the old leader had ACKED.
```

**Notice there are two separate failures stacked on top of each other.** Losing `dave` and `eve` is bad but at least it's *absence*. The primary-key collision is worse: it's *wrong data that looks right*, and it propagated into a second system. That's how a replication bug became a privacy breach.

---

## 8. Implementation of replication logs — four methods

### ① Statement-based replication

> The leader logs every write **request (statement)** it executes, and sends that statement log to followers. Each follower **parses and executes that SQL statement as if it had been received from a client.**

```
   ⚠️ WAYS THIS BREAKS DOWN

   • NON-DETERMINISTIC FUNCTIONS
        NOW()   → a different timestamp on every replica
        RAND()  → a different random number on every replica

   • AUTO-INCREMENTING COLUMNS, or statements depending on existing data
        UPDATE ... WHERE <some condition>
     must execute in EXACTLY THE SAME ORDER on each replica, otherwise
     they have a different effect. Limiting when there are multiple
     concurrent transactions.

   • SIDE EFFECTS — triggers, stored procedures, user-defined functions
     may produce different side effects on each replica unless they are
     ABSOLUTELY deterministic.
```

**Workarounds exist** (the leader can replace non-deterministic calls with a fixed value when logging), **"however, because there are so many edge cases, other replication methods are now generally preferred."**

| System | Status |
|---|---|
| MySQL before 5.1 | Used it |
| MySQL today | **Switches to row-based** if there is any non-determinism |
| VoltDB | Uses it, **made safe by requiring transactions to be deterministic** |

### ② Write-ahead log (WAL) shipping

Callback to Chapter 3 — in both storage-engine families, **every write is appended to a log**:

```
   LOG-STRUCTURED engine  → the log IS the main storage
   B-TREE engine          → every modification first goes to a WAL

   Either way, the log is an APPEND-ONLY SEQUENCE OF BYTES containing
   all writes. So: besides writing the log to disk, the leader also
   SENDS IT ACROSS THE NETWORK to its followers.

   The follower processing this log builds A COPY OF THE EXACT SAME
   DATA STRUCTURES found on the leader.
```

**Used by PostgreSQL and Oracle.**

```
   ❌ THE MAIN DISADVANTAGE — and it's an operational one

   The log describes data at a VERY LOW LEVEL: which BYTES were changed
   in which DISK BLOCK. This makes replication CLOSELY COUPLED TO THE
   STORAGE ENGINE.

   ➜ If the DB changes its storage format between versions, you typically
     CANNOT run different versions on leader and followers.

   ➜ WHY THAT MATTERS:
        if the protocol ALLOWED a version mismatch, you could do a
        ZERO-DOWNTIME UPGRADE: upgrade followers first, then fail over
        to make an upgraded node the new leader.
        With WAL shipping, SUCH UPGRADES REQUIRE DOWNTIME.
```

### ③ Logical (row-based) log replication

> Use **different log formats for replication and for the storage engine.** This decouples the replication log from storage internals. Called a **logical log**, to distinguish it from the storage engine's **physical** representation.

```
   A LOGICAL LOG for a relational DB is a sequence of records describing
   writes AT THE GRANULARITY OF A ROW:

   ┌──────────┬────────────────────────────────────────────────────────┐
   │ INSERT   │ the NEW VALUES OF ALL COLUMNS                          │
   ├──────────┼────────────────────────────────────────────────────────┤
   │ DELETE   │ enough info to UNIQUELY IDENTIFY the deleted row —     │
   │          │ typically the primary key; if there is no PK, the OLD  │
   │          │ VALUES OF ALL COLUMNS                                  │
   ├──────────┼────────────────────────────────────────────────────────┤
   │ UPDATE   │ enough info to identify the row, PLUS the new values   │
   │          │ of all columns (or at least all that changed)          │
   └──────────┴────────────────────────────────────────────────────────┘

   A transaction modifying several rows generates several such records,
   followed by a COMMIT record.

   ➜ MySQL's BINLOG (in row-based mode) works this way.
```

```
   ✅ TWO BIG ADVANTAGES

   ① BACKWARD COMPATIBILITY is easier, so leader and follower can run
      DIFFERENT VERSIONS of the database software — or even DIFFERENT
      STORAGE ENGINES.
      (This is exactly the Chapter 4 rolling-upgrade argument, now
       applied to the database itself.)

   ② EASIER FOR EXTERNAL APPLICATIONS TO PARSE. Useful for sending DB
      contents to a data warehouse for offline analysis, or building
      custom indexes and caches.
      ➜ This is called CHANGE DATA CAPTURE (Chapter 11).
```

### ④ Trigger-based replication

> The approaches so far are implemented **by the database system, without involving application code.** In many cases that's what you want — **but sometimes more flexibility is needed:**

```
   WHEN YOU NEED TO MOVE REPLICATION UP TO THE APPLICATION LAYER:
      • replicate only a SUBSET of the data
      • replicate from one KIND of database to ANOTHER
      • you need CUSTOM CONFLICT RESOLUTION LOGIC

   HOW A TRIGGER WORKS:
      register custom application code in the DB so it runs automatically
      when a write transaction occurs
              │
              ▼
      the trigger LOGS THE CHANGE INTO A SEPARATE TABLE
              │
              ▼
      an EXTERNAL PROCESS reads that table, applies any application
      logic, and replicates the change to another system

   → Databus (Oracle), Bucardo (Postgres). Oracle GoldenGate instead
     reads the database log.

   ⚠️ "Typically has GREATER OVERHEADS than other replication methods, and
      is MORE PRONE TO BUGS AND LIMITATIONS than built-in replication.
      However, it can nevertheless be useful due to its FLEXIBILITY."
```

### 📊 Summary comparison

| Method | Coupling | Cross-version? | External parsing? | Main risk |
|---|---|---|---|---|
| **Statement** | Very loose | Yes | Easy | **Non-determinism** |
| **WAL shipping** | **Tight** (storage engine) | ❌ **No** → downtime upgrades | Hard | Version lock-in |
| **Logical/row** | Loose | ✅ Yes | ✅ Easy (→ CDC) | More verbose |
| **Trigger** | Application-level | Yes | Yes | **Overhead, bugs** |

---
---

## 9. Problems With Replication Lag

### The read-scaling architecture, and why it forces async

```
   Leader-based replication requires ALL WRITES through one node, but
   READ-ONLY QUERIES CAN GO TO ANY REPLICA.

   For read-heavy workloads (a common web pattern): create MANY
   FOLLOWERS and distribute reads across them.

              ┌────────┐   writes
    clients ─►│ LEADER │
              └───┬────┘
      ┌───────┬───┴───┬───────┬───────┐
      ▼       ▼       ▼       ▼       ▼
    ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
    │ F │  │ F │  │ F │  │ F │  │ F │  ← add more to scale reads
    └───┘  └───┘  └───┘  └───┘  └───┘
      ▲      ▲      ▲      ▲      ▲
      └──────┴── reads ────┴──────┘

   ⚠️ BUT: "this approach only realistically works with ASYNCHRONOUS
     replication — if you tried to synchronously replicate to all
     followers, A SINGLE NODE FAILURE OR NETWORK OUTAGE WOULD MAKE THE
     ENTIRE SYSTEM UNAVAILABLE FOR WRITING. And the more nodes you have,
     the likelier that one is down."
```

### 🔑 Eventual consistency, defined precisely

> If you run the same query on the leader and a follower at the same time, **you may get different results.** This inconsistency is just a temporary state — **if you stop writing and wait a while, the followers will eventually catch up.** For that reason, this effect is known as **eventual consistency**.

```
   ⚠️ "THE TERM *EVENTUALLY* IS DELIBERATELY VAGUE: in general, there is
      NO LIMIT to how far a replica can fall behind."

   normal operation      → REPLICATION LAG is a fraction of a second,
                           not noticeable in practice
   near capacity /       → the lag can easily increase to SEVERAL SECONDS
   network problems        OR MINUTES
```

> 📖 *Footnote worth knowing:* the term was coined by Douglas Terry et al., popularized by Werner Vogels, and became **"the battle cry of many NoSQL projects."** But **"not only NoSQL databases are eventually consistent: followers in an asynchronously replicated RELATIONAL database have the same characteristics."**

---

### ANOMALY 1 — Reading your own writes

### 🔷 Figure 5-3 — A user writes, then reads from a stale replica

```
             insert into comments                select * from comments
             (author, reply_to, message)         where reply_to = 55555
             values(1234, 55555,'Sounds good!')            │
                       │                                   │       TIME ────►
   User 1234 ──────────●───────────────────────────────────●──► NO RESULTS!
                       │           ▲                       │
                       ▼           │ insert ok             │
   Leader    ──────────●───────────●───────────────────────┼──────────────►
                       │ ╲                                 │
                       │   ╲ insert into comments...       │
                       ▼     ▼                             │
   Follower 1 ─────────────────●─────────────────────────────────────────►
                       │                                   │
                       │  ╲ insert into comments…          │
                       │    ╲                              │      ╲
                       │      ╲                            │        ▼
   Follower 2 ─────────┼────────────────────────────────────────────●────►
                       │                              read landed HERE,
                       │                              BEFORE the write arrived
```

> **To the user, it looks as though the data they submitted was lost, so they will be understandably unhappy.**

**The guarantee needed: read-after-write consistency** (also *read-your-writes consistency*).

> It is a guarantee that **if the user reloads the page, they will always see any updates they submitted themselves.** It makes **no promises about other users** — other users' updates may not be visible until later. But it reassures the user that **their own input has been saved correctly.**

### The four implementation techniques

```
   ① READ USER-MODIFIABLE DATA FROM THE LEADER
      Requires knowing whether something MIGHT have been modified,
      without querying it.
      ✔ Example: on a social network, a profile is normally editable only
        by its owner. So: ALWAYS READ THE USER'S OWN PROFILE FROM THE
        LEADER, and other users' profiles from a follower.

   ② TIME-BASED / LAG-BASED ROUTING
      If MOST things are user-editable, ① won't work (you'd read
      everything from the leader, negating read scaling). Instead:
        • track the time of the last update; FOR ONE MINUTE AFTER IT,
          all reads go to the leader
        • MONITOR REPLICATION LAG and prevent queries on any follower
          more than one minute behind

   ③ CLIENT REMEMBERS ITS LAST-WRITE TIMESTAMP
      The system ensures the replica serving that user's reads reflects
      updates AT LEAST UNTIL THAT TIMESTAMP. If a replica isn't
      sufficiently up-to-date, either route to another replica, or WAIT
      until it has caught up.
      ⚑ The timestamp can be LOGICAL (e.g. log sequence number) or the
        ACTUAL SYSTEM CLOCK — in which case CLOCK SYNCHRONIZATION becomes
        critical (Chapter 8).

   ④ MULTI-DATACENTER: any request that must be served by the leader
      has to be ROUTED TO THE DATACENTER CONTAINING THE LEADER.
```

### ⚠️ Cross-device read-after-write consistency

```
   The same user on a DESKTOP BROWSER and a MOBILE APP:

   ❌ Approach ③ breaks: the code on one device doesn't know what updates
      happened on the other. THE METADATA WOULD NEED TO BE CENTRALIZED.

   ❌ No guarantee that connections from different devices are routed to
      the same datacenter. (Desktop on home broadband, mobile on cellular
      → COMPLETELY DIFFERENT NETWORK ROUTES.)
      If your approach requires reading from the leader, you may first
      need to ROUTE ALL OF A USER'S DEVICES TO THE SAME DATACENTER.
```

### 💻 Simulated, with the fix applied

```
Figure 5-3 -- user posts a comment, then reloads the page:
   read from leader     : 'Sounds good!'
   read from follower 1 : 'Sounds good!'
   read from follower 2 : None            <- 'my comment vanished!'
   FIX (read own writes from leader): 'Sounds good!'
```

---

### ANOMALY 2 — Monotonic reads

> Our second anomaly: **it's possible for a user to see things moving backwards in time.**

### 🔷 Figure 5-4 — Reading from a fresh replica, then a stale one

```
             insert into comments values(1234, 55555, 'Sounds good!')
                       │                                   TIME ──────────►
   User 1234 ──────────●──────────────────────────────────────────────────►
                       │      ▲ insert ok
                       ▼      │
   Leader    ──────────●──────●───────────────────────────────────────────►
                       │ ╲
                       │   ╲ insert into comments…
                       ▼     ▼
   Follower 1 ─────────────────●─────────────────────────────────────────►
                                        ▲
                                        │ 1 RESULT
                       │  ╲  insert into comments…              ╲
                       │    ╲                                     ▼
   Follower 2 ─────────┼──────────────────────────────────────────●───────►
                                                     ▲
                                                     │ NO RESULTS!
   User 2345 ──────────────────────────●─────────────●─────────────────────►
                                   read #1        read #2
                                (fresh replica)  (lagging replica)
                                  sees comment   comment is GONE
```

> It wouldn't be so bad if the *first* query hadn't returned anything, because user 2345 probably wouldn't know a comment existed. **However, it's very confusing for user 2345 if they first see user 1234's comment appear, and then see it disappear again.**

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  MONOTONIC READS is STRONGER than eventual consistency,            │
   │                     but WEAKER than strong consistency.            │
   │                                                                    │
   │  It does NOT say you'll see fresh data. You may see an old value.  │
   │  It ONLY says: if one user makes several reads in sequence, THEY   │
   │  WILL NOT SEE TIME GO BACKWARDS — they will not read older data    │
   │  after having previously read newer data.                          │
   ├────────────────────────────────────────────────────────────────────┤
   │  THE FIX: make sure each user ALWAYS READS FROM THE SAME REPLICA   │
   │  (different users can use different replicas).                     │
   │                                                                    │
   │      replica = hash(user_id) % n     ← NOT random                  │
   └────────────────────────────────────────────────────────────────────┘
```

**Note the cause in my simulation:** *"This scenario is quite likely if the user refreshes a web page, and each request is routed to a random server."* A plain round-robin load balancer is enough to produce it.

---

### ANOMALY 3 — Consistent prefix reads

> Our third anomaly concerns **violation of causality.**

**The dialogue:**

```
   Mr Poons:  "How far into the future can you see, Mrs Cake?"
   Mrs Cake:  "About ten seconds usually, Mr Poons."

   There is a CAUSAL RELATIONSHIP: Mrs Cake heard the question,
   and answered it.
```

### 🔷 Figure 5-5 — An observer sees the answer before the question

```
                  "How far into the future can you see, Mrs Cake?"
                              │                          TIME ───────────►
   Mr Poons  ─────────────────●────────────────────────────────────────►
                              │
   Partition 1 LEADER ────────●────────────────────────────────────────►
                              │ ╲
                              │   ╲  SLOW replication
                              │     ╲
   Partition 1 FOLLOWER ──────┼───────────────────────────────●────────►
                              │                                ╲
                              │                                  ╲
                                        "About ten seconds usually"
                                              │                    ╲
   Mrs Cake  ─────────────────────────────────●─────────────────────╲──►
                                              │                       ╲
   Partition 2 LEADER ────────────────────────●────────────────────────╲►
                                              │ ╲                       ╲
                                              │   ╲ FAST replication     ╲
   Partition 2 FOLLOWER ───────────────────────────●────────────────────►
                                                    ╲                    │
                                                      ▼                  ▼
   Observer  ──────────────────────────────────────────●────────────────●►
                                             "About ten seconds…"  "How far into
                                                                   the future…?"
                                             ▲
                                    THE ANSWER ARRIVES FIRST.
                          "Such psychic powers are impressive, but
                           also very confusing."
```

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  CONSISTENT PREFIX READS: if a sequence of writes happens in a     │
   │  certain order, then anyone reading those writes will see them     │
   │  appear IN THE SAME ORDER.                                         │
   ├────────────────────────────────────────────────────────────────────┤
   │  ⚑ THIS IS A PARTICULAR PROBLEM IN PARTITIONED (SHARDED) DATABASES.│
   │                                                                    │
   │  If the database always applies writes in the same order, reads    │
   │  always see a consistent prefix and this CANNOT HAPPEN.            │
   │  BUT in many distributed DBs, DIFFERENT PARTITIONS OPERATE         │
   │  INDEPENDENTLY, so there is NO GLOBAL ORDERING OF WRITES — a user  │
   │  may see some parts of the database in an older state and some in  │
   │  a newer state.                                                    │
   ├────────────────────────────────────────────────────────────────────┤
   │  FIXES:                                                            │
   │   • make sure causally-related writes go to THE SAME PARTITION     │
   │     (but "in some applications that can't be done efficiently")    │
   │   • in general, requires a DISTRIBUTED TRANSACTION with a          │
   │     guarantee such as SNAPSHOT ISOLATION (Chapter 7)               │
   └────────────────────────────────────────────────────────────────────┘
```

### 💻 Reproduced with differential partition lag

```
   causal order written:
      Mr Poons: How far into the future can you see, Mrs Cake?
      Mrs Cake: About ten seconds usually, Mr Poons.

   order the OBSERVER sees:
      Mrs Cake: About ten seconds usually, Mr Poons.
      Mr Poons: How far into the future can you see, Mrs Cake?
```

Partition 1 had a lag of 5 ticks and partition 2 a lag of 1. **Nothing was faulty — the partitions simply replicated at different speeds**, which is enough to invert causality.

---

### Solutions for replication lag — the meta-lesson

> When working with an eventually consistent system, **it is worth thinking about how the application behaves if the replication lag increases to several minutes or even hours.** If the answer is "no problem", that's great. However, if the result is a bad experience for users, **it's important to design the system to provide a stronger guarantee.**

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "PRETENDING THAT REPLICATION IS SYNCHRONOUS, WHEN IN FACT IT IS      ║
   ║   ASYNCHRONOUS, IS A RECIPE FOR PROBLEMS DOWN THE LINE."              ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   An application CAN provide a stronger guarantee than the database
   (e.g. by reading certain things from the leader) — but this ADDS
   COMPLEXITY TO THE APPLICATION, AND IS EASY TO GET WRONG.

   ➜ "It would be better if application developers didn't have to worry
     about subtle replication issues, and could just trust their database
     to 'do the right thing'. THIS IS WHY TRANSACTIONS EXIST: they are a
     way for a database to provide stronger guarantees, so that the
     application can be simpler."

   ⚠️ AND A POINTED REMARK:
     "In the move to distributed databases, many systems have abandoned
      [transactions], claiming that transactions are too expensive in
      terms of performance and availability, and asserting that eventual
      consistency is inevitable in a scalable system.
                        THAT IS NOT NECESSARILY TRUE."
```

---
---

## 10. Multi-leader replication

> Leader-based replication has one major downside: **there is only one leader, and all writes must go through it.** If you can't connect to the leader for any reason, **you can't write to the database.**

**The extension:** allow more than one node to accept writes. **Each leader simultaneously acts as a follower to the other leaders.**

> 📖 *Footnote:* if the database is partitioned, **each partition has one leader.** Different partitions may have leaders on different nodes, but each partition must have one leader node.

### Use case 1 — Multi-datacenter operation

### 🔷 Figure 5-6 — Multi-leader replication across datacenters

```
   ╔═══════════════════════════════════╗ ╔═══════════════════════════════════╗
   ║         DATACENTER 1              ║ ║         DATACENTER 2              ║
   ║                                   ║ ║                                   ║
   ║  ┌──────────────┐                 ║ ║                 ┌──────────────┐  ║
   ║  │  CONFLICT    │                 ║ ║                 │  CONFLICT    │  ║
   ║  │  RESOLUTION  │                 ║ ║                 │  RESOLUTION  │  ║
   ║  └──────┬───────┘                 ║ ║                 └───────┬──────┘  ║
   ║         │                         ║ ║                         │         ║
   ║    ┌────▼─────┐      ┌──────────┐ ║ ║  ┌──────────┐     ┌─────▼────┐    ║
   ║    │  LEADER  │─────►│ FOLLOWER │ ║ ║  │ FOLLOWER │◄────│  LEADER  │    ║
   ║    │          │      └──────────┘ ║ ║  └──────────┘     │          │    ║
   ║    └────┬─────┘                   ║ ║                   └─────┬────┘    ║
   ║         │  ▲                      ║ ║                      ▲  │         ║
   ╚═════════│══│══════════════════════╝ ╚══════════════════════│══│═════════╝
             │  └──────────── changes ─────────────────────────-┘  │
             └───────────────  changes  ──────────────────────────►┘
             │                                                     │
        read-write queries                              read-write queries

   WITHIN each datacenter : regular leader-follower replication
   BETWEEN datacenters    : each DC's leader replicates to the others
```

### The three-way comparison

| Dimension | **Single-leader** | **Multi-leader** |
|---|---|---|
| **Performance** | Every write must go **over the internet** to the leader's DC. Significant write latency — **"might contravene the purpose of having multiple datacenters in the first place"** | Every write processed **in the local datacenter**, replicated async. **Inter-DC delay is hidden from users** |
| **Tolerance of DC outages** | Failover can promote a follower in another DC | **Each DC continues operating independently**; replication catches up when the failed DC returns |
| **Tolerance of network problems** | **Very sensitive** — writes go synchronously over the (less reliable) public internet link | **Usually tolerates it better** — a temporary interruption doesn't prevent writes |

### ⚠️ But the book is unusually blunt about the downsides

> **"The same data may be concurrently modified in two different datacenters, and those write conflicts must be resolved."**
>
> As multi-leader replication is **a somewhat retrofitted feature in many databases**, there are often **subtle configuration pitfalls and surprising interactions with other database features.** For example, **auto-incrementing keys, triggers and integrity constraints can be problematic.** For this reason, **multi-leader replication is often considered dangerous territory that should be avoided if possible.**

Implementations: **Tungsten Replicator** (MySQL), **BDR** (PostgreSQL), **GoldenGate** (Oracle).

### Use case 2 — Clients with offline operation

```
   Calendar apps on your phone, laptop, other devices.
   You must be able to READ and WRITE meetings AT ANY TIME, regardless
   of internet connectivity. Changes sync when the device is next online.

   ┌──────────────────────────────────────────────────────────────────┐
   │  EVERY DEVICE HAS A LOCAL DATABASE THAT ACTS AS A LEADER.        │
   │  An asynchronous multi-leader replication process (sync) runs    │
   │  between the replicas on all your devices.                       │
   │  The replication lag may be HOURS OR EVEN DAYS.                  │
   └──────────────────────────────────────────────────────────────────┘

   "From an architectural point of view, this is ESSENTIALLY THE SAME AS
    MULTI-LEADER REPLICATION BETWEEN DATACENTERS, TAKEN TO THE EXTREME:
    EACH DEVICE IS A 'DATACENTER', and the network connection between
    them is EXTREMELY UNRELIABLE."

   ⚑ "As the RICH HISTORY OF BROKEN CALENDAR SYNC IMPLEMENTATIONS
      demonstrates, multi-leader replication is a tricky thing to get
      right."     (CouchDB is designed for this mode of operation.)
```

### Use case 3 — Collaborative editing

```
   Etherpad, Google Docs: several people editing simultaneously.

   "We don't usually think of collaborative editing as a database
    replication problem, but it has a lot in common with offline editing."

   When one user edits, changes are INSTANTLY applied to their LOCAL
   REPLICA (the document state in their browser), and ASYNCHRONOUSLY
   replicated to the server and other users.

   ┌───────────────────────────────┬───────────────────────────────────┐
   │ WANT NO CONFLICTS?            │ WANT FAST COLLABORATION?          │
   ├───────────────────────────────┼───────────────────────────────────┤
   │ The app must obtain a LOCK on │ Make the unit of change VERY      │
   │ the document before editing.  │ SMALL (e.g. a single keystroke)   │
   │ Others wait for the lock.     │ and AVOID LOCKING.                │
   │                               │                                   │
   │ ⚑ Equivalent to SINGLE-LEADER │ ⚑ Brings ALL the challenges of    │
   │   replication with            │   multi-leader replication,       │
   │   transactions on the leader. │   including CONFLICT RESOLUTION.  │
   └───────────────────────────────┴───────────────────────────────────┘
```

---

## 11. Handling write conflicts

### 🔷 Figure 5-7 — A write conflict from two concurrent leaders

```
                          update pages set title='B' where id=123
                                    │                    TIME ─────────────►
   User 1  ─────────────────────────●──────────────────────────────────────►
                                    │       ▲ ok
                                    ▼       │
   Leader 1 ────────────────────────●───────●─────────────────────────✗─────►
                initially there is       ╲                     CONFLICT: can't
                a page id=123, title=A     ╲ change id=123      change title
                                             ╲ old=A, new=B     from A to C,
                                               ╲                because title
                                              ╱  ╲              is now B
                              change id=123  ╱     ╲
                              old=A, new=C  ╱        ▼
   Leader 2 ────────────────────────●──────●───────────────────────────✗────►
                                    │      ▲                   CONFLICT: can't
                                    │      │ ok                change title
                                    ▲      │                   from A to B,
   User 2  ─────────────────────────●──────●────────────────────────────────►
                          update pages set title='C' where id=123

   ➜ "This problem DOES NOT OCCUR in a single-leader database."
```

### Synchronous vs asynchronous conflict detection

```
   ┌────────────────────────────────┬────────────────────────────────────┐
   │  SINGLE-LEADER                 │  MULTI-LEADER                      │
   ├────────────────────────────────┼────────────────────────────────────┤
   │  The second writer either      │  BOTH WRITES SUCCEED, and the      │
   │  BLOCKS and waits, or the      │  conflict is only detected         │
   │  transaction ABORTS and the    │  ASYNCHRONOUSLY at some later      │
   │  user retries.                 │  point in time.                    │
   │                                │  ⚠️ "At that time, IT MAY BE TOO   │
   │                                │     LATE TO ASK THE USER TO        │
   │                                │     RESOLVE THE CONFLICT."         │
   └────────────────────────────────┴────────────────────────────────────┘

   ⚑ Could you make detection synchronous? Yes — wait for the write to
     replicate to all replicas before reporting success.
     BUT: "by doing so, you would LOSE THE MAIN ADVANTAGE of multi-leader
     replication. IF YOU WANT SYNCHRONOUS CONFLICT DETECTION, YOU MIGHT
     AS WELL JUST USE SINGLE-LEADER REPLICATION."
```

### Strategy 1 — Conflict avoidance

> **The simplest strategy for dealing with conflicts is to avoid them.** If the application can ensure that **all writes for a particular record go through the same leader**, then conflicts cannot occur. Since many implementations handle conflicts quite poorly, **avoiding conflicts is a frequently recommended approach.**

```
   Route all requests from a particular user to THE SAME DATACENTER, and
   use the leader there for both reads and writes.

   Different users have different "HOME" datacenters (picked by geographic
   proximity), but FROM ANY ONE USER'S POINT OF VIEW THE CONFIGURATION IS
   ESSENTIALLY SINGLE-LEADER.

   ⚠️ BUT IT BREAKS DOWN when you need to CHANGE the designated leader:
      • a datacenter has failed and traffic must be rerouted
      • a user has MOVED and is now closer to a different datacenter
      → then you must deal with concurrent writes on different leaders.
```

### Strategy 2 — Converging towards a consistent state

```
   A SINGLE-LEADER database applies writes SEQUENTIALLY: the last write
   determines the final value.

   In MULTI-LEADER there is NO DEFINED ORDERING:
      at leader 1: title goes A → B → C
      at leader 2: title goes A → C → B
   "NEITHER ORDER IS 'MORE CORRECT' THAN THE OTHER."

   If each replica simply applied writes in the order it saw them:
      final value = C at leader 1,  B at leader 2   ← NOT ACCEPTABLE

   ➜ "Every replication scheme must ensure that the data is EVENTUALLY
     THE SAME IN ALL REPLICAS. The database must resolve the conflict in
     a CONVERGENT way — all replicas must arrive at the SAME FINAL VALUE."
```

**The four convergent resolution techniques:**

| # | Technique | Verdict |
|---|---|---|
| 1 | **Last write wins (LWW)** — unique ID per write (timestamp, random number, UUID, hash); highest ID wins, discard the rest | **"Although this technique is popular, it is DANGEROUSLY PRONE TO DATA LOSS"** |
| 2 | **Higher-numbered replica wins** — writes from a higher-numbered replica take precedence | **"This also implies data loss"** |
| 3 | **Merge the values** — e.g. order alphabetically and concatenate (title becomes `"B/C"`) | Preserves data, may be nonsense |
| 4 | **Record the conflict explicitly** in a data structure that preserves all information, and write application code to resolve it later (perhaps by prompting the user) | Most work, least loss |

### Strategy 3 — Custom conflict resolution logic

```
   ┌──────────────────────────────┬─────────────────────────────────────┐
   │  ON WRITE                    │  ON READ                            │
   ├──────────────────────────────┼─────────────────────────────────────┤
   │  As soon as the DB detects a │  All conflicting writes are STORED. │
   │  conflict in the log of      │  Next time the data is read, the    │
   │  replicated changes, it      │  MULTIPLE VERSIONS are returned to  │
   │  calls the CONFLICT HANDLER. │  the application, which may prompt  │
   │                              │  the user or resolve automatically, │
   │  ⚠️ Typically CANNOT PROMPT A │  then write the result back.        │
   │    USER — it runs in a       │                                     │
   │    background process and    │  → CouchDB works this way.          │
   │    MUST EXECUTE QUICKLY.     │                                     │
   │  → Bucardo (a Perl snippet). │                                     │
   └──────────────────────────────┴─────────────────────────────────────┘

   ⚠️ "Conflict resolution usually applies at the level of an INDIVIDUAL
     ROW OR DOCUMENT, NOT FOR AN ENTIRE TRANSACTION. If you have a
     transaction that atomically makes several different writes, EACH
     WRITE IS STILL CONSIDERED SEPARATELY for conflict resolution."
```

### 📦 Sidebar: Automatic conflict resolution

> **The Amazon shopping cart example** is frequently cited: for some time, the conflict resolution logic **"would preserve items added to the cart, but not items removed from the cart. Thus, customers would sometimes see items RE-APPEARING in their cart even though they had previously been removed."**

```
   THREE LINES OF RESEARCH:

   ① CRDTs (Conflict-free Replicated Data Types)
      A family of data structures — sets, maps, ordered lists, counters —
      which can be CONCURRENTLY EDITED BY MULTIPLE USERS and which
      AUTOMATICALLY RESOLVE CONFLICTS IN SENSIBLE WAYS.
      (Implemented in Riak 2.0.)

   ② MERGEABLE PERSISTENT DATA STRUCTURES
      Track history explicitly, similarly to GIT, and use a THREE-WAY
      MERGE function (whereas CRDTs use two-way merges).

   ③ OPERATIONAL TRANSFORMATION
      The algorithm behind Etherpad and Google Docs. Designed
      particularly for concurrent editing of AN ORDERED LIST OF ITEMS,
      such as the characters constituting a text document.
```

### ❓ What is a conflict? — the subtle case

```
   OBVIOUS: two writes concurrently set the same field to different
            values (Figure 5-7).

   SUBTLE:  A MEETING ROOM BOOKING SYSTEM.
            The app must ensure each room is booked by only one group at
            any one time (no overlapping bookings).

            A conflict arises if two different bookings are created for
            the same room at the same time.

            ⚠️ "EVEN IF THE APPLICATION CHECKS AVAILABILITY BEFORE
               ALLOWING A USER TO MAKE A BOOKING, there can be a conflict
               if the two bookings are made ON TWO DIFFERENT LEADERS."

            ➜ "Solutions have been proposed, but can be hard to implement
              in practice. For now, DETECTING CONFLICTS IS A QUESTION TO
              THINK ABOUT WHEN DESIGNING A REPLICATED SYSTEM, BUT THERE
              ISN'T A QUICK READY-MADE ANSWER."
```

**This is the important case**, because the check-then-act pattern is everywhere: usernames, inventory, seat reservations, unique constraints. Chapter 7's write skew covers it properly.

---

## 12. Multi-leader replication topologies

### 🔷 Figure 5-8 — Three topologies

```
   (a) CIRCULAR              (b) STAR                  (c) ALL-TO-ALL
                                                        (most general)
        ┌───┐                      ┌───┐                    ┌───┐
        │ 1 │                      │ 1 │                    │ 1 │
        └─┬─┘                      └─┬─┘                    └─┬─┘
     ┌────┘   └────┐                 │                  ┌─────┼─────┐
     ▼             ▲            ┌────▼────┐             │   ╱   ╲   │
   ┌───┐         ┌───┐          │  ROOT   │             ▼  ▼     ▼  ▼
   │ 4 │         │ 2 │      ┌───┤    0    ├───┐       ┌───┐     ┌───┐
   └─┬─┘         └─┬─┘      │   └────┬────┘   │       │ 4 │◄───►│ 2 │
     ▲             │        ▼        ▼        ▼       └─┬─┘     └─┬─┘
     └────┐   ┌────┘      ┌───┐    ┌───┐    ┌───┐       │  ╲   ╱  │
        ┌─┴───┴─┐         │ 2 │    │ 3 │    │ 4 │       │   ╳     │
        │   3   │         └───┘    └───┘    └───┘       ▼  ╱ ╲    ▼
        └───────┘                                       ┌───┐
                                                        │ 3 │
   MySQL default      can be generalised                └───┘
                      to a TREE                    every leader sends to
                                                   EVERY other leader
```

### Loop prevention

```
   In circular and star topologies, a write may pass through SEVERAL
   NODES before reaching all replicas. So nodes must FORWARD changes
   they receive from other nodes.

   ⚑ TO PREVENT INFINITE REPLICATION LOOPS:
     each node is given a UNIQUE IDENTIFIER, and in the replication log
     EACH WRITE IS TAGGED WITH THE IDENTIFIERS OF ALL THE NODES IT HAS
     PASSED THROUGH.

     When a node receives a change tagged with ITS OWN IDENTIFIER, THAT
     CHANGE IS IGNORED — it knows it has already processed it.

        write ──[tagged: 1]──► node2 ──[tagged: 1,2]──► node3
                                                          │
              node1 ◄──[tagged: 1,2,3]───────────────────┘
                │
                └─ sees its own id "1" in the tag → DROPS IT. No loop.
```

### 💻 I measured fault tolerance across the three topologies

```
   write from node 0 reaches all others:
      circular     hops {1:1, 2:2, 3:3}    ← up to n-1 hops
      star         hops {1:1, 2:1, 3:1}
      all-to-all   hops {1:1, 2:1, 3:1}

   NODE 1 FAILS:
      circular     write from node 0 now reaches []       <- BROKEN
      star         write from node 0 now reaches [2, 3]
      all-to-all   write from node 0 now reaches [2, 3]
```

> **"A problem with circular and star topologies is that if JUST ONE NODE FAILS, it can interrupt the flow of replication messages between other nodes."** The topology could be reconfigured around the failure, **"but in most deployments such reconfiguration would have to be done MANUALLY."**
>
> **"The fault tolerance of a more densely connected topology (such as all-to-all) is better, because it allows messages to travel along DIFFERENT PATHS, avoiding a single point of failure."**

### ⚠️ But all-to-all has its own problem

### 🔷 Figure 5-9 — Writes arriving in the wrong order

```
             insert into data (key,value) values ('x', 1)
                       │                                 TIME ────────────►
   Client A  ──────────●──────────────────────────────────────────────────►
                       │      ▲ ok
                       ▼      │
   Leader 1  ──────────●──────●───────────────────────────────────────────►
                        ╲      ╲
                         ╲  insert… value=1   (SLOW link)
                          ╲                    ╲
   Leader 2  ──────────────●────────────────────────────────●─────────────►
                         insert…                     ▲   update… value=2
                         value=1                     │
                                                     └─ DEPENDENT UPDATE
                                                        ARRIVES BEFORE
                                       ╱                THE INSERT
                          update… value=2  (FAST link)
                        ╱
   Leader 3  ──────────●────────────────────────────────────────────────►
                       │  ▲ ok
                       ▲  │
   Client B  ──────────●──●───────────────────────────────────────────────►
                 update data set value = value + 1 where key = 'x'
```

> This is a problem of **causality**, similar to consistent prefix reads: **the update depends on the prior insert**, so all nodes must process the insert first.
>
> ⚠️ **"Simply attaching a TIMESTAMP to every write is NOT SUFFICIENT, because CLOCKS CANNOT BE TRUSTED to be sufficiently in sync"** (Chapter 8). → **The technique needed is version vectors.**

**And a warning about the state of implementations:**

> Conflict detection techniques are **poorly implemented in many multi-leader replication systems.** At the time of writing, **PostgreSQL BDR does not provide causal ordering of writes**, and **Tungsten Replicator for MySQL doesn't even try to detect conflicts.**
>
> **"It is worth being aware of these issues, carefully reading the documentation, and THOROUGHLY TESTING YOUR DATABASE to ensure that it really does provide the guarantees you believe it to have."**

---
---

## 13. Leaderless replication

> Some systems abandon the concept of a leader, **allowing any replica to directly accept writes from clients.**

```
   HISTORY: "Some of the EARLIEST replicated data systems were leaderless,
   but the idea was MOSTLY FORGOTTEN during the era of dominance of
   relational databases. It once again became a fashionable architecture
   after AMAZON USED IT FOR THEIR IN-HOUSE DYNAMO SYSTEM."

   → Riak, Cassandra, Voldemort  ("Dynamo-style")

   📖 CONFUSING FOOTNOTE WORTH KNOWING:
      "Dynamo is not available to users outside of Amazon. Confusingly,
       AWS offers a hosted database product called DynamoDB, which uses a
       COMPLETELY DIFFERENT ARCHITECTURE: it is based on SINGLE-LEADER
       replication."
```

> In some implementations the client **directly sends its writes to several replicas**; in others, a **coordinator node** does this on behalf of the client. **However, unlike a leader database, that coordinator DOES NOT ENFORCE A PARTICULAR ORDERING OF WRITES. As we shall see, this has profound consequences.**

### 🔷 Figure 5-10 — Quorum write, quorum read, and read repair

```
        set key = users.1234.picture_url, value = 'me-new.jpg'
                       │                              TIME ────────────────►
   User 1234 ──────────●─────────────────────────────────────────────────►
                     ╱ │ ╲      ▲ ok      ▲ ok
                   ╱   │   ╲    │         │
                 ▼     ▼     ▼  │         │
   Replica 1 ────●──────────────●──────────────────────────────────────────►
                                          value='me-new.jpg', version=7
   Replica 2 ──────────●────────────────●─────────────────────────────────►
                                          value='me-new.jpg', version=7
   Replica 3 ─────── NODE OFFLINE ──────────────────────────────────────►
                     ✗ MISSES THE WRITE       │
                                              │ value='me-old.jpg', version=6
                                              ▼                          ▲
   User 2345 ─────────────────────────────────●──────────────────────────●►
                                       get key=…picture_url      set value=
                                       (reads from all 3)        'me-new.jpg'
                                                                 version=7
                                                                 ▲
                                                         ══ READ REPAIR ══
                                       the client SEES replica 3 is stale
                                       and WRITES THE NEWER VALUE BACK
```

**Note there is no failover at all:**

> In a leader-based configuration, if you want to continue processing writes, you may need to perform a failover. **In a leaderless configuration, FAILOVER DOES NOT EXIST.** The client sends the write to all three replicas in parallel, two accept it, the unavailable one misses it — **and the client simply ignores the fact that one of the replicas missed the write.**

### The two catch-up mechanisms

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  ① READ REPAIR                                                       │
   │     When a client reads from several nodes in parallel, it can       │
   │     DETECT STALE RESPONSES (via version numbers) and WRITE THE       │
   │     NEWER VALUE BACK to the stale replica.                           │
   │     ✅ "Works well for values that are FREQUENTLY READ."              │
   ├──────────────────────────────────────────────────────────────────────┤
   │  ② ANTI-ENTROPY PROCESS                                              │
   │     A BACKGROUND PROCESS that constantly looks for differences       │
   │     between replicas and copies missing data from one to another.    │
   │     ⚠️ Unlike the replication log in leader-based replication, it     │
   │       DOES NOT COPY WRITES IN ANY PARTICULAR ORDER, and there may    │
   │       be a SIGNIFICANT DELAY before data is copied.                  │
   └──────────────────────────────────────────────────────────────────────┘

   ⚠️ NOT ALL SYSTEMS IMPLEMENT BOTH. Voldemort has no anti-entropy.
     "Without an anti-entropy process, VALUES THAT ARE RARELY READ MAY BE
      MISSING FROM SOME REPLICAS and thus have REDUCED DURABILITY, because
      read repair only happens when a value is read by the application."
```

**That last point is quietly alarming.** In a read-repair-only system, *the data you never look at is the data most likely to be silently under-replicated.*

---

## 14. Quorums for reading and writing

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║                          w  +  r  >  n                                ║
   ║                                                                       ║
   ║   n = number of replicas a value is stored on                         ║
   ║   w = nodes that must confirm a WRITE for it to be successful         ║
   ║   r = nodes that must be queried for each READ                        ║
   ║                                                                       ║
   ║   "As long as w + r > n, we expect to get an up-to-date value when    ║
   ║    reading, BECAUSE AT LEAST ONE OF THE r NODES WE'RE READING FROM    ║
   ║    MUST BE UP-TO-DATE."                                               ║
   ║                                                                       ║
   ║   Think of r and w as the MINIMUM NUMBER OF VOTES required for the    ║
   ║   read or write to be valid.                                          ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 🔷 Figure 5-11 — The overlap, with n=5, w=3, r=3

```
                                  write
                                    │
        ┌───────────────────────────┼───────────────────────┐
        │        w = 3 successful writes                    │
        ▼        ▼        ▼                                 │
   ┌─────────┬─────────┬─────────┬─────────┬─────────┐      │
   │Replica 1│Replica 2│Replica 3│Replica 4│Replica 5│  n = 5 replicas
   └─────────┴─────────┴────▲────┴────▲────┴────▲────┘
                            │         │         │
                            └─────────┴─────────┘
                              r = 3 successful reads
                                    │
                                  read

           3 + 3 = 6  >  5      ➜  THE SETS MUST OVERLAP
                                   (pigeonhole principle)
        Replica 3 is in BOTH sets → the read sees the latest write
```

### The tolerance arithmetic

```
   • w < n  →  we can still process WRITES if a node is unavailable
   • r < n  →  we can still process READS if a node is unavailable

   n=3, w=2, r=2  →  tolerate 1 unavailable node
   n=5, w=3, r=3  →  tolerate 2 unavailable nodes

   COMMON CHOICE: n odd (3 or 5), w = r = (n+1)/2 rounded up.

   ⚑ BUT YOU CAN TUNE IT:
     a few writes, many reads → w = n, r = 1
     ✅ makes reads FAST
     ❌ but JUST ONE FAILED NODE CAUSES ALL WRITES TO FAIL

   ⚑ IMPORTANT DETAIL: reads and writes are ALWAYS SENT TO ALL n REPLICAS
     IN PARALLEL. w and r determine only HOW MANY WE WAIT FOR.
```

> 📖 *Note:* there may be more than n nodes in the cluster, but **any given value is stored on only n nodes.** This allows the dataset to be partitioned (Chapter 6).

### 💻 I verified the quorum condition with 200,000 Monte Carlo trials per configuration

```
  n   w   r   w+r>n?   fresh reads   tolerates
────────────────────────────────────────────────────
  3   2   2      YES       100.00%   1 node(s) down
  3   3   1      YES       100.00%   0 node(s) down
  3   1   3      YES       100.00%   0 node(s) down
  3   1   1       no        33.26%   2 node(s) down
  5   3   3      YES       100.00%   2 node(s) down
  5   2   2       no        70.07%   3 node(s) down
  5   1   1       no        20.17%   4 node(s) down
  5   5   1      YES       100.00%   0 node(s) down
```

**Every `w+r>n` row is exactly 100.00%, not 99.9%.** That's the giveaway that this is **set overlap, not probability** — the pigeonhole principle guarantees it. And notice the trade-off in the right-hand column: `n=5,w=1,r=1` tolerates four dead nodes but reads correctly only 20% of the time. **Availability and freshness are being traded directly against each other, and w+r is the dial.**

---

## 15. Limitations of quorum consistency

> **"Although quorums APPEAR to guarantee that a read returns the latest written value, IN PRACTICE IT IS NOT SO SIMPLE."**

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  SIX EDGE CASES WHERE w + r > n STILL RETURNS A STALE VALUE           ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  ① SLOPPY QUORUM — the w writes may land on DIFFERENT NODES than      ║
   ║     the r reads, so there is NO LONGER A GUARANTEED OVERLAP.          ║
   ║                                                                       ║
   ║  ② TWO CONCURRENT WRITES — it is not clear which happened first.      ║
   ║     The only safe solution is to MERGE them. If a winner is picked    ║
   ║     by timestamp (LWW), WRITES CAN BE LOST DUE TO CLOCK SKEW.         ║
   ║                                                                       ║
   ║  ③ A WRITE CONCURRENT WITH A READ — the write may be reflected on     ║
   ║     only some replicas; IT IS UNDETERMINED whether the read returns   ║
   ║     the old or the new value.                                         ║
   ║                                                                       ║
   ║  ④ A PARTIALLY-FAILED WRITE — succeeded on some replicas, failed on   ║
   ║     others, and succeeded on FEWER THAN w overall.                    ║
   ║     ⚠️ IT IS NOT ROLLED BACK ON THE REPLICAS WHERE IT SUCCEEDED.       ║
   ║     So if a write was reported as FAILED, subsequent reads MAY OR     ║
   ║     MAY NOT return the value from that write.                         ║
   ║                                                                       ║
   ║  ⑤ A NODE CARRYING A NEW VALUE FAILS and is restored from a replica   ║
   ║     carrying an OLD value → the count of replicas with the new value  ║
   ║     may fall BELOW w, BREAKING THE QUORUM CONDITION.                  ║
   ║                                                                       ║
   ║  ⑥ UNLUCKY TIMING — even when everything works correctly              ║
   ║     (→ "Linearizability and quorums", Chapter 9)                      ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

**Point ④ deserves a second look.** A write that returns an *error* to your client may still be *partially durable* and may show up in later reads. The failure signal doesn't mean "nothing happened."

> **The bottom line:** *"Dynamo-style databases are generally optimized for use cases that can tolerate eventual consistency. The parameters w and r allow you to ADJUST THE PROBABILITY of stale values being read, BUT IT'S WISE TO NOT TAKE THEM AS ABSOLUTE GUARANTEES."*
>
> In particular, **you usually do NOT get read-your-writes, monotonic reads, or consistent prefix reads.** Stronger guarantees generally require **transactions or consensus.**

### Monitoring staleness

```
   ┌──────────────────────────────┬───────────────────────────────────────┐
   │ LEADER-BASED                 │ LEADERLESS                            │
   ├──────────────────────────────┼───────────────────────────────────────┤
   │ ✅ EASY. Writes are applied   │ ❌ HARD. "There is NO FIXED ORDER in   │
   │   to leader and followers IN │   which writes are applied."          │
   │   THE SAME ORDER, and each   │                                       │
   │   node has a POSITION in the │ ❌ And if the DB only uses read repair │
   │   replication log.           │   (no anti-entropy) "THERE IS NO      │
   │                              │   LIMIT TO HOW OLD A VALUE MIGHT BE — │
   │   lag = leader position      │   if a value is only infrequently     │
   │       − follower position    │   read, the value returned by a stale │
   │                              │   replica may be ANCIENT."            │
   └──────────────────────────────┴───────────────────────────────────────┘

   ➜ There has been research on PREDICTING the expected percentage of
     stale reads from n, w and r (Bailis et al., PBS).
     "This is unfortunately NOT YET COMMON PRACTICE, but it would be good
      to include staleness measurements in the standard set of metrics.
      EVENTUAL CONSISTENCY IS A DELIBERATELY VAGUE GUARANTEE, BUT FOR
      OPERABILITY IT'S IMPORTANT TO BE ABLE TO QUANTIFY 'EVENTUAL'."
```

---

## 16. Sloppy quorums and hinted handoff

```
   THE PROBLEM: "A network interruption can easily cut off a client from
   a large number of database nodes. ALTHOUGH THOSE NODES ARE ALIVE, and
   other clients may be able to connect to them, TO A CLIENT THAT IS CUT
   OFF THEY MIGHT AS WELL BE DEAD."

   In a LARGE cluster (many more than n nodes), the client can probably
   reach SOME database nodes — just not the n nodes that hold this value.

   ➜ THE DESIGNER'S CHOICE:
      • return errors for everything we can't reach a quorum for?
      • OR accept the writes anyway, onto nodes that ARE reachable but
        AREN'T among the n "home" nodes for that value?
```

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  SLOPPY QUORUM — the second option                                    ║
   ║                                                                       ║
   ║  Writes and reads still require w and r successful responses, BUT     ║
   ║  THOSE MAY INCLUDE NODES THAT ARE NOT AMONG THE DESIGNATED n          ║
   ║  "HOME" NODES.                                                        ║
   ║                                                                       ║
   ║  📖 THE BOOK'S ANALOGY:                                                ║
   ║     "If you locked yourself out of your house, you may knock on the   ║
   ║      NEIGHBOR'S door and ask whether you may STAY ON THEIR COUCH      ║
   ║      temporarily."                                                    ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  HINTED HANDOFF — the cleanup                                         ║
   ║                                                                       ║
   ║  Once the interruption is fixed, writes temporarily accepted on       ║
   ║  behalf of another node are SENT TO THE APPROPRIATE "HOME" NODES.     ║
   ║                                                                       ║
   ║     "Once you find the keys to your house again, your neighbor        ║
   ║      politely asks you to get off their couch and go home."           ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### ⚠️ The consequence that catches people out

> Sloppy quorums are particularly useful for **increasing write availability**: as long as **any** w nodes are available, the database can accept writes. **However, this means that even when w + r > n, you cannot be sure to read the latest value**, because it may have been temporarily written to nodes outside of n.

```
   ➜ "Thus, A SLOPPY QUORUM ACTUALLY ISN'T A QUORUM AT ALL IN THE
     TRADITIONAL SENSE. It's only an ASSURANCE OF DURABILITY — namely
     that the data is stored on w nodes SOMEWHERE. There is NO GUARANTEE
     that a read of r nodes will see it until the hinted handoff has
     completed."
```

**Defaults matter here:** enabled by default in **Riak**; **disabled** by default in **Cassandra** and **Voldemort**.

### Multi-datacenter operation, leaderless style

| System | Approach |
|---|---|
| **Cassandra, Voldemort** | n **includes nodes in all datacenters**; config specifies how many of the n replicas in each DC. Writes are sent to all replicas regardless of DC, **but the client usually only waits for a quorum within its LOCAL datacenter**, so it's unaffected by cross-DC delays. Higher-latency cross-DC writes are often configured to happen asynchronously |
| **Riak** | All client↔node communication stays **local to one datacenter**, so n describes replicas **within one DC**. Cross-DC replication happens asynchronously in the background, **in a style similar to multi-leader replication** |

---

## 17. Detecting concurrent writes

### 🔷 Figure 5-12 — No well-defined ordering

```
             set X = A
                 │                                        TIME ─────────►
   Client A ─────●──────────────────────────────────────────────────────►
                  ╲
                    ╲
   Node 1  ──────────●────── NODE UNRESPONSIVE ─────────────────► X = A
                                                    (never got B)
                    ╱ ╲
   Node 2  ────────●─────●──────────────────────────────────────► X = B
                   A     B      (received A, then B)
                          ╲
   Node 3  ────────────●───●────────────────────────────────────► X = A
                       B    A    (received B, then A)
                      ╱
   Client B ─────●──────────────────────────────●───────────────────────►
             set X = B                        get X  → ???

   "If each node simply OVERWROTE the value whenever it received a write,
    THEY WOULD BECOME PERMANENTLY INCONSISTENT: node 2 thinks X is B,
    the other nodes think X is A."
```

> ⚠️ **"One might hope that replicated databases would handle this automatically, but unfortunately MOST IMPLEMENTATIONS ARE QUITE POOR: if you want to avoid losing data, YOU — THE APPLICATION DEVELOPER — NEED TO KNOW A LOT ABOUT THE INTERNALS of your database's conflict handling."**

### Last write wins — and why the scare quotes matter

```
   The idea: each replica stores only the most 'recent' value; 'older'
   values are overwritten and discarded.

   ⚠️ "As indicated by the SCARE QUOTES around 'recent', THIS IDEA IS
      ACTUALLY QUITE MISLEADING. In the example of Figure 5-12, NEITHER
      CLIENT KNEW ABOUT THE OTHER when it sent its write, so it's not
      clear which one happened first. IN FACT, IT DOESN'T REALLY MAKE
      SENSE TO SAY THAT EITHER HAPPENED 'FIRST': we say the writes are
      CONCURRENT, so their order is UNDEFINED."

   LWW forces an ARBITRARY order: attach a timestamp, biggest wins,
   discard the rest.

   ┌──────────────────────────────────────────────────────────────────┐
   │ ✅ achieves EVENTUAL CONVERGENCE                                  │
   │ ❌ AT THE COST OF DURABILITY: if several concurrent writes hit    │
   │    the same key, EVEN IF ALL WERE REPORTED SUCCESSFUL TO THE      │
   │    CLIENT (because they reached w replicas), ONLY ONE SURVIVES.   │
   │    The others are SILENTLY DISCARDED.                             │
   │ ❌ "LWW may even drop writes that ARE NOT CONCURRENT" (clock skew)│
   └──────────────────────────────────────────────────────────────────┘

   The ONLY supported conflict resolution method in CASSANDRA;
   an OPTIONAL feature in RIAK.
```

> **The only safe way to use a database with LWW** is to ensure that **a key is only written once and thereafter treated as immutable**, avoiding concurrent updates entirely. *(A recommended way to use Cassandra: use a UUID as the key, giving every write operation a unique key.)*

### 💻 LWW destroying data, measured

```
three CONCURRENT writes, all acknowledged as successful to their clients:
   ts=100  client A: add item X
   ts=101  client B: add item Y
   ts=99   client C: add item Z

final stored value: 'client B: add item Y'
=> 2 of 3 acknowledged writes were SILENTLY DISCARDED.

--- worse: clock skew drops writes that were NOT concurrent ---
   v1 written at 12:00:05 by a node whose clock runs FAST  (ts=1205)
   v2 written at 12:00:10 by a node whose clock runs SLOW  (ts=1158)
   stored value is 'v1'
   => a causally LATER write lost to clock skew.
```

The second case is the dangerous one. The first is at least *defensible* — the writes really were concurrent, so someone had to lose. In the second, there is a genuine happens-before relationship and LWW inverts it.

---

## 18. The "happens-before" relationship

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  An operation A HAPPENS BEFORE another operation B if B KNOWS ABOUT   ║
   ║  A, or DEPENDS ON A, or BUILDS UPON A in some way.                    ║
   ║                                                                       ║
   ║  Two operations are CONCURRENT if NEITHER HAPPENS BEFORE THE OTHER,   ║
   ║  i.e. NEITHER KNOWS ABOUT THE OTHER.                                  ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   THREE POSSIBILITIES for any two operations A and B:
        A happened before B   │   B happened before A   │   A ∥ B (concurrent)
              ▼                            ▼                       ▼
        later overwrites            later overwrites        CONFLICT — must
        earlier                     earlier                 be resolved

   EXAMPLES FROM THE BOOK:
   • Figure 5-9: NOT concurrent. A's insert happens before B's increment,
     because the value B incremented is the value A inserted. B's
     operation BUILDS UPON A's. We say B is CAUSALLY DEPENDENT on A.
   • Figure 5-12: CONCURRENT. When each client starts, it does not know
     another client is also operating on the same key.
```

### 📦 Sidebar: Concurrency, time, and relativity

> It may seem that two operations should be called concurrent if they occur **"at the same time"** — **but in fact, it is not important whether they literally overlap in time.** Because of problems with clocks in distributed systems, **it is quite difficult to tell whether two things literally happened at the same time.**
>
> **"For defining concurrency, EXACT TIME DOESN'T MATTER: we simply call two operations concurrent if they are BOTH UNAWARE OF EACH OTHER, regardless of the physical time at which they occurred."**

```
   THE RELATIVITY ANALOGY (and its limit):

   PHYSICS: information cannot travel faster than light. Two events some
            distance apart cannot affect each other if the time between
            them is shorter than light's travel time.

   COMPUTERS: "two operations might be concurrent EVEN THOUGH THE SPEED
              OF LIGHT WOULD IN PRINCIPLE HAVE ALLOWED ONE TO AFFECT THE
              OTHER. If the network was slow or interrupted, two
              operations can occur SOME TIME APART and still be
              concurrent, because the network problems PREVENTED ONE
              FROM BEING ABLE TO KNOW ABOUT THE OTHER."
```

**So concurrency here is an epistemic notion, not a temporal one.** It's about *what a node could have known*, not about clocks at all. That reframing is what makes version vectors work.

---

## 19. Tracking happens-before: the shopping cart

### 🔷 Figures 5-13 and 5-14 — Two clients editing a cart

```
   STEP  WHO      ACTION              SENDS version  SERVER STATE AFTER
   ────────────────────────────────────────────────────────────────────────
    1   client1  + milk               (none)         v1: [milk]
    2   client2  + eggs               (none)         v1: [milk]
                                                     v2: [eggs]
    3   client1  + flour              based on v1    v2: [eggs]
                 sends [milk,flour]                  v3: [milk, flour]
                                                     ↑ v1 OVERWRITTEN
                                                       (v3 supersedes it)
                                                       but v2 KEPT
                                                       (concurrent)
    4   client2  + ham                based on v2    v3: [milk, flour]
                 sends [eggs,milk,ham]               v4: [eggs, milk, ham]
                                                     ↑ v2 overwritten
    5   client1  + bacon              based on v3    v4: [eggs, milk, ham]
                 sends [milk,flour,                  v5: [milk, flour,
                        eggs,bacon]                      eggs, bacon]
```

**Figure 5-14 — the causal dependency graph:**

```
                                                            [milk, flour,
            [milk]                    [milk, flour]         eggs, bacon]
   +milk ──────────► +flour ──────────────────────► +bacon ──────────►
     ▲                                                 ▲
     │                                                 │
   empty                                               │ (merged in)
     │                                                 │
     ▼                                                 │
   +eggs ─────────► +ham ────────────────────────────► │
            [eggs]        [eggs, milk, ham]

   Arrows = "happened before". Two chains that KEEP CROSSING but never
   fully synchronize — which is why two siblings survive at the end.
```

### 🔑 The algorithm, in four rules

```
   ① THE SERVER maintains a VERSION NUMBER for every key, INCREMENTS it
      on every write, and stores the new version number with the value.

   ② WHEN A CLIENT READS a key, the server returns ALL VALUES THAT HAVE
      NOT BEEN OVERWRITTEN, plus the latest version number.
      ⚑ A CLIENT MUST READ A KEY BEFORE WRITING.

   ③ WHEN A CLIENT WRITES, it must include the VERSION NUMBER FROM THE
      PRIOR READ, and it must MERGE TOGETHER ALL VALUES it received in
      that read.

   ④ WHEN THE SERVER RECEIVES a write with a particular version number,
      it can OVERWRITE ALL VALUES AT OR BELOW that version (it knows they
      have been merged into the new value), but it MUST KEEP ALL VALUES
      WITH A HIGHER VERSION NUMBER (those are concurrent with the
      incoming write).

   ➜ "When a write includes the version number from a prior read, THAT
     TELLS US WHICH PREVIOUS STATE THE WRITE IS BASED ON. If you make a
     write WITHOUT a version number, it is concurrent to ALL other
     writes, so it will not overwrite anything."

   ⚑ CRUCIALLY: "the server can determine whether two operations are
     concurrent BY LOOKING AT THE VERSION NUMBERS — IT DOES NOT NEED TO
     INTERPRET THE VALUE ITSELF (so the value could be any data
     structure)."
```

### 💻 I implemented this and it reproduces Figure 5-13 exactly

```
  1. client1 +milk   (based on v0)
     -> version 1, siblings: [milk]
  2. client2 +eggs   (based on v0)
     -> version 2, siblings: [milk] | [eggs]
  3. client1 +flour  (based on v1)
     -> version 3, siblings: [eggs] | [flour, milk]
  4. client2 +ham    (based on v2)
     -> version 4, siblings: [flour, milk] | [eggs, ham, milk]
  5. client1 +bacon  (based on v3)
     -> version 5, siblings: [eggs, ham, milk] | [bacon, eggs, flour, milk]

final siblings: [('bacon','eggs','flour','milk'), ('eggs','ham','milk')]
application merges by UNION -> ['bacon','eggs','flour','ham','milk']
NOTHING WAS LOST.
```

Exactly the book's outcome. **"In this example, the clients are never fully up to date with the data on the server, since there is always another operation going on concurrently. But old versions of the value DO get overwritten eventually, and NO WRITES ARE LOST."**

### Merging concurrently written values

```
   The algorithm ensures no data is silently dropped, but IT REQUIRES
   THE CLIENTS TO DO EXTRA WORK: merging the concurrent values.
   Riak calls these concurrent values SIBLINGS.

   ✅ FOR A SHOPPING CART, a reasonable merge is THE UNION:
      [milk, flour, eggs, bacon] ∪ [eggs, milk, ham]
        = [milk, flour, eggs, bacon, ham]    (no duplicates)

   ❌ BUT UNION BREAKS IF YOU ALLOW REMOVAL:
      "if you merge two sibling carts, and an item has been REMOVED in
       only one of them, THEN THE REMOVED ITEM WOULD REAPPEAR."

   ➜ THE FIX: "an item CANNOT SIMPLY BE DELETED from the database when it
     is removed; instead, the system must LEAVE A MARKER with an
     appropriate version number to indicate that the item has been
     removed when merging siblings. Such a deletion marker is known as a
     TOMBSTONE."
     (Same word as Chapter 3's LSM-tree deletions — same idea, different
      layer: you cannot represent 'absence' by being absent.)
```

**💻 I demonstrated the reappearance bug:**

```
   sibling A = ['bacon','eggs','flour','milk']
   sibling B = ['eggs','ham','milk']   (flour & bacon deliberately removed)
   naive union = ['bacon','eggs','flour','ham','milk']  <- REMOVED ITEMS BACK
```

That's the Amazon cart bug, reproduced in four lines.

---

## 20. Version vectors

> The example used only a **single replica**. **How does the algorithm change when there are multiple replicas but no leader?**

```
   A single version number is NOT SUFFICIENT when multiple replicas accept
   writes concurrently. Instead:

   ┌──────────────────────────────────────────────────────────────────────┐
   │  USE A VERSION NUMBER PER REPLICA AS WELL AS PER KEY.                │
   │                                                                      │
   │  Each replica increments ITS OWN version number when processing a    │
   │  write, and ALSO KEEPS TRACK of the version numbers it has seen      │
   │  FROM ALL THE OTHER REPLICAS.                                        │
   │                                                                      │
   │  The collection of version numbers from all replicas is called a     │
   │                        VERSION VECTOR                                │
   │                                                                      │
   │          { replica1: 2, replica2: 1, replica3: 5 }                   │
   └──────────────────────────────────────────────────────────────────────┘

   The most interesting variant is the DOTTED VERSION VECTOR, used in
   Riak 2.0.
```

### 💻 The comparison rule, implemented and tested

**A dominates B if A[i] ≥ B[i] for every replica i, and A ≠ B. Otherwise they're concurrent.**

```
version vector A      version vector B      verdict
────────────────────────────────────────────────────────────────────────
{'r1': 2, 'r2': 1}    {'r1': 1, 'r2': 1}    A happened AFTER B  → A overwrites B
{'r1': 2, 'r2': 1}    {'r1': 1, 'r2': 3}    CONCURRENT → keep BOTH as siblings
{'r1': 1, 'r2': 1}    {'r1': 1, 'r2': 1}    EQUAL
{'r1': 5}             {'r1': 5, 'r2': 1}    B happened AFTER A  → B overwrites A
```

**Look at row 2.** A is ahead on r1, B is ahead on r2. Neither dominates. That partial order — not a total order — *is* concurrency, detected mechanically without a single clock reading.

> Version vectors are sent **from the replicas to clients when values are read, and must be sent back when a value is written.** This lets the database distinguish overwrites from concurrent writes.
>
> **"The version vector structure ensures that IT IS SAFE TO READ FROM ONE REPLICA AND SUBSEQUENTLY WRITE BACK TO ANOTHER REPLICA: this may result in siblings being created, but NO DATA IS LOST as long as siblings are merged correctly."**

> 📖 **Version vectors vs vector clocks** — *"the difference is subtle. One way of looking at it: **version vectors are for client-server systems, and vector clocks are for peer-to-peer systems.**"*

---

## 21. 💻 Bonus: a CRDT that removes the need for merge code

The book mentions CRDTs in a sidebar without showing one. I implemented an **OR-Set (Observed-Remove Set)** — the structure that fixes the Amazon cart bug structurally.

**The trick:** tag every `add` with a unique identifier. A `remove` tombstones only the tags it has *observed*. Merge is a pure union of both sets.

```
replica A value: ['bacon', 'eggs', 'flour', 'milk']
replica B value: ['eggs', 'ham', 'milk']
automatic merge: ['bacon','eggs','flour','ham','milk']  <- no handler written

A keeps 'milk', B removes it     -> merged: []        (remove wins — B
                                                       OBSERVED that add)
concurrent add vs remove         -> merged: ['milk']  (add wins — B never
                                                       observed A's add)
```

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  WHY THIS IS DIFFERENT FROM THE UNION MERGE ABOVE                    │
   │                                                                      │
   │  Naive union: "was this item in either set?"  → removals reappear    │
   │  OR-Set:      "was this SPECIFIC ADD EVENT tombstoned by someone     │
   │                who SAW it?"                  → removals stick        │
   │                                                                      │
   │  The merge function is COMMUTATIVE, ASSOCIATIVE AND IDEMPOTENT, so   │
   │  replicas converge NO MATTER WHAT ORDER messages arrive in, and      │
   │  duplicates are harmless. No conflict handler. No user prompt.       │
   └──────────────────────────────────────────────────────────────────────┘
```

The Amazon bug is **structurally impossible** here, which is the whole argument for CRDTs.

---

## 22. Chapter Summary

**Replication serves four purposes:**

```
   • HIGH AVAILABILITY      keep running when a machine (or several, or a
                            whole datacenter) goes down
   • DISCONNECTED OPERATION continue working during a network interruption
   • LATENCY                place data geographically close to users
   • SCALABILITY            handle more reads than one machine could, by
                            reading from replicas
```

> **"Despite being a simple goal — a copy of the same data on several machines — replication turns out to be a REMARKABLY TRICKY PROBLEM. It requires carefully thinking about concurrency and about all the things that can go wrong."**

**The three approaches:**

| | **Single-leader** | **Multi-leader** | **Leaderless** |
|---|---|---|---|
| **Writes go to** | One node (the leader) | Any of several leaders | Several nodes in parallel |
| **Reads from** | Any replica (may be stale) | Any replica | Several nodes in parallel |
| **Conflicts?** | ✅ **None to worry about** | ⚠️ Yes | ⚠️ Yes |
| **Why choose it** | **"Fairly easy to understand"** | Robust to faults, network interruptions, latency spikes | Same, plus no failover at all |
| **Cost** | Single point for writes | **"Harder to reason about, only very weak consistency guarantees"** | Same |

**On synchronous vs asynchronous:**

> Although asynchronous replication can be fast when the system is running smoothly, **it's important to figure out what happens when replication lag increases and servers fail.** If a leader fails and you promote an asynchronously updated follower, **recently committed data may be lost.**

**The three consistency models for reasoning about replication lag:**

```
   ┌────────────────────────┬────────────────────────────────────────────┐
   │ READ-AFTER-WRITE       │ a user should always see data that THEY    │
   │                        │ THEMSELVES submitted                       │
   ├────────────────────────┼────────────────────────────────────────────┤
   │ MONOTONIC READS        │ after seeing the data at one point in      │
   │                        │ time, they shouldn't LATER see the data    │
   │                        │ from some EARLIER point in time            │
   ├────────────────────────┼────────────────────────────────────────────┤
   │ CONSISTENT PREFIX READS│ users should see the data in a state that  │
   │                        │ MAKES CAUSAL SENSE — a question and its    │
   │                        │ reply in the correct order                 │
   └────────────────────────┴────────────────────────────────────────────┘
```

Finally: multi-leader and leaderless both allow concurrent writes, **so conflicts may occur.** We examined **an algorithm to determine whether one operation happened before another, or whether they happened concurrently**, and **methods for resolving conflicts by merging concurrent updates.**

> *"In the next chapter we will continue looking at data distributed across multiple machines, through the counterpart of replication: splitting a large dataset into partitions."*

---

# 23. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA PART II + CH.5 — REPLICATION                                            ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  DISTRIBUTE FOR: scalability · fault tolerance/HA · latency                   ║
║  shared-memory (cost SUPER-LINEAR, one location) · shared-disk (contention +  ║
║  locking) · SHARED-NOTHING (nodes coordinate in software over a network)      ║
║  REPLICATION = same data on many nodes.  PARTITIONING = split data up. Both.  ║
║  ⚠️ "a single-threaded program can beat a 100-core cluster" — measure first.   ║
║                                                                               ║
║  🔑 "ALL THE DIFFICULTY IN REPLICATION LIES IN HANDLING *CHANGES* TO DATA."    ║
║                                                                               ║
║  ── SINGLE-LEADER ──────────────────────────────────────────────────────────  ║
║  All writes → leader → replication log → followers apply IN THE SAME ORDER.   ║
║  Reads from anywhere; writes only on the leader. Postgres/MySQL/Oracle/       ║
║  SQL Server/MongoDB/RethinkDB/Espresso/Kafka/RabbitMQ.                        ║
║  SYNC  ✅ guaranteed up-to-date copy  ❌ one slow follower BLOCKS ALL WRITES    ║
║  ASYNC ✅ leader never blocks          ❌ acked writes CAN STILL BE LOST        ║
║  → SEMI-SYNCHRONOUS in practice: exactly ONE sync follower, rest async.       ║
║  NEW FOLLOWER: snapshot → copy → request changes since the log position       ║
║      (PG: log sequence number / MySQL: binlog coordinates) → caught up.       ║
║  FAILOVER = detect (timeout) + elect (CONSENSUS, ch.9) + reconfigure.         ║
║     ⚠️ lost writes · GITHUB: stale follower promoted → auto-inc PK REUSED →    ║
║       collided with Redis → PRIVATE DATA LEAKED · SPLIT BRAIN (fencing /      ║
║       STONITH, but you may shut down BOTH) · timeout too short = needless     ║
║       failover that makes an overloaded system WORSE.                         ║
║     ➜ many teams do failover MANUALLY on purpose.                             ║
║                                                                               ║
║  REPLICATION LOGS                                                             ║
║   STATEMENT  ❌ NOW()/RAND(), auto-inc, triggers. MySQL<5.1; VoltDB (forces    ║
║              determinism).                                                    ║
║   WAL SHIP   Postgres/Oracle. ❌ TIED TO STORAGE FORMAT → no version mismatch  ║
║              → upgrades need DOWNTIME.                                        ║
║   LOGICAL/ROW MySQL binlog. ✅ decoupled → cross-version, cross-engine, and    ║
║              parseable by outsiders → CHANGE DATA CAPTURE (ch.11).            ║
║   TRIGGER    app-level. ✅ flexible (subsets, cross-DB, custom conflicts)      ║
║              ❌ overhead + bugs. Databus, Bucardo.                             ║
║                                                                               ║
║  ── REPLICATION LAG: "eventually" HAS NO UPPER BOUND ───────────────────────  ║
║  Read scaling REQUIRES async (sync to all = one node down kills writes).      ║
║  ① READ-AFTER-WRITE  "my own comment vanished"                                ║
║     fix: read user-editable data from LEADER · 1-min-after-write rule ·       ║
║          client remembers its last-write timestamp · route to leader's DC.    ║
║     ⚠️ CROSS-DEVICE needs CENTRALIZED metadata + same-DC routing.              ║
║  ② MONOTONIC READS  "the comment appeared, then disappeared" (time went back) ║
║     fix: STICKY REPLICA — replica = hash(user_id) % n, never random.          ║
║  ③ CONSISTENT PREFIX  "the answer arrives before the question"                ║
║     Caused by DIFFERENT PARTITIONS replicating at different speeds.           ║
║     fix: causally-related writes to the SAME partition, or snapshot isolation.║
║  ➜ "Pretending replication is synchronous when it is asynchronous is a recipe ║
║     for problems." App-level fixes are complex & easy to get wrong → THIS IS  ║
║     WHY TRANSACTIONS EXIST.                                                   ║
║                                                                               ║
║  ── MULTI-LEADER ───────────────────────────────────────────────────────────  ║
║  Use cases: MULTI-DATACENTER (local writes, hides inter-DC latency, survives  ║
║  DC outage & flaky links) · OFFLINE CLIENTS (every device IS a datacenter) ·  ║
║  COLLABORATIVE EDITING (lock = single-leader; no lock = full multi-leader).   ║
║  ⚠️ "often considered DANGEROUS TERRITORY" — retrofitted; auto-inc keys,       ║
║     triggers and constraints all misbehave.                                   ║
║  CONFLICTS: detected ASYNCHRONOUSLY, too late to ask the user.                ║
║     AVOID (route a record's writes to one leader — breaks on DC failure or a  ║
║     user moving) · CONVERGE (LWW ❌data loss · higher replica ID ❌data loss ·   ║
║     merge · record explicitly) · CUSTOM on-write (fast, no prompting) or      ║
║     on-read (siblings returned, CouchDB).                                     ║
║     ⚠️ resolution is PER ROW, NOT PER TRANSACTION.                             ║
║     AUTO: CRDTs · mergeable persistent structures (3-way) · operational       ║
║     transformation (Docs/Etherpad).  Amazon cart: removed items REAPPEARED.   ║
║     SUBTLE CONFLICT: meeting-room double booking — check-then-act across two  ║
║     leaders. "There isn't a quick ready-made answer."                         ║
║  TOPOLOGIES: circular (MySQL default) · star/tree · ALL-TO-ALL.               ║
║     Loops prevented by TAGGING each write with every node it passed through.  ║
║     circular/star: ONE node failure BREAKS replication. all-to-all: messages  ║
║     OVERTAKE → update arrives before its insert → need VERSION VECTORS        ║
║     (timestamps can't fix it — clock skew).                                   ║
║                                                                               ║
║  ── LEADERLESS (Dynamo-style: Riak, Cassandra, Voldemort) ──────────────────  ║
║  NO FAILOVER EXISTS. Client writes to many, reads from many, compares         ║
║  versions. (AWS DynamoDB is NOT this — it's single-leader.)                   ║
║  Catch-up: READ REPAIR (only for frequently-read values!) + ANTI-ENTROPY      ║
║     (background, unordered, slow; Voldemort has none → rare values lose       ║
║     durability).                                                              ║
║  🔑 QUORUM  w + r > n  → the read set and write set MUST OVERLAP (pigeonhole, ║
║     not probability). n=3,w=r=2 tolerates 1 down. n=5,w=r=3 tolerates 2.      ║
║     All n are always contacted; w and r are just how many you WAIT FOR.       ║
║  ⚠️ STALE ANYWAY: sloppy quorum · concurrent writes · write during read ·      ║
║     FAILED WRITES ARE NOT ROLLED BACK on replicas that succeeded · a new-value║
║     node restored from an old backup · unlucky timing.                        ║
║     → w,r adjust PROBABILITY, they are NOT ABSOLUTE GUARANTEES. You do NOT    ║
║       get read-your-writes / monotonic reads / consistent prefix.             ║
║  SLOPPY QUORUM + HINTED HANDOFF: write to the neighbour's couch, hand back    ║
║     later. ✅ write availability ❌ "isn't a quorum at all" — durability only. ║
║     ON by default in Riak; OFF in Cassandra & Voldemort.                      ║
║  MONITOR STALENESS: easy with a leader (log positions); hard without one.     ║
║     "For operability it's important to be able to QUANTIFY 'eventual'."       ║
║                                                                               ║
║  ── CONCURRENCY DETECTION ──────────────────────────────────────────────────  ║
║  A HAPPENS BEFORE B if B knows about / depends on / builds upon A.            ║
║  CONCURRENT = neither knows about the other. NOT about wall-clock overlap —   ║
║     it's about WHAT A NODE COULD HAVE KNOWN.                                  ║
║  LWW: converges, but SILENTLY DISCARDS acked writes, and clock skew drops     ║
║     even NON-concurrent ones. Cassandra's only option. Safe ONLY if keys are  ║
║     written once and immutable (→ use a UUID key).                            ║
║  VERSION NUMBERS (1 replica): server keeps a version per key; client MUST     ║
║     READ BEFORE WRITE, echoes the version, and MERGES what it read. Server    ║
║     overwrites ≤ that version, KEEPS anything higher (= concurrent SIBLINGS). ║
║     Server never looks at the VALUE — only the versions.                      ║
║  VERSION VECTORS (n replicas): one counter PER REPLICA per key. A dominates B ║
║     iff A[i] ≥ B[i] ∀i and A≠B; otherwise CONCURRENT. Dotted version vectors  ║
║     in Riak 2.0. Makes read-here-write-there SAFE.                            ║
║  MERGING SIBLINGS: union works for adds; REMOVALS NEED TOMBSTONES or deleted  ║
║     items REAPPEAR. CRDTs (OR-Set etc.) do this automatically.                ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 24. ✅ Test yourself

1. **Why is it impractical to make all followers synchronous?**
   → Any single node outage would halt every write, since the leader must block until that follower acknowledges. In practice you get semi-synchronous: one synchronous follower guarantees an up-to-date second copy, and another async follower is promoted to synchronous if it falls over.

2. **An asynchronously replicated leader acknowledges a write, then dies permanently. What happened to the write?**
   → It's gone. Acknowledgement to the client does not imply durability under async replication — that's the trade you made for leader throughput. This is why "committed" means something weaker than most developers assume.

3. **Why did the GitHub incident leak private data, when the underlying bug was just a lagging replica?**
   → The promoted follower's auto-increment counter was behind, so it reissued primary keys the old leader had already used. Those keys were also referenced from Redis, so the collision pointed Redis entries at the wrong rows. The lesson: discarded writes are dangerous precisely when *other systems* hold references to the database's identifiers.

4. **Your users complain that a comment appears on refresh and then vanishes. Which guarantee is missing, and what's the one-line fix?**
   → Monotonic reads. The load balancer is sending successive reads to replicas with different lag. Pin each user to a replica deterministically, e.g. `hash(user_id) % n`, instead of routing randomly.

5. **Why does WAL shipping force downtime during database upgrades, when logical replication doesn't?**
   → The WAL describes byte-level changes to disk blocks, so it's tightly coupled to the storage format. Leader and follower must run the same version. A logical log describes row-level changes, so it stays compatible across versions — which is what allows upgrade-followers-then-fail-over.

6. **With n=5, w=3, r=3, is a stale read possible?**
   → Yes, despite w+r>n. Sloppy quorums break the overlap; concurrent writes have no defined winner; a write that failed overall isn't rolled back where it succeeded; and restoring a node from an old backup can drop the new value below w. The condition is necessary, not sufficient.

7. **Why is a sloppy quorum "not a quorum at all"?**
   → Because the w writes may land on nodes outside the n home nodes, while the r reads poll the home nodes. There's no guaranteed intersection until hinted handoff completes. It buys durability — the data is on w nodes *somewhere* — not freshness.

8. **Two writes happen ten minutes apart on opposite sides of a network partition. Concurrent or not?**
   → Concurrent. Concurrency here means neither operation knew about the other, not that they overlapped in time. The partition prevented knowledge from flowing, so neither happens-before the other regardless of the ten-minute gap.

9. **Version vectors A={r1:2, r2:1} and B={r1:1, r2:3}. What should the server do?**
   → Keep both as siblings. A is ahead on r1, B is ahead on r2, so neither dominates — they're concurrent, and discarding either would lose data. Note the server decides this without inspecting the values at all.

10. **Why does union merging make deleted shopping-cart items reappear, and what fixes it?**
    → Because absence and deletion look identical to a union: if one sibling lacks an item, union can't tell whether it was never added or deliberately removed. The fix is a tombstone — an explicit deletion marker carrying a version — or a CRDT like an OR-Set, which tags each add and only tombstones adds that were actually observed.

---

*All quoted material, figures, examples and statistics are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Part II introduction and Chapter 5. Diagrams have been redrawn in ASCII from the book's originals. The simulations, measured outputs, and the OR-Set CRDT in §21 are supplementary material I added; every number quoted was produced by running the accompanying code.*
