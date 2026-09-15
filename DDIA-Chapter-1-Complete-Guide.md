# DDIA — Chapter 1: Reliable, Scalable and Maintainable Applications
### A complete study guide — every concept, every figure, plus extra material and code

> *"The Internet was done so well that most people think of it as a natural resource like the Pacific Ocean, rather than something that was man-made. When was the last time a technology with a scale like that was so error-free?"*
> — Alan Kay, quoted at the top of the chapter

---

## 0. The 30-second version

Chapter 1 is the **vocabulary chapter**. It builds no systems and gives no algorithms. It does one thing: it takes three words engineers throw around loosely — *reliable*, *scalable*, *maintainable* — and forces each one to mean something precise enough to argue about.

| Word | Loose meaning | Kleppmann's precise meaning |
|---|---|---|
| **Reliable** | "It doesn't break" | Continues working **correctly** even when **faults** occur |
| **Scalable** | "It handles a lot" | Has **strategies** for keeping performance good as **load parameters** grow |
| **Maintainable** | "The code is clean" | Operability + Simplicity + Evolvability — three human-facing properties |

Everything else in the 500-page book is machinery in service of these three words.

---

## 1. Setting the scene: data-intensive vs compute-intensive

**Compute-intensive** applications are bottlenecked on raw CPU — think video encoding, protein folding, ray tracing. You throw FLOPS at them.

**Data-intensive** applications are bottlenecked on something else entirely:

1. **Amount** of data — more than fits in one machine's memory or disk
2. **Complexity** of data — tangled relationships, many shapes, many sources
3. **Speed of change** — the data mutates fast, and so do the *requirements* about it

Almost every application you'll build professionally is data-intensive. Raw CPU is rarely the wall you hit.

### The standard building blocks

Kleppmann lists five, and this list is the skeleton of the whole book:

```
┌──────────────────┬─────────────────────────────────────┬──────────────────────┐
│ Building block   │ What it's for                       │ Examples             │
├──────────────────┼─────────────────────────────────────┼──────────────────────┤
│ Databases        │ Store data so it can be found later │ Postgres, MySQL,     │
│                  │                                     │ MongoDB, Cassandra   │
├──────────────────┼─────────────────────────────────────┼──────────────────────┤
│ Caches           │ Remember the result of an expensive │ Redis, Memcached     │
│                  │ operation, to speed up reads        │                      │
├──────────────────┼─────────────────────────────────────┼──────────────────────┤
│ Search indexes   │ Search by keyword / filter          │ Elasticsearch, Solr  │
├──────────────────┼─────────────────────────────────────┼──────────────────────┤
│ Stream processing│ Send a message to another process   │ Kafka, RabbitMQ,     │
│                  │ to be handled asynchronously        │ Flink                │
├──────────────────┼─────────────────────────────────────┼──────────────────────┤
│ Batch processing │ Periodically crunch a large amount  │ Hadoop, Spark        │
│                  │ of accumulated data                 │                      │
└──────────────────┴─────────────────────────────────────┴──────────────────────┘
```

**Why this seems obvious:** because these abstractions are *so successful* we stop noticing them. Nobody writes a storage engine from scratch — a database is a perfectly good tool for the job.

**Why it isn't obvious:** the categories leak. There are *many* databases with wildly different characteristics, several ways to build a search index, various approaches to caching. Picking wrong is expensive. And sometimes no single tool can do the job at all.

---

## 2. Thinking About Data Systems

### Why lump databases, queues, and caches together?

A database and a message queue both "store data for a while," but their **access patterns** are totally different, so their performance characteristics and implementations are totally different. So why one umbrella term — *data systems*?

Two reasons:

**Reason 1 — the categories are blurring.**

| Tool | Traditional category | But also used as |
|---|---|---|
| **Redis** | Cache / key-value store | Message queue (`LPUSH`/`BRPOP`, Streams) |
| **Kafka** | Message queue | Durable log with database-like guarantees |
| **Postgres** | Relational DB | Document store (JSONB), queue (`SKIP LOCKED`), search (tsvector) |
| **Elasticsearch** | Search index | Analytics store, primary datastore (risky!) |

**Reason 2 — one tool is no longer enough.** Requirements have become so wide-ranging that work gets broken into tasks, each handed to the tool that does it efficiently, and the tools get **stitched together with application code**.

That stitching is the key insight of the chapter. If you have a memcached layer and an Elasticsearch index alongside Postgres, it is **your application code's job** to keep them in sync. Nobody does it for you.

### 🔷 Figure 1-1 — One possible architecture for a data system combining several components

```
                    ┌───────────────────────────────────────────┐
                    │                  API                      │  ← clients see ONE service
                    └───────────────────────────────────────────┘
                                        ▲
                                        │ client requests
                                        ▼
  ┌────────────┐   1. read: check cache first    ┌──────────────────────┐
  │ In-memory  │◄───────────────────────────────►│                      │
  │   cache    │   4. invalidate / update cache  │   APPLICATION CODE   │
  └────────────┘                                 │                      │
                                                 └───┬────────┬─────┬───┘
                          2. cache misses & writes   │        │     │  3. search requests
                     ┌──────────────────────────────-┘        │     └──────────────┐
                     ▼                                        ▼                    ▼
             ┌───────────────┐                        ┌───────────────┐    ┌───────────────┐
             │   PRIMARY     │   capture changes      │  FULL-TEXT    │    │    MESSAGE    │
             │   DATABASE    │──────────────────────► │    INDEX      │    │     QUEUE     │
             │ (source of    │   (app code applies    │               │    │               │
             │   truth)      │    updates to index)   └───────────────┘    └───────┬───────┘
             └───────────────┘                                                     │
                                                                                   ▼
                                                                          ┌────────────────┐
                                                                          │ APPLICATION    │
                                                                          │ CODE (async)   │
                                                                          │ e.g. send email│
                                                                          └────────┬───────┘
                                                                                   ▼
                                                                           "OUTSIDE WORLD"
```

**How to read this diagram, arrow by arrow:**

1. A client hits **the API**. It has no idea any of this exists — that's the point.
2. Application code checks the **in-memory cache** first (fast path).
3. On a **cache miss**, or on any **write**, it goes to the **primary database** — the source of truth.
4. After a write, app code must **invalidate or update the cache**, or clients read stale data.
5. Changes are **captured** from the database and **applied to the full-text index** — again by app code (this is *change data capture*, Chapter 11).
6. **Search requests** bypass the DB and hit the index directly.
7. Side effects that don't need to block the response (emails, notifications) go on a **message queue** and get handled asynchronously by another piece of app code.

### 💡 The consequence: you are now a data system designer

> When you combine several tools to provide a service, the service's API hides those implementation details from clients. **You have essentially created a new, special-purpose data system out of smaller, general-purpose components.**

Your composite system now makes *promises* — "the cache will be correctly invalidated on writes, so clients see consistent results." Which raises the hard questions the whole book answers:

- How do you keep data **correct and complete** when things go wrong internally?
- How do you give **consistently good performance** even when parts are degraded?
- How do you **scale** to handle increased load?
- What does a **good API** for this service look like?

### The three concerns

Many factors shape a data system's design — team skills, legacy dependencies, delivery timescale, organizational risk tolerance, regulation. Those are situational. The book focuses on three that matter in *most* systems:

| Concern | One-line definition |
|---|---|
| **Reliability** | Work correctly (right function, right performance) *even in the face of adversity* — hardware faults, software faults, human error |
| **Scalability** | As the system grows in data volume / traffic / complexity, there are *reasonable ways* of dealing with that growth |
| **Maintainability** | Many different people will work on this over time — all of them should be able to work on it *productively* |

---

## 3. RELIABILITY

### The intuition

Software is "working correctly" when:

- ✅ It **performs the function** the user expected
- ✅ It **tolerates user mistakes** and unexpected usage
- ✅ Its **performance is good enough** for the use case, under expected load and data volume
- ✅ It **prevents unauthorized access and abuse**

> **Reliability ≈ "continuing to work correctly, even when things go wrong."**

### 🔑 Fault ≠ Failure — the most important distinction in the chapter

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │                                                                      │
   │   FAULT                     FAULT TOLERANCE                FAILURE   │
   │   ─────                     ───────────────                ───────   │
   │   ONE COMPONENT         ┌──────────────────────┐      THE SYSTEM     │
   │   deviates from    ───► │  retries, replicas,  │ ──►  AS A WHOLE     │
   │   its spec              │  failover, checksums │      stops serving  │
   │                         │  redundancy, quorums │      the user       │
   │   e.g. a disk dies      └──────────────────────┘                     │
   │        a packet drops             │                                  │
   │        a node hangs               │                                  │
   │                                   ▼                                  │
   │                        ✅ fault absorbed, NO failure                  │
   │                                                                      │
   │   ── Probability of a fault can NEVER be reduced to zero ──          │
   │   ── So: design mechanisms that stop faults BECOMING failures ──     │
   └──────────────────────────────────────────────────────────────────────┘
```

- A **fault** = one *component* deviating from its spec.
- A **failure** = the *system as a whole* stops providing the required service to the user.
- A system that anticipates faults and copes is **fault-tolerant** or **resilient**.

The term "fault-tolerant" is slightly misleading — it suggests tolerance of *every* possible fault, which isn't feasible. Kleppmann's joke: if the entire planet Earth were swallowed by a black hole, tolerating that fault would require web hosting in space — good luck getting the budget approved. **You only ever tolerate certain *types* of fault.** Naming which ones is part of the design.

### 🐒 The counter-intuitive bit: deliberately *increase* the fault rate

Many critical bugs are actually caused by **poor error handling** — the recovery path is the code path nobody tested. So: trigger faults on purpose, e.g. by randomly killing processes without warning.

This keeps the fault-tolerance machinery **continually exercised**, so you have real confidence it works when a fault arrives naturally at 3 a.m. **Netflix's Chaos Monkey** is the canonical example.

### ⚠️ When prevention beats cure

Generally, *tolerate* faults rather than *prevent* them. But some faults have no cure — **security** is the example. If an attacker has compromised a system and read sensitive data, that event **cannot be undone**. No amount of failover un-leaks a database. For those, prevention is the only option.

---

### 3.1 Hardware faults

Disks crash, RAM goes bad, the power grid blacks out, someone unplugs the wrong cable. At data-center scale these happen **constantly**.

**The arithmetic that makes this concrete:**

Hard disks have a **MTTF (mean time to failure)** of roughly 10–50 years.

```
10,000 disks ÷ (10 years × 365 days)  =  2.74 disk deaths per DAY
10,000 disks ÷ (50 years × 365 days)  =  0.55 disk deaths per DAY
```

So "on a storage cluster with 10,000 disks, expect on average one disk to die per day." At 10,000 disks, a "once in 10 years" event happens before lunch.

**The classic response — hardware redundancy:**

| Component | Redundancy mechanism |
|---|---|
| Disks | RAID |
| Power | Dual power supplies, UPS batteries, diesel generators |
| CPUs | Hot-swappable CPUs |
| Network | Bonded NICs, redundant switches |

This is well understood and can keep a machine up for years. But it **cannot completely prevent** hardware problems causing failures.

**Why the industry shifted from hardware to software fault tolerance:**

```
  PAST                                    PRESENT
  ────                                    ───────
  Few machines                            Many machines
       │                                       │
       ▼                                       ▼
  Single-machine failure is RARE          Fault rate scales with machine count
       │                                       │
       ▼                                       ▼
  Hardware redundancy is enough           Cloud VMs vanish without warning
  Restore a backup if it dies             (AWS prioritizes elasticity over
       │                                    single-machine reliability)
       ▼                                       │
  Multi-machine redundancy only for            ▼
  the few apps needing high availability  SOFTWARE fault tolerance:
                                          tolerate loss of ENTIRE machines
```

**The underrated operational bonus:** a single-server system needs **planned downtime** to reboot for an OS security patch. A system that tolerates machine failure can be patched **one node at a time, with zero downtime for the system as a whole**. Fault tolerance buys you rolling upgrades — which is often the reason you actually feel the benefit day to day.

---

### 3.2 Software errors (systematic faults)

Hardware faults are **random and largely uncorrelated** — one machine's disk failing doesn't imply another's will. There may be weak correlations (a hot server rack) but mass simultaneous hardware failure is unlikely.

**Software faults are the opposite: systematic and correlated across nodes.** Which is exactly why they cause far more system failures than hardware faults do. Your redundancy doesn't help — all your replicas run the same buggy code.

Examples from the book:

| Type | Concrete example |
|---|---|
| Bad-input crash | A bug crashing every app server given a particular input. **The leap second of June 30, 2012** hung many applications simultaneously due to a Linux kernel bug |
| Runaway resource use | A process eats all the CPU / memory / disk / bandwidth |
| Degraded dependency | A service you depend on slows down, becomes unresponsive, or returns **corrupted** responses |
| **Cascading failure** | A small fault in one component triggers a fault in another, which triggers more |

> These bugs **lie dormant for a long time** until triggered by unusual circumstances. What's revealed is that the software made an **assumption about its environment** — usually true, until one day it isn't.

**There is no quick solution.** Lots of small things help:

- Carefully thinking about **assumptions and interactions** in the system
- Thorough **testing**
- **Process isolation**
- Allowing processes to **crash and restart**
- **Measuring, monitoring, analyzing** behavior in production
- **Self-checking invariants**: if a message queue guarantees `messages in == messages out`, have it continuously verify that while running and raise an alert on discrepancy

---

### 3.3 Human errors

Humans design, build, and operate these systems, and humans are unreliable even with the best intentions.

> **One study of large internet services found that configuration errors by operators were the leading cause of outages — while hardware faults played a role in only 10–25% of outages.**

Sit with that number. Your careful RAID setup is addressing a quarter of your outage risk at most. The fingers on the keyboard are the bigger threat.

**The six approaches (best systems combine several):**

```
1. MINIMIZE OPPORTUNITY FOR ERROR
   Well-designed abstractions, APIs, admin interfaces make the RIGHT thing easy
   and the WRONG thing hard.
   ⚠️ Balance: too restrictive → people work around them → benefit negated.

2. DECOUPLE mistake-places from failure-places
   Fully-featured non-production SANDBOXES with real data, where people can
   explore and experiment without touching real users.

3. TEST THOROUGHLY at all levels
   Unit → integration → whole-system → manual.
   Automated tests are especially valuable for the CORNER CASES that rarely
   arise in normal operation.

4. ALLOW QUICK RECOVERY
   Fast config rollback. Gradual code rollout (bugs hit a small subset of users).
   Tools to RECOMPUTE data when the old computation turns out to be wrong.

5. DETAILED, CLEAR MONITORING  (other disciplines call this TELEMETRY)
   Performance metrics + error rates.
   → early warning signals
   → check whether assumptions/constraints are being violated
   → invaluable for diagnosis when something breaks
   📌 "Once a rocket has left the ground, telemetry is essential."

6. GOOD MANAGEMENT PRACTICES AND TRAINING
   Complex, important, and explicitly out of scope for this book.
```

---

### 3.4 How important is reliability?

Not just nuclear power stations and air traffic control. Mundane applications too:

- Bugs in **business applications** → lost productivity, and legal risk if figures are reported incorrectly
- Outages of **e-commerce** sites → lost revenue and reputation

And the argument that lands hardest: consider a **parent who stores all pictures and videos of their children in your photo app**. How would they feel if that database were suddenly corrupted? Would they even know how to restore from a backup?

**When can you cut corners?** Sometimes, deliberately:
- Reducing **development cost** — e.g. a prototype for an unproven market
- Reducing **operational cost** — e.g. a service with very narrow profit margins

The rule isn't "never cut corners." It's: **be very conscious of when you are cutting them.**

---

## 4. SCALABILITY

### ❗ The first thing to unlearn

> **It is meaningless to say "X is scalable" or "Y doesn't scale."**

Scalability is **not a one-dimensional label** you attach to a system. It is a *question*:

> *"If the system grows in a particular way, what are our options for coping with that growth? How can we add computing resources to handle the additional load?"*

The three-step method the whole section builds:

```
   ┌────────────────┐      ┌────────────────────┐      ┌─────────────────┐
   │ 1. DESCRIBE    │ ──►  │ 2. DESCRIBE        │ ──►  │ 3. COPE with    │
   │    THE LOAD    │      │    PERFORMANCE     │      │    growth       │
   │                │      │                    │      │                 │
   │ load parameters│      │ throughput /       │      │ scale up / out, │
   │ (numbers)      │      │ response time      │      │ elastic/manual, │
   │                │      │ percentiles        │      │ stateless first │
   └────────────────┘      └────────────────────┘      └─────────────────┘
```

---

### 4.1 Describing load — *load parameters*

You cannot ask "what if load doubles?" until you can say what load *is*. Load is described with a few numbers called **load parameters**. Which numbers depends entirely on your architecture:

- Requests per second to a web server
- Ratio of reads to writes in a database
- Number of simultaneously active users in a chat room
- Hit rate on a cache
- …or something specific to your domain

Sometimes the **average** is what matters. Sometimes your bottleneck is dominated by **a small number of extreme cases** — and picking the average would hide the whole problem. (The Twitter example is exactly this.)

---

### 🐦 The Twitter case study (data from November 2012)

Two main operations:

| Operation | Load |
|---|---|
| **Post tweet** | 4.6k requests/sec average, **over 12k/sec at peak** |
| **Home timeline** | **300k requests/sec** — view recent tweets from people you follow |

**Handling 12,000 writes/sec would be fairly easy.** That's not the problem.

The problem is **fan-out**.

> 📖 **Fan-out** — a term borrowed from electronic engineering, where it means the number of logic gate inputs attached to another gate's output (the output must supply enough current to drive all attached inputs). In transaction processing, it means **the number of requests to other services needed to serve one incoming request**.

Each user follows many people, and is followed by many people. That's the multiplier.

#### Approach 1 — Fan-out on READ

Posting a tweet just inserts into a global collection. Reading a timeline does the join at read time.

##### 🔷 Figure 1-2 — Simple relational schema for a Twitter home timeline

```
   currently logged-in user: 17055506
             │
             ▼
   ┌──────────────────────────────┐          ┌──────────────────────────────────────────────────┐
   │       follows table          │          │                  tweets table                    │
   ├──────────────┬───────────────┤          ├──────┬───────────┬──────────────────┬────────────┤
   │ follower_id  │  followee_id  │          │  id  │ sender_id │       text       │ timestamp  │
   ├──────────────┼───────────────┤          ├──────┼───────────┼──────────────────┼────────────┤
   │  17055506    │      12       │          │  20  │    12     │ just setting up  │ 1142974214 │
   └──────────────┴───────┬───────┘          │      │           │    my twttr      │            │
                          │                  └──────┴─────┬─────┴──────────────────┴────────────┘
                          │                               │
                          │        ┌──────────────────────┴──────────────────────┐
                          └───────►│                users table                  │◄──── join
                                   ├──────┬──────────────┬───────────────────────┤
                                   │  id  │ screen_name  │     profile_image     │
                                   ├──────┼──────────────┼───────────────────────┤
                                   │  12  │    jack      │      1234567.jpg      │
                                   └──────┴──────────────┴───────────────────────┘
```

The query:

```sql
SELECT tweets.*, users.*
FROM tweets
  JOIN users   ON tweets.sender_id      = users.id
  JOIN follows ON follows.followee_id   = users.id
WHERE follows.follower_id = current_user;
```

**Read this query as a cost model, not as SQL.** Every single one of the 300k timeline requests per second must:
1. look up everyone `current_user` follows,
2. find recent tweets from each of them,
3. merge and sort by time.

Cheap writes. **Brutally expensive reads.**

#### Approach 2 — Fan-out on WRITE

Maintain a **per-user home timeline cache** — like a mailbox. When someone posts, look up all their followers and **push** the tweet into every follower's mailbox. Reading is then just "read my mailbox" — the answer was precomputed.

##### 🔷 Figure 1-3 — Twitter's data pipeline for delivering tweets to followers

```
                                 FAN-OUT: deliver tweet to each        ┌──────────────────────────┐
                                 follower (up to 31 M per user)        │ Timeline for recipient 1 │
                                                                       │  [T7][T5][T3][T1]        │
   ┌──────────────┐      ┌────────────────────────────┐   ┌───────────►└──────────────────────────┘ ─┐
   │ User posts   │      │        ALL TWEETS          │   │            ┌──────────────────────────┐  │
   │    tweet     │─────►│ [T8][T7][T6][T5][T4][T3]   │───┼───────────►│ Timeline for recipient 2 │  ├──► GET HOME
   │              │      │        [T2][T1]            │   │            │  [T8][T6][T5]            │  │    TIMELINE
   └──────────────┘      └────────────────────────────┘   │            └──────────────────────────┘  │  (website, API)
                                                          │            ┌──────────────────────────┐  │
                                                          └───────────►│ Timeline for recipient 3 │ ─┘
                                                                       │  [T8][T7][T5][T4][T3]    │
                                                                       └──────────────────────────┘
      ▲                                    ▲                                        ▲
      │                                    │                                        │
   4.6 k writes/sec                 345 k writes/sec                        300 k reads/sec
   (what users do)              (what the SYSTEM must do)                 (now cheap — O(1))
```

**The arithmetic — this is the whole point of the figure:**

```
   4,600 tweets/sec  ×  ~75 followers on average  =  345,000 timeline-cache writes/sec
   ─────────────────    ─────────────────────────    ────────────────────────────────
   what users do        the FAN-OUT multiplier       what your system actually does
```

Twitter started with **Approach 1** and the system struggled with home timeline queries, so they **switched to Approach 2**. Why it works: the tweet-publish rate is *almost two orders of magnitude lower* than the timeline-read rate (4.6k vs 300k). **So do more work at write time and less at read time.**

#### 💥 The celebrity problem — where the average lies to you

The average of 75 followers **hides** the distribution. Some users have **over 30 million followers**.

```
   Ordinary user tweets  →      75 writes    → done in microseconds
   Celebrity tweets      →  31,000,000 writes → and Twitter targets
                                                delivery within 5 SECONDS
```

**This is why "the distribution of followers per user (weighted by how often those users tweet)" is the key load parameter for Twitter** — not requests per second. It determines the fan-out load. Pick the wrong load parameter and you'd conclude Twitter is an easy system to build.

#### The final twist — the hybrid

Now that Approach 2 works robustly, Twitter has moved to a **hybrid**:

| User type | Strategy |
|---|---|
| Most users | Fan-out on **write** — tweets pushed into follower timelines at post time |
| Small number of **celebrities** | **Excepted from fan-out**. Their tweets are fetched separately at read time and merged into the timeline (Approach 1 style) |

Best of both: no 31-million-write storms, and reads stay cheap because only a handful of celebrity timelines need merging. **This hybrid delivers consistently good performance** — and it's a nice lesson that mature systems rarely pick one textbook answer.

---

### 4.2 Describing performance

Once load is described, ask it two ways:

1. **Fix resources, increase load** → how is performance affected?
2. **Fix performance, increase load** → how much must resources grow?

Both need performance numbers. Which number depends on the system type:

| System type | Metric that matters | Meaning |
|---|---|---|
| **Batch** (e.g. Hadoop) | **Throughput** | Records processed per second, or total job time on a dataset of given size |
| **Online** | **Response time** | Time between client sending a request and receiving a response |

> 📌 **Footnote worth remembering:** ideally, batch job time = dataset size ÷ throughput. In practice it's longer, due to **skew** (data not spread evenly across workers) or **waiting for the slowest task**.

#### 📖 Latency vs Response time — commonly confused, not the same

```
   ├────────────────────── RESPONSE TIME (what the CLIENT sees) ──────────────────────┤

   client                network        QUEUE          SERVICE TIME        network      client
   sends   ─────────────────────────►  ░░░░░░░  ─────────────────────────► ──────────► receives
   request                             waiting        actual processing
                                          │
                                          └──► LATENCY = the duration a request is
                                               WAITING to be handled — during which
                                               it is *latent*, awaiting service
```

- **Response time** = what the client sees = service time **+ network delays + queueing delays**
- **Latency** = the duration a request is waiting to be handled

They get used synonymously all the time. They aren't the same, and the difference matters most exactly when the system is under stress.

#### Response time is a *distribution*, not a number

Even issuing the *same* request repeatedly gives different response times every try. Sources of random extra latency:

- A **context switch** to a background process
- **Loss of a network packet** and TCP retransmission
- A **garbage collection pause**
- A **page fault** forcing a read from disk
- **Mechanical vibrations in the server rack** ← real; the book cites a case of 500 ms disk latency traced to a slightly unbalanced rack

So: think of response time as **a distribution of values you can measure**, never a single number.

##### 🔷 Figure 1-4 — Illustrating mean and percentiles (100 sample requests)

```
  response
    time
      ▲
      │                                                                        █
p99 ──┼────────────────────────────────────────────────────────────────────█───█──── ← 1 in 100 is worse
      │                                        █                           █   █
      │                                        █                           █   █
p95 ──┼──────────────█──────────────────█──────█───────────────────────█───█───█──── ← 5 in 100 are worse
      │              █          █       █      █        █              █   █   █
mean ─┼──█───────────█──────█───█───█───█───█──█────█───█────█─────█────█───█───█──── ← DRAGGED UP by outliers
      │  █   █   █   █  █   █   █   █   █   █  █  █ █   █    █  █  █    █   █   █
p50 ──┼──█───█───█───█──█───█───█───█───█───█──█──█─█───█────█──█──█────█───█───█──── ← half above, half below
      │  █   █   █   █  █   █   █   █   █   █  █  █ █   █    █  █  █    █   █   █
      │  █   █   █   █  █   █   █   █   █   █  █  █ █   █    █  █  █    █   █   █
      └──┴───┴───┴───┴──┴───┴───┴───┴───┴───┴──┴──┴─┴───┴────┴──┴──┴────┴───┴───┴───►
                                    100 requests (each bar = one request)

   Most requests are fast. A few OUTLIERS take much longer.
   Note that mean sits ABOVE the median — that's the outliers pulling it up.
```

#### Why the mean is a bad metric

The *average* (arithmetic mean: sum ÷ n) is commonly reported. But it's a poor description of "typical," **because it doesn't tell you how many users actually experienced that delay.**

**Percentiles are better.** Sort your response times fastest → slowest:

| Percentile | Name | Meaning |
|---|---|---|
| **p50** | median | Half of requests are faster, half slower. Good "typical" metric |
| **p95** | | 95 of 100 requests are faster than this; 5 of 100 take this long or more |
| **p99** | | 1 in 100 requests is worse |
| **p999** | | 1 in 1,000 requests is worse |

**Here's what that looks like on real generated data (100,000 requests, code in the appendix):**

```
   mean :   140.5 ms      ← nobody's experience looks like this
   p50  :   101.1 ms      ← the actual typical request
   p90  :   124.4 ms
   p95  :   176.6 ms
   p99  :  1158.3 ms      ← 8× worse than "average"!
   p99.9:  1531.2 ms
```

The mean (140 ms) is **higher than 90% of all requests** and **11× better than the p99**. It describes essentially no one. That's the argument, in numbers.

> ⚠️ **Subtle but important:** the median is the half-way point *for a single request*. If a user makes several requests — over a session, or because one page loads several resources — the probability that **at least one** is slower than the median is **much greater than 50%**.

#### Why high percentiles matter — the Amazon argument

**Amazon specifies internal service response times at the p99.9**, even though it affects only 1 in 1,000 requests. Why bother?

> **Because the customers with the slowest requests are often those who have made the most purchases** — they have the most data, so their requests are the most expensive. **They are your most valuable customers.**

The supporting numbers cited in the chapter:

| Finding | Source |
|---|---|
| **100 ms** increase in response time → **1% reduction in sales** | Amazon |
| **1-second** slowdown → **16% drop** in a customer satisfaction metric | Others |

**But there's a stopping point.** Amazon deemed optimizing the **p99.99** (slowest 1 in 10,000) **too expensive and not worth the benefit.** Very high percentiles are easily affected by random events outside your control, and the returns diminish sharply. Knowing where to stop is part of the skill.

#### SLOs and SLAs

Percentiles show up in contracts:

- **SLO** — Service Level *Objective*: internal performance target
- **SLA** — Service Level *Agreement*: contract defining expected performance and availability

Example SLA from the book:

```
   ┌────────────────────────────────────────────────────────────┐
   │  The service is considered UP if:                          │
   │      • median response time  <  200 ms                     │
   │      • 99th percentile       <    1 s                      │
   │        (if it's slower than that, it might as well be down)│
   │                                                            │
   │  The service must be UP at least 99.9% of the time.        │
   │                                                            │
   │  → Sets expectations for clients                           │
   │  → Allows customers to demand a REFUND if unmet            │
   └────────────────────────────────────────────────────────────┘
```

#### Queueing delays and head-of-line blocking

At high percentiles, **queueing delay is often the dominant component** of response time.

```
   A server processes only a few things in parallel (limited by CPU cores).

   ┌────────────────────────────────────────────────────────────────────────┐
   │  arriving requests                                                     │
   │                                                                        │
   │  [fast][fast][fast]  ►►►  [ SLOOOOW REQUEST ]  ◄── holds the line      │
   │                            ▲                                           │
   │  these are quick to        │  all cores busy                           │
   │  process, but they must    │                                           │
   │  WAIT ─────────────────────┘                                           │
   └────────────────────────────────────────────────────────────────────────┘
                    = HEAD-OF-LINE BLOCKING
   The client sees a slow response even though ITS request was fast to process.
```

**Two practical consequences:**

1. **Measure response times on the client side**, not the server side. The server thinks it was fast. The client knows better.
2. **When load-testing, the generating client must keep sending requests independently of response time.** If the client waits for the previous request to finish before sending the next, it **artificially keeps queues shorter than reality** and your measurements are wrong. (This is the bug known as *coordinated omission* — see §7.4.)

#### 📦 Percentiles in Practice (the sidebar)

**Tail amplification.** High percentiles matter most in backend services called **multiple times** to serve one end-user request. Even if calls are parallel, the end-user request waits for **the slowest** one. It takes **just one slow call** to make the whole request slow.

##### 🔷 Figure 1-5 — One slow backend slows the entire end-user request

```
                              END-USER REQUEST
                                     ▲
                 ┌───────────────────┴───────────────────┐
                 │          WEB APPLICATION              │
                 │   waits for ALL backends to return    │
                 │   → total = MAX(all) = 487 ms         │
                 └───────────────────┬───────────────────┘
       ┌───────┬───────┬─────────┬───┴─────┬────────┬────────┬────────┐
       ▼       ▼       ▼         ▼         ▼        ▼        ▼        ▼
   ┌───────┬───────┬────────┬────────┬────────┬──────────┬────────┐
   │Backend│Backend│Backend │Backend │Backend │ Backend  │Backend │
   │   1   │   2   │   3    │   4    │   5    │    6     │   7    │
   ├───────┼───────┼────────┼────────┼────────┼──────────┼────────┤
   │ 92 ms │ 76 ms │ 103 ms │ 143 ms │  86 ms │ ▓487 ms▓ │ 133 ms │
   └───────┴───────┴────────┴────────┴────────┴─────┬────┴────────┘
                                                    │
                                        THIS ONE decides everything.
                                        Six fast backends bought you nothing.
```

**The probability, computed (code in appendix):** if just **1% of backend calls are slow**:

```
     1 backend call   →   1.0%  of user requests hit a slow call
     5 backend calls  →   4.9%
    10 backend calls  →   9.6%
    50 backend calls  →  39.5%
   100 backend calls  →  63.4%   ← your "1% problem" now affects 2 users in 3
```

A rare backend problem becomes a **common** user problem, purely through fan-out. This is the essence of Dean & Barroso's *"The Tail at Scale."*

**Computing percentiles efficiently.** For a dashboard you want, say, a rolling 10-minute window, recalculated every minute.

- **Naïve:** keep every response time in the window and sort the list each minute. Works; often too expensive.
- **Better:** approximation algorithms at minimal CPU/memory cost:
  - **Forward decay**
  - **t-digest**
  - **HdrHistogram**

> ⚠️ **Beware:** **averaging percentiles is mathematically meaningless** — whether to reduce time resolution or to combine several machines. **The right way to aggregate response time data is to add the histograms.**

**Here's that mistake measured (code in appendix), across 3 servers with uneven traffic:**

```
   per-server p99  : 1115 ms, 1162 ms, 1544 ms
   average of p99s : 1274 ms   ✗ WRONG — off by 224 ms
   true p99        : 1498 ms   ✓ RIGHT — computed from merged raw data
```

The average of the p99s isn't even *close*, and it errs **optimistically** — it tells you things are better than they are. Note *why*: averaging throws away the fact that server 3 handled 80% of the traffic. Percentiles are positions in a distribution; you cannot average positions.

---

### 4.3 Approaches for coping with load

> **An architecture appropriate for one level of load is unlikely to cope with ten times that load.** On a fast-growing service, expect to **re-think your architecture on every order of magnitude** load increase — perhaps more often.

#### Scaling up vs scaling out

```
   ┌──────────────────────────────────┐    ┌──────────────────────────────────────┐
   │  SCALING UP (vertical)           │    │  SCALING OUT (horizontal)            │
   │                                  │    │  a.k.a. SHARED-NOTHING architecture  │
   │        ┌──────────┐              │    │                                      │
   │        │          │              │    │   ┌────┐ ┌────┐ ┌────┐ ┌────┐        │
   │        │  ONE     │              │    │   │ sm │ │ sm │ │ sm │ │ sm │        │
   │        │  BIG     │              │    │   └────┘ └────┘ └────┘ └────┘        │
   │        │  MACHINE │              │    │   ┌────┐ ┌────┐ ┌────┐ ┌────┐        │
   │        │          │              │    │   │ sm │ │ sm │ │ sm │ │ sm │        │
   │        └──────────┘              │    │   └────┘ └────┘ └────┘ └────┘        │
   │                                  │    │                                      │
   │  ✅ SIMPLER                       │    │  ✅ cheaper per unit of capacity      │
   │  ❌ high-end machines get         │    │  ✅ no single-machine ceiling         │
   │     VERY expensive                │    │  ❌ distributed-systems complexity    │
   └──────────────────────────────────┘    └──────────────────────────────────────┘

   REALITY: good architectures use a PRAGMATIC MIXTURE.
   Several fairly powerful machines can be simpler AND cheaper
   than a large number of small VMs.
```

Note the book's specific pushback on cargo-culting: "a large number of small virtual machines" is not automatically the right answer, and often isn't.

#### Elastic vs manually scaled

| | **Elastic** | **Manually scaled** |
|---|---|---|
| How | Automatically adds resources on detecting load increase | A human analyses capacity and adds machines |
| Good when | Load is **highly unpredictable** | You want **simplicity** |
| Trade-off | More moving parts | **Fewer operational surprises** |

The book leans toward manual for predictable load — "fewer operational surprises" is doing real work in that sentence, and it links forward to rebalancing partitions (Chapter 6).

#### ⚠️ Stateless is easy; stateful is hard

```
   STATELESS SERVICE                          STATEFUL DATA SYSTEM
   ─────────────────                          ────────────────────
   Distributing across machines               Going from single node to
   is FAIRLY STRAIGHTFORWARD                  distributed introduces A LOT
   (any node can serve any request)           of additional complexity
                                              (replication, partitioning,
                                               consistency, consensus…)
```

> **Common wisdom until recently: keep your database on a single node (scale up) until scaling cost or high-availability requirements force you to distribute it.**

Kleppmann then hedges thoughtfully: as tools and abstractions improve, this may change. It's conceivable distributed data systems become the default even for modest data volumes. (Writing in 2026, that prediction has largely come true — managed distributed stores like Spanner, CockroachDB, DynamoDB, and Aurora are now the default choice for a lot of teams who don't have "big data" at all.)

#### 🪄 There is no magic scaling sauce

> The architecture of systems that operate at large scale is **usually highly specific to the application.** There is no generic, one-size-fits-all scalable architecture — informally known as **magic scaling sauce**.

The bottleneck could be volume of reads, volume of writes, volume of data to store, complexity of data, response-time requirements, access patterns — or, usually, **some mixture of all of these**.

**The illustration that makes it click:**

```
   SYSTEM A                            SYSTEM B
   100,000 requests/sec × 1 kB   vs    3 requests/min × 2 GB
   = 100 MB/sec                        = 100 MB/sec

   ↑ SAME DATA THROUGHPUT, COMPLETELY DIFFERENT SYSTEMS ↑

   A: connection handling, request      B: streaming, chunked transfer,
      routing, tiny fast lookups,          huge buffers, object storage,
      cache-friendly, latency-bound        resumable uploads, throughput-bound
```

**A scalable architecture is built around assumptions about which operations are common and which are rare — i.e. the load parameters.** If those assumptions turn out wrong, the scaling effort is **at best wasted, at worst counter-productive.**

> 💡 **In an early-stage startup or unproven product, it's usually more important to be able to iterate quickly on product features than to scale to some hypothetical future load.**

Reassuringly, though: scalable architectures are still assembled from **general-purpose building blocks arranged in familiar patterns** — which is what the rest of the book teaches.

---

## 5. MAINTAINABILITY

> **The majority of the cost of software is not in its initial development, but in its ongoing maintenance** — fixing bugs, keeping systems operational, investigating failures, adapting to new platforms, modifying for new use cases, repaying technical debt, adding features.

And yet many engineers dislike maintaining "legacy" systems — fixing other people's mistakes, outdated platforms, systems forced to do things they were never intended for. *"Every legacy system is unpleasant in its own way,"* so general advice is hard.

The response: **design software so it minimizes pain during maintenance — and so we avoid creating legacy software ourselves.** Three design principles:

```
   ╔═══════════════════════════════════════════════════════════════════════════╗
   ║                          MAINTAINABILITY                                  ║
   ╠═══════════════════╦═══════════════════════╦═══════════════════════════════╣
   ║   OPERABILITY     ║      SIMPLICITY       ║        EVOLVABILITY           ║
   ║                   ║                       ║                               ║
   ║ Make it easy for  ║ Make it easy for NEW  ║ Make it easy for engineers    ║
   ║ OPERATIONS teams  ║ ENGINEERS to under-   ║ IN FUTURE to change the       ║
   ║ to keep the       ║ stand the system, by  ║ system, adapting it for       ║
   ║ system running    ║ removing as much      ║ unanticipated use cases as    ║
   ║ smoothly          ║ complexity as possible║ requirements change           ║
   ║                   ║                       ║                               ║
   ║   → the PRESENT   ║  ⚠️ NOT the same as    ║ a.k.a. extensibility,         ║
   ║     of the system ║  simplicity of the UI ║ modifiability, plasticity     ║
   ║                   ║   → UNDERSTANDING it  ║   → the FUTURE of the system  ║
   ╚═══════════════════╩═══════════════════════╩═══════════════════════════════╝
```

A neat way to hold them: **operability is about *today*, simplicity is about *understanding*, evolvability is about *tomorrow*.**

---

### 5.1 Operability: making life easy for operations

> **"Good operations can often work around the limitations of bad (or incomplete) software, but good software cannot run reliably with bad operations."**

Some operations can and should be automated — but humans still have to set up that automation and verify it works.

**What a good operations team does:**

- 🔍 **Monitor** system health, and quickly restore service if it goes bad
- 🕵️ **Track down the cause** of problems — failures, degraded performance
- 🔄 **Keep software and platforms up to date**, including security patches
- 🔗 **Keep tabs on how systems affect each other**, so a problematic change is avoided *before* it causes damage
- 📈 **Anticipate future problems** and solve them before they occur — e.g. capacity planning
- 🛠️ **Establish good practices and tools** for deployment, configuration management
- 🚚 **Perform complex maintenance** — e.g. moving an application from one platform to another
- 🔒 **Maintain security** as configuration changes are made
- 📋 **Define processes** that make operations predictable and keep production stable
- 🧠 **Preserve the organization's knowledge** about the system, even as individual people come and go

> **Good operability means making routine tasks easy, so the operations team can focus effort on high-value activities.**

**What data systems can do to make routine tasks easy:**

| Property | What it means |
|---|---|
| **Visibility** | Good monitoring into runtime behavior and internals |
| **Automation support** | Good integration with standard tools |
| **No machine dependency** | Machines can be taken down for maintenance while the system keeps running |
| **Documentation** | Clear operational model: *"if I do X, Y will happen"* |
| **Good defaults** | But administrators can **override** them when needed |
| **Self-healing** | Where appropriate — but with **manual control** available |
| **Predictability** | Minimize surprises |

Notice the pattern in the last three: **sensible automatic behavior, with an escape hatch.** Automation that can't be overridden is a liability during an incident.

---

### 5.2 Simplicity: managing complexity

Small projects can have "delightfully simple and expressive code." As they grow they often become very complex and hard to understand. Complexity **slows down everyone**, which further increases maintenance cost. A project mired in complexity is a **big ball of mud**.

**Symptoms of complexity:**

```
   🔸 explosion of the state space
   🔸 tight coupling of modules
   🔸 tangled dependencies
   🔸 inconsistent naming and terminology
   🔸 hacks aimed at solving performance problems
   🔸 special-casing to work around issues elsewhere
```

**Consequences:** budgets and schedules overrun; and there's a **greater risk of introducing bugs** when making a change — hidden assumptions, unintended consequences, and unexpected interactions are more easily overlooked when the system is hard to reason about.

#### 🔑 Essential vs Accidental complexity

> **Moseley and Marks:** complexity is **accidental** if it is *not inherent in the problem the software solves (as seen by the users)*, but **arises only from the implementation.**

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │  ESSENTIAL complexity          │  ACCIDENTAL complexity             │
   │  ────────────────────          │  ─────────────────────             │
   │  Inherent in the problem       │  Arises only from HOW you built it │
   │  as the USER sees it           │                                    │
   │                                │                                    │
   │  e.g. "tax rules genuinely     │  e.g. tax rules scattered across   │
   │  differ across 30 countries"   │  30 copy-pasted if-branches in 8   │
   │                                │  services with 3 naming schemes    │
   │                                │                                    │
   │  ❌ CANNOT be removed           │  ✅ CAN and SHOULD be removed       │
   │     (only managed)             │                                    │
   └─────────────────────────────────────────────────────────────────────┘
```

**Making a system simpler does not necessarily mean reducing its functionality.** It can mean removing *accidental* complexity. This distinction is what lets "simplify it" be a real engineering instruction rather than a request to delete features.

#### Abstraction: the best tool we have

> **A good abstraction can hide a great deal of implementation detail behind a clean, simple-to-understand façade.**

A good abstraction is also **reusable across a wide range of applications** — which is not just more efficient than re-implementing, it produces **higher-quality software**, because quality improvements in the abstracted component benefit **every** application using it.

**Examples given:**

| Abstraction | What it hides |
|---|---|
| **High-level programming languages** | Machine code, CPU registers, syscalls |
| **SQL** | Complex on-disk and in-memory data structures, concurrent requests from other clients, inconsistencies after crashes |

> When programming in a high-level language, **we are still using machine code** — we just aren't using it *directly*, because the abstraction saves us from having to think about it.

**But — finding good abstractions is very hard.** In distributed systems specifically, there are many good *algorithms*, but it's much less clear **how to package them into abstractions** that keep system complexity manageable. (That open problem is essentially why Part II of the book exists.)

---

### 5.3 Evolvability: making change easy

> **It's extremely unlikely that your system's requirements will remain unchanged forever.**

Why requirements change:
- You learn new facts
- Previously unanticipated use cases emerge
- Business priorities change
- Users request new features
- New platforms replace old platforms
- Legal or regulatory requirements change
- Growth of the system forces architectural changes

**Organizationally:** *agile* working patterns provide a framework for adapting to change, plus technical tools like **TDD** and **refactoring**.

**But there's a scope gap:**

```
   AGILE / TDD / refactoring          THIS BOOK
   ─────────────────────────          ─────────
   Small, LOCAL scale                 LARGER DATA SYSTEM level
   A couple of source files           Several applications/services
   within the same application        with different characteristics

              ↓                                 ↓
        "refactoring"                     "EVOLVABILITY"
```

**The motivating question Kleppmann poses:** *how would you "refactor" Twitter's architecture for assembling home timelines from Approach 1 to Approach 2?* You can't do that with an IDE refactor button. It's a data migration, a dual-write period, a backfill, a cutover, and a rollback plan. That's what evolvability means at the system level.

**And note the link back:** evolvability is closely tied to **simplicity and abstractions** — simple, easy-to-understand systems are usually easier to modify than complex ones. The three maintainability principles are not independent; simplicity is upstream of evolvability.

---

## 6. Chapter Summary (as the book states it)

An application must meet:

| Requirement type | Meaning | Examples |
|---|---|---|
| **Functional** | What it should do | Store, retrieve, search, process data |
| **Non-functional** | General properties | Security, reliability, compliance, scalability, compatibility, maintainability |

The three explored in detail:

**🛡️ RELIABILITY** — making systems work correctly, even when faults occur.

| Fault source | Character |
|---|---|
| Hardware | Typically **random and uncorrelated** |
| Software | Bugs are typically **systematic and hard to deal with** (correlated!) |
| Humans | Inevitably make mistakes from time to time |

Fault-tolerance techniques can **hide certain types of fault from the end user**.

**📈 SCALABILITY** — strategies for keeping performance good as load increases. Requires describing load and performance **quantitatively** — Twitter home timelines for load, response-time **percentiles** for performance. In a scalable system, you can **add processing capacity to remain reliable under high load.**

**🔧 MAINTAINABILITY** — in essence, **making life better for the engineering and operations teams** who work with the system. Good **abstractions** reduce complexity and ease modification. Good **operability** means visibility into health plus effective ways of managing it.

> **There is unfortunately no quick answer to making applications reliable, scalable or maintainable.** But certain **patterns and techniques keep re-appearing** across different kinds of application — and that's what the book covers.

---
---

# 7. 🎁 EXTRA MATERIAL (not in the book — added to round out the chapter)

Everything above is Kleppmann's. This section is supplementary — things that connect directly to Chapter 1 concepts and that I've found make them stick.

## 7.1 The "nines" — what availability targets actually cost

The book mentions "99.9% of the time" in the SLA example without unpacking it. Here's the downtime budget each nine buys, computed:

| Availability | Downtime / year | Downtime / month | Realistic meaning |
|---|---|---|---|
| **99%** ("two nines") | 5,259.6 min (~3.65 days) | ~7.3 hours | Hobby project |
| **99.9%** ("three nines") | 526.0 min (~8.8 hours) | ~43.8 min | Typical SaaS SLA |
| **99.99%** ("four nines") | 52.6 min | ~4.4 min | Serious infrastructure |
| **99.999%** ("five nines") | 5.3 min | ~26 seconds | Telco / payments |

**The thing to notice:** at five nines, your **entire annual budget** is 5 minutes. A single unplanned reboot blows the year. This is why the book insists fault tolerance must let you patch **one node at a time** — with five nines you cannot afford planned downtime *either*.

Note also: each nine costs roughly **10× more** than the previous one, for **10× less** downtime. This is the same diminishing-returns curve as Amazon's decision to stop at p99.9 rather than p99.99. Same shape, different axis.

## 7.2 Little's Law — why queues explode near saturation

Chapter 1 says "queueing delays are often a large part of the response time at high percentiles" but doesn't say *why* it gets bad so suddenly. Queueing theory does.

For an M/M/1 queue, with **utilisation ρ** (arrival rate ÷ service rate):

```
   Average time in system  T  =  service_time / (1 − ρ)
```

```
   ρ = 0.50  →  T = 2.0 × service time      "we're at half capacity, all fine"
   ρ = 0.80  →  T = 5.0 × service time
   ρ = 0.90  →  T = 10.0 × service time
   ρ = 0.95  →  T = 20.0 × service time
   ρ = 0.99  →  T = 100.0 × service time    ← 💥
```

```
   response
     time  ▲                                              ┃
           │                                             ┃
           │                                            ┃
           │                                         ┏━━┛
           │                                  ┏━━━━━━┛
           │                    ┏━━━━━━━━━━━━━┛
           │━━━━━━━━━━━━━━━━━━━━┛
           └──────┬──────┬──────┬──────┬──────┬──────┬───► utilisation ρ
                 0.5    0.6    0.7    0.8    0.9    1.0
                                              ▲
                                    THE CLIFF — this is why systems
                                    seem "fine" and then fall over
                                    with no warning
```

**Why this matters for Chapter 1:** it explains the *mechanism* behind the p99 blowing up while the p50 looks healthy. It also explains why capacity planning targets ~50–70% utilisation rather than 95% — you are buying headroom against the cliff, not wasting money.

## 7.3 RED and USE — practical monitoring frameworks

The book says "set up detailed and clear monitoring" without saying *what to monitor*. Two standard answers:

**RED — for services (request-driven, user-facing):**
```
   R ate      — requests per second           ← your load parameter
   E rrors    — failed requests per second    ← reliability
   D uration  — response time DISTRIBUTION    ← percentiles, not averages!
```

**USE — for resources (CPU, disk, network, memory):**
```
   U tilisation — % time the resource was busy       ← feeds Little's Law
   S aturation  — how much queued work is waiting    ← the early warning
   E rrors      — error event count
```

Rule of thumb: **RED tells you the user is unhappy. USE tells you why.**

## 7.4 Coordinated omission — the load-testing bug the book warns about

Chapter 1 says the load-generating client "needs to keep sending requests independently of the response time." Gil Tene named this failure mode **coordinated omission**, and it's worth seeing concretely:

```
   ❌ CLOSED LOOP (broken)             ✅ OPEN LOOP (correct)
   ────────────────────────            ──────────────────────
   send → wait for response → send     send at fixed rate, regardless
                                       of whether responses come back

   When the server stalls for 1s,      When the server stalls for 1s,
   the client STOPS SENDING.           1000 requests pile up (at 1000 rps)
   Those requests never existed,       and each records its FULL wait.
   so they never record a bad time.

   → the stall is INVISIBLE            → the stall shows up in the p99
   → your p99 looks great              → your p99 tells the truth
   → production falls over
```

The system "coordinates" with your measurement tool to hide its own worst behaviour. Tools like `wrk2` and HdrHistogram were built specifically to correct for it.

## 7.5 The Universal Scalability Law — why scaling out has a ceiling

The book says "an architecture appropriate for one level of load is unlikely to cope with ten times that load." Gunther's USL says *why* mathematically:

```
                        N
   Speedup(N) = ─────────────────────────
                1 + α(N−1) + βN(N−1)

   α = contention  (serialization — queueing for a shared resource)
   β = coherency   (crosstalk — nodes must agree with each other)
```

- With **α = β = 0**: perfect linear scaling. Never happens.
- With **α > 0, β = 0**: this is Amdahl's Law — you plateau.
- With **β > 0**: **you get WORSE past a certain N.** Adding machines *reduces* throughput, because coordination overhead grows as N².

That β term is the mathematical shape of "distributing stateful data systems introduces a lot of additional complexity." Replication and consensus are coherency costs. It's also why the book's "keep your database on one node until forced otherwise" advice is more than conservatism.

## 7.6 A concrete map of Chapter 1 to real tools (2026)

| Ch.1 concept | Where you meet it in practice |
|---|---|
| Fault vs failure | Kubernetes pod restarts (fault) vs. 5xx to users (failure) |
| Chaos Monkey | Chaos Mesh, Gremlin, AWS Fault Injection Simulator |
| Percentiles / histograms | Prometheus `histogram_quantile()`, OpenTelemetry, Datadog |
| Don't average percentiles | Prometheus native histograms; HDR histogram merging |
| Fan-out on write | Redis lists / sorted sets per user timeline |
| Hybrid fan-out | Basically every social feed: push for normal users, pull for celebrities |
| Change data capture (Fig 1-1) | Debezium → Kafka → Elasticsearch |
| Cache invalidation (Fig 1-1) | Redis + write-through / TTL / event-driven invalidation |
| Elastic scaling | Kubernetes HPA, AWS Auto Scaling, serverless |
| SLO / error budgets | Google SRE workbook, Nobl9, Sloth |

---

# 8. 💻 CODE APPENDIX

All of these are runnable. The outputs quoted throughout this guide came from actually executing them.

## 8.1 Mean vs percentiles — see the lie yourself

```python
import random, statistics as st
random.seed(42)

def sample_latency():
    """95% fast requests; 5% tail (GC pause, disk seek, TCP retransmit)."""
    if random.random() < 0.95:
        return random.gauss(100, 15)     # ~100 ms
    return random.gauss(900, 300)        # the tail

lat = sorted(max(1, sample_latency()) for _ in range(100_000))

def pct(sorted_vals, p):
    """Linear-interpolated percentile."""
    k = (len(sorted_vals) - 1) * p / 100
    f, c = int(k), min(int(k) + 1, len(sorted_vals) - 1)
    return sorted_vals[f] + (sorted_vals[c] - sorted_vals[f]) * (k - f)

print(f"mean : {st.mean(lat):7.1f} ms")
for p in (50, 90, 95, 99, 99.9):
    print(f"p{p:<5}: {pct(lat, p):7.1f} ms")
```

```
mean :   140.5 ms      ← describes nobody
p50   :   101.1 ms
p90   :   124.4 ms
p95   :   176.6 ms
p99   :  1158.3 ms
p99.9 :  1531.2 ms
```

## 8.2 Proving that averaging percentiles is meaningless

```python
servers = []
for i in range(3):
    n     = [10_000, 10_000, 80_000][i]   # uneven traffic — server 3 is busiest
    shift = [0, 0, 400][i]                # and server 3 is sick
    servers.append(sorted(max(1, sample_latency() + shift) for _ in range(n)))

per_server_p99 = [pct(s, 99) for s in servers]
print("per-server p99 :", [f"{v:.0f}" for v in per_server_p99])
print(f"average of p99s: {st.mean(per_server_p99):.0f} ms   <-- WRONG")

merged = sorted(v for s in servers for v in s)   # merge RAW data
print(f"true p99       : {pct(merged, 99):.0f} ms   <-- RIGHT")
```

```
per-server p99 : ['1115', '1162', '1544']
average of p99s: 1274 ms   <-- WRONG
true p99       : 1498 ms   <-- RIGHT
```

**Why it's wrong:** averaging discards the fact that server 3 served 80% of traffic. Percentiles are *positions in a distribution*. You cannot average positions — you must merge the distributions (or their histograms).

## 8.3 Tail amplification — Figure 1-5 as probability

```python
p_slow = 0.01   # only 1% of backend calls are slow
for n in (1, 5, 10, 50, 100):
    p_user_slow = 1 - (1 - p_slow) ** n
    print(f"{n:3d} parallel backend calls -> {p_user_slow*100:5.1f}% of user requests are slow")
```

```
  1 parallel backend calls ->   1.0%
  5 parallel backend calls ->   4.9%
 10 parallel backend calls ->   9.6%
 50 parallel backend calls ->  39.5%
100 parallel backend calls ->  63.4%
```

Formula: **P(at least one slow) = 1 − (1 − p)ⁿ**. Your 1% backend problem becomes a 63% user problem at n=100.

## 8.4 Twitter's two approaches, in code

```python
from collections import defaultdict
import bisect

tweets   = {}                        # tweet_id -> (sender_id, text, timestamp)
follows  = defaultdict(set)          # follower_id -> {followee_id, ...}
followers_of = defaultdict(set)      # followee_id -> {follower_id, ...}


# ─────────────── APPROACH 1: FAN-OUT ON READ ───────────────
# cheap write, expensive read (the SQL JOIN version)

def post_tweet_v1(sender_id, text, ts):
    tid = len(tweets)
    tweets[tid] = (sender_id, text, ts)
    return tid                                    # O(1) — done.

def home_timeline_v1(user_id, limit=50):
    followees = follows[user_id]                  # who do I follow?
    relevant = [(ts, tid) for tid, (s, _, ts) in tweets.items()
                if s in followees]                # scan + filter  ← EXPENSIVE
    relevant.sort(reverse=True)                   # merge by time
    return [tid for _, tid in relevant[:limit]]
    # Cost: O(total_tweets) per read, 300,000 times per second. Ouch.


# ─────────────── APPROACH 2: FAN-OUT ON WRITE ───────────────
# expensive write, cheap read (the mailbox version)

timeline_cache = defaultdict(list)   # user_id -> [(ts, tid), ...] newest last

def post_tweet_v2(sender_id, text, ts):
    tid = len(tweets)
    tweets[tid] = (sender_id, text, ts)
    for follower in followers_of[sender_id]:      # ← THE FAN-OUT
        bisect.insort(timeline_cache[follower], (ts, tid))
    return tid
    # Cost: O(number_of_followers). For a celebrity: 31,000,000 inserts.

def home_timeline_v2(user_id, limit=50):
    return [tid for _, tid in reversed(timeline_cache[user_id][-limit:])]
    # Cost: O(limit). Precomputed. This is why Twitter switched.


# ─────────────── THE HYBRID (what Twitter actually does now) ───────────────

CELEBRITY_THRESHOLD = 1_000_000

def is_celebrity(user_id):
    return len(followers_of[user_id]) > CELEBRITY_THRESHOLD

def post_tweet_hybrid(sender_id, text, ts):
    tid = len(tweets)
    tweets[tid] = (sender_id, text, ts)
    if not is_celebrity(sender_id):               # normal user → PUSH
        for follower in followers_of[sender_id]:
            bisect.insort(timeline_cache[follower], (ts, tid))
    # celebrity → do nothing at write time; we PULL at read time instead
    return tid

def home_timeline_hybrid(user_id, limit=50):
    # 1. precomputed part (cheap)
    result = list(timeline_cache[user_id][-limit:])
    # 2. merge in celebrities followed by this user (few, so still cheap)
    for celeb in (u for u in follows[user_id] if is_celebrity(u)):
        result += [(ts, tid) for tid, (s, _, ts) in tweets.items() if s == celeb]
    result.sort(reverse=True)
    return [tid for _, tid in result[:limit]]
```

**Read the three `post_tweet` functions side by side.** That's the entire scalability section in miniature: the same feature, three cost models, and the right choice depends entirely on your load parameters.

## 8.5 A tiny fault-injection wrapper (Chaos Monkey in 15 lines)

```python
import random, functools

def chaos(failure_rate=0.05, latency_ms=0):
    """Deliberately inject faults so the recovery path is actually exercised."""
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            if random.random() < failure_rate:
                raise ConnectionError(f"chaos: injected fault in {fn.__name__}")
            if latency_ms:
                import time; time.sleep(latency_ms / 1000)
            return fn(*args, **kwargs)
        return wrapper
    return decorator


@chaos(failure_rate=0.30, latency_ms=50)
def fetch_user(user_id):
    return {"id": user_id, "name": "alice"}


# The point: your RETRY logic now runs in dev, not for the first time at 3am.
def fetch_user_with_retry(user_id, attempts=3):
    for i in range(attempts):
        try:
            return fetch_user(user_id)
        except ConnectionError:
            if i == attempts - 1:
                raise
            # exponential backoff + jitter (avoids thundering herd)
            import time; time.sleep((2 ** i) * 0.1 * random.uniform(0.5, 1.5))
```

> **"Many critical bugs are actually due to poor error handling."** The `except` block above is code that, without chaos injection, might never run before production.

## 8.6 A self-checking invariant (the message-queue example)

```python
class SelfCheckingQueue:
    """The book's example: if messages_in != messages_out, raise an alert."""
    def __init__(self):
        self._items, self.enqueued, self.dequeued, self.in_flight = [], 0, 0, 0

    def put(self, item):
        self._items.append(item); self.enqueued += 1

    def get(self):
        item = self._items.pop(0); self.in_flight += 1; return item

    def ack(self):
        self.in_flight -= 1; self.dequeued += 1

    def check_invariant(self):
        expected = self.dequeued + self.in_flight + len(self._items)
        if expected != self.enqueued:
            raise AssertionError(
                f"🚨 INVARIANT VIOLATED: enqueued={self.enqueued} but "
                f"accounted for {expected} — messages have been LOST"
            )
        return True
```

Run `check_invariant()` on a timer in production. It's cheap, and it turns silent data loss into a page.

## 8.7 Percentiles over a rolling window (what a dashboard actually does)

```python
import time
from collections import deque

class RollingPercentiles:
    """Naive version — the book notes this may be too inefficient at scale,
       in which case use t-digest / HdrHistogram / forward decay instead."""
    def __init__(self, window_seconds=600):
        self.window = window_seconds
        self.samples = deque()               # (timestamp, value)

    def record(self, value_ms):
        now = time.time()
        self.samples.append((now, value_ms))
        cutoff = now - self.window
        while self.samples and self.samples[0][0] < cutoff:
            self.samples.popleft()           # evict outside the window

    def snapshot(self):
        vals = sorted(v for _, v in self.samples)
        if not vals:
            return {}
        return {f"p{p}": pct(vals, p) for p in (50, 90, 95, 99, 99.9)}
```

**Upgrade path when this gets too slow:** `pip install hdrhistogram` or `tdigest`. Both let you **merge** across machines correctly, which — as §8.2 showed — averaging cannot do.

---

# 9. 📌 ONE-PAGE CHEAT SHEET

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.1 — RELIABLE, SCALABLE, MAINTAINABLE                                ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  DATA-INTENSIVE ≠ compute-intensive. Bottleneck = amount / complexity /      ║
║  rate of change of data — not CPU.                                           ║
║                                                                              ║
║  BUILDING BLOCKS: databases · caches · search indexes · stream processing ·  ║
║  batch processing.  You STITCH them with app code → you are now a data       ║
║  system designer, and keeping them in sync is YOUR job.                      ║
║                                                                              ║
║  ── RELIABILITY ───────────────────────────────────────────────────────────  ║
║  FAULT = one component off-spec.  FAILURE = system stops serving users.      ║
║  Goal: stop faults from becoming failures. Faults can't be driven to zero.   ║
║    • Hardware — random, uncorrelated. MTTF 10–50y → 1 disk/day per 10k.      ║
║    • Software — SYSTEMATIC, CORRELATED. Worse. Redundancy doesn't help.      ║
║    • Human   — config errors are the #1 cause; HW only 10–25% of outages.    ║
║  Deliberately inject faults (Chaos Monkey). Prevention > cure for SECURITY.  ║
║                                                                              ║
║  ── SCALABILITY ───────────────────────────────────────────────────────────  ║
║  "X is scalable" is MEANINGLESS. Ask: if it grows THIS way, what are our     ║
║  options?                                                                    ║
║    1. DESCRIBE LOAD → load parameters. Twitter: follower distribution,       ║
║       NOT tweets/sec. Averages hide celebrities (75 avg vs 31M max).         ║
║       Fan-out on read (cheap write) vs on write (cheap read) vs HYBRID.      ║
║    2. DESCRIBE PERFORMANCE → batch: throughput. Online: response time.       ║
║       Response time = service + network + QUEUEING. Latency = the waiting.   ║
║       Use PERCENTILES not means. p50/p95/p99/p999. Amazon SLOs at p99.9      ║
║       (slow customers = valuable customers). Stop at p99.99 — too costly.    ║
║       Head-of-line blocking → MEASURE CLIENT-SIDE. Load-gen must not wait.   ║
║       Tail amplification: 1% slow × 100 backends = 63% slow user requests.   ║
║       ⚠️ NEVER AVERAGE PERCENTILES. Add the histograms.                       ║
║    3. COPE → scale up (simpler) vs out (shared-nothing). Mix pragmatically.  ║
║       Elastic (unpredictable load) vs manual (fewer surprises).              ║
║       Stateless = easy. Stateful = hard. NO MAGIC SCALING SAUCE.             ║
║       Early-stage: iterate on features > scale for hypothetical load.        ║
║                                                                              ║
║  ── MAINTAINABILITY ───────────────────────────────────────────────────────  ║
║  Most software cost is MAINTENANCE, not initial development.                 ║
║    • OPERABILITY  — visibility, automation, no machine dependency, good      ║
║                     defaults + overrides, predictability. (today)            ║
║    • SIMPLICITY   — remove ACCIDENTAL complexity (implementation-caused),    ║
║                     not functionality. Best tool = ABSTRACTION. (understand) ║
║    • EVOLVABILITY — agility at the DATA SYSTEM level, not the file level.    ║
║                     Downstream of simplicity. (tomorrow)                     ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

# 10. ✅ Test yourself

Cover the answers.

1. **A disk fails in your RAID array and the service keeps running. Fault or failure?**
   → A **fault**. The redundancy prevented it from becoming a failure. That's the whole distinction.

2. **Why do software faults cause more system failures than hardware faults, despite hardware failing more often?**
   → Software faults are **systematic and correlated across nodes** — every replica runs the same bug, so redundancy provides no protection. Hardware faults are largely independent.

3. **Your p50 is 80 ms and your p99 is 4 s. Your mean is 150 ms. What's likely going on?**
   → A heavy tail. The mean is being dragged up by outliers while describing nobody's actual experience. Investigate GC pauses, queueing/head-of-line blocking, cold caches, or intrinsically expensive requests.

4. **Why is Twitter's key load parameter the follower distribution rather than tweets per second?**
   → Because fan-out determines the real write volume. 4.6k tweets/sec becomes 345k cache writes/sec on average — and a single celebrity tweet becomes 31M writes. Tweets/sec alone would make the problem look trivial.

5. **Three servers report p99s of 100, 120 and 900 ms. What is the fleet p99?**
   → **Unknown from this data.** You cannot average percentiles. You need the raw data or mergeable histograms — and the answer depends on how much traffic each server took.

6. **Give an example of accidental complexity vs essential complexity.**
   → Essential: genuinely different tax rules in 30 countries. Accidental: those rules copy-pasted across 8 services with 3 naming conventions. Only the second can be removed without losing functionality.

7. **When is preventing a fault better than tolerating it?**
   → When there's no cure — **security** breaches. If an attacker has read sensitive data, that cannot be undone by any failover mechanism.

8. **Your load test shows a beautiful p99. Production falls over. Name a likely measurement bug.**
   → **Coordinated omission**: the load generator waited for each response before sending the next, so during stalls it stopped generating load and never recorded the requests that would have queued.

---

*Sources: all quoted material, figures, and statistics are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 1. Section 7 and the code appendix are supplementary material I added; the computed outputs come from actually running the code shown.*
