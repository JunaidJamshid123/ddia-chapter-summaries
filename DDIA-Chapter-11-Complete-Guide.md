# DDIA — Chapter 11: Stream Processing
### Complete study guide — theory, every diagram redrawn, worked examples, and what changed since 2017

> *A complex system that works is invariably found to have evolved from a simple system that works. The inverse proposition also appears to be true: A complex system designed from scratch never works and cannot be made to work.*
> — John Gall, *Systemantics* (1975)

---

## 0. The map of this chapter

Chapter 10 assumed **bounded** input. This chapter removes that assumption, and almost everything changes as a consequence.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  ① TRANSMITTING EVENT STREAMS                                                │
│       events · producers/consumers · direct messaging · MESSAGE BROKERS      │
│       (AMQP/JMS style) vs PARTITIONED LOGS (Kafka style)                     │
│                                                                              │
│  ② DATABASES AND STREAMS                                                     │
│       the dual-writes race · CHANGE DATA CAPTURE · EVENT SOURCING            │
│       · state as the integral of an event stream · immutability              │
│                                                                              │
│  ③ PROCESSING STREAMS                                                        │
│       CEP · analytics · materialized views · REASONING ABOUT TIME            │
│       · windows · THREE KINDS OF JOIN · fault tolerance & exactly-once       │
│                                                                              │
│  🔑 THE ONE ASSUMPTION THAT CHANGES EVERYTHING: THE INPUT NEVER ENDS.        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Why batch isn't enough

> *"one big assumption remained throughout Chapter 10: THAT THE INPUT IS BOUNDED… For example, the sorting operation that is central to MapReduce MUST READ ITS ENTIRE INPUT before it can start producing output."*
>
> *"In reality, A LOT OF DATA IS UNBOUNDED because it arrives gradually over time: your users produced data yesterday and today, and they will continue to produce more data tomorrow. **Unless you go out of business, this process NEVER ENDS, and so the data is NEVER 'COMPLETE' in any meaningful way.**"*

```
   ➜ Batch processors must ARTIFICIALLY DIVIDE the data into chunks of fixed
     duration: a day's worth at the end of every day, an hour's at the end of
     every hour.

   ⚠️ "The problem with DAILY batch processes is that changes in the input are
      only reflected in the output A DAY LATER, which is TOO SLOW for many
      impatient users."

   ➜ Run more frequently — every second — "OR EVEN CONTINUOUSLY, ABANDONING
     THE FIXED TIME-SLICES ENTIRELY, and simply PROCESSING EVERY EVENT AS IT
     HAPPENS. THAT IS THE IDEA BEHIND STREAM PROCESSING."
```

> 📖 **The concept is everywhere already:** *"stdin and stdout of Unix, programming languages (LAZY LISTS), filesystem APIs (Java's FileInputStream), TCP CONNECTIONS, delivering AUDIO AND VIDEO over the internet."*

---

# PART A — TRANSMITTING EVENT STREAMS

## 1. Events, producers and consumers

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  AN EVENT = "a SMALL, SELF-CONTAINED, IMMUTABLE OBJECT containing     ║
   ║  the details of SOMETHING THAT HAPPENED at some point in time."       ║
   ║  Usually contains A TIMESTAMP.                                        ║
   ║                                                                       ║
   ║  It's the streaming word for what batch calls A RECORD.               ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   THE TERMINOLOGY MAPPING:
   ┌──────────────────────────┬──────────────────────────────────────────┐
   │  BATCH                   │  STREAM                                  │
   ├──────────────────────────┼──────────────────────────────────────────┤
   │  record                  │  EVENT                                   │
   │  file written once,      │  event generated once by a PRODUCER      │
   │  read by many jobs       │  (publisher/sender), processed by many   │
   │                          │  CONSUMERS (subscribers/recipients)      │
   │  a FILENAME identifies   │  a TOPIC or STREAM groups related events │
   │  a set of records        │                                          │
   └──────────────────────────┴──────────────────────────────────────────┘
```

### Why polling isn't good enough

> *"In principle, a file or database is SUFFICIENT to connect producers and consumers: a producer writes every event to the datastore, and each consumer PERIODICALLY POLLS… This is essentially what a batch process does."*
>
> ⚠️ *"However, when moving towards continual processing with low delays, **POLLING BECOMES EXPENSIVE**… THE MORE OFTEN YOU POLL, THE LOWER THE PERCENTAGE OF REQUESTS THAT RETURN NEW EVENTS, and thus the higher the overheads."*
>
> *Databases handle this badly: triggers "are VERY LIMITED in what they can do, and have been SOMEWHAT OF AN AFTERTHOUGHT in database design."*

---

## 2. Messaging systems — the two questions that classify them

> *"A direct communication channel like a Unix pipe or TCP connection would be a simple way… However, Unix pipes and TCP connect EXACTLY ONE SENDER WITH ONE RECIPIENT, whereas a messaging system allows MULTIPLE producer nodes and MULTIPLE consumer nodes."*

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  QUESTION 1: WHAT IF PRODUCERS SEND FASTER THAN CONSUMERS CAN         ║
   ║  PROCESS? Three options:                                              ║
   ║                                                                       ║
   ║    ① DROP MESSAGES                                                    ║
   ║    ② BUFFER in a queue  → "what happens as that queue GROWS? Does the ║
   ║         system CRASH if the queue no longer fits in memory? Or does   ║
   ║         it WRITE MESSAGES TO DISK, but how does that affect           ║
   ║         performance?"                                                 ║
   ║    ③ APPLY BACKPRESSURE (block the producer)                          ║
   ║         ← what Unix pipes and TCP do: small fixed buffer, SENDER      ║
   ║           BLOCKED until the recipient takes data out                  ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  QUESTION 2: WHAT IF NODES CRASH OR GO OFFLINE? ARE MESSAGES LOST?    ║
   ║    "durability may require some combination of WRITING TO DISK and/or ║
   ║     REPLICATION, WHICH HAS A COST. If you can afford to sometimes     ║
   ║     lose messages, you can probably get HIGHER THROUGHPUT AND LOWER   ║
   ║     LATENCY on the same hardware."                                    ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### ⚠️ When is losing messages acceptable?

```
   ✅ PERIODIC SENSOR READINGS AND METRICS — "an occasional missing data
      point is perhaps not important, since an UPDATED VALUE WILL BE SENT A
      SHORT TIME LATER anyway."
      ⚠️ BUT: "beware that IF A LARGE NUMBER OF MESSAGES IS DROPPED, IT MAY
         NOT BE IMMEDIATELY APPARENT THAT THE METRICS ARE INCORRECT."

   ❌ COUNTING EVENTS — "it is MORE IMPORTANT that they are delivered
      reliably, since EVERY LOST MESSAGE MEANS INCORRECT COUNTERS."
```

### Direct messaging from producers to consumers

```
   • UDP MULTICAST — "widely used in the FINANCIAL INDUSTRY for stock market
     feeds, where LOW LATENCY is important." UDP is unreliable, but
     application-level protocols can recover lost packets.
   • BROKERLESS LIBRARIES — ZeroMQ, nanomsg: pub-sub over TCP or IP multicast.
   • StatsD / Brubeck — unreliable UDP for metrics. "In the StatsD protocol,
     COUNTER METRICS ARE ONLY CORRECT IF ALL MESSAGES ARE RECEIVED; using UDP
     makes the metrics AT BEST APPROXIMATE."
   • WEBHOOKS — "a callback URL of one service is registered with another
     service, and it makes a request to that URL whenever an event occurs."

   ⚠️ THE COMMON WEAKNESS: "they generally require THE APPLICATION CODE TO BE
      AWARE OF THE POSSIBILITY OF MESSAGE LOSS… they generally assume that
      PRODUCERS AND CONSUMERS ARE CONSTANTLY ONLINE.
      If a consumer is OFFLINE, it may MISS MESSAGES sent while it is
      unreachable… [retry] may break down IF THE PRODUCER CRASHES, LOSING THE
      BUFFER OF MESSAGES IT WAS SUPPOSED TO RETRY."
```

### Message brokers

> *"a MESSAGE BROKER is essentially **A KIND OF DATABASE THAT IS OPTIMIZED FOR HANDLING MESSAGE STREAMS.** It runs as a server, with producers and consumers connecting to it as clients."*
>
> *"By CENTRALIZING the data in the broker, these systems can more easily TOLERATE CLIENTS THAT COME AND GO, and the question of durability is MOVED TO THE BROKER."*

```
   ⚑ CONSEQUENCE — ASYNCHRONY: "when a producer sends a message, it normally
     ONLY WAITS FOR THE BROKER TO CONFIRM that it has BUFFERED the message —
     IT DOES NOT WAIT FOR THE MESSAGE TO BE PROCESSED BY CONSUMERS."
```

### 📊 Message brokers vs databases — four differences

| | **Database** | **Message broker** |
|---|---|---|
| **Retention** | Keeps data **forever until explicitly deleted** | **Automatically deletes** a message once successfully delivered → *"NOT SUITABLE FOR LONG-TERM DATA STORAGE"* |
| **Working set** | Assumes large datasets | *"assumes their working set is FAIRLY SMALL, i.e. the queues are SHORT."* If consumers are slow and messages spill to disk, *"each individual message takes longer to process, and THE OVERALL THROUGHPUT MAY DEGRADE"* |
| **Selecting data** | Secondary indexes, arbitrary queries | **Subscribing to topics matching a pattern** — *"The mechanisms are different, but both are essentially ways for a client to SELECT THE PORTION OF THE DATA IT WANTS"* |
| **Change notification** | Result is a **point-in-time snapshot**; the client isn't told when it goes stale (*unless it polls*) | **No arbitrary queries, but they DO NOTIFY clients when data changes** |

**This last row is the crux.** Databases have data but no notifications; brokers have notifications but no durable data. The whole second half of the chapter is about getting both.

*Implemented in RabbitMQ, ActiveMQ, HornetQ, Qpid, TIBCO EMS, IBM MQ, Azure Service Bus; standardized in JMS and AMQP.*

---

## 3. Multiple consumers: two patterns

### 🔷 Figure 11-1 — Load balancing vs fan-out

```
   (a) LOAD BALANCING                          (b) FAN-OUT
       "shared subscription" (JMS)                 "topic subscription" (JMS)
       multiple consumers on one                   "exchange bindings" (AMQP)
       queue (AMQP)
                                        TIME ──►                        TIME ──►
   Producer 1 ──●──────●───────────            Producer 1 ──●──────●───────────
   Producer 2 ──────●──────●───────            Producer 2 ──────●──────●───────
                    │                                           │
   Broker     [m1][m2][m3][m4][m5]            Broker     [m1][m2][m3][m4][m5]
                    │                                           │
              ┌─────┴─────┐                              ┌──────┴──────┐
              ▼           ▼                              ▼             ▼
   Consumer 1   [m2]    [m4]                  Consumer 1  [m1][m2][m3][m4]
   Consumer 2 [m1]  [m3]                      Consumer 2  [m1][m2][m3][m4]

   EACH MESSAGE TO **ONE** CONSUMER          EACH MESSAGE TO **ALL** CONSUMERS
   "useful when the messages are             "the streaming equivalent of
    EXPENSIVE TO PROCESS, and so you          having SEVERAL DIFFERENT BATCH
    want to ADD CONSUMERS TO                  JOBS THAT READ THE SAME INPUT
    PARALLELIZE the processing."              FILE."

   ⚑ THEY COMBINE: "two separate GROUPS of consumers may each subscribe to a
     topic, such that EACH GROUP COLLECTIVELY RECEIVES ALL MESSAGES, but
     WITHIN each group only ONE of the nodes receives each message."
```

### Acknowledgements and redelivery

> *"a client must EXPLICITLY TELL THE BROKER when it has finished processing a message, so that the broker can remove it from the queue. If the connection is closed or times out without the broker receiving an acknowledgement, it ASSUMES THE MESSAGE WAS NOT PROCESSED, and therefore it DELIVERS THE MESSAGE AGAIN to another consumer."*
>
> ⚠️ *"Note that it COULD HAPPEN THAT THE MESSAGE ACTUALLY WAS FULLY PROCESSED, BUT THE ACKNOWLEDGEMENT WAS LOST IN THE NETWORK."* ← Chapter 8's indistinguishable failures

### 🔷 Figure 11-2 — Load balancing + redelivery breaks ordering

```
                                                              TIME ─────────►
   Producer 1 ──●────────●──────────●────────────────────────────────────────►
   Producer 2 ─────●──────────●─────────────────────────────────────────────►

   Broker      [m1]  [m2]  [m3]  [m4]  [m5]

                        ▲ ack              ▲ ack
   Consumer 1 ─────────[m2]───────────────[m4]────────[m3]────────[m5]──────►
                                                        ▲
                   ▲ ack                                │ REDELIVERED
   Consumer 2 ────[m1]──────────[m3]✗ CRASHED ──────────┘

   💥 CONSUMER 1 PROCESSES IN THE ORDER  m4, m3, m5
      "Thus, m3 and m4 are NOT DELIVERED IN THE SAME ORDER AS THEY WERE SENT."

   🔑 "Even if the message broker OTHERWISE TRIES TO PRESERVE THE ORDER of
     messages (AS REQUIRED BY BOTH THE JMS AND AMQP STANDARDS), THE
     COMBINATION OF LOAD BALANCING WITH REDELIVERY INEVITABLY LEADS TO
     MESSAGES BEING REORDERED."

   "This is NOT A PROBLEM if messages are COMPLETELY INDEPENDENT of each
    other, but IT CAN BE IMPORTANT IF THERE ARE CAUSAL DEPENDENCIES."
```

---

## 4. Partitioned logs — the hybrid

### 🔑 The mindset difference the whole section turns on

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  MESSAGING MINDSET                ║  DATABASE / FILESYSTEM MINDSET    ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  "Sending a packet… is normally A ║  "everything that is written is   ║
   ║   TRANSIENT OPERATION THAT LEAVES ║   normally expected to be         ║
   ║   NO PERMANENT TRACE."            ║   PERMANENTLY RECORDED, at least  ║
   ║  Even durable brokers "QUICKLY    ║   until someone EXPLICITLY        ║
   ║  DELETE THEM AGAIN after they     ║   CHOOSES TO DELETE it."          ║
   ║  have been delivered."            ║                                   ║
   ╠═══════════════════════════════════╩═══════════════════════════════════╣
   ║  ⚠️ THE CONSEQUENCE FOR DERIVED DATA:                                  ║
   ║  "RECEIVING A MESSAGE IS DESTRUCTIVE if processing it causes it to be ║
   ║   deleted from the broker, so YOU CANNOT RUN THE SAME CONSUMER AGAIN  ║
   ║   AND EXPECT TO GET THE SAME RESULT."                                 ║
   ║  "If you ADD A NEW CONSUMER, it typically only starts receiving       ║
   ║   messages FROM THE TIME IT WAS REGISTERED, BUT NO PRIOR MESSAGES."   ║
   ║                                                                       ║
   ║  ➜ "WHY CAN WE NOT HAVE A HYBRID, combining THE DURABLE STORAGE       ║
   ║    APPROACH OF DATABASES with THE LOW-LATENCY NOTIFICATION            ║
   ║    FACILITIES OF MESSAGING?"                                          ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 🔷 Figure 11-3 — Partitioned logs

```
              ┌─────────────────────────────────────────┐
   TOPIC A    │ Partition 0 │1│2│3│4│5│6│7│8│9│10│ ◄── append ── PRODUCER
              │ Partition 1 │1│2│3│4│5│6│7│8│      ◄── append ── PRODUCER
              └─────────────────────────────────────────┘

              ┌─────────────────────────────────────────┐   CONSUMER GROUP
   TOPIC B    │ Partition 0 │1│2│3│4│                   │   ┌──────────────┐
              │             ────────▲───────────────────┼──►│ CONSUMER     │
              │ Partition 1 │1│2│3│4│5│6│7│             │   │ offset B.0=4 │
              │             ──────────▲─────────────────┼──►│ offset B.1=5 │
              │                                         │   └──────────────┘
              │ Partition 2 │1│2│3│4│5│6│7│8│9│10│11│12││   ┌──────────────┐
              │             ──────────────────▲─────────┼──►│ CONSUMER     │
              └─────────────────────────────────────────┘   │ offset B.2=9 │
                                    read sequentially       └──────────────┘

   🔑 "Within each partition, the broker assigns A MONOTONICALLY INCREASING
     SEQUENCE NUMBER, OR OFFSET, to every message. Such a sequence number
     makes sense because A PARTITION IS APPEND-ONLY, so the messages within a
     partition are TOTALLY ORDERED. **THERE IS NO ORDERING GUARANTEE ACROSS
     DIFFERENT PARTITIONS.**"

   → Kafka, Amazon Kinesis Streams, Twitter's DistributedLog.
   📊 "Even though they write all messages to disk, they are able to achieve
      throughput of MILLIONS OF MESSAGES PER SECOND by PARTITIONING across
      multiple machines, and fault tolerance by REPLICATING."
```

### ⚖️ Logs vs traditional messaging — the honest trade-off

```
   ✅ FAN-OUT IS TRIVIAL: "several consumers can INDEPENDENTLY READ THE LOG
      WITHOUT AFFECTING EACH OTHER."

   LOAD BALANCING IS COARSE-GRAINED: "instead of assigning individual messages
   to consumer clients, the broker assigns ENTIRE PARTITIONS to nodes."

   ❌ DOWNSIDE 1: "The number of nodes sharing the work can be AT MOST THE
      NUMBER OF LOG PARTITIONS in that topic."
   ❌ DOWNSIDE 2: "If a SINGLE MESSAGE IS SLOW TO PROCESS, it HOLDS UP THE
      PROCESSING OF SUBSEQUENT MESSAGES in that partition — a form of
      HEAD-OF-LINE BLOCKING."       ← Chapter 1, again

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE DECISION RULE:                                                   ║
   ║  "in situations where messages may be EXPENSIVE TO PROCESS and you    ║
   ║   want to PARALLELIZE ON A MESSAGE-BY-MESSAGE BASIS, and where        ║
   ║   MESSAGE ORDERING IS NOT SO IMPORTANT → the JMS/AMQP STYLE IS        ║
   ║   PREFERABLE.                                                         ║
   ║   On the other hand, in situations with HIGH MESSAGE THROUGHPUT,      ║
   ║   where each message is FAST TO PROCESS and where MESSAGE ORDERING IS ║
   ║   IMPORTANT → THE LOG-BASED APPROACH WORKS VERY WELL."                ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### Consumer offsets — and the database analogy

> *"Consuming a partition sequentially makes it EASY TO TELL WHICH MESSAGES HAVE BEEN PROCESSED: all messages with an offset LESS THAN the consumer's current offset have already been processed… Thus, **the broker DOES NOT NEED TO TRACK ACKNOWLEDGEMENTS FOR EVERY SINGLE MESSAGE** — it only needs to PERIODICALLY RECORD THE CONSUMER OFFSETS."*

```
   🔑 "This offset is IN FACT VERY SIMILAR TO THE LOG SEQUENCE NUMBER that is
     commonly found in SINGLE-LEADER DATABASE REPLICATION…
     **THE MESSAGE BROKER BEHAVES LIKE A LEADER DATABASE, AND THE CONSUMER
     LIKE A FOLLOWER.**"                                    ← Chapter 5

   ⚠️ "If the consumer had processed subsequent messages, BUT NOT YET RECORDED
      THEIR OFFSET, those messages will be PROCESSED A SECOND TIME on restart."
```

### 📐 Disk space: the back-of-the-envelope calculation

```
   "the log is divided into SEGMENTS, and from time to time old segments are
    DELETED or MOVED TO ARCHIVE STORAGE."
   ➜ "Effectively, the log implements A BOUNDED-SIZE BUFFER that discards old
     messages when it gets full, also known as A CIRCULAR BUFFER or RING
     BUFFER. **However, since that buffer is ON DISK, IT CAN BE QUITE LARGE.**"

   ┌──────────────────────────────────────────────────────────────────────┐
   │  6 TB drive ÷ 150 MB/s sequential write ≈ 11 HOURS TO FILL THE DRIVE │
   │                                                                      │
   │  "Thus, the disk can buffer 11 HOURS WORTH OF MESSAGES, after which  │
   │   it will start overwriting old messages. THIS RATIO REMAINS THE     │
   │   SAME, EVEN IF YOU USE MANY HARD DRIVES AND MACHINES.               │
   │   In practice, deployments RARELY USE THE FULL WRITE BANDWIDTH, so   │
   │   the log can typically keep SEVERAL DAYS OR EVEN WEEKS."            │
   └──────────────────────────────────────────────────────────────────────┘

   💡 THE OPERATIONAL ADVANTAGE: "You can MONITOR HOW FAR A CONSUMER IS BEHIND
      the head of the log, and RAISE AN ALERT if it falls behind. As the buffer
      is large, THERE IS ENOUGH TIME FOR A HUMAN TO FIX THE SLOW CONSUMER…
      And even if the consumer does fall too far behind, **IT ONLY AFFECTS
      ITSELF, BUT IT DOES NOT DISRUPT THE SERVICE FOR OTHER CONSUMERS.**"
```

### 🔑 Replaying old messages — the property that matters most

> *"in a log-based message broker, CONSUMING MESSAGES IS MORE LIKE READING FROM A FILE: **it is a READ-ONLY OPERATION THAT DOES NOT CHANGE THE LOG.** The only side-effect is that THE CONSUMER OFFSET MOVES FORWARD. But the offset is UNDER THE CONSUMER'S CONTROL, so it can easily be manipulated: for example, you can **start a copy of a consumer with YESTERDAY'S OFFSETS, and write the output to a DIFFERENT LOCATION**, in order to re-process the last day's worth of messages. You can repeat this any number of times, varying the processing code."*
>
> *"This aspect makes log-based messaging MORE LIKE THE BATCH PROCESSES of the last chapter, where derived data is clearly separated from input data through a REPEATABLE TRANSFORMATION PROCESS. It allows MORE EXPERIMENTATION AND EASIER RECOVERY FROM ERRORS AND BUGS."*

**That's Chapter 10's human fault tolerance, delivered for streams.**

---
---

# PART B — DATABASES AND STREAMS

## 5. A write to a database *is* an event

> *"The thing that happened may be a user action, or a sensor reading, **but it may also be A WRITE TO A DATABASE.** The fact that something was written to a database is AN EVENT THAT CAN BE CAPTURED, STORED AND PROCESSED. This suggests that the connection between databases and streams runs deeper than just the physical storage of logs on disk — **IT IS QUITE FUNDAMENTAL.**"*

```
   ⚑ "a REPLICATION LOG is A STREAM OF DATABASE WRITE EVENTS, produced by the
     leader as it processes transactions."
   ⚑ STATE MACHINE REPLICATION (Ch.9): "if every event represents a write, and
     every replica processes the same events IN THE SAME ORDER, then the
     replicas will all end up IN THE SAME FINAL STATE.
     **IT'S JUST ANOTHER CASE OF EVENT STREAMS!**"
```

## 6. Keeping systems in sync — and why dual writes fail

```
   "there is NO SINGLE SYSTEM that can satisfy all needs… an OLTP database to
    serve user requests, a CACHE to speed up common requests, a FULL-TEXT
    INDEX for search, and a DATA WAREHOUSE for analytics. Each of these has
    ITS OWN COPY of the data."

   ❌ DUAL WRITES: "the application code EXPLICITLY WRITES TO EACH OF THE
      SYSTEMS when data changes: first writing to the database, then updating
      the search index, then invalidating the cache entries."
```

### 🔷 Figure 11-4 — The dual-writes race condition

```
                  set X = A                        set X = A
                      │                                │      TIME ────────►
   Client 1 ──────────●────────────────────────────────●────────────────────►
                      │   ▲ ok                         │   ▲ ok
                      ▼   │                            │   │
   Database ──────────●───●──────────●─────────────────┼───┼────────────────►
                                     │  ▲ ok           │   │
                              FINAL VALUE: X = B       │   │
                                                       ▼   │
   Search index ───────────────────────────────────────●───●────────────────►
                                          ▲
                              FINAL VALUE: X = A   💥 PERMANENTLY INCONSISTENT
                                 │             │
   Client 2 ─────────────────────●─────────────●───────────────────────────►
                             set X = B     set X = B

   THE DATABASE sees A then B  →  ends up B
   THE INDEX    sees B then A  →  ends up A

   "The two systems are now PERMANENTLY INCONSISTENT with each other, EVEN
    THOUGH NO ERROR OCCURRED IN THE EXECUTION."

   ⚠️ "Unless you have some ADDITIONAL CONCURRENCY TRACKING MECHANISM, such as
      VERSION VECTORS, YOU WILL NOT EVEN NOTICE that concurrent writes
      occurred — one value will SIMPLY SILENTLY OVERWRITE another."
```

```
   ❌ AND A SECOND, SEPARATE PROBLEM: "one of the writes may FAIL while the
      other SUCCEEDS. This is a FAULT-TOLERANCE problem rather than a
      concurrency problem, but it ALSO has the effect of the two systems
      becoming inconsistent. Ensuring both succeed or both fail is A CASE OF
      THE ATOMIC COMMIT PROBLEM, WHICH IS EXPENSIVE TO SOLVE."   ← Ch.9's 2PC

   🔑 THE DIAGNOSIS: "in Figure 11-4 THERE ISN'T A SINGLE LEADER: the database
     may have a leader and the search index may have a leader, BUT NEITHER
     FOLLOWS THE OTHER, and so conflicts can occur."   ← multi-leader, Ch.5

   ➜ "The situation would be better IF THERE REALLY WAS ONLY ONE LEADER, for
     example the database, AND IF WE COULD MAKE THE SEARCH INDEX A FOLLOWER
     OF THE DATABASE. But is this possible in practice?"
```

## 7. Change data capture

> *"The problem with most databases' replication logs is that they have long been considered to be AN INTERNAL IMPLEMENTATION DETAIL OF THE DATABASE, **NOT A PUBLIC API.** Clients are supposed to query the database through its data model and query language, NOT PARSE THE REPLICATION LOGS."*
>
> **CDC** = *"the process of OBSERVING ALL DATA CHANGES written to a database, and EXTRACTING THEM IN A FORM IN WHICH THEY CAN BE REPLICATED TO OTHER SYSTEMS."*

### 🔷 Figure 11-5 — CDC turns one database into the leader

```
                  SYSTEM OF RECORD                        DERIVED DATA SYSTEMS
   Client 1 ──┐
   set X = A  │   ┌──────────────┐
              ├──►│   DATABASE   │                            ┌───────────────┐
   Client 2 ──┘   └──────┬───────┘                       ┌───►│ SEARCH INDEX  │
   set X = B             │                               │    └───────────────┘
                    CHANGE DATA CAPTURE                  │
                         │              append           │    ┌───────────────┐
                         ▼                               ├───►│ DATA WAREHOUSE│
              … [ X=A ][ X=B ] …  ───────────────────────┤    └───────────────┘
              LOG OF DATA CHANGES                        │    ┌───────────────┐
                                                         └───►│ CACHE         │
              log consumers APPLY CHANGES IN ORDER            └───────────────┘

   🔑 "Essentially, change data capture MAKES ONE DATABASE THE LEADER (the one
     from which the changes are captured), AND TURNS THE OTHERS INTO
     FOLLOWERS."

   ⚑ "A LOG-BASED MESSAGE BROKER IS WELL SUITED for transporting the change
     events, since IT PRESERVES THE ORDERING OF MESSAGES (avoiding the
     reordering issue of Figure 11-2)."
```

### How it's implemented

```
   ❌ DATABASE TRIGGERS — "they tend to be FRAGILE and have SIGNIFICANT
      PERFORMANCE OVERHEADS."
   ✅ PARSING THE REPLICATION LOG — "a MORE ROBUST approach, although it also
      comes with challenges, SUCH AS HANDLING SCHEMA CHANGES."

   IN THE WILD: LinkedIn's DATABUS · Facebook's WORMHOLE · Yahoo's SHERPA ·
   BOTTLED WATER (Postgres WAL decoding) · MAXWELL and DEBEZIUM (MySQL binlog)
   · MONGORIVER (MongoDB oplog) · GOLDENGATE (Oracle).

   ⚠️ "Like message brokers, CDC is usually ASYNCHRONOUS: the system of record
      DOES NOT WAIT for the change to be applied to consumers before
      committing. ✅ adding a slow consumer does not slow down the system of
      record, ❌ BUT ALL THE ISSUES OF REPLICATION LAG APPLY."   ← Chapter 5
```

### Initial snapshot and log compaction

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │ THE PROBLEM: "Building a new full-text index requires A FULL COPY of  │
   │ the entire database — it is NOT SUFFICIENT to only apply a log of     │
   │ recent changes, since it would be MISSING ITEMS THAT WERE NOT         │
   │ RECENTLY UPDATED."                                                    │
   │                                                                      │
   │ ANSWER 1 — INITIAL SNAPSHOT, which "must CORRESPOND TO A KNOWN        │
   │ POSITION OR OFFSET in the change log, so that you know AT WHICH POINT │
   │ TO START APPLYING CHANGES after the snapshot."      ← Ch.5's LSN      │
   │                                                                      │
   │ ANSWER 2 — LOG COMPACTION (Ch.3's hash indexes, reused):              │
   │   "the storage engine periodically looks for log records WITH THE     │
   │    SAME KEY, THROWS AWAY DUPLICATES and keeps ONLY THE MOST RECENT    │
   │    UPDATE for each key." A null value = a deletion (a TOMBSTONE).     │
   │                                                                      │
   │   🔑 "The disk space required depends ONLY ON THE CURRENT CONTENTS OF │
   │     THE DATABASE, NOT THE NUMBER OF WRITES that have ever occurred."  │
   │                                                                      │
   │   ➜ "you can start a new consumer FROM OFFSET 0 of the log-compacted  │
   │     topic… The log is GUARANTEED TO CONTAIN THE MOST RECENT VALUE FOR │
   │     EVERY KEY — in other words, IT CAN OBTAIN A FULL COPY OF THE      │
   │     DATABASE CONTENTS WITHOUT HAVING TO TAKE ANOTHER SNAPSHOT."       │
   │                                                                      │
   │   ⚑ "it allows THE MESSAGE BROKER TO BE USED FOR DURABLE STORAGE,     │
   │     NOT JUST FOR TRANSIENT MESSAGING."                                │
   └──────────────────────────────────────────────────────────────────────┘
```

> **Change streams as a first-class API:** RethinkDB subscriptions, Firebase, CouchDB change feeds, Meteor's use of the MongoDB oplog, and **Kafka Connect** — *"an effort to integrate CDC tools for a wide range of database systems with Kafka."*

---

## 8. Event sourcing

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  CHANGE DATA CAPTURE              ║  EVENT SOURCING                   ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  "the application uses the        ║  "the APPLICATION LOGIC IS        ║
   ║   database IN A MUTABLE WAY,      ║   EXPLICITLY BUILT ON THE BASIS   ║
   ║   updating and deleting records   ║   OF IMMUTABLE EVENTS written to  ║
   ║   AT WILL."                       ║   an event log. The event store   ║
   ║                                   ║   is APPEND-ONLY, and updates or  ║
   ║  Changes extracted AT A LOW LEVEL ║   deletes are DISCOURAGED OR      ║
   ║  (parsing the replication log).   ║   PROHIBITED."                    ║
   ║                                   ║                                   ║
   ║  🔑 "THE APPLICATION WRITING TO    ║  "Events are CAREFULLY DESIGNED   ║
   ║    THE DATABASE DOES NOT NEED TO  ║   TO MIRROR THINGS THAT HAPPENED  ║
   ║    BE AWARE THAT CDC IS           ║   AT THE APPLICATION LEVEL."      ║
   ║    OCCURRING."                    ║                                   ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝

   Developed in the DOMAIN-DRIVEN DESIGN (DDD) community. Similar to the
   CHRONICLE data model, and to the FACT TABLE of a star schema (Ch.3).
```

### Deriving current state — and why log compaction differs

```
   "An event log BY ITSELF IS NOT VERY USEFUL, because users generally expect
    to see THE CURRENT STATE of the system, not the history of modifications.
    On a shopping website, users expect to see THE CURRENT CONTENTS OF THEIR
    CART, not an append-only list of all the changes they have ever made."

   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  CDC EVENT                       │  EVENT-SOURCED EVENT                 │
   │  "typically contains THE ENTIRE  │  "modeled at A HIGHER LEVEL: an      │
   │   NEW VERSION OF THE RECORD"     │   event typically expresses THE      │
   │  → the current value is fully    │   INTENT OF A USER ACTION, not the   │
   │    determined by THE MOST RECENT │   mechanics of the state update."    │
   │    event for that key            │  → "LATER EVENTS TYPICALLY DO NOT    │
   │  ✅ LOG COMPACTION WORKS          │     OVERRIDE PRIOR EVENTS, and so    │
   │                                  │     YOU NEED THE FULL HISTORY."      │
   │                                  │  ❌ "LOG COMPACTION IS NOT POSSIBLE   │
   │                                  │     IN THE SAME WAY."                │
   └──────────────────────────────────┴──────────────────────────────────────┘

   ⚑ Snapshots exist, but "this is ONLY A PERFORMANCE OPTIMIZATION to speed up
     reads and recovery; THE INTENTION IS THAT THE SYSTEM IS ABLE TO STORE ALL
     RAW EVENTS FOREVER, AND RE-PROCESS THE FULL EVENT LOG WHENEVER REQUIRED."
```

### 🔑 Commands vs events — the distinction that makes event sourcing work

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │                                                                      │
   │   USER REQUEST                                                       │
   │        │                                                             │
   │        ▼                                                             │
   │   ┌─────────┐   "at this point IT MAY STILL FAIL, for example        │
   │   │ COMMAND │    because some INTEGRITY CONDITION IS VIOLATED."      │
   │   └────┬────┘                                                        │
   │        │  VALIDATE  (is the username free? is the seat free?)        │
   │        │  ← must happen SYNCHRONOUSLY                                │
   │        ▼                                                             │
   │   ┌─────────┐   "which is DURABLE AND IMMUTABLE… At the point when   │
   │   │  EVENT  │    the event is generated, IT BECOMES A FACT."         │
   │   └─────────┘                                                        │
   │                                                                      │
   │   ⚠️ "A CONSUMER OF THE EVENT STREAM IS NOT ALLOWED TO REJECT AN      │
   │      EVENT: by the time the consumer sees it, IT IS ALREADY AN        │
   │      IMMUTABLE PART OF THE LOG, and it MAY HAVE ALREADY BEEN SEEN BY  │
   │      OTHER CONSUMERS."                                               │
   └──────────────────────────────────────────────────────────────────────┘

   ⚑ "Even if the customer LATER DECIDES TO CHANGE OR CANCEL the reservation,
     THE FACT REMAINS TRUE that they formerly held a reservation — and the
     change or cancellation is A SEPARATE EVENT ADDED LATER."

   💡 THE ASYNCHRONOUS ALTERNATIVE: split into TWO events — "first a TENTATIVE
      RESERVATION, and then a separate CONFIRMATION EVENT once the reservation
      has been validated."     ← exactly Chapter 9's total-order-broadcast
                                  username-claiming protocol
```

---

## 9. State, streams and immutability

### 🔷 Figure 11-6 — The calculus analogy

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║                                                                       ║
   ║                    ──────── integrate over time ────────►             ║
   ║     EVENT STREAM                                    APPLICATION STATE ║
   ║     (changelog)                                     (current values)  ║
   ║                    ◄──── differentiate by time ─────                  ║
   ║                                                                       ║
   ║  "the application state is WHAT YOU GET WHEN YOU INTEGRATE AN EVENT   ║
   ║   STREAM OVER TIME, and a change stream is WHAT YOU GET WHEN YOU      ║
   ║   DIFFERENTIATE THE STATE BY TIME."                                   ║
   ║                                                                       ║
   ║  📖 "The analogy has limitations (for example, THE SECOND DERIVATIVE  ║
   ║     OF STATE DOES NOT SEEM TO BE MEANINGFUL), but it's a useful       ║
   ║     starting point."                                                  ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   🔑 "MUTABLE STATE AND AN APPEND-ONLY LOG OF IMMUTABLE EVENTS DO NOT
     CONTRADICT EACH OTHER: THEY ARE TWO SIDES OF THE SAME COIN."

   Examples: "your list of currently available seats is THE RESULT OF THE
   RESERVATIONS you have processed; the current account balance is THE RESULT
   OF THE CREDITS AND DEBITS; the response time graph is AN AGGREGATION OF THE
   INDIVIDUAL RESPONSE TIMES."
```

> **Pat Helland, quoted in full because it's the thesis of the section:**
> *"Transaction logs record all the changes made to the database. High-speed appends are the only way to change the log. From this perspective, **the contents of the database hold A CACHING OF THE LATEST RECORD VALUES IN THE LOGS. THE TRUTH IS THE LOG. THE DATABASE IS A CACHE OF A SUBSET OF THE LOG.**"*

### 💰 The accounting argument

```
   "Immutability in databases is AN OLD IDEA. Accountants have been using
    immutability FOR CENTURIES in financial bookkeeping. When a transaction
    occurs, it is recorded in AN APPEND-ONLY LEDGER."

   "If a mistake is made, ACCOUNTANTS DON'T ERASE OR CHANGE THE INCORRECT
    TRANSACTION — instead, THEY ADD ANOTHER TRANSACTION THAT COMPENSATES for
    the mistake… The incorrect transaction STILL REMAINS IN THE LEDGER
    FOREVER, because it might be important FOR AUDITING REASONS.
    THIS PROCESS IS ENTIRELY NORMAL IN ACCOUNTING."
```

### Three advantages of immutable events

```
   ① RECOVERY FROM BUGS — "if you accidentally deploy buggy code that writes
      bad data, RECOVERY IS MUCH HARDER IF THE CODE IS ABLE TO DESTRUCTIVELY
      OVERWRITE DATA."                        ← Ch.10's human fault tolerance

   ② THEY CAPTURE MORE INFORMATION THAN THE CURRENT STATE
      "a customer may ADD AN ITEM TO THEIR CART AND THEN REMOVE IT AGAIN.
       Although the second event CANCELS OUT the first from the point of view
       of ORDER FULFILLMENT, it may be useful to know FOR ANALYTICS PURPOSES
       that the customer WAS CONSIDERING A PARTICULAR ITEM BUT THEN DECIDED
       AGAINST IT… THIS INFORMATION WOULD BE LOST in a database that deletes
       items when they are removed from the cart."

   ③ SEVERAL VIEWS FROM ONE LOG → CQRS
      "if you want to introduce a NEW FEATURE that presents your existing data
       in some new way, you can use the event log to BUILD A SEPARATE
       READ-OPTIMIZED VIEW, and RUN IT ALONGSIDE THE EXISTING SYSTEMS WITHOUT
       HAVING TO MODIFY THEM. Once the old system is no longer needed, YOU CAN
       SIMPLY SHUT IT DOWN."

      ➜ COMMAND QUERY RESPONSIBILITY SEGREGATION (CQRS)

      🔑 "The traditional approach to database and schema design is based on
        THE FALLACY THAT DATA MUST BE WRITTEN IN THE SAME FORM AS IT WILL BE
        QUERIED. Debates about NORMALIZATION AND DENORMALIZATION BECOME
        LARGELY IRRELEVANT if you can translate data from a write-optimized
        event log to read-optimized application state."
```

**The Twitter timeline, revisited.** *"home timelines are HIGHLY DENORMALIZED, since your tweets are duplicated in all of the timelines of the people following you. However, the fan-out service KEEPS THIS DUPLICATED STATE IN SYNC, WHICH KEEPS THE DUPLICATION MANAGEABLE."* — Chapter 1's example, now recognizable as a materialized view maintained by a stream processor.

### ⚠️ Concurrency control and deletion — the two honest downsides

```
   ❌ READ-YOUR-WRITES: "consumers of the event log are usually ASYNCHRONOUS,
      so a user may make a write, then read from a log-derived view, and FIND
      THAT THEIR WRITE HAS NOT YET BEEN REFLECTED."   ← Ch.5, again
      FIXES: update the read view SYNCHRONOUSLY (needs a transaction across
      both, or both in one storage system), or use total order broadcast.

   ✅ BUT IT ALSO SIMPLIFIES CONCURRENCY: "Much of the need for MULTI-OBJECT
      TRANSACTIONS stems from a single user action requiring data to be
      changed in SEVERAL DIFFERENT PLACES. With event sourcing, you can design
      an event such that it is A SELF-CONTAINED DESCRIPTION OF A USER ACTION —
      so the user action requires ONLY A SINGLE WRITE IN ONE PLACE."
      And if log and state are PARTITIONED THE SAME WAY, "a straightforward
      SINGLE-THREADED log consumer NEEDS NO CONCURRENCY CONTROL FOR WRITES…
      THE LOG REMOVES THE NON-DETERMINISM OF CONCURRENCY BY DEFINING A SERIAL
      ORDER OF EVENTS IN A PARTITION."       ← Ch.7's actual serial execution

   ⚠️ TRULY DELETING DATA: privacy regulations may require it, and then "it's
      NOT SUFFICIENT to just append another event saying the prior data should
      be considered deleted — YOU ACTUALLY WANT TO REWRITE HISTORY AND PRETEND
      THAT THE DATA WAS NEVER WRITTEN."
      (Datomic calls this EXCISION; Fossil calls it SHUNNING.)
      "Truly deleting data is SURPRISINGLY HARD, since COPIES CAN LIVE IN MANY
       PLACES… **DELETION IS MORE A MATTER OF 'MAKING IT HARDER TO RETRIEVE
       THE DATA' THAN 'MAKING IT IMPOSSIBLE TO RETRIEVE THE DATA'.**"
```

---
---

# PART C — PROCESSING STREAMS

## 10. Three things you can do with a stream

```
   ① WRITE IT TO A DATABASE, cache, search index → "the streaming equivalent
      of what we discussed in 'The output of batch workflows'."
   ② PUSH IT TO USERS — email alerts, push notifications, a realtime
      dashboard. "In this case, A HUMAN IS THE ULTIMATE CONSUMER."
   ③ PROCESS INPUT STREAMS TO PRODUCE OUTPUT STREAMS   ← the rest of the chapter

   The code that does ③ is an OPERATOR or a JOB. "a stream processor consumes
   input streams IN A READ-ONLY FASHION, and writes its output to a different
   location IN AN APPEND-ONLY FASHION."   ← identical to MapReduce

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  🔑 "The ONE CRUCIAL DIFFERENCE to batch jobs is that A STREAM NEVER   ║
   ║  ENDS. This difference has MANY IMPLICATIONS:"                        ║
   ║                                                                       ║
   ║   • "SORTING DOES NOT MAKE SENSE with an unbounded dataset, and so    ║
   ║      SORT-MERGE JOINS DO NOT APPLY."                                  ║
   ║   • "with a batch job that has been running for A FEW MINUTES, a      ║
   ║      failed task can simply be RESTARTED FROM THE BEGINNING — but     ║
   ║      with a stream job that has been running FOR SEVERAL YEARS,       ║
   ║      RESTARTING FROM THE BEGINNING MAY NOT BE A VIABLE OPTION."       ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

## 11. Four uses of stream processing

### ① Complex event processing (CEP)

```
   Developed in the 1990s. "Similarly to the way that A REGULAR EXPRESSION
   allows you to search for certain patterns of CHARACTERS IN A STRING, CEP
   allows you to specify rules to search for certain PATTERNS OF EVENTS IN A
   STREAM."

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  🔑 THE RELATIONSHIP BETWEEN QUERIES AND DATA IS REVERSED:            ║
   ║                                                                       ║
   ║  DATABASE                      │  CEP ENGINE                          ║
   ║  ────────                      │  ──────────                          ║
   ║  data stored PERSISTENTLY      │  QUERIES stored LONG-TERM            ║
   ║  queries are TRANSIENT:        │  "events from the input streams       ║
   ║  "the database searches for    │   CONTINUOUSLY FLOW PAST THE QUERIES  ║
   ║   data matching the query,     │   in search of a query that matches   ║
   ║   AND THEN FORGETS ABOUT       │   the event."                         ║
   ║   THE QUERY."                  │                                      ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   USES: fraud detection · algorithmic trading · factory monitoring ·
   military/intelligence tracking.
   IMPLEMENTATIONS: Esper, IBM InfoSphere Streams, Apama, TIBCO StreamBase,
   SQLstream.
```

### ② Stream analytics

```
   "the boundary between CEP and stream analytics is BLURRY, but as a general
    rule, analytics tends to be LESS INTERESTED IN FINDING SPECIFIC EVENT
    SEQUENCES, and is MORE ORIENTED TOWARDS AGGREGATIONS AND STATISTICAL
    METRICS over a large number of events":
      • the RATE of some type of event
      • a ROLLING AVERAGE over some time period
      • COMPARING CURRENT STATISTICS TO PREVIOUS INTERVALS

   ⚑ PROBABILISTIC ALGORITHMS are common: BLOOM FILTERS (set membership),
     HYPERLOGLOG (cardinality estimation), percentile estimation.
     "they produce APPROXIMATE RESULTS, but require SIGNIFICANTLY LESS MEMORY."

   ⚠️ "This use of approximation algorithms SOMETIMES LEADS PEOPLE TO BELIEVE
      THAT STREAM PROCESSING SYSTEMS ARE ALWAYS LOSSY AND INEXACT, **BUT THAT
      IS WRONG: THERE IS NOTHING INHERENTLY APPROXIMATE ABOUT STREAM
      PROCESSING.**"

   FRAMEWORKS: Storm, Spark Streaming, Flink, Concord, Samza, Kafka Streams.
   HOSTED: Google Cloud Dataflow, Azure Stream Analytics.
```

### ③ Maintaining materialized views

```
   "deriving an ALTERNATIVE VIEW onto some dataset, so that you can query it
    efficiently, and UPDATING THAT VIEW WHENEVER THE UNDERLYING DATA CHANGES."

   ⚠️ THE KEY DIFFERENCE FROM ANALYTICS: "it is usually NOT SUFFICIENT to
      consider only events within some TIME WINDOW: building the materialized
      view requires ALL EVENTS THAT EVER HAPPENED… **IN EFFECT, YOU NEED A
      WINDOW THAT STRETCHES ALL THE WAY BACK TO THE BEGINNING OF TIME.**"

   "the need to maintain events forever RUNS COUNTER TO THE ASSUMPTIONS OF
    SOME ANALYTICS-ORIENTED FRAMEWORKS. Samza and Kafka Streams support this
    kind of usage, building upon KAFKA'S SUPPORT FOR LOG COMPACTION."
```

### ④ Search on streams

```
   "media monitoring services subscribe to feeds of news articles and search
    for any news mentioning companies, products or topics of interest… by
    FORMULATING A SEARCH QUERY IN ADVANCE, and then CONTINUALLY MATCHING the
    stream of news items against this query."

   🔑 "Conventional search engines FIRST INDEX THE DOCUMENTS AND THEN RUN
     QUERIES over the index. By contrast, searching a stream TURNS THE
     PROCESSING ON ITS HEAD: **THE QUERIES ARE STORED, AND THE DOCUMENTS RUN
     PAST THE QUERIES.**"

   "To optimize, it is possible to INDEX THE QUERIES AS WELL AS THE DOCUMENTS."
```

### 📦 Sidebar: why actors aren't stream processors

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  ACTOR FRAMEWORKS                │  STREAM PROCESSORS                   │
   ├──────────────────────────────────┼──────────────────────────────────────┤
   │  primarily a mechanism for       │  primarily A DATA MANAGEMENT         │
   │  MANAGING CONCURRENCY and        │  MECHANISM                           │
   │  distributed execution           │                                      │
   │  communication is EPHEMERAL      │  event logs are DURABLE and          │
   │  and ONE-TO-ONE                  │  MULTI-SUBSCRIBER                    │
   │  can communicate in ARBITRARY    │  usually set up in ACYCLIC PIPELINES │
   │  ways, INCLUDING CYCLIC          │  where every stream is the output of │
   │  request-response                │  one job                             │
   └──────────────────────────────────┴──────────────────────────────────────┘
   (Though Storm's DISTRIBUTED RPC blurs the line.)
```

---

## 12. Reasoning about time — the hardest part of the chapter

> *"It MIGHT SEEM that the meaning of 'LAST FIVE MINUTES' should be unambiguous and clear, but unfortunately **THE NOTION IS SURPRISINGLY TRICKY.**"*

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  EVENT TIME                       ║  PROCESSING TIME                  ║
   ║  the timestamp EMBEDDED IN THE    ║  the LOCAL SYSTEM CLOCK on the    ║
   ║  EVENT                            ║  processing machine               ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  What BATCH processes must use:   ║  "This approach has the advantage ║
   ║  "There is NO POINT in looking at ║   of being SIMPLE, and it is      ║
   ║   the system clock of the         ║   REASONABLE IF THE DELAY between ║
   ║   machines running the batch      ║   event creation and processing   ║
   ║   process, because THE TIME AT    ║   IS NEGLIGIBLY SHORT."           ║
   ║   WHICH THE PROCESS IS RUN HAS    ║                                   ║
   ║   NOTHING TO DO WITH THE TIME AT  ║  ❌ "IT BREAKS DOWN IF THERE IS    ║
   ║   WHICH THE EVENTS ACTUALLY       ║     ANY SIGNIFICANT PROCESSING    ║
   ║   OCCURRED."                      ║     LAG."                         ║
   ║                                   ║                                   ║
   ║  ✅ AND IT MAKES PROCESSING        ║                                   ║
   ║     DETERMINISTIC.                ║                                   ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝
```

### 🎬 The Star Wars analogy — the clearest explanation of the distinction

```
   RELEASE ORDER (processing time):  IV(1977) V(1980) VI(1983) I(1999)
                                     II(2002) III(2005) VII(2015)
   NARRATIVE ORDER (event time):     I II III IV V VI VII

   "If you watched the movies IN THE ORDER THEY CAME OUT, the order in which
    you PROCESSED the movies is INCONSISTENT WITH THE ORDER OF THEIR
    NARRATIVE. (The EPISODE NUMBER is like the EVENT TIMESTAMP, and the DATE
    WHEN YOU WATCHED the movie is the PROCESSING TIME.)

    **As humans, WE ARE ABLE TO COPE with such discontinuities, but stream
    processing algorithms NEED TO BE SPECIFICALLY WRITTEN to accommodate such
    timing and ordering issues.**"
```

### 🔷 Figure 11-7 — Windowing by processing time creates phantom spikes

```
                                                            TIME ──────────►
   Web server    ████████████████████████████████████████████████████████
                 (steady, constant rate of requests)

   Stream        ██████████████░░░░░░░░░░░░████████████████████████████████
   processor                   └ RESTARTING ┘

   RATE AS MEASURED BY PROCESSING TIME
      10 ┤                              ╭──╮
         │                              │  │  ← 💥 PHANTOM SPIKE
         │─────────────╮                │  │     (the backlog, processed
       0 ┤             ╰────────────────╯  ╰──   all at once)
                        └ nothing, while down ┘

   ACTUAL REQUEST RATE
      10 ┤──────────────────────────────────────────
         │                                              ← perfectly steady
       0 ┤

   "If you measure the rate based on the PROCESSING TIME, IT WILL LOOK AS IF
    THERE WAS A SUDDEN ANOMALOUS SPIKE OF REQUESTS while processing the
    backlog, WHEN IN FACT THE REAL RATE OF REQUESTS WAS STEADY."
```

### ⏰ Knowing when you're ready — stragglers and watermarks

> *"A tricky problem when defining windows in terms of event time is that **YOU CAN NEVER BE SURE WHEN YOU HAVE RECEIVED ALL OF THE EVENTS for a particular window**, or whether there are some events still to come."*

```
   THE SCENARIO: you're counting requests per minute. You have events for the
   37th minute; now most incoming events are in the 38th and 39th.
   **WHEN DO YOU DECLARE THE 37TH MINUTE FINISHED, AND OUTPUT ITS COUNTER?**

   "In general, IT'S IMPOSSIBLE TO BE SURE. You can TIME OUT and declare a
    window ready after you have not seen any new events for a while, BUT IT
    COULD STILL HAPPEN THAT SOME EVENTS WERE BUFFERED ON ANOTHER MACHINE
    somewhere, which couldn't yet be sent due to a network interruption."

   ┌──────────────────────────────────────────────────────────────────────┐
   │  TWO OPTIONS FOR STRAGGLERS:                                         │
   │                                                                      │
   │  ① IGNORE THEM — "they are probably A SMALL PERCENTAGE in normal     │
   │     circumstances. TRACK THE NUMBER OF DROPPED EVENTS AS A METRIC,   │
   │     AND ALERT if you start dropping a significant amount of data."   │
   │                                                                      │
   │  ② RECALCULATE and ISSUE A CORRECTION — "publishing the updated      │
   │     value (POSSIBLY RETRACTING THE PREVIOUS OUTPUT FIRST)."          │
   └──────────────────────────────────────────────────────────────────────┘

   A LOW WATERMARK indicates "FROM NOW ON THERE WILL BE NO MORE MESSAGES WITH
   A TIMESTAMP EARLIER THAN t", and consumers wait for it.
   ⚠️ "However, IF TIMESTAMPS ARE GENERATED BY CLIENTS, you cannot be sure
      whether there are still any pending events somewhere, SO STRAGGLERS ARE
      STILL POSSIBLE."
```

### 📱 Whose clock are you using? — the three-timestamp trick

```
   THE PROBLEM: "a mobile app may be used WHILE OFFLINE, buffering events
   locally and sending them HOURS OR DAYS LATER. To any consumers of this
   stream, the events will appear as EXTREMELY DELAYED STRAGGLERS."

   "the timestamp SHOULD really be the time at which THE USER INTERACTION
    OCCURRED, according to the DEVICE'S local clock. However, **THE CLOCK ON A
    USER-CONTROLLED DEVICE OFTEN CANNOT BE TRUSTED**, as it may be
    accidentally or DELIBERATELY set to the wrong time."   ← Chapter 8

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE FIX — LOG THREE TIMESTAMPS:                                      ║
   ║    ① the time the EVENT OCCURRED,      per the DEVICE clock           ║
   ║    ② the time the event was SENT,      per the DEVICE clock           ║
   ║    ③ the time the event was RECEIVED,  per the SERVER clock           ║
   ║                                                                       ║
   ║  ➜ ③ − ② ESTIMATES THE CLOCK OFFSET between device and server         ║
   ║    (assuming network delay is negligible), so you can SUBTRACT IT     ║
   ║    FROM ① to get the true time the event occurred.                    ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ⚑ "This problem is NOT UNIQUE to stream processing — BATCH PROCESSING
     SUFFERS FROM EXACTLY THE SAME ISSUE. It is just MORE NOTICEABLE in a
     streaming context, where WE ARE MORE AWARE OF THE PASSAGE OF TIME."
```

### 🔷 The four types of window

```
   ① TUMBLING — fixed length, EVERY EVENT BELONGS TO EXACTLY ONE WINDOW
      │───1min───│───1min───│───1min───│───1min───│
      10:03      10:04      10:05      10:06      10:07
      "implement by taking each event timestamp and ROUNDING IT TO THE
       NEAREST MINUTE."

   ② HOPPING — fixed length, WINDOWS OVERLAP, "in order to provide some
      SMOOTHING." A 5-min window with a 1-min hop:
      │─────────5 min─────────│
           │─────────5 min─────────│
                │─────────5 min─────────│
      10:03  10:04  10:05 …
      "implement by first calculating 1-MINUTE TUMBLING WINDOWS, and then
       AGGREGATING OVER SEVERAL ADJACENT WINDOWS."

   ③ SLIDING — "contains ALL THE EVENTS THAT OCCUR WITHIN SOME INTERVAL OF
      EACH OTHER" — NO FIXED BOUNDARIES.
      events at 10:03:39 and 10:08:12 ARE in the same 5-min sliding window
      (they're <5 min apart) but would NOT be in the same tumbling or
      hopping window.
      "implement by keeping A BUFFER OF EVENTS SORTED BY TIME, and REMOVING
       OLD EVENTS when they expire."

   ④ SESSION — **NO FIXED DURATION.**
      │──events──│           │─events─│        │──events──│
      └ session ─┘  30 min   └session─┘ 30min  └─session──┘
                    idle                idle
      "grouping together all events FOR THE SAME USER that occur CLOSELY
       TOGETHER IN TIME, and the window ENDS WHEN THE USER HAS BEEN INACTIVE
       FOR SOME TIME."
      ⚑ SESSIONIZATION — the streaming version of Ch.10's GROUP BY use case.
```

---

## 13. Stream joins — three kinds

> *"Since stream processing GENERALIZES DATA PIPELINES TO INCREMENTAL PROCESSING of unbounded datasets, there is EXACTLY THE SAME NEED FOR JOINS. However, the fact that NEW EVENTS CAN APPEAR ANYTIME makes joins on streams MORE CHALLENGING."*

### ① Stream-stream join (window join)

```
   THE EXAMPLE: measuring SEARCH RESULT CLICK-THROUGH RATE. You log a SEARCH
   event and, if the user clicks, a CLICK event — connected by SESSION ID.

   ┌──────────────────────────────────────────────────────────────────────┐
   │   search stream   ──●───────────●──────────●──────────────────       │
   │                      ╲           ╲          ╲                        │
   │                       ╲  join     ╲ join     ╲ NO CLICK EVER COMES   │
   │                        ▼           ▼                                 │
   │   click stream    ──────●───────────●────────────────────────────    │
   │                                                                      │
   │   ⚠️ "the time between the search and the click may be HIGHLY          │
   │      VARIABLE: in many cases A FEW SECONDS, but it could be AS LONG  │
   │      AS DAYS OR WEEKS (if a user runs a search, FORGETS ABOUT THAT   │
   │      BROWSER TAB, and then returns and clicks a result later)."      │
   │   ⚠️ "Due to variable network delays, THE CLICK EVENT MAY EVEN ARRIVE │
   │      BEFORE THE SEARCH EVENT."                                       │
   └──────────────────────────────────────────────────────────────────────┘

   🔑 WHY YOU CAN'T JUST EMBED THE SEARCH DETAILS IN THE CLICK EVENT:
     "that would ONLY TELL YOU ABOUT THE CASES WHERE THE USER CLICKED a search
      result, BUT NOT ABOUT THE SEARCHES WHERE THE USER DID NOT CLICK ANY of
      the results. In order to measure search quality you need ACCURATE
      CLICK-THROUGH RATES, for which YOU NEED BOTH."

   IMPLEMENTATION: "the stream processor needs to MAINTAIN STATE: all the
   events that occurred in the last hour, INDEXED BY SESSION ID. Whenever a
   search or click event occurs, it is ADDED TO THE APPROPRIATE INDEX, and it
   ALSO CHECKS THE OTHER INDEX to see if another event for the same session ID
   has already arrived."
```

### ② Stream-table join (stream enrichment)

```
   Chapter 10's Figure 10-2 join, done continuously.

   ┌──────────────────────────────────────────────────────────────────────┐
   │  activity events ──────────►┌───────────────────┐────► ENRICHED      │
   │  (user_id, url)             │ STREAM PROCESSOR  │      activity      │
   │                             │  ┌─────────────┐  │      events        │
   │  CDC changelog ────────────►│  │ LOCAL COPY  │  │      (user_id,     │
   │  of the profiles DB         │  │ OF THE DB   │  │       url, dob)    │
   │                             │  └─────────────┘  │                    │
   │                             └───────────────────┘                    │
   └──────────────────────────────────────────────────────────────────────┘

   ❌ "querying a REMOTE DATABASE is likely to be SLOW, and RISKS OVERLOADING
      the remote database."             ← identical to Ch.10's argument
   ✅ "load A COPY OF THE DATABASE INTO THE STREAM PROCESSOR, so that it can
      be queried LOCALLY WITHOUT A NETWORK ROUND-TRIP. This is VERY SIMILAR TO
      THE HASH JOINS we discussed in Map-side joins."

   🔑 THE DIFFERENCE FROM BATCH: "a batch job uses A POINT-IN-TIME SNAPSHOT,
     whereas a stream processor is LONG-RUNNING, and the contents of the
     database is LIKELY TO CHANGE OVER TIME, so the local copy NEEDS TO BE
     KEPT UP-TO-DATE. **THIS IS SOLVED BY CHANGE DATA CAPTURE**: the stream
     processor SUBSCRIBES TO A CHANGELOG of the profiles database AS WELL AS
     the stream of activity events."
```

### ③ Table-table join (materialized view maintenance)

```
   The Twitter timeline cache, as a stream job. FOUR EVENT TYPES TO HANDLE:

   ┌──────────────────────────────────────────────────────────────────────┐
   │  • user u SENDS A TWEET      → add to the timeline of EVERY FOLLOWER │
   │  • user DELETES a tweet      → remove from ALL users' timelines      │
   │  • u1 STARTS FOLLOWING u2    → add u2's RECENT TWEETS to u1's timeline│
   │  • u1 UNFOLLOWS u2           → REMOVE u2's tweets from u1's timeline │
   └──────────────────────────────────────────────────────────────────────┘

   "Another way of looking at this is that IT MAINTAINS A MATERIALIZED VIEW
    for a query that JOINS TWO TABLES":

      SELECT follows.follower_id AS timeline_id,
        array_agg(tweets.* ORDER BY tweets.timestamp DESC)
      FROM tweets
      JOIN follows ON follows.followee_id = tweets.sender_id
      GROUP BY follows.follower_id;

   "THE JOIN OF THE STREAMS CORRESPONDS DIRECTLY TO THE JOIN OF THE TABLES in
    that query. The timelines are effectively A CACHE OF THE RESULT OF THIS
    QUERY, UPDATED EVERY TIME THE UNDERLYING TABLES CHANGE."

   📖 THE LOVELIEST FOOTNOTE IN THE BOOK:
      "If you regard a stream as THE DERIVATIVE OF A TABLE, and regard a join
       as A PRODUCT of two tables u·v, something interesting happens: the
       stream of changes to the materialized join FOLLOWS THE PRODUCT RULE
                          (u·v)′ = u′v + uv′
       In words: WHENEVER THE TWEETS CHANGE, IT IS JOINED WITH THE CURRENT
       FOLLOWS, AND WHENEVER THE FOLLOWS CHANGE, IT IS JOINED WITH THE CURRENT
       TWEETS."
```

### ⚠️ Time-dependence of joins — the subtle killer

```
   "The order of the events that maintain the state IS IMPORTANT (it matters
    whether you FIRST FOLLOW AND THEN UNFOLLOW, or the other way round). In a
    partitioned log, the ordering WITHIN A SINGLE PARTITION is preserved, but
    **THERE IS TYPICALLY NO ORDERING GUARANTEE ACROSS DIFFERENT STREAMS OR
    PARTITIONS.**"

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE QUESTION: "if a user UPDATES THEIR PROFILE, which activity       ║
   ║  events are joined with THE OLD PROFILE, and which with THE NEW?      ║
   ║  Put another way: **IF STATE CHANGES OVER TIME, AND YOU JOIN WITH     ║
   ║  SOME STATE, WHAT POINT IN TIME DO YOU USE FOR THE JOIN?**"           ║
   ║                                                                       ║
   ║  ⚠️ "If the ordering across streams is UNDETERMINED, THE JOIN BECOMES  ║
   ║     NON-DETERMINISTIC, which means YOU CANNOT RE-RUN THE SAME JOB ON  ║
   ║     THE SAME INPUT AND NECESSARILY GET THE SAME RESULT."              ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 14. Fault tolerance and exactly-once semantics

> *"Batch processing… ensures that the output is THE SAME AS IF NOTHING HAD GONE WRONG. It APPEARS AS THOUGH every input record was processed EXACTLY ONCE… Although restarting tasks means that records MAY IN FACT BE PROCESSED MULTIPLE TIMES, **the VISIBLE EFFECT in the output is AS IF they had only been processed once**, a principle known as EXACTLY-ONCE SEMANTICS."*
>
> ⚠️ *"The same issue arises in stream processing, but IT IS LESS STRAIGHTFORWARD: **waiting until a task is finished before making its output visible IS NOT AN OPTION, BECAUSE A STREAM IS INFINITE AND SO YOU CAN NEVER FINISH PROCESSING IT.**"*

### ① Microbatching and checkpointing

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  MICROBATCHING (Spark Streaming) │  CHECKPOINTING (Flink)               │
   ├──────────────────────────────────┼──────────────────────────────────────┤
   │  "break the stream into SMALL    │  "periodically generates ROLLING     │
   │   BLOCKS, and treat each block   │   CHECKPOINTS of state and writes    │
   │   like A MINIATURE BATCH         │   them to DURABLE STORAGE. If a      │
   │   PROCESS."                      │   stream operator crashes, it can    │
   │                                  │   RESTART FROM ITS MOST RECENT       │
   │  Batch size ≈ 1 SECOND — "a      │   CHECKPOINT, and DISCARD ANY OUTPUT │
   │  PERFORMANCE COMPROMISE: SMALLER │   generated between the last         │
   │  batches incur GREATER           │   checkpoint and the crash."         │
   │  SCHEDULING AND COORDINATION     │                                      │
   │  OVERHEAD, while LARGER batches  │  Triggered by BARRIERS in the        │
   │  mean A LONGER DELAY."           │  message stream — "similar to the    │
   │                                  │  boundaries between microbatches,    │
   │  ⚑ IMPLICITLY provides a         │  BUT WITHOUT FORCING A PARTICULAR    │
   │    TUMBLING WINDOW equal to the  │  WINDOW SIZE."                       │
   │    batch size, WINDOWED BY       │                                      │
   │    PROCESSING TIME.              │                                      │
   └──────────────────────────────────┴──────────────────────────────────────┘

   ⚠️ THE LIMIT OF BOTH: "as soon as OUTPUT LEAVES THE STREAM PROCESSOR (by
      writing to a database, sending messages to an external broker, OR
      SENDING EMAILS), the framework IS NO LONGER ABLE TO DISCARD THE OUTPUT
      of a failed batch. Restarting a failed task causes the external
      side-effect TO HAPPEN TWICE."
```

### ② Atomic commit revisited

```
   "we need to ensure that ALL OUTPUTS AND SIDE-EFFECTS of processing an event
    TAKE EFFECT IF AND ONLY IF THE PROCESSING IS SUCCESSFUL."

   ┌──────────────────────────────────────────────────────────────────────┐
   │  THE FULL LIST OF THINGS THAT MUST COMMIT ATOMICALLY:                │
   │    • messages sent to DOWNSTREAM OPERATORS                           │
   │    • messages to EXTERNAL MESSAGING SYSTEMS (including email/push)   │
   │    • any DATABASE WRITES                                             │
   │    • any changes to OPERATOR STATE                                   │
   │    • any ACKNOWLEDGEMENT OF INPUT MESSAGES — including MOVING THE    │
   │      CONSUMER OFFSET FORWARD                    ← easy to forget     │
   └──────────────────────────────────────────────────────────────────────┘

   "If this sounds familiar, it is because we discussed it in EXACTLY-ONCE
    MESSAGE PROCESSING in the context of DISTRIBUTED TRANSACTIONS AND 2PC."

   ✅ "in MORE RESTRICTED ENVIRONMENTS it is possible to implement such an
      atomic commit facility EFFICIENTLY… The approach relies on WRITING THE
      TRANSACTION COMMIT AS A SINGLE OBJECT to a fault-tolerant datastore,
      since A SINGLE-OBJECT WRITE CAN BE MADE ATOMIC FAIRLY EASILY."
      (Used in Google Cloud Dataflow; "plans to add similar features to Kafka.")
```

### ③ Idempotence — the pragmatic alternative

```
   "An IDEMPOTENT operation is one that you can perform MULTIPLE TIMES, and it
    has THE SAME EFFECT AS IF YOU PERFORMED IT ONLY ONCE."

   ✅ setting a key to a FIXED VALUE   ❌ INCREMENTING a counter

   🔑 THE TRICK: "Even if an operation is NOT NATURALLY IDEMPOTENT, it can
     often BE MADE IDEMPOTENT WITH A BIT OF EXTRA METADATA. When consuming
     from Kafka, every message has a PERSISTENT, MONOTONICALLY INCREASING
     OFFSET. When writing a value to an external database, **YOU CAN INCLUDE
     THE OFFSET OF THE MESSAGE THAT TRIGGERED THE LAST WRITE WITH THE VALUE.**
     Thus, YOU CAN TELL WHETHER AN UPDATE HAS ALREADY BEEN APPLIED."

        UPDATE counters
        SET value = 42, last_offset = 1234
        WHERE key = 'foo' AND last_offset < 1234;   ← the whole technique

   ⚠️ THE ASSUMPTIONS IT RELIES ON:
      • "restarting a failed task must REPLAY THE SAME MESSAGES IN THE SAME
        ORDER (A LOG-BASED MESSAGE BROKER DOES THIS)"
      • "the processing must be DETERMINISTIC"
      • "NO OTHER NODE MAY CONCURRENTLY UPDATE THE SAME VALUE"
      • "FENCING may be required to prevent interference from a node that is
        THOUGHT TO BE DEAD BUT IS ACTUALLY ALIVE"     ← Chapter 8's tokens

   ➜ "Despite all those caveats, idempotent operations can be AN EFFECTIVE WAY
     of achieving exactly-once semantics WITH ONLY A SMALL OVERHEAD."
```

### ④ Rebuilding state after a failure

```
   ❌ "keep the state in a REMOTE DATASTORE and replicate it" — "having to
      query a remote database for EACH INDIVIDUAL MESSAGE CAN BE SLOW."
   ✅ "keep state LOCAL to the stream processor, and REPLICATE IT
      PERIODICALLY."
      • FLINK: "periodically captures SNAPSHOTS of operator state and writes
        them to DURABLE STORAGE such as HDFS"
      • SAMZA and KAFKA STREAMS: "replicate state changes by SENDING THEM TO A
        DEDICATED KAFKA TOPIC WITH LOG COMPACTION, SIMILAR TO CHANGE DATA
        CAPTURE"      ← the chapter's own CDC idea, turned inward

   💡 SOMETIMES YOU DON'T NEED TO REPLICATE AT ALL:
      • "if the state consists of aggregations over A FAIRLY SHORT WINDOW, it
        may be FAST ENOUGH TO SIMPLY REPLAY THE INPUT EVENTS."
      • "if the state is a LOCAL REPLICA OF A DATABASE maintained by CDC, the
        database can ALSO BE REBUILT FROM THE LOG-COMPACTED CHANGE STREAM."
```

---

## 15. Chapter Summary

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "In some ways, stream processing is VERY MUCH LIKE THE BATCH         ║
   ║   PROCESSING of Chapter 10, but done CONTINUOUSLY on an UNBOUNDED     ║
   ║   stream rather than on a fixed-size input. From this perspective,    ║
   ║   **MESSAGE BROKERS AND EVENT LOGS SERVE AS THE STREAMING EQUIVALENT  ║
   ║   OF A FILESYSTEM.**"                                                 ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### The two broker types

| | **AMQP/JMS-style** | **Log-based** |
|---|---|---|
| **Assignment** | Broker assigns **individual messages** to consumers | Broker assigns **all messages in a partition** to the same consumer node |
| **Ordering** | Not preserved under load balancing + redelivery | **Always delivers in the same order** |
| **Progress** | Per-message **acknowledgements**; messages deleted once acked | Consumers **checkpoint the offset** of the last message processed |
| **Replay** | ❌ Destructive | ✅ **Messages retained on disk — jump back and re-read** |
| **Use when** | Messages are **expensive to process**, order doesn't matter, no need to re-read — *"an asynchronous form of RPC, for example in a task queue"* | **High throughput**, each message fast to process, **order matters**, derived state |

### The three joins

| Join | Inputs | What it does |
|---|---|---|
| **Stream-stream** | Two activity streams | *"Matching two events that occur within some window of time"* |
| **Stream-table** | Activity events + a **database changelog** | The changelog keeps a **local copy** up-to-date; activity events query it and emit **enriched** events |
| **Table-table** | Two **database changelogs** | *"every change on one side is joined with the LATEST STATE of the other side. The result is a stream of changes to the MATERIALIZED VIEW of the two tables"* |

### And the closing thought

> *"Representing databases as streams opens up POWERFUL OPPORTUNITIES FOR INTEGRATING SYSTEMS. You can keep derived data systems CONTINUALLY UP-TO-DATE by consuming the log of changes… **You can even build FRESH VIEWS onto existing data by STARTING FROM SCRATCH and consuming the log of changes FROM THE BEGINNING ALL THE WAY TO THE PRESENT.**"*

---
---

# 16. 🎁 WHAT'S CHANGED SINCE 2017

This chapter has aged **remarkably well** — better than Chapter 10. The reason is that Kleppmann wasn't describing a product landscape; he was describing an architectural idea (the log as the integration backbone) that turned out to be right.

## 16.1 Kafka delivered on almost everything the chapter hoped for

```
   The chapter says "there are PLANS to add similar features [atomic commit]
   to Apache Kafka." Those plans shipped, and then some:

   ✅ EXACTLY-ONCE SEMANTICS (Kafka 0.11, late 2017; production-hardened in
      2.5+). Built from exactly the ingredients the chapter describes:
        • IDEMPOTENT PRODUCER — sequence numbers per producer/partition
          deduplicate retries at the broker
        • TRANSACTIONS — atomically commit {messages produced + consumer
          offsets advanced}, which is precisely the chapter's list of things
          that must commit together
        • PRODUCER EPOCHS — a FENCING TOKEN that stops a zombie producer after
          a failover                          ← Chapter 8's fix, again
      ➜ `processing.guarantee=exactly_once_v2` in Kafka Streams is a one-line
        config for what this chapter presents as an open problem.

   ✅ KRAFT (2022 production-ready; ZooKeeper REMOVED entirely in Kafka 4.0,
      2025). Kafka runs its own Raft quorum for metadata — see Chapter 9.

   ✅ TIERED STORAGE (KIP-405, GA in Kafka 3.9/4.0). This one matters for this
      chapter specifically: the book's "11 hours to fill a 6 TB drive"
      calculation assumed local disk bounded the retention window. With cold
      segments offloaded to object storage, **INFINITE RETENTION BECAME
      PRACTICAL.** The book's aspiration that a log can be "used for DURABLE
      STORAGE, NOT JUST TRANSIENT MESSAGING" is now the default assumption.

   🆕 DISKLESS / S3-NATIVE BROKERS — WarpStream, AutoMQ, Bufstream, and
      KIP-1150 push this further: brokers with NO LOCAL DISK AT ALL, writing
      straight to object storage. Trades latency (hundreds of ms) for
      dramatically lower cost and zero rebalancing pain.

   🆕 REDPANDA — a C++ Kafka-protocol-compatible broker, no JVM, no
      ZooKeeper, thread-per-core. Competes on tail latency.
   🆕 APACHE PULSAR — separates serving from storage (BookKeeper), and
      natively supports BOTH broker styles the chapter contrasts: queue
      semantics (shared subscriptions) AND log semantics, in one system.
      That's a genuine answer to the chapter's "pick one" framing.
```

## 16.2 Flink won stream processing

```
   The chapter lists "Storm, Spark Streaming, Flink, Concord, Samza, Kafka
   Streams" as roughly co-equal. Today:

   • FLINK is the default choice for serious stateful streaming. Its
     checkpointing model (the one the chapter describes) proved the right
     abstraction, and it got much better: INCREMENTAL CHECKPOINTS, UNALIGNED
     CHECKPOINTS, RocksDB state backend, and now DISAGGREGATED STATE (Flink
     2.0, 2025) storing state in object storage.
   • FLINK SQL made windows and joins declarative — you write
     `TUMBLE(...)`, `HOP(...)`, `SESSION(...)` and `FOR SYSTEM_TIME AS OF`
     for temporal joins, rather than hand-coding state management.
   • STORM, SAMZA, CONCORD: effectively legacy.
   • SPARK STRUCTURED STREAMING replaced the DStream microbatching described
     here, and added CONTINUOUS PROCESSING mode; still strongest when you're
     already in a Spark shop.
   • KAFKA STREAMS remains excellent for library-style, no-cluster-needed
     stream processing embedded in a normal JVM app.

   🆕 APACHE BEAM / THE DATAFLOW MODEL deserves special credit: the paper the
      chapter cites as reference [1] became the industry's shared vocabulary.
      WATERMARKS, TRIGGERS, ALLOWED LATENESS and ACCUMULATION MODE are now
      standard concepts in Flink, Beam and Spark alike. The chapter's "two
      options for stragglers" (drop them, or retract and re-emit) became a
      formal, configurable dimension.
```

## 16.3 CDC became infrastructure, not a research topic

```
   The chapter describes CDC as an emerging idea with a list of bespoke tools.
   It's now a standard building block:

   • DEBEZIUM is the de facto standard (Postgres, MySQL, MongoDB, SQL Server,
     Oracle, Db2, Cassandra), and Debezium Server can run without Kafka
     Connect. Bottled Water, Mongoriver and Maxwell are historical footnotes.
   • DATABASES ADDED FIRST-CLASS CHANGE STREAMS, which the chapter predicted:
     MongoDB CHANGE STREAMS (3.6), Postgres LOGICAL REPLICATION + pgoutput,
     MySQL binlog tooling, DynamoDB Streams, Cosmos DB change feed, and
     Snowflake/Databricks CHANGE DATA FEED.
   • THE TRANSACTIONAL OUTBOX PATTERN became the standard answer to the
     dual-writes problem in Figure 11-4 — write the business row AND an
     outbox row in ONE LOCAL TRANSACTION, then let CDC publish the outbox.
     This is the practical, XA-free resolution the chapter gestures toward.

   🆕 ZERO-ETL is the marketing term for managed CDC pipelines (Aurora→Redshift,
      Snowflake/Databricks connectors). Same idea, fewer moving parts to run.
```

## 16.4 Incremental view maintenance became a product category

```
   The chapter's "maintaining materialized views" section describes something
   the tools of 2017 could only approximate. It's now its own category:

   • MATERIALIZE — incremental view maintenance from the Differential Dataflow
     research line. You write ordinary SQL (including multi-way joins and
     recursion); it maintains the answer incrementally as inputs change.
   • RISINGWAVE — similar, Postgres-wire-compatible, cloud-native.
   • FELDERA / DBSP — a formal theory of incremental computation, with a
     proof that any query can be incrementalized.
   • CLICKHOUSE MATERIALIZED VIEWS, SNOWFLAKE DYNAMIC TABLES, DATABRICKS
     MATERIALIZED VIEWS — the warehouses grew the feature too.

   🔑 WHY THIS MATTERS FOR THE CHAPTER: the product rule footnote —
     (u·v)′ = u′v + uv′ — is literally the mathematics these systems
     implement. What the book presents as a charming aside turned out to be
     the foundation of a whole product category.
```

## 16.5 Kappa vs Lambda — a debate that resolved

```
   The book predates the clear articulation of this, but it's the natural
   next question after this chapter:

   LAMBDA ARCHITECTURE (Nathan Marz): run a BATCH layer for correctness and a
   SPEED layer for freshness, then merge at query time.
      ❌ You maintain TWO IMPLEMENTATIONS of the same logic, forever.

   KAPPA ARCHITECTURE (Jay Kreps, 2014): just the stream layer. Need to
   recompute? REPLAY THE LOG FROM THE BEGINNING into a new view, then switch
   over — which is exactly the "replaying old messages" capability this
   chapter describes in §4.

   ➜ 2026 VERDICT: Kappa's *reasoning* won (one codebase, replay to recompute),
     but in practice most organizations run a "STREAMING LAKEHOUSE": Flink or
     Spark writing into Iceberg/Delta tables, where the SAME TABLE serves both
     streaming and batch consumers. The layers merged rather than one winning.
```

## 16.6 The immutability-vs-privacy tension got much sharper

```
   The chapter's §9 warning — "deletion is more a matter of MAKING IT HARDER
   to retrieve the data" — was written months before GDPR came into force
   (May 2018), and was then joined by CCPA and others.

   🆕 THE PRACTICAL ANSWER: CRYPTO-SHREDDING. Encrypt each subject's personal
      data with a PER-SUBJECT KEY, store keys in a separate mutable keystore,
      and on a deletion request DESTROY THE KEY. The immutable log keeps its
      ciphertext, which is now permanently unreadable.
      ┌──────────────────────────────────────────────────────────────────┐
      │  immutable log:  [ …encrypted with key_K… ]  ← unchanged         │
      │  keystore:       key_K  ───► DELETED         ← the only mutation │
      │  result: the event still exists, but IS NO LONGER PERSONAL DATA  │
      └──────────────────────────────────────────────────────────────────┘
   🆕 Table formats (Iceberg/Delta) also added row-level DELETE specifically
      so that lakes could comply — see Chapter 10's §16.
```

## 16.7 What has aged perfectly

```
   ✅ THE TWO BROKER STYLES and the decision rule between them. Unchanged, and
      still the first question to ask.
   ✅ FIGURE 11-4 (dual writes). Still the single best argument for CDC, and
      still a mistake teams make weekly.
   ✅ EVENT TIME vs PROCESSING TIME, the Star Wars analogy, and Figure 11-7's
      phantom spike. This is now standard vocabulary — the chapter taught it
      before the industry had settled on the terms.
   ✅ THE FOUR WINDOW TYPES. Exactly what Flink SQL and Beam implement.
   ✅ THE THREE JOIN TYPES. Exactly how Flink and Kafka Streams categorize them.
   ✅ "state is the INTEGRAL of a stream; a stream is the DERIVATIVE of state."
      Still the single most useful sentence for thinking about this material.
   ✅ THE IDEMPOTENCE-WITH-OFFSET trick. Still the cheapest route to
      effectively-exactly-once when you write to an external system.
   ✅ "THERE IS NOTHING INHERENTLY APPROXIMATE ABOUT STREAM PROCESSING."
      A misconception that still needs correcting.

   ⚠️ ONE THING THAT AGED OSTENTATIOUSLY: the phrase "exactly-once" itself
      became contested. The consensus refinement is that you get EXACTLY-ONCE
      *STATE UPDATES* within a closed system, and AT-LEAST-ONCE DELIVERY plus
      IDEMPOTENCE at the edges — which is, in fairness, precisely what the
      chapter actually describes. It's the marketing that overreached, not
      the book.
```

## 16.8 A 2026 decision table

```
   ┌────────────────────────────────────┬────────────────────────────────────┐
   │  YOU NEED…                         │  REACH FOR                         │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  a durable, replayable event log   │  KAFKA (or Redpanda / WarpStream    │
   │                                    │  for cost/latency variants)        │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  a task queue, order irrelevant    │  RabbitMQ / SQS — NOT a log        │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  keep a search index / cache in    │  DEBEZIUM CDC → Kafka → consumer.  │
   │  sync with a database              │  Never dual writes.                │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  write DB + publish event          │  TRANSACTIONAL OUTBOX + CDC        │
   │  atomically                        │                                    │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  complex stateful streaming, event │  FLINK (SQL if you can, DataStream │
   │  time, large state                 │  if you must)                      │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  simple streaming inside an app    │  KAFKA STREAMS — no cluster needed │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  an always-fresh SQL view          │  MATERIALIZE / RisingWave, or your │
   │                                    │  warehouse's dynamic tables        │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  one table for both stream and     │  ICEBERG or DELTA, written by Flink│
   │  batch consumers                   │  ("streaming lakehouse")           │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  GDPR deletion over an immutable   │  CRYPTO-SHREDDING (+ row-level     │
   │  log                               │  deletes in the table format)      │
   └────────────────────────────────────┴────────────────────────────────────┘
```

*(As with earlier chapters: this reflects developments through my knowledge cutoff, and the area moves quickly — treat specific products, versions and claims as a starting point to verify.)*

---

# 17. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.11 — STREAM PROCESSING                                               ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  Batch assumes BOUNDED input. Reality is UNBOUNDED — "unless you go out of     ║
║  business, this process NEVER ENDS." Batch chunks it artificially; streaming   ║
║  abandons the chunks. EVENT = small, self-contained, IMMUTABLE, timestamped.   ║
║  PRODUCER → TOPIC → CONSUMER(S). Polling is expensive → push notifications.    ║
║                                                                               ║
║  ── MESSAGING ─────────────────────────────────────────────────────────────── ║
║  TWO CLASSIFYING QUESTIONS: ① producers faster than consumers? → DROP, BUFFER, ║
║  or BACKPRESSURE (Unix pipes/TCP do the last). ② nodes crash? → durability     ║
║  costs throughput. Metrics can tolerate loss; COUNTERS CANNOT.                 ║
║  DIRECT MESSAGING (UDP multicast, ZeroMQ, StatsD, WEBHOOKS): assumes producer  ║
║  AND consumer are ALWAYS ONLINE; the app must handle loss itself.              ║
║  MESSAGE BROKER = "A DATABASE OPTIMIZED FOR MESSAGE STREAMS." vs a real DB:    ║
║  deletes on delivery · assumes SHORT queues · pattern subscriptions instead of ║
║  indexes · NOTIFIES on change (a DB gives you a snapshot and never tells you   ║
║  it went stale).                                                              ║
║  LOAD BALANCING (one message → ONE consumer) vs FAN-OUT (→ ALL). Combinable.   ║
║  ⚠️ LOAD BALANCING + REDELIVERY ⇒ REORDERING, INEVITABLY (Fig 11-2), even       ║
║     though JMS and AMQP both promise ordering.                                ║
║                                                                               ║
║  ── PARTITIONED LOGS (Kafka, Kinesis, DistributedLog) ─────────────────────── ║
║  MINDSET SHIFT: messaging is TRANSIENT, databases are PERMANENT. Consuming     ║
║  from a queue is DESTRUCTIVE → you can't re-run and get the same result, and   ║
║  a NEW consumer sees NOTHING from before it registered.                       ║
║  THE LOG: append-only, PARTITIONED, each partition TOTALLY ORDERED by OFFSET.  ║
║  NO ORDERING ACROSS PARTITIONS. Broker ≈ LEADER, consumer ≈ FOLLOWER;          ║
║  the offset ≈ the LOG SEQUENCE NUMBER of Ch.5.                                ║
║  ✅ fan-out is free · offsets replace per-message acks (less bookkeeping)      ║
║  ❌ parallelism ≤ PARTITION COUNT · one slow message blocks the partition      ║
║     (HEAD-OF-LINE BLOCKING)                                                   ║
║  RULE: expensive messages + order unimportant → JMS/AMQP. High throughput +    ║
║  fast messages + ORDER MATTERS → LOG.                                         ║
║  DISK: a 6 TB drive at 150 MB/s = 11 HOURS to fill → days/weeks in practice.   ║
║  A CIRCULAR BUFFER ON DISK. Monitor consumer lag; a slow consumer HURTS ONLY   ║
║  ITSELF. 🔑 REPLAY: consuming is READ-ONLY, offsets are client-controlled →     ║
║  re-run yesterday's data with new code into a new location. BATCH-LIKE         ║
║  HUMAN FAULT TOLERANCE, for streams.                                          ║
║                                                                               ║
║  ── DATABASES AS STREAMS ──────────────────────────────────────────────────── ║
║  A replication log IS a stream of write events. State machine replication is   ║
║  just event streaming.                                                        ║
║  ❌ DUAL WRITES (Fig 11-4): two clients, DB ends at B, index ends at A —        ║
║     PERMANENTLY INCONSISTENT WITH NO ERROR RAISED. Plus the atomic-commit      ║
║     problem if one write fails. DIAGNOSIS: there's NO SINGLE LEADER.          ║
║  ✅ CDC: make the DB the LEADER and everything else a FOLLOWER. Parse the       ║
║     replication log (robust) rather than use triggers (fragile, slow).        ║
║     Debezium/Maxwell/Bottled Water/GoldenGate. ASYNC → replication lag applies.║
║  BOOTSTRAPPING: INITIAL SNAPSHOT tied to a known log offset, OR LOG COMPACTION ║
║     (keep only the latest value per key; size depends on DB CONTENTS, not      ║
║     write count) → a new consumer starts at OFFSET 0 and gets a FULL COPY.    ║
║     ⇒ the broker becomes DURABLE STORAGE, not transient messaging.            ║
║  EVENT SOURCING vs CDC: CDC is LOW-LEVEL and INVISIBLE TO THE APP; event       ║
║     sourcing is APP-LEVEL, events express USER INTENT. CDC events can be       ║
║     log-compacted; EVENT-SOURCED ONES USUALLY CAN'T (later events don't        ║
║     override earlier ones) — snapshots are only a PERFORMANCE OPTIMIZATION.   ║
║  COMMAND (may fail, VALIDATE SYNCHRONOUSLY) → EVENT (immutable FACT; consumers ║
║     MAY NOT REJECT IT). Or split into TENTATIVE + CONFIRMATION.               ║
║  🔑 STATE = ∫ EVENTS dt.  EVENT STREAM = d(STATE)/dt.  "THE TRUTH IS THE LOG;  ║
║     THE DATABASE IS A CACHE OF A SUBSET OF THE LOG." (Helland)                ║
║  Accountants have done this FOR CENTURIES: never erase, ADD A COMPENSATING     ║
║  ENTRY. Immutability buys: bug recovery · MORE INFORMATION (the cart item      ║
║  added then removed) · MANY READ VIEWS FROM ONE LOG = CQRS, which makes the    ║
║  normalize/denormalize debate "LARGELY IRRELEVANT."                           ║
║  ❌ costs: async read views break READ-YOUR-WRITES; TRUE DELETION IS HARD       ║
║     ("making it HARDER to retrieve, not IMPOSSIBLE"). Datomic EXCISION.       ║
║  ✅ but it SIMPLIFIES concurrency: one self-contained event = ONE WRITE; a      ║
║     single-threaded partitioned consumer needs NO CONCURRENCY CONTROL.        ║
║                                                                               ║
║  ── PROCESSING ────────────────────────────────────────────────────────────── ║
║  USES: CEP (QUERIES ARE STORED, EVENTS FLOW PAST THEM — the reverse of a DB) · ║
║  ANALYTICS (windows, bloom filters, HyperLogLog — but "NOTHING IS INHERENTLY   ║
║  APPROXIMATE ABOUT STREAM PROCESSING") · MATERIALIZED VIEWS (need a window     ║
║  stretching BACK TO THE BEGINNING OF TIME) · SEARCH ON STREAMS (index the      ║
║  QUERIES, run the documents past them).                                       ║
║  ⏰ EVENT TIME vs PROCESSING TIME. 🎬 Star Wars: episode number = event time,   ║
║     viewing date = processing time. Fig 11-7: windowing by processing time     ║
║     invents a PHANTOM SPIKE after a restart, on a perfectly steady stream.    ║
║  STRAGGLERS: you can NEVER be sure a window is complete. Either DROP (and      ║
║     ALERT on the drop count) or RECALCULATE AND RETRACT. LOW WATERMARK = "no   ║
║     more events earlier than t" — but CLIENT-GENERATED timestamps break it.   ║
║  UNTRUSTED DEVICE CLOCKS → LOG THREE TIMESTAMPS (device-occurred,             ║
║     device-sent, server-received) and infer the offset.                       ║
║  WINDOWS: TUMBLING (disjoint) · HOPPING (fixed length, OVERLAPPING) ·          ║
║     SLIDING (events within an interval OF EACH OTHER, no fixed boundaries) ·   ║
║     SESSION (NO FIXED DURATION; ends after inactivity).                       ║
║  JOINS: STREAM-STREAM (window join; keep both sides indexed by key — note you  ║
║     CAN'T just embed search details in the click event, or you lose the        ║
║     NON-clicks) · STREAM-TABLE (enrichment; LOCAL COPY kept fresh BY CDC) ·    ║
║     TABLE-TABLE (materialized view; the Twitter timeline; (u·v)′ = u′v + uv′). ║
║     ⚠️ TIME-DEPENDENCE: no ordering guarantee ACROSS streams → the join can be  ║
║        NON-DETERMINISTIC, so re-running may not reproduce the result.         ║
║  FAULT TOLERANCE: you can't wait for a stream to "finish."                     ║
║     MICROBATCHING (Spark, ~1s, implicit processing-time tumbling window) ·     ║
║     CHECKPOINTING (Flink, BARRIERS, no forced window size).                    ║
║     ⚠️ BOTH FAIL once output LEAVES the framework (DB writes, emails).          ║
║     ATOMIC COMMIT must cover: downstream messages + external messages +        ║
║     DB writes + operator state + THE CONSUMER OFFSET. Feasible in restricted   ║
║     settings via a SINGLE-OBJECT WRITE.                                       ║
║     🔑 IDEMPOTENCE is the cheap alternative: STORE THE TRIGGERING OFFSET       ║
║        ALONGSIDE THE VALUE and skip replays. Requires: same replay order       ║
║        (a LOG gives you this) · determinism · no concurrent writers ·          ║
║        FENCING against zombies.                                               ║
║     STATE RECOVERY: local state + periodic replication (Flink → durable        ║
║     snapshots; Samza/Kafka Streams → a LOG-COMPACTED KAFKA TOPIC). Sometimes   ║
║     just REPLAY the input instead.                                            ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 18. ✅ Test yourself

1. **Why does adding a consumer to a JMS-style broker behave so differently from adding one to Kafka?**
   → In a JMS-style broker, acknowledging a message deletes it, so a new consumer only ever sees messages sent after it registered. A log is read-only from the consumer's perspective — the offset is under the consumer's control — so a new consumer can start at offset 0 and read the entire retained history.

2. **JMS and AMQP both require the broker to preserve message order. Why do you still get reordering?**
   → Because load balancing combined with redelivery breaks it. If a consumer crashes mid-message, that message is redelivered to a different consumer that has already moved on to later messages. The order of *sending* is preserved; the order of *processing* is not.

3. **Two clients concurrently update the same item, and the app dual-writes to a database and a search index. What goes wrong, and why won't you notice?**
   → The two systems can apply the writes in different orders and end up permanently inconsistent. No error is raised anywhere — one value silently overwrites another in each system — so without version vectors or similar you'd never know a concurrent write occurred.

4. **How does CDC fix that, in one sentence?**
   → It makes the database the single leader and every other system a follower of its change log, so all derived systems apply the same writes in the same order.

5. **Why can CDC events be log-compacted but event-sourced events usually can't?**
   → A CDC event typically carries the entire new row, so the latest event for a key fully determines its current value and earlier ones are redundant. Event-sourced events express user *intent* and build on each other, so you generally need the full history to reconstruct state.

6. **Explain "state is the integral of an event stream."**
   → Current state is what you get by accumulating all events over time; the change stream is what you get by differentiating state with respect to time. Mutable state and an immutable log aren't in conflict — they're two representations of the same information, and the log is the more fundamental one.

7. **Your request-rate dashboard shows a huge spike right after a deploy, but traffic was flat. What happened?**
   → You're windowing by processing time. The processor was down, the backlog accumulated, and on restart it processed everything at once — so the spike is an artifact of when events were *processed*, not when they *occurred*. Window by event time instead.

8. **Why can't you just embed the search details in the click event and avoid a stream-stream join?**
   → Because you'd only ever learn about searches that *were* clicked. Click-through rate needs the denominator — the searches with no click — which only exists in the search stream.

9. **Microbatching gives exactly-once semantics. Why isn't that enough?**
   → Only inside the framework. Once an effect escapes — a database write, an email, a message to an external broker — the framework can no longer discard it, so a restart causes the side effect to happen twice. You need atomic commit or idempotence at that boundary.

10. **What's the cheapest practical route to exactly-once effects on an external database?**
    → Store the triggering message's offset alongside the value and make the write conditional on the stored offset being lower. A replay then becomes a no-op. It relies on the log replaying in the same order, deterministic processing, no concurrent writers, and fencing against a zombie processor.

---

*All quoted material, figures and examples attributed to the book are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 11. Diagrams have been redrawn in ASCII from the book's originals. Section 16 (post-2017 developments) is supplementary material I added; that landscape moves quickly, so treat specific products and claims there as a starting point worth verifying.*
