# DDIA — Chapter 9: Consistency and Consensus
### Complete study guide — theory, every diagram redrawn, worked examples, and what changed since 2017

> *Is it better to be alive and wrong or right and dead?*
> — Jay Kreps, *A few notes on Kafka and Jepsen* (2013)

---

## 0. The map of this chapter

Chapter 8 was the problems. **Chapter 9 is the solutions** — and the proof of exactly how far those solutions can go.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  ① LINEARIZABILITY — "pretend there is only ONE copy of the data"            │
│       what it is · when you need it · how to implement it · what it costs    │
│       · the CAP theorem (and why the book says to stop using it)            │
│                                                                              │
│  ② ORDERING GUARANTEES                                                       │
│       causality is a PARTIAL order · linearizability is a TOTAL order        │
│       Lamport timestamps · why they're not enough · TOTAL ORDER BROADCAST    │
│                                                                              │
│  ③ DISTRIBUTED TRANSACTIONS AND CONSENSUS                                    │
│       2PC and its blocking problem · XA · fault-tolerant consensus           │
│       · epoch numbers and quorums · ZooKeeper                                │
│                                                                              │
│  🔑 THE PUNCHLINE: a whole family of apparently unrelated problems turn out   │
│     to be THE SAME PROBLEM — and that problem is CONSENSUS.                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

> **The strategy, stated up front:** *"The best way of building fault tolerant systems is to find some GENERAL-PURPOSE ABSTRACTIONS WITH USEFUL GUARANTEES, implement them once, and then let applications rely on those guarantees. This is the same approach as we used with transactions in Chapter 7."*

---

## 1. Why eventual consistency is hard to live with

> Most replicated databases provide at least **eventual consistency**: *"if you stop writing to the database and wait for some unspecified length of time, then eventually all read requests will return the same value."*
>
> **A better name would be CONVERGENCE**, *"as we expect all replicas to eventually converge to the same value."*

```
   ⚠️ "This is A VERY WEAK GUARANTEE — it DOESN'T SAY ANYTHING ABOUT WHEN the
      replicas will converge. UNTIL THE TIME OF CONVERGENCE, READS COULD
      RETURN ANYTHING OR NOTHING."
```

### 🔑 Why developers find it so hard

> **"Eventual consistency is hard for application developers because IT IS SO DIFFERENT FROM THE BEHAVIOR OF VARIABLES IN A NORMAL SINGLE-THREADED PROGRAM. If you assign a value to a variable, and then read it shortly afterwards, you don't expect to read back the old value, or for the read to fail. A DATABASE LOOKS SUPERFICIALLY LIKE A VARIABLE THAT YOU CAN READ AND WRITE, BUT IN FACT ITS SEMANTICS IS MUCH MORE COMPLICATED."**

```
   ⚠️ AND THE TESTING PROBLEM:
   "Bugs are often SUBTLE AND HARD TO FIND BY TESTING, because the application
    MAY WORK WELL MOST OF THE TIME. The edge cases of eventual consistency only
    become apparent WHEN THERE IS A FAULT in the system, or AT HIGH
    CONCURRENCY."
```

### 📖 Consistency models vs isolation levels — don't conflate them

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  TRANSACTION ISOLATION (Ch.7)    │  DISTRIBUTED CONSISTENCY (Ch.9)      │
   ├──────────────────────────────────┼──────────────────────────────────────┤
   │  about avoiding RACE CONDITIONS  │  about COORDINATING THE STATE OF     │
   │  due to concurrently executing   │  REPLICAS in the face of DELAYS AND  │
   │  transactions                    │  FAULTS                              │
   └──────────────────────────────────┴──────────────────────────────────────┘
   "Although there is some overlap, they are MOSTLY INDEPENDENT CONCERNS."
```

---
---

# PART A — LINEARIZABILITY

## 2. What linearizability is

> **The idea:** *"Wouldn't it be a lot simpler if the database could give the ILLUSION THAT THERE IS ONLY ONE REPLICA, i.e. only one copy of the data?"*

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  LINEARIZABILITY — also known as ATOMIC CONSISTENCY, STRONG           ║
   ║  CONSISTENCY, IMMEDIATE CONSISTENCY, or EXTERNAL CONSISTENCY          ║
   ║                                                                       ║
   ║  "make a system appear AS IF THERE WAS ONLY ONE COPY OF THE DATA,     ║
   ║   and ALL OPERATIONS ON IT ARE ATOMIC."                               ║
   ║                                                                       ║
   ║  🔑 LINEARIZABILITY IS A **RECENCY GUARANTEE**:                        ║
   ║  "as soon as one client successfully completes a write, ALL CLIENTS   ║
   ║   READING FROM THE DATABASE MUST BE ABLE TO SEE THE VALUE JUST        ║
   ║   WRITTEN… not from a stale cache or replica."                        ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 🔷 Figure 9-1 — The football fans

```
              insert into final_scores
              (player1, score1, player2, score2)
              values('Germany', 1, 'Argentina', 0)
                     │                                          TIME ───────►
   Referee ──────────●─────────────────────────────────────────────────────►
                     │   ▲ ok
                     ▼   │
   Leader ───────────●───●────────────────────────────────────────────────►
                      ╲ insert…
                       ╲
   Follower 1 ──────────●───────────────────────────────────────────────────►
                            ▲
                            │ select * from final_scores
   Alice ───────────────────●───────────────────────────────────────────────►
                            │
                            │      ╲ insert…  (SLOW)
   Follower 2 ──────────────────────────────●──────────────────────────────►
                                                      ▲
                                                      │ select * (STALE!)
   Bob ───────────────────────────────────────────────●─────────────────────►

        ┌──────────────────────────┐      ┌────────────────────────────────┐
        │ "Hey, Germany has won    │ ───► │ "Really? The website says      │
        │  the world cup!"         │      │  they're still playing."       │
        └──────────────────────────┘      └────────────────────────────────┘
              ▲                                        ▲
           ALICE                                      BOB
```

> **Why this is a violation, precisely:** *"If Alice and Bob had hit reload AT THE SAME TIME, it wouldn't have been surprising if they had got two different query results… However, **Bob knows that he hit the reload button AFTER he heard Alice exclaim the final score**, and therefore he EXPECTS HIS QUERY RESULT TO BE AT LEAST AS RECENT AS ALICE'S."*

---

## 3. Nailing down the definition — three diagrams

### 🔷 Figure 9-2 — Concurrent reads may return either value

```
                                                                  TIME ──────►
   Client A   ├─read(x)⇒0─┤      ├─read(x) ⇒ 0 or 1─┤      ├─read(x)⇒1─┤

   Client B          ├─read(x) ⇒ 0 or 1─┤     ├─read(x) ⇒ 0 or 1─┤

   Client C                  ├──────── write(x, 1) ⇒ ok ────────┤

   ⚑ Each BAR is a request: it STARTS when the client sends it and ENDS when
     the response arrives. "a client DOESN'T KNOW EXACTLY WHEN the database
     processed its request."

   • A's FIRST read COMPLETES BEFORE THE WRITE BEGINS → must return 0
   • A's LAST read BEGINS AFTER THE WRITE COMPLETED  → must return 1
   • Anything OVERLAPPING the write → 0 or 1, we can't say
```

> 📖 **Footnote worth knowing:** this diagram *"assumes the existence of a GLOBAL CLOCK… this assumption is ok: for purposes of ANALYZING a distributed algorithm we may pretend that an accurate global clock exists, AS LONG AS THE ALGORITHM DOESN'T HAVE ACCESS TO IT."*

### 🔷 Figure 9-3 — The extra constraint: no flipping back

```
                                                                  TIME ──────►
   Client A   ├─read(x)⇒0─┤      ├─read(x)⇒1─┤      ├─read(x)⇒1─┤
                                       │
                                       │ ONCE A HAS SEEN 1…
                                       ▼
   Client B          ├─read(x)⇒0─┤        ├─read(x)⇒1─┤
                                          ▲
                          …B, STARTING STRICTLY LATER, MUST ALSO SEE 1
                          (even though C's write is STILL ONGOING)

   Client C                  ├──────── write(x, 1) ⇒ ok ────────┤

   ⚠️ WITHOUT this constraint, "readers could see a value FLIP BACK AND FORTH
      between the old and the new value SEVERAL TIMES while a write is going
      on. That is not what we expect of a system that emulates A SINGLE COPY
      OF THE DATA."

   ➜ There must be SOME POINT IN TIME at which x ATOMICALLY FLIPS from 0 to 1.
```

> 📖 The weaker semantics where reads may flip is called a **regular register**.

### 🔷 Figure 9-4 — The full picture, with compare-and-set

```
                                                                  TIME ──────►
   Client A     ├─write(x,1)⇒ok─┤              ├──read(x)⇒4──┤

   Client B ├─read(x)⇒1─┤ ├─cas(x,1,2)⇒ok─┤          ├─read(x)⇒2─┤
                                                      ▓▓▓▓▓▓▓▓▓▓▓▓
                                                      NOT LINEARIZABLE
   Client C      ├─read(x)⇒1─┤ ├─read(x)⇒2─┤  ├─cas(x,2,4)⇒ok─┤

   Client D  ├─write(x,0)⇒ok─┤   ├─cas(x,0,3)⇒ERROR─┤

   Database ──●──●───●───●──────●──●────●──●───●────●────────────────────────►
             x=0 x=1 rd  rd    x=2 rd   rd x=4 rd   rd
             └──────── the operations, laid on a SINGLE TIMELINE ────────┘

   🔑 "The requirement of linearizability is that the lines joining up the
     operation markers ALWAYS MOVE FORWARDS IN TIME (from left to right),
     NEVER BACKWARDS."
```

**The four details the book draws out — each one is a common misconception:**

```
   ① B's read returned 1 even though D's write(0) and A's write(1) were sent
      LATER. ✅ FINE: the three requests are CONCURRENT, so the DB may have
      processed D's write, then A's write, then B's read. "Perhaps B's read
      request was SLIGHTLY DELAYED IN THE NETWORK."

   ② B's read returned 1 BEFORE A received its "ok". ✅ FINE: "it doesn't mean
      the value was read before it was written, it just means the 'ok'
      RESPONSE was slightly delayed."

   ③ NO TRANSACTION ISOLATION IS ASSUMED — "another client may change a value
      at any time." C reads 1 then 2, because B changed it in between. D's cas
      FAILS because by the time it was processed, x was no longer 0.

   ④ B's FINAL read is NOT LINEARIZABLE. It's concurrent with C's cas(2→4), so
      returning 2 would be fine IN ISOLATION — but A HAS ALREADY READ 4 BEFORE
      B'S READ STARTED. "B is not allowed to read an older value than A."
      ← the Alice and Bob situation, again
```

> **Testing for it:** *"It is possible (though COMPUTATIONALLY EXPENSIVE) to test whether a system's behavior is linearizable by recording the timings of all requests and responses, and checking whether they can be arranged into a valid sequential order."*

### 📦 Sidebar: Linearizability vs serializability — the most confused pair in the book

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  SERIALIZABILITY (Ch.7)           ║  LINEARIZABILITY (Ch.9)           ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  An ISOLATION property of         ║  A RECENCY GUARANTEE on reads and ║
   ║  TRANSACTIONS, each of which may  ║  writes of a SINGLE REGISTER      ║
   ║  read/write MULTIPLE OBJECTS.     ║  (an individual object).          ║
   ║                                   ║                                   ║
   ║  Transactions behave as if they   ║  "It DOESN'T GROUP OPERATIONS     ║
   ║  ran in SOME SERIAL ORDER.        ║   TOGETHER INTO TRANSACTIONS, so  ║
   ║  ⚑ "It is OK for that serial      ║   it DOES NOT PREVENT problems    ║
   ║    order to be DIFFERENT from the ║   such as WRITE SKEW."            ║
   ║    order in which transactions    ║                                   ║
   ║    were actually run."            ║  ⚑ It DOES care about real time.  ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝

   BOTH TOGETHER = STRICT SERIALIZABILITY (a.k.a. strong-1SR)

   • 2PL and actual serial execution → TYPICALLY LINEARIZABLE ✅
   • SSI → NOT LINEARIZABLE ❌ "by design, it makes reads from a CONSISTENT
     SNAPSHOT… the whole point of a consistent snapshot is that IT DOES NOT
     INCLUDE WRITES MORE RECENT THAN THE SNAPSHOT."
```

**That last bullet is worth pausing on.** Snapshot isolation and linearizability are in direct tension: one deliberately looks at the past, the other insists on the present.

---

## 4. When do you actually need linearizability?

### ① Locking and leader election

```
   "A system that uses single-leader replication needs to ensure that there is
    indeed ONLY ONE LEADER, not several (SPLIT BRAIN). One way of electing a
    leader is to use a LOCK… NO MATTER HOW THIS LOCK IS IMPLEMENTED, IT MUST
    BE LINEARIZABLE: all nodes must agree which node owns the lock, OTHERWISE
    IT IS USELESS."

   → ZooKeeper and etcd, via consensus algorithms.
   → Apache Curator provides higher-level "recipes" on top of ZooKeeper.
   → Oracle RAC uses a lock PER DISK PAGE, with a DEDICATED CLUSTER
     INTERCONNECT NETWORK because the locks are on the critical path.
```

> ⚠️ **Important footnote:** *"Strictly speaking, ZooKeeper and etcd provide LINEARIZABLE WRITES, but READS MAY BE STALE, since by default they can be served by any one of the replicas. You can OPTIONALLY REQUEST a linearizable read: etcd calls this a QUORUM READ, and in ZooKeeper you need to call `sync()` before the read."*

### ② Constraints and uniqueness guarantees

```
   Usernames · email addresses · file paths · bank balance never negative ·
   don't sell more items than you have · two people don't book the same seat

   "These constraints all require there to be A SINGLE UP-TO-DATE VALUE that
    all nodes agree on."
```

### 💡 …but you may be able to cheat — and the book is refreshingly practical here

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │ • Two people grab the same username? "You can SEND ONE OF THEM AN     │
   │   EMAIL TO APOLOGIZE, and ask them to choose a different one."        │
   │   → a COMPENSATING TRANSACTION.                                       │
   │                                                                       │
   │ • Oversold stock? "Order in more stock, apologize for the delay, and   │
   │   offer a discount. THIS IS ACTUALLY THE SAME AS YOU'D HAVE TO DO IF   │
   │   A FORK-LIFT TRUCK RUNS OVER ONE OF THE ITEMS IN YOUR WAREHOUSE…      │
   │   Thus, THE APOLOGY WORKFLOW ALREADY NEEDS TO BE PART OF YOUR          │
   │   BUSINESS PROCESSES ANYWAY."                                         │
   │                                                                       │
   │ • Overdrawn account? "You can CHARGE THEM AN OVERDRAFT PENALTY FEE     │
   │   AND LAUGH ALL THE WAY TO THE BANK. By limiting the maximum amount    │
   │   that can be withdrawn per day, THE RISK TO THE BANK IS BOUNDED."     │
   └──────────────────────────────────────────────────────────────────────┘
```

**This is the most underrated passage in the chapter.** The question isn't "is this constraint important?" but "what does violating it actually cost, and do I already have a process for that?" Physical reality violates your inventory count anyway.

### ③ Cross-channel timing dependencies

### 🔷 Figure 9-5 — The image resizer race condition

```
      1. upload image              2. store full-size image
   ─────────────────────►┌────────────┐──────────────────────►┌──────────────┐
                         │ WEB SERVER │                       │ FILE STORAGE │
                         └──────┬─────┘                       └──────┬───────┘
                                │                             ▲      │
                    3. send     │                   5. fetch  │      │ 6. store
                    message     │                   full-size │      │ resized
                                ▼                             │      ▼
                         ┌──────────────┐  4. deliver  ┌──────┴──────────────┐
                         │ MESSAGE QUEUE│─────────────►│    IMAGE RESIZER    │
                         └──────────────┘              └─────────────────────┘

   ⚠️ TWO COMMUNICATION CHANNELS between the web server and the resizer:
      THE FILE STORAGE and THE MESSAGE QUEUE.

   "the message queue (steps 3 and 4) MIGHT BE FASTER THAN THE INTERNAL
    REPLICATION INSIDE THE STORAGE SERVICE. In this case, when the resizer
    fetches the image (step 5), IT MIGHT SEE AN OLD VERSION OF THE IMAGE, OR
    NOTHING AT ALL… the full-size and the resized images become PERMANENTLY
    INCONSISTENT."
```

> **The structural insight:** this is *exactly* Figure 9-1. Alice's voice was the second channel; here it's the message queue. **Linearizability violations only become observable when there's a side channel** — but the corruption happens either way.

---

## 5. Implementing linearizable systems

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  SINGLE-LEADER REPLICATION      →  POTENTIALLY linearizable           ║
   ║    Reads from the leader (or SYNCHRONOUSLY updated followers) CAN be. ║
   ║    ❌ but not always: snapshot isolation, or concurrency bugs.         ║
   ║    ❌ "Using the leader for reads RELIES ON THE ASSUMPTION THAT YOU    ║
   ║       KNOW FOR SURE WHO THE LEADER IS… if the DELUSIONAL LEADER       ║
   ║       continues to serve requests, it is likely to VIOLATE            ║
   ║       LINEARIZABILITY."                                               ║
   ║    ❌ async failover may LOSE DATA → violates durability AND          ║
   ║       linearizability.                                                ║
   ╟───────────────────────────────────────────────────────────────────────╢
   ║  CONSENSUS ALGORITHMS           →  LINEARIZABLE ✅                     ║
   ║    "consensus protocols contain MEASURES TO PREVENT SPLIT-BRAIN AND   ║
   ║     STALE REPLICAS." → ZooKeeper, etcd.                               ║
   ╟───────────────────────────────────────────────────────────────────────╢
   ║  MULTI-LEADER REPLICATION       →  NOT linearizable ❌                 ║
   ║    "they concurrently process writes on multiple nodes and            ║
   ║     ASYNCHRONOUSLY replicate them."                                   ║
   ╟───────────────────────────────────────────────────────────────────────╢
   ║  LEADERLESS REPLICATION         →  PROBABLY NOT linearizable ❌        ║
   ║    LWW with time-of-day clocks: "ALMOST CERTAINLY NON-LINEARIZABLE."  ║
   ║    Sloppy quorums: "RUIN ANY CHANCE of linearizability."              ║
   ║    Even STRICT quorums: see Figure 9-6.                               ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 🔷 Figure 9-6 — Strict quorums are NOT enough

```
              set x = 1                                          TIME ───────►
   Writer ────●────────────────────────────────────────────●─────────────────►
             ╱│╲                                         ▲ ok
            ╱ │ ╲
   Replica 1 ─┼──────────────────────────────●───────────●────────────────────►
              │                              0          ok
   Replica 2 ─●──────────●──────────────●────────────────────────────────────►
             ok          0              0
   Replica 3 ────────────●──────────────────────────────────────────────────►
                         1
                         │
   Reader A ──────────├──●──get x ⇒ 1──┤
                          (sees 1 on replica 3, 0 on replica 2)

   Reader B ────────────────────────├──get x ⇒ 0──┤
                                     (sees 0 on BOTH replicas it polls)

   n = 3, w = 3, r = 2  →  w + r > n  ✅ QUORUM CONDITION MET

   ❌ BUT: "B's request BEGINS AFTER A'S REQUEST COMPLETES, but B returns the
      OLD value while A returns the NEW value."
      ← the Alice and Bob situation, for the third time
```

**How to fix it, and why nobody does:**

```
   ✅ POSSIBLE at the cost of performance:
      • a READER must perform READ REPAIR SYNCHRONOUSLY before returning
      • a WRITER must READ THE LATEST STATE of a quorum before writing

   ❌ "However, RIAK DOES NOT DO THIS due to the performance penalty.
      CASSANDRA DOES wait for read repair on quorum reads, BUT IT LOSES
      LINEARIZABILITY IF THERE ARE MULTIPLE CONCURRENT WRITES to the same key,
      due to its use of LAST-WRITE-WINS."

   ⚠️ AND THE HARD LIMIT: "only linearizable READ AND WRITE operations can be
      implemented in this model, BUT A LINEARIZABLE COMPARE-AND-SET OPERATION
      CANNOT — IT REQUIRES A CONSENSUS ALGORITHM."

   ➜ "it is SAFEST TO ASSUME that a leaderless system with Dynamo-style
     replication DOES NOT PROVIDE LINEARIZABILITY."
```

---

## 6. The cost of linearizability, and the CAP theorem

### 🔷 Figure 9-7 — A network interruption forces a choice

```
   ╔═══════════════════════════════╗           ╔═══════════════════════════════╗
   ║        DATACENTER 1           ║           ║        DATACENTER 2           ║
   ║                               ║           ║                               ║
   ║  ┌─────────────┐  ┌────────┐  ║  ╳╳╳╳╳╳   ║  ┌────────┐  ┌─────────────┐  ║
   ║  │ Application │──│   DB   │══╬══╳╳╳╳╳╳═══╬══│   DB   │──│ Application │  ║
   ║  └──────┬──────┘  └────────┘  ║  ╳╳╳╳╳╳   ║  └────────┘  └──────┬──────┘  ║
   ║         │                     ║     ▲     ║                     │         ║
   ╚═════════│═════════════════════╝     │     ╚═════════════════════│═════════╝
             │           NETWORK INTERRUPTION                        │
             ▼           (network partition)                         ▼
          Clients                                                 Clients

   MULTI-LEADER : each DC CONTINUES OPERATING NORMALLY. Writes queue up and
                  are exchanged when connectivity returns. ✅ AVAILABLE
   SINGLE-LEADER: the leader is in ONE DC. Clients in the OTHER DC cannot make
                  writes, nor linearizable reads. ❌ UNAVAILABLE THERE
```

### The trade-off, stated precisely

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  • If your application REQUIRES LINEARIZABILITY, and some replicas    ║
   ║    are disconnected, "SOME REPLICAS CANNOT PROCESS REQUESTS while     ║
   ║    they are disconnected: they must either WAIT until the network     ║
   ║    problem is fixed, or RETURN AN ERROR (either way, THEY BECOME      ║
   ║    UNAVAILABLE)."                                                     ║
   ║                                                                       ║
   ║  • If it DOES NOT require linearizability, "each replica can process  ║
   ║    requests INDEPENDENTLY… the application can REMAIN AVAILABLE, but  ║
   ║    ITS BEHAVIOR IS NOT LINEARIZABLE."                                 ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   This insight is popularly known as THE CAP THEOREM, named by Eric Brewer in
   2000 — "although the trade-off was ALREADY KNOWN to designers of
   distributed databases SINCE THE 1970s."
```

### ⚠️ The book's verdict on CAP is unusually blunt

> **"The CAP theorem, as formally defined, is OF VERY NARROW SCOPE: it only considers ONE CONSISTENCY MODEL (namely linearizability) and ONE KIND OF FAULT (network partitions). It doesn't say anything about network delays, dead nodes, or other trade-offs. Thus, although CAP has been historically influential, IT HAS LITTLE PRACTICAL VALUE FOR DESIGNING SYSTEMS… CAP has now been SUPERSEDED BY MORE PRECISE RESULTS, so it is OF MOSTLY HISTORICAL INTEREST TODAY."**

### 📦 Sidebar: The Unhelpful CAP Theorem

```
   ❌ "Consistency, Availability, Partition tolerance: PICK 2 OUT OF 3"
      "putting it this way is MISLEADING, because NETWORK PARTITIONS ARE A
       KIND OF FAULT, so they aren't something you would NORMALLY CHOOSE:
       EITHER THEY HAPPEN OR THEY DON'T."

   ✅ A BETTER PHRASING:
            "either CONSISTENT or AVAILABLE when PARTITIONED"

   ⚠️ "there are SEVERAL CONTRADICTORY DEFINITIONS of the term AVAILABILITY,
      and the formalization as a theorem DOES NOT MATCH ITS USUAL MEANING.
      Many so-called 'highly available' systems ACTUALLY DO NOT MEET CAP'S
      IDIOSYNCRATIC DEFINITION of availability."

   ➜ "there is A LOT OF MISUNDERSTANDING AND CONFUSION around CAP, and IT DOES
     NOT HELP US UNDERSTAND SYSTEMS BETTER, SO CAP IS BEST AVOIDED."
```

> 📖 Also note the footnote on **CP/AP labels**: *"this classification scheme has SEVERAL FLAWS, so it is BEST AVOIDED."*

### 🔑 The real reason linearizability is rare: it's just slow

> **"Even RAM on a modern multicore CPU is NOT LINEARIZABLE"** — each core has its own cache and store buffer, so a write by one core isn't immediately visible to another unless you use a **memory barrier / fence**.

```
   🔑 "It makes NO SENSE to use the CAP theorem to justify the multi-core
     memory consistency model: WITHIN ONE COMPUTER WE USUALLY ASSUME RELIABLE
     COMMUNICATION… THE REASON FOR DROPPING LINEARIZABILITY IS PERFORMANCE,
     NOT FAULT TOLERANCE."

   "The same is true of many distributed databases that choose not to provide
    linearizable guarantees: they do so PRIMARILY TO INCREASE PERFORMANCE, not
    so much for fault tolerance. LINEARIZABILITY IS SLOW — AND THIS IS TRUE
    ALL THE TIME, NOT ONLY DURING A NETWORK FAULT."
```

**And there is a proof that you can't do better:**

> **Attiya and Welch prove that if you want linearizability, THE RESPONSE TIME OF READ AND WRITE REQUESTS IS AT LEAST PROPORTIONAL TO THE UNCERTAINTY OF DELAYS IN THE NETWORK.** *"A FASTER ALGORITHM FOR LINEARIZABILITY DOES NOT EXIST, but weaker consistency models can be much faster."*

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  response time of linearizable ops  ∝  network delay UNCERTAINTY     │
   │                                                                      │
   │  Not average delay. UNCERTAINTY. The jitter, from Chapter 8.         │
   │  This is why linearizability hurts most in geo-distributed systems.  │
   └──────────────────────────────────────────────────────────────────────┘
```

---
---

# PART B — ORDERING GUARANTEES

## 7. Ordering and causality

> **"Looking at it like this, MANY OF THE LAST 200 PAGES OF THIS BOOK COULD BE BOILED DOWN TO THE THREE WORDS: 'WATCH YOUR CAUSALITY!'"**

### Every place causality has already appeared

```
   ┌─────────────────────────┬────────────────────────────────────────────┐
   │ Consistent prefix reads │ the ANSWER arrives before the QUESTION     │
   │ (Fig 5-5)               │ (Mr Poons and Mrs Cake)                    │
   ├─────────────────────────┼────────────────────────────────────────────┤
   │ Multi-leader overtaking │ "an UPDATE to a row THAT DID NOT EXIST"     │
   │ (Fig 5-9)               │ — a row must be CREATED before UPDATED     │
   ├─────────────────────────┼────────────────────────────────────────────┤
   │ Happens-before (Ch.5)   │ A before B, B before A, or CONCURRENT      │
   ├─────────────────────────┼────────────────────────────────────────────┤
   │ Snapshot isolation      │ "CONSISTENT" means CONSISTENT WITH         │
   │ (Ch.7)                  │ CAUSALITY: "if the snapshot contains an    │
   │                         │ ANSWER, it must also contain the QUESTION" │
   │                         │ → READ SKEW = reading a state that         │
   │                         │   VIOLATES CAUSALITY                       │
   ├─────────────────────────┼────────────────────────────────────────────┤
   │ Write skew (Fig 7-8)    │ going off-call is causally dependent on    │
   │                         │ OBSERVING who is on-call. SSI detects      │
   │                         │ write skew by TRACKING CAUSAL DEPENDENCIES │
   ├─────────────────────────┼────────────────────────────────────────────┤
   │ Alice and Bob (Fig 9-1) │ Alice's exclamation is causally dependent  │
   │                         │ on the announcement of the score           │
   └─────────────────────────┴────────────────────────────────────────────┘
```

### 🔑 The causal order is a PARTIAL order; linearizability is a TOTAL order

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  TOTAL ORDER                      ║  PARTIAL ORDER                    ║
   ║  like NATURAL NUMBERS             ║  like MATHEMATICAL SETS           ║
   ║  "5 and 13? 13 is greater."       ║  "is {a,b} greater than {b,c}?    ║
   ║  ANY two elements comparable      ║   You can't really compare them." ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  LINEARIZABILITY                  ║  CAUSALITY                        ║
   ║  "for ANY two operations we can   ║  "two events are ORDERED if they  ║
   ║   ALWAYS say which one happened   ║   are CAUSALLY RELATED, but they  ║
   ║   first."                         ║   are INCOMPARABLE if they are    ║
   ║                                   ║   CONCURRENT."                    ║
   ║  ➜ THERE ARE NO CONCURRENT        ║                                   ║
   ║    OPERATIONS in a linearizable   ║  ➜ the timeline BRANCHES AND      ║
   ║    datastore. A SINGLE TIMELINE.  ║    MERGES — like GIT.             ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝

        LINEARIZABLE                        CAUSALLY CONSISTENT
        ───────────                         ───────────────────
        ●──●──●──●──●──●──●                      ●──●──●
        one straight line                       ╱        ╲
                                            ●──●          ●──●
                                                ╲        ╱
                                                 ●──●──●
                                         branches and merges (like git)
```

> **The relationship:** *"LINEARIZABILITY IMPLIES CAUSALITY: any system that is linearizable will preserve causality correctly."* And critically: *"if there are MULTIPLE COMMUNICATION CHANNELS in a system, linearizability ensures that causality is AUTOMATICALLY PRESERVED WITHOUT HAVING TO DO ANYTHING SPECIAL."*

### 💡 The good news: a middle ground exists

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "CAUSAL CONSISTENCY IS THE STRONGEST POSSIBLE CONSISTENCY MODEL THAT ║
   ║   DOES NOT SLOW DOWN DUE TO NETWORK DELAYS, AND REMAINS AVAILABLE IN  ║
   ║   THE FACE OF NETWORK FAILURES."                                      ║
   ║                                                                       ║
   ║  ➜ "In many cases, systems that APPEAR to require linearizability in  ║
   ║    fact ONLY REALLY REQUIRE CAUSAL CONSISTENCY, which can be          ║
   ║    implemented MORE EFFICIENTLY."                                     ║
   ║                                                                       ║
   ║  ⚑ AND NOTE: "in particular, THE CAP THEOREM DOES NOT APPLY."         ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

*(Written in 2017: "not much of it has yet made its way into production systems." See §18 for how that changed.)*

---

## 8. Sequence numbers and Lamport timestamps

> **Why not just track causality directly?** *"Actually keeping track of ALL causal dependencies CAN BECOME IMPRACTICAL. In many applications, clients READ LOTS OF DATA before writing something, and then it is NOT CLEAR whether the write is causally dependent on all or only some of those prior reads."*

### Three sequence-number schemes that DON'T work

```
   ❌ ① ODD/EVEN (or node-ID bits): "Each node may process A DIFFERENT NUMBER
        OF OPERATIONS PER SECOND. Thus the counter for even numbers may LAG
        BEHIND the counter for odd numbers."
   ❌ ② PHYSICAL TIMESTAMPS: subject to CLOCK SKEW → Figure 8-3.
   ❌ ③ BLOCK ALLOCATION (node A gets 1–1000, node B gets 1001–2000): "one
        operation may be given a number in 1,001–2,000, and A CAUSALLY LATER
        OPERATION may be given a number in 1–1,000."

   ➜ All three "generate a UNIQUE, APPROXIMATELY INCREASING sequence number"
     but they are NOT CONSISTENT WITH CAUSALITY.
```

> 📖 **Footnote worth keeping:** *"A total order that is INCONSISTENT with causality is easy to create, but NOT VERY USEFUL. For example, you can generate a RANDOM UUID for each operation… This is a valid total order, but the random UUIDs TELL YOU NOTHING about which operation actually happened first."*

### 🔷 Figure 9-8 — Lamport timestamps

```
   A Lamport timestamp is the pair (counter, node ID).

              write      │                              write        write
              max=0      │                              max=1        max=5
   Client A ────●────────┼──────────────────────────────────●──────────●──────►
               (1,1)     │                                            (6,1)
                 │       │                                    ▲         │
   Node 1 ───────●───────┼────────────────────────────────────┼─────────●──────►
                c=1      │                                  c=6       c=6
                         │                                    │
                         │                             (5,2)  │
   Node 2 ──●─────●─────●──────●──────────────────────────●────┴──────────●────►
           c=1   c=2   c=3    c=4                        c=5             c=6
          (1,2) (2,2) (3,2)  (4,2)                                      (6,2)
   Client B ●─────●─────●──────●──────────────────────────────────────────●────►
          write write write  write                                     write
          max=0 max=1 max=2  max=3                                     max=4

   🔑 THE KEY IDEA: "EVERY NODE AND EVERY CLIENT KEEPS TRACK OF THE MAXIMUM
     COUNTER VALUE IT HAS SEEN SO FAR, AND INCLUDES THAT MAXIMUM ON EVERY
     REQUEST. When a node receives a request with a maximum counter GREATER
     THAN ITS OWN, it IMMEDIATELY INCREASES ITS OWN COUNTER TO THAT MAXIMUM."

   In the diagram: client A receives counter 5 from node 2, sends max=5 to
   node 1. Node 1's counter was only 1 → IT JUMPS TO 5 → next op is 6.
```

**The comparison rule:** greater counter wins; ties broken by greater node ID.

```python
# Lamport clock, in full
class LamportClock:
    def __init__(self, node_id):
        self.node_id, self.counter = node_id, 0

    def local_event(self):
        self.counter += 1
        return (self.counter, self.node_id)

    def on_receive(self, remote_max):
        self.counter = max(self.counter, remote_max) + 1   # ← THE WHOLE TRICK
        return (self.counter, self.node_id)
```

### ⚠️ Lamport timestamps vs version vectors — a real distinction

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  VERSION VECTORS (Ch.5)          │  LAMPORT TIMESTAMPS                  │
   ├──────────────────────────────────┼──────────────────────────────────────┤
   │  Can DISTINGUISH whether two     │  ALWAYS ENFORCE A TOTAL ORDERING.    │
   │  operations are CONCURRENT or    │                                      │
   │  one is CAUSALLY DEPENDENT.      │  "From the total ordering you CANNOT │
   │                                  │   TELL whether two operations are    │
   │  → detect conflicts, keep        │   CONCURRENT or CAUSALLY DEPENDENT." │
   │    SIBLINGS                      │                                      │
   └──────────────────────────────────┴──────────────────────────────────────┘
   Lamport timestamps CAPTURE all causality, but IMPOSE MORE ORDERING THAN
   STRICTLY REQUIRED — concurrent ops get ordered arbitrarily.
```

---

## 9. Why timestamp ordering is still not enough

**The username uniqueness example — read this carefully, it's the hinge of the chapter:**

```
   You might think: "if two accounts with the same username are created, PICK
   THE ONE WITH THE LOWER TIMESTAMP as the winner." Since timestamps are
   totally ordered, this comparison is always valid.

   ✅ "This approach works for determining THE WINNER AFTER THE FACT: once you
      have COLLECTED ALL the username creation operations, you can compare."

   ❌ "However, IT IS NOT SUFFICIENT WHEN A NODE HAS JUST RECEIVED A REQUEST
      FROM A USER and NEEDS TO DECIDE RIGHT NOW whether it should succeed or
      fail. AT THAT MOMENT, THE NODE DOES NOT KNOW whether another node is
      CONCURRENTLY in the process of creating an account with the same name."

   To be sure, "you would have to CHECK WITH EVERY OTHER NODE to see what it
   is doing. IF ONE OF THE OTHER NODES HAS FAILED or cannot be reached, THIS
   SYSTEM WOULD GRIND TO A HALT."
```

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  🔑 THE PROBLEM: "the total order of operations ONLY EMERGES AFTER YOU ║
   ║  HAVE COLLECTED ALL OF THE OPERATIONS."                               ║
   ║                                                                       ║
   ║  "in order to implement something like a uniqueness constraint, IT'S  ║
   ║   NOT SUFFICIENT TO HAVE A TOTAL ORDERING OF OPERATIONS — YOU ALSO    ║
   ║   NEED TO KNOW WHEN THAT ORDER IS FINALIZED."                         ║
   ║                                                                       ║
   ║            ➜ This idea is captured in TOTAL ORDER BROADCAST.          ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 10. Total order broadcast

> Also called **atomic broadcast** — *"the term is traditional, but it is VERY CONFUSING as it's inconsistent with other uses of the word atomic: IT HAS NOTHING TO DO WITH ATOMICITY IN ACID TRANSACTIONS."*

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  TWO PROPERTIES, ALWAYS SATISFIED — even if nodes or the network fail ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  RELIABLE DELIVERY                                                    ║
   ║     "No messages are lost: IF A MESSAGE IS DELIVERED TO ONE NODE, IT  ║
   ║      IS DELIVERED TO ALL NODES."                                      ║
   ║  TOTALLY ORDERED DELIVERY                                             ║
   ║     "Messages are delivered to EVERY NODE IN THE SAME ORDER."         ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### What you can build with it

```
   ① DATABASE REPLICATION — "if every message represents a write, and every
      replica processes the same writes in the same order, then THE REPLICAS
      WILL REMAIN CONSISTENT."  = STATE MACHINE REPLICATION

   ② SERIALIZABLE TRANSACTIONS — "if every message represents a DETERMINISTIC
      transaction to be executed as a stored procedure" (← Ch.7 serial
      execution, distributed)

   ③ A LOG — "delivering a message is LIKE APPENDING TO THE LOG."

   ④ A LOCK SERVICE WITH FENCING TOKENS — "Every request to acquire the lock
      is APPENDED AS A MESSAGE, and all messages are SEQUENTIALLY NUMBERED…
      The sequence number can then serve as FENCING TOKEN. In ZooKeeper, this
      sequence number is called zxid."      ← Ch.8's fencing, delivered
```

### 🔑 The property that makes it stronger than timestamps

> **"An important aspect of total order broadcast is that THE ORDER IS FIXED AT THE TIME THE MESSAGES ARE DELIVERED: a node is NOT ALLOWED TO RETROACTIVELY INSERT A MESSAGE INTO AN EARLIER POSITION in the order if subsequent messages have already been delivered. THIS FACT MAKES TOTAL ORDER BROADCAST STRONGER THAN TIMESTAMP ORDERING."**

```
   LAMPORT TIMESTAMPS          TOTAL ORDER BROADCAST
   ──────────────────          ─────────────────────
   ●───●───●───?───●           ●───●───●───●───●
           ▲                               ▲
   a message with a lower      once delivered, the prefix
   timestamp might STILL       is FINAL. Nothing can be
   ARRIVE and be inserted      inserted behind it.
   here. You can never know
   when the order is settled.  ➜ THIS is what lets you DECIDE NOW.
```

### Building linearizable storage on top — the username example, solved

```
   For every possible username, imagine a linearizable register with an atomic
   compare-and-set. Initially null. To claim a username:

   ① APPEND a message to the log, tentatively claiming the username.
   ② READ the log, and WAIT FOR YOUR OWN MESSAGE TO BE DELIVERED BACK TO YOU.
   ③ CHECK for any messages claiming that username.
        • if the FIRST such message is YOURS  → SUCCESS, commit the claim
        • if the FIRST such message is SOMEONE ELSE'S → ABORT

   "Because log entries are delivered to all nodes IN THE SAME ORDER, if there
    are several concurrent writes, ALL NODES WILL AGREE ON WHICH ONE CAME
    FIRST."
```

> ⚠️ **This gives linearizable WRITES but not linearizable READS.** *"if you read from a store that is ASYNCHRONOUSLY UPDATED from the log, IT MAY BE STALE. (To be precise, the procedure above provides SEQUENTIAL CONSISTENCY, sometimes also known as TIMELINE CONSISTENCY, a slightly weaker guarantee.)"*

```
   THREE WAYS TO GET LINEARIZABLE READS:
   ① SEQUENCE READS THROUGH THE LOG — append a message, wait for it to come
      back, then read. "The message's POSITION IN THE LOG defines the point in
      time at which the read happens." (etcd QUORUM READS work like this.)
   ② FETCH THE LATEST LOG POSITION LINEARIZABLY, wait for all entries up to it
      to be delivered, then read. (ZooKeeper's sync().)
   ③ READ FROM A SYNCHRONOUSLY UPDATED REPLICA. (chain replication.)
```

### And the reverse construction

```
   TOTAL ORDER BROADCAST  ⇄  LINEARIZABLE COMPARE-AND-SET REGISTER

   To build broadcast FROM a linearizable register with increment-and-get:
      for each message, INCREMENT-AND-GET the register, attach the value as a
      sequence number, send to all nodes; recipients deliver CONSECUTIVELY.

   🔑 THE CRITICAL DIFFERENCE FROM LAMPORT TIMESTAMPS:
     "the numbers you get from incrementing the linearizable register form
      A SEQUENCE WITH NO GAPS. Thus, if a node has delivered message 4 and
      receives message 6, IT KNOWS THAT IT MUST WAIT FOR MESSAGE 5."

   ➜ "if you think hard enough about linearizable sequence number generators,
     YOU INEVITABLY END UP WITH A CONSENSUS ALGORITHM."
```

> **"It can be proved that a linearizable compare-and-set register and total order broadcast are BOTH EQUIVALENT TO CONSENSUS. That is, IF YOU CAN SOLVE ONE OF THESE PROBLEMS, YOU CAN TRANSFORM IT INTO A SOLUTION FOR THE OTHERS. THIS IS QUITE A PROFOUND AND SURPRISING INSIGHT!"**

---
---

# PART C — DISTRIBUTED TRANSACTIONS AND CONSENSUS

## 11. Two-phase commit

> **"Consensus is one of the most important and fundamental problems in distributed computing. On the surface, it seems simple… You might think that this shouldn't be too hard. UNFORTUNATELY, MANY BROKEN SYSTEMS HAVE BEEN BUILT IN THE MISTAKEN BELIEF THAT THIS PROBLEM IS EASY."**

### 📦 Sidebar: The impossibility of consensus (FLP)

```
   THE FLP RESULT (Fischer, Lynch, Paterson) proves "there is NO ALGORITHM
   which reliably achieves consensus IF THERE IS A RISK THAT A NODE MAY
   CRASH." …and yet here we are, discussing consensus algorithms.

   THE RESOLUTION: "the FLP result is proved in A VERY RESTRICTIVE THEORETICAL
   SYSTEM MODEL, assuming a DETERMINISTIC algorithm that CANNOT USE ANY CLOCKS
   OR TIMEOUTS."

      ✅ allow TIMEOUTS (or any way of suspecting crashes) → SOLVABLE
      ✅ allow RANDOM NUMBERS                              → SOLVABLE

   ➜ "although the FLP result is of GREAT THEORETICAL IMPORTANCE, distributed
     systems can USUALLY ACHIEVE CONSENSUS IN PRACTICE."
```

### Why one-phase commit doesn't work

```
   "It is NOT SUFFICIENT to simply send a commit request to all nodes and
    independently commit on each one":
   • some nodes may DETECT A CONSTRAINT VIOLATION while others can commit
   • some COMMIT REQUESTS MIGHT BE LOST, timing out, while others get through
   • some nodes may CRASH BEFORE THE COMMIT RECORD is fully written

   🔑 "once a transaction has been committed on one node, IT CANNOT BE
     RETRACTED AGAIN if it later turns out that it was aborted on another…
     A TRANSACTION COMMIT MUST BE IRREVOCABLE."
```

### 🔷 Figure 9-9 — A successful 2PC

```
                write data     write data      prepare        commit
                     │             │              │              │   TIME ───►
   Coordinator ──────●─────────────●──────────────●──────────────●──────────►
                     │  ▲ ok       │  ▲ ok        │  ▲ yes       │  ▲ ok
                     ▼  │          │  │           ▼  │           ▼  │
   Database 1 ───────●──●──────────┼──┼───────────●──●───────────●──●────────►
                     ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                     ▼  ▲          ▼  ▲           ▼  ▲           ▼  ▲
   Database 2 ───────●──●──────────●──●───────────●──●───────────●──●────────►
                     ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                     └──── ░ = LOCKS HELD BY THE TRANSACTION ────┘
                                        ├── PHASE 1 ──┤├─ PHASE 2 ─┤
```

> ⚠️ **"Don't confuse 2PC and 2PL.** Two-phase commit provides ATOMIC COMMIT in a distributed database; two-phase locking provides SERIALIZABLE ISOLATION. **To avoid confusion, it's best to think of them as ENTIRELY SEPARATE CONCEPTS, and to ignore the unfortunate similarity in name."**

### 🔑 The system of promises — why 2PC works and 1PC doesn't

```
   ① App requests a GLOBALLY UNIQUE TRANSACTION ID from the coordinator.
   ② App begins a SINGLE-NODE transaction on each participant, tagged with
      that global ID. Anything goes wrong here → anyone can abort.
   ③ Coordinator sends PREPARE to all participants.
   ④ ⚑ POINT OF NO RETURN #1 — the PARTICIPANT'S PROMISE
      "the participant MAKES SURE THAT IT CAN DEFINITELY COMMIT THE
       TRANSACTION IN ALL CIRCUMSTANCES. This includes WRITING ALL TRANSACTION
       DATA TO DISK (a crash, a power failure or running out of disk space are
       NOT ACCEPTABLE EXCUSES for refusing to commit later), and checking for
       conflicts or constraint violations.
       By replying 'yes', THE NODE GIVES UP THE RIGHT TO ABORT THE
       TRANSACTION, BUT WITHOUT ACTUALLY COMMITTING IT."
   ⑤ ⚑ POINT OF NO RETURN #2 — the COORDINATOR'S DECISION
      "The coordinator MUST WRITE THAT DECISION TO ITS TRANSACTION LOG ON
       DISK… This is called THE COMMIT POINT."
   ⑥ Commit/abort sent to all. "If the request fails or times out, the
      coordinator MUST RETRY FOREVER UNTIL IT SUCCEEDS. THERE IS NO MORE GOING
      BACK."

   ➜ "Single-node atomic commit LUMPS THESE TWO EVENTS INTO ONE: writing the
     commit record to the transaction log."
```

### 💒 The marriage analogy, which genuinely helps

```
   "the minister asks the bride and groom INDIVIDUALLY whether they want to
    marry the other… After receiving BOTH acknowledgements, the minister
    pronounces the couple husband and wife: THE TRANSACTION IS COMMITTED."

   "before saying 'I do', you have the freedom to ABORT by saying 'no way!'.
    However, AFTER saying 'I do', you CANNOT RETRACT that statement.

    IF YOU FAINT AFTER SAYING 'I DO', and you don't hear the minister speak
    the words 'you are now husband and wife', THAT DOESN'T CHANGE THE FACT
    THAT THE TRANSACTION WAS COMMITTED. When you recover consciousness, you
    can find out whether you are married BY QUERYING THE MINISTER for the
    status of your global transaction ID."
```

### 🔷 Figure 9-10 — Coordinator failure: the fatal flaw

```
                write data      prepare          commit
                     │             │                │         TIME ─────────►
   Coordinator ──────●─────────────●────────────────●═══ CRASHED ═══════════
                     │  ▲ ok       │  ▲ yes         │
                     ▼  │          ▼  │             │
   Database 1 ───────●──●──────────●──●─────────────┼──── STUCK IN "PREPARED"
                     │  ▲ ok       │  ▲ yes         │      ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                     ▼  │          ▼  │             ▼  ▲ ok  HOLDING LOCKS
   Database 2 ───────●──●──────────●──●─────────────●──●───────────────────►
                                                    (committed!)
                     ├──── phase 1 ────┤├──── phase 2 ────┤

   💥 DB1 voted YES, so it CANNOT UNILATERALLY ABORT. DB2 has COMMITTED.
      "Even a TIMEOUT DOES NOT HELP HERE: if database 1 unilaterally aborts
       after a timeout, it will END UP INCONSISTENT with database 2."
```

> **The participant's state is called IN DOUBT or UNCERTAIN.** *"Without hearing from the coordinator, the participant HAS NO WAY OF KNOWING whether to commit or abort… **THE ONLY WAY HOW 2PC CAN COMPLETE IS BY WAITING FOR THE COORDINATOR TO RECOVER.**"*
>
> **The reduction:** *"the commit point of 2PC REDUCES DOWN TO A REGULAR SINGLE-NODE ATOMIC COMMIT ON THE COORDINATOR."*

### Why not three-phase commit?

```
   "2PC is called a BLOCKING atomic commit protocol… As an alternative, an
    algorithm called THREE-PHASE COMMIT (3PC) has been proposed. HOWEVER, the
    standard formulation of 3PC ASSUMES A NETWORK WITH BOUNDED DELAY and nodes
    with BOUNDED RESPONSE TIMES; in most practical systems IT CANNOT GUARANTEE
    ATOMICITY."

   🔑 "In general, NON-BLOCKING ATOMIC COMMIT REQUIRES A PERFECT FAILURE
     DETECTOR, i.e. a RELIABLE mechanism for telling whether a node is crashed.
     In a network with unbounded delay, A TIMEOUT IS NOT A RELIABLE FAILURE
     DETECTOR."       ← exactly Chapter 8's conclusion
```

---

## 12. Distributed transactions in practice

```
   "Distributed transactions have A MIXED REPUTATION… they are criticized for
    CAUSING OPERATIONAL PROBLEMS, KILLING PERFORMANCE, and PROMISING MORE THAN
    THEY CAN DELIVER."

   📊 "distributed transactions in MySQL are reported to be OVER 10 TIMES
      SLOWER than single-node transactions."
      Cost: additional DISK FORCING (fsync) for crash recovery + extra
      NETWORK ROUND-TRIPS.
```

### Two very different things get conflated

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │ DATABASE-INTERNAL                │ HETEROGENEOUS                        │
   │ distributed transactions         │ distributed transactions             │
   ├──────────────────────────────────┼──────────────────────────────────────┤
   │ All nodes run THE SAME DATABASE  │ TWO OR MORE DIFFERENT TECHNOLOGIES:  │
   │ SOFTWARE. VoltDB, FoundationDB,  │ databases from different vendors, or │
   │ MySQL Cluster NDB.               │ even non-database systems such as    │
   │                                  │ MESSAGE BROKERS.                     │
   │ ✅ "can use ANY protocol and      │                                      │
   │    apply OPTIMIZATIONS specific  │ ❌ "a lot more challenging"           │
   │    to that technology" → often   │                                      │
   │    work QUITE WELL               │                                      │
   └──────────────────────────────────┴──────────────────────────────────────┘
```

### Exactly-once message processing

```
   "a message from a message queue can be ACKNOWLEDGED AS PROCESSED IF AND
    ONLY IF the database transaction for processing the message was
    SUCCESSFULLY COMMITTED."

   ┌──────────────────────────────────────────────────────────────────────┐
   │  ATOMICALLY COMMIT { message acknowledgement + database writes }     │
   │                                                                      │
   │  Either fails → BOTH aborted → the broker SAFELY REDELIVERS later.   │
   │  ➜ the message is EFFECTIVELY PROCESSED EXACTLY ONCE.                │
   └──────────────────────────────────────────────────────────────────────┘

   ⚠️ "only possible if ALL systems affected are able to use THE SAME ATOMIC
      COMMIT PROTOCOL. For example, if a side-effect is to SEND AN EMAIL, and
      the email server does not support 2PC, then the email MIGHT BE SENT TWO
      OR MORE TIMES."
```

### XA transactions

```
   X/Open XA (eXtended Architecture), introduced 1991.
   Supported by PostgreSQL, MySQL, DB2, SQL Server, Oracle, ActiveMQ,
   HornetQ, MSMQ, IBM MQ.

   ⚑ "XA IS NOT A NETWORK PROTOCOL — it is merely A C API for interfacing with
     a transaction coordinator." (Java: JTA, over JDBC and JMS drivers.)

   ⚠️ "in practice the coordinator is often SIMPLY A LIBRARY LOADED INTO THE
      SAME PROCESS AS THE APPLICATION (not a separate service)."
      → "If the application process crashes… THE COORDINATOR GOES WITH IT.
         Any participants with prepared but uncommitted transactions are then
         STUCK IN DOUBT."
      → the coordinator's log is on THE APPLICATION SERVER'S LOCAL DISK, so
        THAT SERVER MUST BE RESTARTED to recover.
```

### ⚠️ Holding locks while in doubt — the operational nightmare

```
   "when using 2-phase commit, a transaction MUST HOLD ONTO THE LOCKS
    THROUGHOUT THE TIME IT IS IN DOUBT.

    If the coordinator has crashed and takes 20 MINUTES to start up again,
    THOSE LOCKS WILL BE HELD FOR 20 MINUTES.
    If the coordinator's log is ENTIRELY LOST, THOSE LOCKS WILL BE HELD
    FOREVER — at least until MANUALLY RESOLVED BY AN ADMINISTRATOR."

   ➜ "This can cause LARGE PARTS OF YOUR APPLICATION TO BECOME UNAVAILABLE."
```

```
   💀 ORPHANED IN-DOUBT TRANSACTIONS DO OCCUR — "e.g. because the transaction
      log is LOST OR CORRUPTED due to a software bug. These transactions
      CANNOT BE RESOLVED AUTOMATICALLY, so they SIT FOREVER in the database,
      holding locks and blocking other transactions."

      "Even REBOOTING your database servers WOULD NOT FIX THIS, since a
       correct implementation of 2PC MUST PRESERVE THE LOCKS OF AN IN-DOUBT
       TRANSACTION EVEN ACROSS REBOOTS. IT'S A STICKY SITUATION."

   🚨 HEURISTIC DECISIONS — the emergency escape hatch that lets a participant
      unilaterally decide. "To be clear, HEURISTIC here is A EUPHEMISM FOR
      PROBABLY BREAKING ATOMICITY."
```

### The four limitations

```
   ① THE COORDINATOR IS ITSELF A KIND OF DATABASE, "and so it needs to be
      approached with THE SAME CARE as any other important database."
      "Surprisingly, MANY COORDINATOR IMPLEMENTATIONS ARE NOT HIGHLY AVAILABLE
       BY DEFAULT."
   ② IT BREAKS STATELESSNESS — "suddenly the coordinator's logs become A
      CRUCIAL PART OF THE DURABLE SYSTEM STATE — as important as the databases
      themselves. SUCH APPLICATION SERVERS ARE NO LONGER STATELESS."
   ③ XA IS A LOWEST COMMON DENOMINATOR — "it CANNOT DETECT DEADLOCKS across
      different systems, and IT DOES NOT WORK WITH SERIALIZABLE SNAPSHOT
      ISOLATION."
   ④ 🔑 THEY AMPLIFY FAILURES — "for 2PC to successfully commit, ALL
      PARTICIPANTS MUST RESPOND. Consequently, IF ANY PART OF THE SYSTEM IS
      BROKEN, THE TRANSACTION ALSO FAILS. Distributed transactions thus have a
      tendency of AMPLIFYING FAILURES, WHICH RUNS COUNTER TO OUR GOAL OF
      BUILDING FAULT TOLERANT SYSTEMS."
```

**Point ④ is the deepest criticism.** A mechanism intended to increase safety *decreases* availability multiplicatively: with n participants each up 99.9% of the time, the transaction succeeds only 99.9%ⁿ of the time.

---

## 13. Fault-tolerant consensus

### The formal definition

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "one or more nodes may PROPOSE values, and the consensus algorithm   ║
   ║   DECIDES on one of those values."                                    ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  UNIFORM AGREEMENT   No two nodes decide differently.       │ SAFETY  ║
   ║  INTEGRITY           No node decides twice.                 │ SAFETY  ║
   ║  VALIDITY            If a node decides v, then v was        │ SAFETY  ║
   ║                      PROPOSED BY SOME NODE.                 │         ║
   ║  TERMINATION         Every node that does not crash         │ LIVENESS║
   ║                      EVENTUALLY DECIDES some value.         │         ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ⚑ VALIDITY "exists mostly to RULE OUT TRIVIAL SOLUTIONS: you could have an
     algorithm that ALWAYS DECIDES null… it would satisfy agreement and
     integrity, but not validity."

   ⚑ TERMINATION "formalizes the idea of FAULT TOLERANCE… it must MAKE
     PROGRESS. Even if some nodes fail, the other nodes must still reach a
     decision." → 2PC DOES NOT MEET THIS.
```

### The crash model, vividly put

> *"When a node 'crashes', it SUDDENLY DISAPPEARS AND NEVER COMES BACK. (Instead of a software crash, imagine that there is AN EARTHQUAKE, and the datacenter containing your node is DESTROYED BY A LANDSLIDE. You must assume that your node is BURIED UNDER 30 FEET OF MUD and is NEVER GOING TO COME BACK ONLINE.)"*

```
   🔑 THE FUNDAMENTAL LIMIT:
   "it can be proved that ANY CONSENSUS ALGORITHM REQUIRES AT LEAST A MAJORITY
    OF NODES TO BE FUNCTIONING CORRECTLY in order to assure termination."

   ⚑ BUT THE SAFETY PROPERTIES STILL HOLD: "most implementations ensure that
     agreement, integrity and validity ARE ALWAYS MET, EVEN IF A MAJORITY OF
     NODES FAILS or there is a severe network problem.

     Thus, a large-scale outage CAN STOP THE SYSTEM FROM PROCESSING REQUESTS,
     BUT IT CANNOT CORRUPT THE CONSENSUS SYSTEM by causing it to make invalid
     decisions."

   ← this is EXACTLY the safety/liveness split from Chapter 8, in production
```

### Consensus ≡ total order broadcast

```
   "total order broadcast is EQUIVALENT TO REPEATED ROUNDS OF CONSENSUS (each
    consensus decision corresponding to ONE MESSAGE DELIVERY)":

   ┌────────────────────┬───────────────────────────────────────────────────┐
   │ AGREEMENT      →   │ all nodes deliver THE SAME MESSAGES IN THE SAME   │
   │                    │ ORDER                                             │
   │ INTEGRITY      →   │ messages are NOT DUPLICATED                       │
   │ VALIDITY       →   │ messages are NOT CORRUPTED or FABRICATED          │
   │ TERMINATION    →   │ messages are NOT LOST                             │
   └────────────────────┴───────────────────────────────────────────────────┘

   THE ALGORITHMS: Viewstamped Replication (VSR) · Paxos · Raft · Zab
   "VSR, Raft and Zab implement total order broadcast DIRECTLY, because that
    is MORE EFFICIENT than doing repeated rounds of one-value-at-a-time
    consensus. In Paxos, this optimization is known as MULTI-PAXOS."

   ⚠️ "unless you're IMPLEMENTING A CONSENSUS SYSTEM YOURSELF (WHICH IS
      PROBABLY NOT ADVISABLE — IT'S HARD)"
```

### 🐔🥚 The chicken-and-egg problem, and its resolution

```
   "It seems that IN ORDER TO ELECT A LEADER, WE FIRST NEED A LEADER. In order
    to solve consensus, we must first solve consensus. HOW DO WE BREAK OUT OF
    THIS CONUNDRUM?"
```

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE ANSWER: EPOCH NUMBERING                                          ║
   ║  (BALLOT number in Paxos, VIEW number in VSR, TERM number in Raft)    ║
   ║                                                                       ║
   ║  "they DON'T GUARANTEE THAT THE LEADER IS UNIQUE. Instead they make   ║
   ║   A WEAKER GUARANTEE: WITHIN EACH EPOCH, THE LEADER IS UNIQUE."       ║
   ║                                                                       ║
   ║   epoch 1 ──────────► epoch 2 ──────────► epoch 3 ──────────►         ║
   ║   leader A            leader B            leader C                    ║
   ║   (thought dead)      (thought dead)      (current)                   ║
   ║                                                                       ║
   ║  "If there is a conflict between two different leaders in two         ║
   ║   different epochs, THE LEADER WITH THE HIGHER EPOCH NUMBER PREVAILS."║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 🔑 The two-quorum argument — the heart of why this is safe

```
   "For EVERY DECISION that a leader wants to make, it must send the proposal
    to the other nodes and WAIT FOR A MAJORITY to respond in favor. A node
    votes in favor ONLY IF IT IS NOT AWARE OF ANY OTHER LEADER WITH A HIGHER
    EPOCH."

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "Since A NODE REQUIRES A MAJORITY OF VOTES TO BECOME LEADER, and A   ║
   ║   PROPOSAL REQUIRES A MAJORITY TO BE DECIDED, we can be sure that AT  ║
   ║   LEAST ONE OF THE NODES VOTING ON A PROPOSAL WILL HAVE SEEN A LEADER ║
   ║   ELECTION IF ONE HAS HAPPENED."                                      ║
   ║                                                                       ║
   ║      election quorum ████████░░░░░                                    ║
   ║      proposal quorum ░░░░████████                                     ║
   ║                          ▲▲▲▲  ← TWO MAJORITIES MUST OVERLAP          ║
   ║                                  (Chapter 8's pigeonhole, again)      ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 2PC vs consensus — the comparison that explains everything

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  TWO-PHASE COMMIT                │  FAULT-TOLERANT CONSENSUS            │
   ├──────────────────────────────────┼──────────────────────────────────────┤
   │  requires a YES VOTE FROM EVERY  │  requires votes from only A MAJORITY │
   │  PARTICIPANT                     │  OF NODES                            │
   │                                  │                                      │
   │  ❌ one dead participant blocks   │  ✅ tolerates a minority failing      │
   │                                  │                                      │
   │  no recovery process for a dead  │  ✅ DEFINES A RECOVERY PROCESS by     │
   │  coordinator                     │     which nodes get into a           │
   │                                  │     consistent state after a new     │
   │                                  │     leader is elected                │
   └──────────────────────────────────┴──────────────────────────────────────┘
   "THESE DIFFERENCES ARE KEY TO THE FAULT TOLERANCE OF A CONSENSUS ALGORITHM."
```

### ⚠️ The five limitations of consensus

```
   ① IT IS SYNCHRONOUS REPLICATION — "databases are often configured to use
      ASYNCHRONOUS replication… many people choose to accept [potential data
      loss] for the sake of BETTER PERFORMANCE."
   ② STRICT MAJORITY REQUIRED — minimum 3 nodes to tolerate 1 failure, 5 to
      tolerate 2. "only the majority portion of the network can make progress."
   ③ FIXED MEMBERSHIP — "you CAN'T JUST ADD OR REMOVE NODES. DYNAMIC
      MEMBERSHIP extensions exist but are MUCH LESS WELL UNDERSTOOD."
   ④ TIMEOUT-SENSITIVE — "in environments with HIGHLY VARIABLE NETWORK DELAYS,
      especially GEOGRAPHICALLY DISTRIBUTED systems… FREQUENT LEADER ELECTIONS
      RESULT IN TERRIBLE PERFORMANCE, because the system can end up SPENDING
      MORE TIME CHOOSING A LEADER THAN DOING ANY USEFUL WORK."
   ⑤ 😬 PATHOLOGICAL EDGE CASES — "Raft has been shown to have UNPLEASANT EDGE
      CASES: if the entire network is working correctly EXCEPT FOR ONE
      PARTICULAR NETWORK LINK that is consistently unreliable, Raft can get
      into situations where LEADERSHIP CONTINUALLY BOUNCES BETWEEN TWO NODES,
      or the current leader is CONTINUALLY FORCED TO RESIGN, so THE SYSTEM
      EFFECTIVELY NEVER MAKES PROGRESS."
```

---

## 14. ZooKeeper, etcd, and coordination services

> **"As an application developer, you RARELY NEED TO USE ZOOKEEPER DIRECTLY, because it is actually NOT WELL SUITED AS A GENERAL-PURPOSE DATABASE. It is more likely that you end up relying on it INDIRECTLY via some other project: HBase, Hadoop YARN, OpenStack Nova and Kafka all rely on ZooKeeper running in the background."**

### The four features (only one of which needs consensus)

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │ ① LINEARIZABLE ATOMIC OPERATIONS  ← THE ONLY ONE REQUIRING CONSENSUS │
   │    compare-and-set → a distributed LOCK, usually implemented as a    │
   │    LEASE with an expiry so it's eventually released if a client dies │
   ├──────────────────────────────────────────────────────────────────────┤
   │ ② TOTAL ORDERING OF OPERATIONS                                       │
   │    gives you FENCING TOKENS: the monotonically increasing zxid and   │
   │    cversion.                            ← Chapter 8's fix, delivered │
   ├──────────────────────────────────────────────────────────────────────┤
   │ ③ FAILURE DETECTION                                                  │
   │    long-lived SESSIONS with heartbeats. If heartbeats cease past the │
   │    session timeout, the session is declared dead and locks held by   │
   │    it are auto-deleted (EPHEMERAL NODES).                            │
   ├──────────────────────────────────────────────────────────────────────┤
   │ ④ EVENT NOTIFICATIONS                                                │
   │    clients WATCH keys for changes — find out when another client     │
   │    JOINS the cluster or FAILS, WITHOUT POLLING.                      │
   └──────────────────────────────────────────────────────────────────────┘

   "it is THE COMBINATION OF THESE FEATURES that makes systems like ZooKeeper
    so useful for distributed coordination."
```

### 🔑 Outsourcing coordination

```
   "An application may initially run only on a single node, but eventually may
    grow to THOUSANDS of nodes. TRYING TO PERFORM MAJORITY VOTES OVER SO MANY
    NODES WOULD BE TERRIBLY INEFFICIENT.

    Instead, ZooKeeper runs on A FIXED NUMBER OF NODES (USUALLY 3 OR 5) and
    performs its majority votes among those, WHILE SUPPORTING A POTENTIALLY
    LARGE NUMBER OF CLIENTS."

        3–5 ZooKeeper nodes           thousands of application nodes
        ┌───┬───┬───┐                 ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
        │ Z │ Z │ Z │ ◄───────────────│ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │
        └───┴───┴───┘   clients       └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
        majority votes                     no voting among these
        happen HERE only
```

> **What kind of data:** *"quite SLOW-CHANGING: it represents information like 'the node running on 10.1.1.23 is the leader for partition 7', which may change on a timescale of MINUTES OR HOURS. It is NOT intended for storing the RUNTIME STATE of the application."*

### Service discovery — does it need consensus?

```
   ⚠️ "it is LESS CLEAR whether service discovery ACTUALLY REQUIRES CONSENSUS.
      DNS is the traditional way… Reads from DNS are ABSOLUTELY NOT
      LINEARIZABLE, and it is usually NOT CONSIDERED PROBLEMATIC if the results
      are A LITTLE STALE."

   ✅ "Although SERVICE DISCOVERY does not require consensus, LEADER ELECTION
      DOES." → some consensus systems support READ-ONLY CACHING REPLICAS that
      receive the log but DON'T VOTE, serving non-linearizable reads.
```

---

## 15. Chapter Summary — the consensus equivalence web

> **"With some digging, it turns out that A WIDE RANGE OF PROBLEMS ARE ACTUALLY REDUCIBLE TO CONSENSUS, AND EQUIVALENT TO EACH OTHER."**

```
                    ╔═══════════════════════════════════╗
                    ║          C O N S E N S U S        ║
                    ╚═══════════════════════════════════╝
                                    │
        ┌───────────┬───────────┬───┴───┬───────────┬───────────┐
        ▼           ▼           ▼       ▼           ▼           ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
   │LINEARIZ.│ │ ATOMIC  │ │  TOTAL  │ │ LOCKS  │ │MEMBER- │ │UNIQUENESS│
   │COMPARE- │ │TRANSACT.│ │  ORDER  │ │  AND   │ │  SHIP  │ │CONSTRAINT│
   │AND-SET  │ │ COMMIT  │ │BROADCAST│ │ LEASES │ │SERVICE │ │          │
   └─────────┘ └─────────┘ └─────────┘ └────────┘ └────────┘ └──────────┘

   "if you have a solution for ONE of them, you can EASILY TRANSFORM IT INTO A
    SOLUTION FOR THE OTHERS."
```

### The three ways to handle a failed leader

```
   ① WAIT FOR THE LEADER TO RECOVER. "Many XA/JTA transaction coordinators
      choose this option. THIS APPROACH DOES NOT SOLVE CONSENSUS because it
      does not satisfy TERMINATION: if the leader does not recover, THE SYSTEM
      CAN BE BLOCKED FOREVER."

   ② MANUAL FAILOVER by humans. "Many relational databases take this approach.
      It is A KIND OF CONSENSUS BY 'ACT OF GOD' — the human operator, OUTSIDE
      of the computer system, makes the decision. The speed is limited by THE
      SPEED AT WHICH HUMANS CAN ACT."

   ③ AN ALGORITHM. "This REQUIRES A CONSENSUS ALGORITHM; any system that
      performs automatic failover WITHOUT USING A PROVEN CONSENSUS ALGORITHM
      IS LIKELY TO BEHAVE BADLY in adverse network conditions."
```

> **The "kicking the can" insight:** *"Although a single-leader database can provide linearizability WITHOUT EXECUTING A CONSENSUS ALGORITHM ON EVERY WRITE, it STILL REQUIRES CONSENSUS TO MAINTAIN ITS LEADERSHIP AND FOR LEADERSHIP CHANGES. Thus, in some sense, HAVING A LEADER ONLY 'KICKS THE CAN DOWN THE ROAD': consensus is still required, only IN A DIFFERENT PLACE, AND LESS FREQUENTLY."*

### And the closing door that opens onto Part III

> **"Not every system necessarily requires consensus: leaderless and multi-leader replication systems typically DO NOT use global consensus. The conflicts that occur in these systems are A CONSEQUENCE OF NOT HAVING CONSENSUS across different leaders. BUT MAYBE THAT'S OK: maybe we simply need to COPE WITHOUT LINEARIZABILITY, and LEARN TO WORK BETTER WITH DATA THAT HAS BRANCHING AND MERGING VERSION HISTORIES."**

**Remember that last sentence.** It's the seed of a whole research programme — see §18.7.

---
---

# 16. 🎁 WHAT'S CHANGED SINCE 2017

You asked for modern additions. Here's an honest scorecard, because this chapter has aged unevenly: the **theory is permanent**, but the **practice moved a lot**.

## 16.1 What has aged perfectly

```
   ✅ The consensus equivalence web — a mathematical result, it cannot expire.
   ✅ Linearizability ∝ network delay uncertainty (Attiya & Welch) — still the
      hard floor, still why geo-distributed strong consistency is painful.
   ✅ "CAP is best avoided" — this was a slightly contrarian take in 2017 and
      is now close to consensus opinion among practitioners.
   ✅ 2PC's blocking problem and in-doubt locks — unchanged, and still the
      reason XA is avoided.
   ✅ "Don't implement consensus yourself" — if anything MORE true now that
      good implementations are ubiquitous.
   ✅ The apology/compensating-transaction argument — became the intellectual
      foundation of the Saga pattern in microservices.
```

## 16.2 Raft won, decisively

In 2017 the book lists "VSR, Paxos, Raft and Zab" as roughly co-equal. That's no longer the picture.

```
   RAFT is now the default choice almost everywhere:
      etcd · Consul · TiKV/TiDB · CockroachDB · YugabyteDB · RethinkDB ·
      Neo4j causal clustering · Redpanda · MongoDB (a Raft-like protocol) ·
      Apache Ratis · Hazelcast · ScyllaDB (Raft for schema/topology, 5.x+)

   WHY: Raft was explicitly designed FOR UNDERSTANDABILITY (Ongaro & Ousterhout,
   "In Search of an Understandable Consensus Algorithm", 2014). Paxos is
   notoriously hard to implement correctly; Raft has a reference spec, a TLA+
   model, and dozens of audited implementations.

   PAXOS survives mostly inside Google (Spanner, Chubby) and in academic
   variants. ZAB survives in ZooKeeper. VSR is largely historical.
```

**📌 The biggest single symbol of this shift: Kafka removed ZooKeeper.** KRaft mode (Kafka Raft) became production-ready in 2022 and ZooKeeper was removed entirely in **Kafka 4.0 (2025)**. The book says "Kafka relies on ZooKeeper running in the background" — that sentence is now historically false. Kafka runs its own Raft quorum for metadata.

## 16.3 The "spanner-like" database category became real

The book says of clock-based distributed snapshots: *"these ideas are interesting, but they have NOT YET BEEN IMPLEMENTED IN MAINSTREAM DATABASES OUTSIDE OF GOOGLE."* That is no longer true.

```
   ┌────────────────┬──────────────────────────────────────────────────────┐
   │ Google Spanner │ GA on GCP; TrueTime + commit-wait as described       │
   │ CockroachDB    │ HLC (hybrid logical clocks) + "uncertainty intervals"│
   │                │ instead of atomic clocks — trades a bit of latency   │
   │                │ for commodity hardware                               │
   │ YugabyteDB     │ HLC, Raft per tablet                                 │
   │ TiDB           │ a centralized Timestamp Oracle (Percolator-style)    │
   │ FoundationDB   │ (already in the book) — now Apple-maintained, and    │
   │                │ famous for its DETERMINISTIC SIMULATION TESTING      │
   │ DynamoDB       │ added ACID TRANSACTIONS in 2018 — a leaderless-      │
   │                │ lineage system growing transactions                  │
   └────────────────┴──────────────────────────────────────────────────────┘

   🆕 HYBRID LOGICAL CLOCKS (HLC) deserve a mention the book doesn't give
      them. An HLC is a Lamport timestamp that also tracks physical time:

         (physical_time, logical_counter)

      • behaves like a physical clock when clocks are well-synced (so
        timestamps are HUMAN-MEANINGFUL and close to wall time)
      • falls back to Lamport-style counter increments when they aren't
      • always CONSISTENT WITH CAUSALITY, unlike raw physical timestamps

      This is the practical middle ground between §8's "logical clocks are
      safe" and §7's "physical clocks are useful but wrong." CockroachDB,
      YugabyteDB and MongoDB all use HLCs.

   🆕 AWS TIME SYNC SERVICE (2023) now offers MICROSECOND-accurate clocks with
      published error bounds on EC2 — essentially TrueTime for everyone. The
      book's complaint that "most systems don't expose this uncertainty" is
      becoming less true.
```

## 16.4 Jepsen changed how we know anything

The book cites Kyle Kingsbury's work repeatedly but doesn't fully convey what happened next.

```
   JEPSEN (jepsen.io) became the de facto correctness audit for distributed
   databases. Its method: run a real cluster, inject network partitions,
   clock skew and process pauses, record a history of operations, then use a
   LINEARIZABILITY CHECKER (Knossos, then Elle) to search for a valid
   sequential ordering.

   🔑 ELLE (2020) was a genuine advance: instead of brute-force checking
     linearizability (which is NP-hard), it infers TRANSACTIONAL ANOMALIES
     from cycles in a dependency graph — so it can find G0/G1a/G1b/G1c/G2
     violations efficiently, and NAME the anomaly it found.

   THE CULTURAL EFFECT: vendor claims of "strong consistency" are now expected
   to come with a Jepsen report. Many well-known systems have failed audits and
   subsequently fixed real bugs. This operationalized the book's remark that
   "it is possible, though computationally expensive, to test whether a
   system's behavior is linearizable."
```

## 16.5 XA is dead in new systems; Sagas took its place

```
   The book predicts distributed transactions will remain controversial. In
   microservice architectures they were essentially abandoned.

   ┌──────────────────────────────────────────────────────────────────────┐
   │  THE SAGA PATTERN                                                    │
   │                                                                      │
   │  Instead of ONE atomic transaction across services, run a SEQUENCE   │
   │  OF LOCAL TRANSACTIONS, each with a COMPENSATING ACTION:             │
   │                                                                      │
   │     T1 ──► T2 ──► T3 ──► T4                                          │
   │      ▲      ▲      ▲      │ fails                                    │
   │      C1 ◄── C2 ◄── C3 ◄───┘   ← run compensations in reverse         │
   │                                                                      │
   │  ⚑ NOT atomic and NOT isolated — intermediate states ARE VISIBLE.    │
   │    You trade ACID for availability, and handle the mess in business  │
   │    logic. THIS IS EXACTLY THE BOOK'S "APOLOGY WORKFLOW" ARGUMENT,    │
   │    promoted to an architectural pattern.                             │
   └──────────────────────────────────────────────────────────────────────┘

   🆕 THE TRANSACTIONAL OUTBOX PATTERN solved the specific "update DB + publish
      message atomically" problem WITHOUT 2PC:

        BEGIN;
          UPDATE orders SET status='paid' WHERE id=42;
          INSERT INTO outbox (topic, payload) VALUES ('orders', '{...}');
        COMMIT;                              ← ONE local transaction

        -- then a separate CDC process (Debezium) tails the DB log and
        -- publishes outbox rows to Kafka, at-least-once.

      This is the single most useful practical answer to the book's
      "exactly-once message processing" section, and it needs NO XA at all.

   🆕 KAFKA TRANSACTIONS (0.11, 2017) + the IDEMPOTENT PRODUCER gave
      exactly-once semantics WITHIN Kafka (read-process-write across topics),
      via a transaction coordinator and epoch-fenced producer IDs. Note the
      mechanism: PRODUCER EPOCHS are exactly the book's epoch-fencing idea.

   🆕 DURABLE EXECUTION ENGINES (Temporal, Restate, DBOS, AWS Step Functions)
      are arguably the 2020s answer to the coordinator problem. They persist
      workflow state in a replicated log so a crashed orchestrator resumes
      exactly where it left off — a coordinator that is HIGHLY AVAILABLE BY
      CONSTRUCTION, which is precisely limitation ① the book lists for XA.
```

## 16.6 CAP was replaced by something more useful: PACELC

```
   The book says CAP "has now been superseded by more precise results" without
   naming the most practically useful one.

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  PACELC (Daniel Abadi, 2012)                                          ║
   ║                                                                       ║
   ║     if (P)artition:  choose (A)vailability or (C)onsistency           ║
   ║     (E)lse:          choose (L)atency      or (C)onsistency           ║
   ║                                                                       ║
   ║  🔑 THE POINT: CAP only describes behaviour DURING A FAULT. PACELC    ║
   ║    adds the trade-off that exists ALL THE TIME — which is exactly     ║
   ║    the book's own observation that "LINEARIZABILITY IS SLOW, AND      ║
   ║    THIS IS TRUE ALL THE TIME, NOT ONLY DURING A NETWORK FAULT."       ║
   ║                                                                       ║
   ║  Examples:  DynamoDB = PA/EL   ·  Spanner = PC/EC                     ║
   ║             Cassandra = PA/EL  ·  MongoDB = PA/EC (tunable)           ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   PACELC formalizes the chapter's own best insight better than CAP does.
```

## 16.7 Causal consistency and CRDTs grew up — and Kleppmann led it

This is the most interesting development, because **the chapter's closing sentence predicted it.**

> *"maybe we simply need to cope without linearizability, and LEARN TO WORK BETTER WITH DATA THAT HAS BRANCHING AND MERGING VERSION HISTORIES."*

```
   Two years after DDIA, Kleppmann co-authored "LOCAL-FIRST SOFTWARE: YOU OWN
   YOUR DATA, IN SPITE OF THE CLOUD" (2019) — a manifesto built on exactly
   that idea. The research became production tooling:

   🆕 AUTOMERGE (Kleppmann et al.) — a JSON CRDT library; Automerge 2.0 (2023)
      made it fast enough for real apps.
   🆕 YJS — the CRDT behind most collaborative editors today (used by Notion-
      likes, Jupyter, Rust/JS ecosystems).
   🆕 ELECTRIC SQL, PowerSync, Triplit, Zero — "sync engines" that give you a
      local replica with causal consistency and automatic merge.
   🆕 RIAK-style CRDTs matured into Redis CRDTs (Active-Active), Azure Cosmos
      DB conflict resolution, and Redis Enterprise.

   🔑 THE CONCEPTUAL SHIFT: the book frames concurrency-without-consensus as a
     COMPROMISE you accept when linearizability is too expensive. The
     local-first line of work reframes it as a FEATURE: offline-capable,
     latency-free, user-owned. Same maths, opposite framing.

   ALSO: COPS, Eiger and the causal-consistency research the book calls
   "quite recent" has partly landed — MongoDB's CAUSAL CONSISTENCY SESSIONS
   (3.6, 2017) are the mainstream example: you get read-your-writes and
   monotonic reads across a sharded cluster WITHOUT full linearizability, by
   passing around a cluster time token. That's precisely the book's
   "middle ground."
```

## 16.8 Formal methods went mainstream

```
   The book warns that implementing consensus yourself is hard. What changed is
   HOW THE EXPERTS NOW VERIFY IT:

   • TLA+ (Lamport's own specification language) is used by AWS (famously, on
     S3 and DynamoDB), MongoDB, Azure Cosmos DB, and the Raft spec itself.
     AWS's paper "Use of Formal Methods at Amazon Web Services" reports TLA+
     finding bugs that would have needed very specific fault sequences.
   • Raft ships with a TLA+ spec AND a machine-checked proof of safety.
   • DETERMINISTIC SIMULATION TESTING (FoundationDB, TigerBeetle, Antithesis)
     runs the entire distributed system single-threaded with a seeded random
     scheduler, so any bug found is PERFECTLY REPRODUCIBLE. FoundationDB's
     simulation testing is arguably why it survived Jepsen without a report —
     Kingsbury declined, saying their own testing was more rigorous.

   ➜ The modern answer to "consensus is hard" is not just "use ZooKeeper" but
     "use something that has been model-checked AND fuzz-tested under fault
     injection."
```

## 16.9 Consensus research kept moving

```
   🆕 FLEXIBLE PAXOS (Howard, Malkhi, Spiegelman 2016; deployed later) —
      a genuinely surprising result: the ELECTION quorum and the REPLICATION
      quorum don't BOTH need to be majorities. They only need to INTERSECT
      WITH EACH OTHER.
         → e.g. with n=5 you can use election quorum = 4, replication = 2
         → makes steady-state writes cheaper at the cost of costlier elections
      This directly refines the book's two-quorum overlap argument in §13.

   🆕 EPAXOS / leaderless consensus — avoids the single-leader bottleneck by
      letting any replica commit non-conflicting commands in one round trip.
      Influential in research; adopted cautiously in practice.

   🆕 RAFT MEMBERSHIP CHANGES got properly worked out (joint consensus and
      single-server changes), softening the book's limitation ③.

   🆕 BFT LEFT ACADEMIA. The book says BFT algorithms are "rarely used in
      practice." Post-2017, blockchain funded a decade of BFT engineering:
      Tendermint/CometBFT, HotStuff (and its descendants in Diem/Aptos/Sui),
      Narwhal-Bullshark. Whatever you think of crypto, BFT consensus is now
      genuinely deployed at scale — which was not true when the book was
      written.
```

## 16.10 A 2026 decision table

```
   ┌────────────────────────────────────┬────────────────────────────────────┐
   │  YOU NEED…                         │  REACH FOR                         │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  leader election / config / locks  │  etcd (Raft) or Consul; ZooKeeper  │
   │                                    │  only if you're already on it      │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  linearizable SQL across regions   │  Spanner, CockroachDB, YugabyteDB  │
   │                                    │  — and budget for the latency      │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  atomic "write DB + emit event"    │  TRANSACTIONAL OUTBOX + CDC        │
   │                                    │  (Debezium). NOT XA.               │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  multi-service business workflow   │  SAGA, ideally on a durable        │
   │                                    │  execution engine (Temporal etc.)  │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  ordered log / state machine repl. │  Kafka (KRaft) or Redpanda         │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  offline-capable collaborative app │  CRDTs (Automerge, Yjs) or a sync  │
   │                                    │  engine. Embrace branch-and-merge. │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  read-your-writes on a sharded DB  │  CAUSAL CONSISTENCY SESSIONS —     │
   │                                    │  cheaper than linearizability      │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  to implement consensus yourself   │  DON'T. Still true. More true.     │
   └────────────────────────────────────┴────────────────────────────────────┘
```

*(Caveat worth stating plainly: this section reflects developments through my knowledge cutoff in mid-2026, and this area moves quickly. Treat specific version numbers and product claims as a starting point to verify, not as gospel.)*

---

# 17. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.9 — CONSISTENCY AND CONSENSUS                                        ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  EVENTUAL CONSISTENCY = CONVERGENCE. "Says NOTHING about WHEN." Hard because   ║
║  a database LOOKS like a variable but ISN'T. Bugs appear only under FAULTS or  ║
║  HIGH CONCURRENCY.                                                            ║
║  Isolation levels (Ch.7) ≠ consistency models (Ch.9): RACE CONDITIONS vs       ║
║  COORDINATING REPLICAS. Mostly independent concerns.                          ║
║                                                                               ║
║  ── LINEARIZABILITY ───────────────────────────────────────────────────────── ║
║  = "behave as if there is ONE COPY and all ops are ATOMIC" = a RECENCY         ║
║    GUARANTEE. Once ANY read returns the new value, ALL later reads must too.   ║
║  Alice & Bob (Fig 9-1): Bob reloads AFTER hearing Alice → must not see stale.  ║
║  ⚠️ vs SERIALIZABILITY: serializability = TRANSACTION ISOLATION over MULTIPLE  ║
║     objects, order may differ from real time. Linearizability = RECENCY on ONE ║
║     REGISTER, real-time respected. BOTH = STRICT SERIALIZABILITY.              ║
║     2PL & serial execution ARE linearizable; SSI IS NOT (snapshots look back). ║
║  NEEDED FOR: leader election/locks (else SPLIT BRAIN) · uniqueness constraints ║
║     · CROSS-CHANNEL dependencies (Fig 9-5 image resizer — two paths race).     ║
║  💡 OFTEN AVOIDABLE: compensating transactions. "The APOLOGY WORKFLOW already   ║
║     needs to be part of your business processes anyway" (fork-lift argument).  ║
║  IMPLEMENTATIONS: single-leader = MAYBE (delusional leaders break it) ·        ║
║     consensus = YES · multi-leader = NO · leaderless = PROBABLY NOT — Fig 9-6  ║
║     shows STRICT QUORUMS (w+r>n) still violating it. Linearizable CAS needs    ║
║     consensus; quorums alone can't do it.                                      ║
║  COST: CAP = "either Consistent or Available when Partitioned". Book says CAP  ║
║     is NARROW, its "availability" is idiosyncratic, and it's BEST AVOIDED.     ║
║     🔑 The REAL reason linearizability is rare is PERFORMANCE, NOT FAULT        ║
║       TOLERANCE — even multicore RAM isn't linearizable, for speed.            ║
║       ATTIYA & WELCH: response time ∝ NETWORK DELAY UNCERTAINTY. No faster     ║
║       algorithm exists.                                                       ║
║                                                                               ║
║  ── ORDERING ──────────────────────────────────────────────────────────────── ║
║  "Much of the last 200 pages boils down to: WATCH YOUR CAUSALITY!"             ║
║  CAUSALITY = PARTIAL order (branches & merges, like git). LINEARIZABILITY =    ║
║  TOTAL order (one timeline, NO concurrency exists).                           ║
║  Linearizability IMPLIES causality — and preserves it across MULTIPLE CHANNELS ║
║  for free. But CAUSAL CONSISTENCY is THE STRONGEST MODEL THAT DOESN'T SLOW     ║
║  DOWN WITH NETWORK DELAY and stays AVAILABLE — and CAP DOESN'T APPLY to it.    ║
║  BAD SEQUENCE NUMBERS: odd/even (counters drift) · physical clocks (skew) ·    ║
║     block allocation (later op gets lower block). All INCONSISTENT WITH        ║
║     CAUSALITY.                                                                ║
║  LAMPORT TIMESTAMPS = (counter, node ID). THE TRICK: everyone carries the MAX  ║
║     COUNTER SEEN on every request; receivers jump their counter up to it.      ║
║     ✅ total order consistent with causality                                   ║
║     ❌ CANNOT tell concurrent from causally-dependent (version vectors can)    ║
║     ❌ AND NOT ENOUGH: the total order only emerges AFTER COLLECTING ALL OPS.  ║
║        For a uniqueness constraint you must DECIDE NOW → you'd have to ask     ║
║        EVERY node → one failure halts everything.                             ║
║  ➜ YOU ALSO NEED TO KNOW WHEN THE ORDER IS FINALIZED → TOTAL ORDER BROADCAST. ║
║  TOB = RELIABLE DELIVERY + TOTALLY ORDERED DELIVERY. Order is FIXED AT         ║
║     DELIVERY — nothing can be retroactively inserted. Gives you: state machine ║
║     replication · serializable txns · a LOG · FENCING TOKENS (zxid).           ║
║     Build linearizable CAS on it: append claim → wait for your own message →   ║
║     first claim wins. (Writes linearizable; READS only SEQUENTIALLY CONSISTENT ║
║     unless you sequence them through the log / sync() / read a sync replica.)  ║
║                                                                               ║
║  ── CONSENSUS ─────────────────────────────────────────────────────────────── ║
║  FLP says consensus is impossible — but only for DETERMINISTIC algorithms with ║
║  NO CLOCKS/TIMEOUTS. Add timeouts OR randomness → SOLVABLE.                    ║
║  2PC: prepare → vote → COMMIT POINT (coordinator writes decision to disk) →    ║
║     commit/abort, RETRY FOREVER. TWO POINTS OF NO RETURN: the participant's    ║
║     "yes" (gives up the right to abort) and the coordinator's logged decision. ║
║     ⚠️ 2PC ≠ 2PL. 💒 The marriage analogy: fainting after "I do" changes nothing.║
║     💥 COORDINATOR CRASH → participants stuck IN DOUBT, HOLDING LOCKS, possibly ║
║        FOREVER. Even reboots can't clear them. HEURISTIC DECISIONS = "a        ║
║        euphemism for PROBABLY BREAKING ATOMICITY."                            ║
║     3PC needs a PERFECT FAILURE DETECTOR → impossible with unbounded delay.    ║
║     XA: a C API, coordinator often IN THE APP PROCESS → app servers are NO     ║
║     LONGER STATELESS; lowest common denominator (no cross-system deadlock      ║
║     detection, no SSI); AND IT AMPLIFIES FAILURES (all participants must       ║
║     respond) — the opposite of fault tolerance.                               ║
║  CONSENSUS PROPERTIES: uniform AGREEMENT · INTEGRITY · VALIDITY (all SAFETY)   ║
║     + TERMINATION (LIVENESS, needs a MAJORITY alive). Safety holds EVEN IF a   ║
║     majority fails — an outage can STOP the system but CANNOT CORRUPT it.      ║
║  ALGORITHMS: VSR · Paxos (Multi-Paxos) · Raft · Zab — all implement TOTAL      ║
║     ORDER BROADCAST, which is EQUIVALENT to repeated consensus.                ║
║  🐔🥚 "To elect a leader we first need a leader." RESOLVED BY EPOCH NUMBERS      ║
║     (ballot/view/term): the leader is unique WITHIN AN EPOCH; higher epoch     ║
║     wins. Every decision needs a MAJORITY VOTE, and since election and         ║
║     proposal quorums MUST OVERLAP, a stale leader is always detected.          ║
║  vs 2PC: consensus needs only a MAJORITY (not everyone) and DEFINES A RECOVERY ║
║     PROCESS. That's the whole difference.                                     ║
║  LIMITS: it's SYNCHRONOUS replication · strict majority · fixed membership ·   ║
║     timeout-sensitive (thrashing elections) · Raft's bad-link pathology.       ║
║  ZOOKEEPER/etcd: linearizable CAS (the only part needing consensus) + total    ║
║     ordering (FENCING TOKENS) + failure detection (sessions, EPHEMERAL NODES)  ║
║     + WATCHES. Runs on 3–5 nodes, serves THOUSANDS of clients — coordination   ║
║     is OUTSOURCED. Data is SLOW-CHANGING, NOT app runtime state.              ║
║     Service discovery does NOT need consensus (DNS is stale and fine); LEADER  ║
║     ELECTION DOES.                                                            ║
║                                                                               ║
║  🔑 ALL EQUIVALENT TO CONSENSUS: linearizable CAS · atomic commit · total order ║
║     broadcast · locks/leases · membership · uniqueness constraints.           ║
║  THREE WAYS TO HANDLE A DEAD LEADER: wait (XA — fails TERMINATION) · manual    ║
║     failover ("consensus by ACT OF GOD") · a real algorithm.                   ║
║  A leader "KICKS THE CAN DOWN THE ROAD": consensus is still needed, just LESS  ║
║     OFTEN — for leadership changes.                                           ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 18. ✅ Test yourself

1. **Alice and Bob both reload at the same instant and see different scores. Is that a linearizability violation?**
   → No. Concurrent requests may legitimately return different results. The violation in Figure 9-1 is that Bob reloads *strictly after* hearing Alice, so his read must be at least as recent as hers.

2. **Serializability and linearizability both mean "some sequential order." What's the difference?**
   → Serializability is an isolation property over multi-object transactions, and the equivalent serial order need not match real time. Linearizability is a recency guarantee on a single object, and it does respect real time. SSI is serializable but not linearizable, precisely because snapshots deliberately read the past.

3. **Your Dynamo-style store uses w + r > n. Do you have linearizability?**
   → No. Figure 9-6 shows a strict quorum violating it under variable network delays. You'd additionally need synchronous read repair and read-before-write, which Riak declines on performance grounds. And linearizable compare-and-set is impossible in that model regardless — it needs consensus.

4. **Why does the book say CAP is "best avoided"?**
   → It covers only one consistency model and one fault type, says nothing about delays or dead nodes, uses an idiosyncratic definition of availability that many highly available systems fail, and is often stated as "pick 2 of 3" when partitions aren't something you choose. PACELC captures the trade-off better because it also describes the non-partitioned case.

5. **If linearizability's problem were fault tolerance, what would that fail to explain?**
   → Multicore RAM. CPU caches aren't linearizable even though communication inside one machine is reliable and a core that's disconnected isn't expected to keep working. The reason there is purely performance — which is the same reason most distributed databases skip it.

6. **What exactly makes Lamport timestamps insufficient for a username uniqueness constraint?**
   → They give you a total order, but the order only becomes known once every operation has been collected. The node must decide *now* whether to accept a registration, and it cannot know whether another node is concurrently claiming the same name with a lower timestamp without asking everyone — which breaks under a single failure.

7. **What does total order broadcast add that timestamp ordering lacks?**
   → Finality. Once a message is delivered, nothing may be retroactively inserted before it. That's what lets a node decide immediately rather than waiting for the order to settle.

8. **A 2PC coordinator crashes after participants vote yes. Why can't a participant just time out and abort?**
   → Because another participant may already have committed. Having voted yes, it gave up the right to abort unilaterally, and aborting now would make the participants inconsistent with each other. It must stay in doubt — holding locks — until the coordinator recovers.

9. **How do consensus algorithms escape the "to elect a leader you need a leader" circularity?**
   → By weakening the guarantee. They don't ensure a globally unique leader; they ensure a unique leader *per epoch*, with monotonically increasing epoch numbers. A leader must win a majority to be elected and a majority to decide anything, and since those two majorities must intersect, any stale leader is detected before it can act.

10. **Name three problems that are equivalent to consensus, and say why that matters.**
    → Linearizable compare-and-set, atomic transaction commit, and total order broadcast (also locks/leases, membership, and uniqueness constraints). It matters because a solution to any one transforms into a solution to the others — so recognizing your problem as one of these tells you immediately both that it's solvable and how expensive it will be.

---

*All quoted material, figures and examples attributed to the book are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 9. Diagrams have been redrawn in ASCII from the book's originals. Section 16 (post-2017 developments) is supplementary material I added; that landscape moves quickly, so treat specific products, versions and claims there as a starting point worth verifying against current sources.*
