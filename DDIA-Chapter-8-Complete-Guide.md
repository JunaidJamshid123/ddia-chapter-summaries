# DDIA — Chapter 8: The Trouble with Distributed Systems
### Complete study guide — theory, every diagram redrawn, and measured simulations

> *Hey I just met you / The network's laggy / But here's my data / So store it maybe*
> — Kyle Kingsbury, *Carly Rae Jepsen and the perils of network partitions* (2013)

---

## 0. The map of this chapter

> **"This chapter is a THOROUGHLY PESSIMISTIC AND DEPRESSING OVERVIEW of things that may go wrong in a distributed system."**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  ① FAULTS AND PARTIAL FAILURES                                               │
│       single computer = deterministic · distributed = NON-DETERMINISTIC      │
│                                                                              │
│  ② UNRELIABLE NETWORKS                                                       │
│       you cannot distinguish lost request / dead node / lost response        │
│       → timeouts are the only tool, and there is NO CORRECT VALUE            │
│                                                                              │
│  ③ UNRELIABLE CLOCKS                                                         │
│       time-of-day vs monotonic · drift · LWW loses data · TrueTime           │
│                                                                              │
│  ④ PROCESS PAUSES                                                            │
│       GC, VM suspension, swapping — a thread can freeze ANYWHERE             │
│                                                                              │
│  ⑤ KNOWLEDGE, TRUTH AND LIES                                                 │
│       truth is defined by the MAJORITY · fencing tokens · Byzantine faults   │
│       · system models · safety vs liveness                                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

> **"We will now turn our pessimism to the maximum, and ASSUME THAT ANYTHING THAT CAN GO WRONG WILL GO WRONG."** *(Footnote: with one exception — we assume faults are non-Byzantine.)*

**Why this chapter exists:** Chapter 9 will give solutions. **"But first, in this chapter, we must understand WHAT CHALLENGES WE ARE UP AGAINST."**

---

## 1. Faults and partial failures

### The single computer: an idealized system model

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  "An individual computer with good software is usually EITHER FULLY  │
   │   FUNCTIONAL OR ENTIRELY BROKEN, BUT NOT SOMETHING IN BETWEEN."      │
   │                                                                      │
   │  🔑 THIS IS A DELIBERATE DESIGN CHOICE:                               │
   │  "if an internal fault occurs, WE PREFER A COMPUTER TO CRASH         │
   │   COMPLETELY, RATHER THAN RETURN A WRONG RESULT, because wrong       │
   │   results are difficult and confusing to deal with."                 │
   │                                                                      │
   │  "Thus, computers HIDE THE FUZZY PHYSICAL REALITY on which they are  │
   │   implemented, and present an IDEALIZED SYSTEM MODEL that operates   │
   │   with MATHEMATICAL PERFECTION."                                     │
   └──────────────────────────────────────────────────────────────────────┘
```

### The distributed system: no such luxury

> **"In distributed systems, we are no longer operating in an idealized system model — WE HAVE NO CHOICE BUT TO CONFRONT THE MESSY REALITY OF THE PHYSICAL WORLD."**

**The Coda Hale anecdote, quoted in full because it's the best argument in the section:**

```
   "In my limited experience I've dealt with long-lived NETWORK PARTITIONS in
    a single data center, PDU [power distribution unit] FAILURES, SWITCH
    FAILURES, ACCIDENTAL POWER CYCLES of whole racks, whole-DC BACKBONE
    FAILURES, whole-DC POWER FAILURES, and A HYPOGLYCEMIC DRIVER SMASHING HIS
    FORD PICKUP TRUCK INTO A DC'S HVAC SYSTEM.

    And I'm not even an ops guy."
```

### 🔑 Partial failure

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  PARTIAL FAILURE: "some parts of the system may be broken in some     ║
   ║  unpredictable way, EVEN THOUGH OTHER PARTS ARE WORKING FINE."        ║
   ║                                                                       ║
   ║  "The difficulty is that partial failures are NON-DETERMINISTIC: if   ║
   ║   you try to do anything involving multiple nodes and the network, it ║
   ║   MAY SOMETIMES WORK AND SOMETIMES UNPREDICTABLY FAIL.                ║
   ║                                                                       ║
   ║   As we shall see, YOU MAY NOT EVEN KNOW WHETHER SOMETHING            ║
   ║   SUCCEEDED OR NOT!"                                                  ║
   ║                                                                       ║
   ║  ➜ "This non-determinism and possibility of partial failures is WHAT  ║
   ║     MAKES DISTRIBUTED SYSTEMS HARD TO WORK WITH."                     ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### Two philosophies: supercomputing vs cloud

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  HIGH-PERFORMANCE COMPUTING       ║  CLOUD COMPUTING                  ║
   ║  (supercomputers)                 ║  (internet services)              ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  Thousands of CPUs for weather    ║  Multitenant datacenters,         ║
   ║  forecasting, molecular dynamics  ║  COMMODITY computers, IP/Ethernet,║
   ║  SPECIALIZED hardware, RDMA,      ║  elastic allocation, metered      ║
   ║  shared memory, mesh/torus        ║  billing, Clos topologies         ║
   ║  topologies                       ║                                   ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  FAULT HANDLING: CHECKPOINT to    ║  FAULT HANDLING: build TOLERANCE  ║
   ║  durable storage; on failure,     ║  into the software.               ║
   ║  STOP THE ENTIRE CLUSTER, repair, ║                                   ║
   ║  restart from the last checkpoint.║  • services are ONLINE — stopping ║
   ║                                   ║    for repair is NOT ACCEPTABLE   ║
   ║  🔑 "a supercomputer is MORE LIKE  ║  • commodity machines have HIGHER ║
   ║    A SINGLE-NODE COMPUTER than a  ║    FAILURE RATES                  ║
   ║    distributed system: it deals   ║  • "in a system with THOUSANDS OF ║
   ║    with partial failure BY        ║    NODES, it is reasonable to     ║
   ║    LETTING IT ESCALATE INTO TOTAL ║    assume that SOMETHING IS       ║
   ║    FAILURE."                      ║    ALWAYS BROKEN"                 ║
   ║                                   ║  • tolerating failure enables     ║
   ║                                   ║    ROLLING UPGRADES               ║
   ║                                   ║  • geo-distribution goes over the ║
   ║                                   ║    INTERNET: slow and unreliable  ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝

   ➜ "If the error handling strategy consists of SIMPLY GIVING UP, such a
     large system WOULD NEVER WORK."
```

> **The mandate:** *"we need to BUILD A RELIABLE SYSTEM FROM UNRELIABLE COMPONENTS."*
>
> **And the attitude:** *"It would be unwise to assume that faults are rare and simply hope for the best. It is important to consider a wide range of possible faults — even fairly unlikely ones — and to ARTIFICIALLY CREATE SUCH SITUATIONS IN YOUR TESTING ENVIRONMENT. **In distributed systems, SUSPICION, PESSIMISM AND PARANOIA PAY OFF.**"*

### 📦 Sidebar: Building a reliable system from unreliable components

```
   "Intuitively it may seem like a system can only be as reliable as its LEAST
    RELIABLE COMPONENT (its weakest link). THIS IS NOT THE CASE."

   ┌────────────────────────────┬─────────────────────────────────────────┐
   │ ERROR-CORRECTING CODES     │ accurate transmission over a channel    │
   │                            │ that occasionally gets bits wrong       │
   ├────────────────────────────┼─────────────────────────────────────────┤
   │ IP is unreliable           │ drops, delays, duplicates, reorders     │
   │        ↓                   │                                         │
   │ TCP on top of IP           │ retransmits, dedupes, reorders back     │
   └────────────────────────────┴─────────────────────────────────────────┘

   ⚠️ BUT THERE IS ALWAYS A LIMIT:
      "error-correcting codes can deal with a SMALL NUMBER of single-bit
       errors, but if your signal is SWAMPED BY INTERFERENCE, there is a
       FUNDAMENTAL LIMIT to how much data you can get through.
       TCP can hide packet loss, duplication and reordering from you,
       BUT IT CANNOT MAGICALLY REMOVE DELAYS IN THE NETWORK."
```

---

## 2. Unreliable networks

> Shared-nothing systems communicate **only** over an **asynchronous packet network**. *"The network gives NO GUARANTEES as to WHEN it will arrive, or WHETHER it will arrive at all."*

### 🔷 Figure 8-1 — Six things that can go wrong, three of them indistinguishable

```
                    (a)                  (b)                  (c)        TIME ──►
                          ???                  ???                  ???
   Client ────●───────────────────●───────────────────●────────────────────────►
               ╲                   ╲                   ╲
                ╲  ✗ REQUEST LOST   ╲                   ╲            ok
                 ✗                   ╲                   ▼           ▲
   Network ───────────────────────────╲───────────────────●──────────╱────────►
                                       ╲                             ╱
                                        ▼                     ✗ RESPONSE LOST
   Service ─────────────────── NODE UNRESPONSIVE ────────────●──────────────────►
                                                        (it DID the work!)

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "It's NOT POSSIBLE TO DISTINGUISH whether (a) the request was lost,  ║
   ║   (b) the remote node is down, or (c) the response was lost."         ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

**The six possibilities, in full:**

```
   ① your request may have been LOST (someone unplugged the cable)
   ② your request may be WAITING IN A QUEUE, delivered later (overload)
   ③ the remote node may have FAILED (crashed or powered down)
   ④ the remote node may have TEMPORARILY STOPPED RESPONDING (long GC pause)
      but will start responding again later
   ⑤ the remote node PROCESSED your request, but THE RESPONSE WAS LOST
      (misconfigured switch)
   ⑥ the remote node PROCESSED your request, but the response is DELAYED
```

> **"The sender CAN'T EVEN TELL WHETHER THE PACKET WAS DELIVERED: the only way to tell whether it arrived is for the recipient to send a response message, WHICH MAY IN TURN BE LOST OR DELAYED."**
>
> And the timeout doesn't save you: **"when a timeout occurs, you STILL DON'T KNOW whether the remote node got your request or not (and if the request is still queued somewhere, IT MAY STILL BE DELIVERED, even if the sender has given up on it)."**

### 💻 I tabulated exactly what the client can observe

```
what ACTUALLY happened      what the client SEES      side effect?
──────────────────────────────────────────────────────────────────
request lost                             timeout                no
node down                                timeout                no
response lost                            timeout    YES — it ran!
node slow (GC)                           timeout    YES — it ran!
ok                                      response    YES — it ran!
```

**Four different realities, one observation.** In two of them the operation *executed*. This is precisely why Chapter 4 insisted on idempotence: a retry after a timeout may be a duplicate, and you have no way to know.

### Network faults in practice

```
   "We have been building computer networks for decades — one might hope that
    by now we would have figured out how to make them reliable. HOWEVER, IT
    SEEMS THAT WE HAVE NOT YET SUCCEEDED."

   📊 THE STUDIES:
   • One medium-sized datacenter: ~12 NETWORK FAULTS PER MONTH — half
     disconnected a single machine, half disconnected AN ENTIRE RACK.
   • Another measured top-of-rack switches, aggregation switches, load
     balancers, and found "ADDING REDUNDANT NETWORKING GEAR DOESN'T REDUCE
     FAULTS AS MUCH AS YOU'D HOPE, since IT DOESN'T GUARD AGAINST HUMAN
     ERROR (e.g. misconfigured switches), WHICH IS A MAJOR CAUSE OF OUTAGES."
   • A switch software upgrade triggered a topology reconfiguration during
     which packets were DELAYED FOR MORE THAN A MINUTE.
   • 😬 "a network interface that SOMETIMES DROPS ALL INBOUND PACKETS, BUT
     SENDS OUTBOUND PACKETS SUCCESSFULLY — just because a network link works
     IN ONE DIRECTION doesn't guarantee it's also working in the opposite
     direction."
```

> 📖 **Terminology:** a **network partition** or **netsplit** is when part of the network is cut off. **"In this book we'll stick with the more general term NETWORK FAULT, to avoid confusion with PARTITIONS (SHARDS) of a storage system."** *(Same word-collision warning as Chapter 6.)*

```
   ⚠️ IF FAULT HANDLING IS UNDEFINED AND UNTESTED, "ARBITRARILY BAD THINGS
      COULD HAPPEN: the cluster could become DEADLOCKED and permanently
      unable to serve requests, EVEN WHEN THE NETWORK RECOVERS — or it could
      even DELETE ALL OF YOUR DATA."

   ✅ BUT: "Handling network faults DOESN'T NECESSARILY MEAN TOLERATING THEM.
      If your network is normally fairly reliable, a VALID APPROACH may be to
      simply SHOW AN ERROR MESSAGE to users. However, YOU DO NEED TO KNOW HOW
      YOUR SOFTWARE REACTS, and ensure the system can RECOVER."
```

---

## 3. Detecting faults, and the timeout dilemma

### The few cases where you get explicit feedback

```
   ✅ NO PROCESS LISTENING on the port → the OS closes/refuses the TCP
      connection with RST or FIN.
      ⚠️ "However, if the node CRASHED WHILE IT WAS HANDLING YOUR REQUEST,
         you have NO WAY OF KNOWING HOW MUCH DATA WAS ACTUALLY PROCESSED."
   ✅ PROCESS CRASHED but the OS is running → a script can NOTIFY other nodes
      so another can take over WITHOUT WAITING FOR A TIMEOUT. (HBase does this.)
   ✅ SWITCH MANAGEMENT INTERFACE → detect link failures at the hardware level.
      ⚠️ Ruled out over the internet, in shared datacenters, or if the network
         problem also blocks the management interface.
   ✅ ICMP DESTINATION UNREACHABLE from a router.
      ⚠️ "the router DOESN'T HAVE A MAGIC FAILURE DETECTION CAPABILITY either
         — it is subject to THE SAME LIMITATIONS."

   🔑 "Even if TCP acknowledges that a packet was delivered, THE APPLICATION
     MAY HAVE CRASHED BEFORE HANDLING IT. If you want to be sure that a
     request was successful, YOU NEED A POSITIVE RESPONSE FROM THE
     APPLICATION ITSELF."
```

### ⚖️ The timeout trade-off

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  LONG TIMEOUT                    │  SHORT TIMEOUT                     ║
   ║  ────────────                    │  ─────────────                     ║
   ║  a long wait before a node is    │  faster detection, but A HIGHER    ║
   ║  declared dead — users wait or   │  RISK OF INCORRECTLY DECLARING A   ║
   ║  see error messages              │  NODE DEAD when it only suffered a ║
   ║                                  │  TEMPORARY SLOWDOWN                ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  ⚠️ WHY PREMATURE DEATH IS WORSE THAN IT LOOKS:                        ║
   ║  "as its responsibilities are transferred to other nodes, ADDITIONAL  ║
   ║   LOAD IS PLACED ON OTHER NODES AND THE NETWORK. If the system is     ║
   ║   ALREADY STRUGGLING with high load, declaring nodes dead prematurely ║
   ║   can MAKE THE PROBLEM WORSE, and even cause a CASCADING FAILURE —    ║
   ║   in the extreme case, ALL NODES MAY DECLARE EACH OTHER DEAD, AND     ║
   ║   EVERYTHING STOPS WORKING."                                          ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

**The fictitious system where timeouts would be easy:**

```
   IF every packet arrived within time d (or was lost), AND
   IF a non-failed node always handled a request within time r,
   THEN every successful request responds within  2d + r
        → 2d + r is a reasonable timeout. Problem solved.

   ❌ "UNFORTUNATELY, MOST SYSTEMS WE WORK WITH HAVE NEITHER OF THOSE
      GUARANTEES: asynchronous networks have UNBOUNDED DELAYS, and most
      server implementations CANNOT GUARANTEE a maximum handling time."

   ➜ "For failure detection, IT'S NOT SUFFICIENT FOR THE SYSTEM TO BE FAST
     MOST OF THE TIME: if your timeout is low, IT ONLY TAKES A TRANSIENT
     SPIKE in round-trip-times to throw the system off-balance."
```

### 💻 I measured the trade-off on a realistic latency distribution

200,000 samples: 97% healthy (~5 ms), 3% queueing spikes (~120 ms), 0.1% GC pauses (~2 s).

```
round-trip time: p50=5.1ms  p99=139.9ms  p99.9=226.2ms  max=3314ms

   timeout    FALSE deaths (live node declared dead)    detection delay
──────────────────────────────────────────────────────────────────────────
      10ms                                   3.026%               10ms
      50ms                                   2.897%               50ms
     200ms                                   0.161%              200ms
    1000ms                                   0.082%             1000ms
    5000ms                                   0.000%             5000ms
   30000ms                                   0.000%            30000ms
```

**There is no good row in that table.** A 50 ms timeout wrongly kills a healthy node 3% of the time — in a 100-node cluster that's roughly three spurious failovers per round of health checks. A 5-second timeout is safe but means five seconds of unavailability before failover even starts. **The book's "there is unfortunately no simple answer" is not a dodge; the data has no knee.**

---

## 4. Why delays are unbounded: network congestion and queueing

> **"When driving a car, travel times on road networks often vary most due to TRAFFIC CONGESTION. Similarly, the variability of packet delays on computer networks is most often due to QUEUEING."**

### 🔷 Figure 8-2 — Switch queues fill up

```
   INPUT LINKS              NETWORK SWITCH                     OUTPUT LINKS

   Port 1 ──────┐                                       ┌──────► Port 1
                │                                       │
   Port 2 ──────┼──────►  ┌──────────────┐              ├──────► Port 2
                ├────────►│              │   ░░░░░░░    │
   Port 3 ──────┤         │    SWITCH    │──►░QUEUE░───-┼──────► Port 3 ◄── ALL
                │         │    FABRIC    │   ░░░░░░░    │        THREE WANT
   Port 4 ──────┴────────►│              │   (FILLING)  ├──────► Port 4  THIS PORT
                          └──────────────┘              │
                                                        └
   Ports 1, 2 AND 4 are all sending to port 3.
   The switch must QUEUE them and feed them in ONE BY ONE.
   ⚠️ "If there is so much incoming data that THE SWITCH QUEUE FILLS UP, THE
      PACKET IS DROPPED, so it needs to be re-sent — EVEN THOUGH THE NETWORK
      IS FUNCTIONING FINE."
```

### The five sources of queueing

```
   ① SWITCH QUEUES — network congestion (Figure 8-2)
   ② OS QUEUES at the destination — "if all CPU cores are busy, the incoming
      request is QUEUED BY THE OPERATING SYSTEM until the application is
      ready. Depending on load, this may take AN ARBITRARY LENGTH OF TIME."
   ③ VIRTUAL MACHINE PAUSES — "a running operating system is often PAUSED FOR
      TENS OF MILLISECONDS while another VM uses a CPU core. During this time
      the VM CANNOT CONSUME ANY DATA FROM THE NETWORK."
   ④ TCP FLOW CONTROL — the sender limits its own rate → "ADDITIONAL QUEUEING
      AT THE SENDER, BEFORE THE DATA EVEN ENTERS THE NETWORK."
   ⑤ TCP RETRANSMISSION — "although the application DOES NOT SEE the packet
      loss and retransmission, IT DOES SEE THE RESULTING DELAY."
```

### 💻 I simulated the queue, and the knee is as brutal as advertised

```
   utilisation ρ    avg time in system
   ─────────────────────────────────────────────────
          0.10                 1.1x
          0.30                 1.4x
          0.50                 2.0x
          0.70                 3.3x  █
          0.80                 5.0x  █
          0.90                10.0x  ███
          0.95                20.0x  █████
          0.99               100.0x  █████████████████████████████

   And the actual tail from a discrete-event simulation:
        ρ       p50       p99     p99.9       max
     0.50      1.39      8.92     12.99     19.39
     0.80      3.53     23.77     32.98     39.00
     0.95     13.84    111.31    121.67    127.84
```

**Note the p50 column at ρ=0.95: 13.84.** Your median looks like a 14× degradation while your p99 is a 111× degradation. **This is the mechanism behind Chapter 1's percentile argument and Chapter 7's 2PL latency instability — the same curve, appearing for the third time.**

> **The multitenancy problem:** *"In public clouds and multitenant datacenters, resources are shared among many customers: network links and switches, and even each machine's network interface and CPUs. Batch workloads such as MapReduce can EASILY SATURATE NETWORK LINKS. As you have NO CONTROL OR INSIGHT over other customers' usage, network delays can be HIGHLY VARIABLE if someone near you (a NOISY NEIGHBOR) is using a lot of resources."*

### The better answer: adaptive timeouts

> **"Rather than using configured constant timeouts, systems can CONTINUALLY MEASURE RESPONSE TIMES AND THEIR VARIABILITY (JITTER), and AUTOMATICALLY ADJUST TIMEOUTS according to the observed response time distribution."** Done with a **Phi Accrual failure detector**, used in **Akka and Cassandra**.

### 💻 I implemented Phi Accrual — it outputs suspicion, not a verdict

After 200 healthy heartbeats (mean ≈100 ms, sd ≈8 ms):

```
     silence       phi    interpretation
   ──────────────────────────────────────────────
        50ms      0.00    normal
       100ms      0.30    normal
       130ms      4.53    suspicious
       160ms     15.32    almost certainly dead
       200ms     40.43    almost certainly dead
       300ms    157.37    almost certainly dead
```

**The key difference:** a fixed timeout is a step function at one arbitrary point. Phi is a *continuous suspicion level calibrated to the link's own observed jitter*. On a link that normally varies by ±8 ms, 130 ms of silence is genuinely alarming; on a jittery link with sd=50 ms it wouldn't be. The detector adapts; a constant cannot.

---

## 5. Synchronous vs asynchronous networks

> **"Why can't we solve this at the hardware level, and make the network reliable so the software doesn't need to worry about it?"**

### The telephone network comparison

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  CIRCUIT-SWITCHED (telephone)     ║  PACKET-SWITCHED (Ethernet/IP)    ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  A CIRCUIT: a FIXED, GUARANTEED   ║  Packets OPPORTUNISTICALLY use    ║
   ║  amount of bandwidth reserved     ║  whatever bandwidth is available. ║
   ║  along the ENTIRE ROUTE for the   ║  An idle TCP connection uses      ║
   ║  duration of the call.            ║  NO bandwidth at all.             ║
   ║                                   ║                                   ║
   ║  ISDN: 4,000 frames/sec, each     ║                                   ║
   ║  call gets 16 BITS PER FRAME      ║                                   ║
   ║  → guaranteed 16 bits every       ║                                   ║
   ║    250 MICROSECONDS               ║                                   ║
   ║                                   ║                                   ║
   ║  ✅ NO QUEUEING (space is already  ║  ❌ QUEUEING → UNBOUNDED DELAYS    ║
   ║     reserved at the next hop)     ║                                   ║
   ║  ✅ BOUNDED DELAY                  ║                                   ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝
```

### 🔑 So why did we choose packet switching? Bursty traffic.

```
   "A circuit is good for an AUDIO OR VIDEO CALL, which needs a fairly
    CONSTANT number of bits per second. On the other hand, requesting a web
    page, sending an email or transferring a file DOESN'T HAVE ANY PARTICULAR
    BANDWIDTH REQUIREMENT — WE JUST WANT IT TO COMPLETE AS QUICKLY AS
    POSSIBLE."

   If you transferred a file over a circuit, you'd have to GUESS an allocation:
      guess TOO LOW  → the transfer is unnecessarily slow, capacity unused
      guess TOO HIGH → the circuit CANNOT BE SET UP AT ALL

   ➜ "using circuits for bursty data transfers WASTES NETWORK CAPACITY and
     makes transfers UNNECESSARILY SLOW. By contrast, TCP DYNAMICALLY ADAPTS."
```

> 📖 **Hybrid attempts:** ATM (*"a competitor to Ethernet in the 1980s… nothing to do with Automatic Teller Machines, despite sharing an acronym. Perhaps, in some parallel universe, the internet is based on something like ATM — in that universe, internet video calls are probably a lot more reliable than they are in ours"*). **InfiniBand** does end-to-end link-layer flow control. **QoS + admission control** can emulate circuit switching — **"however, such quality of service is CURRENTLY NOT ENABLED in multitenant datacenters and public clouds, or when communicating via the internet."**

### 📦 Sidebar: Latency and resource utilization — the deepest point in the chapter

```
   "You can think of variable delays as a consequence of DYNAMIC RESOURCE
    PARTITIONING."

   ┌──────────────────────────────────┬──────────────────────────────────┐
   │ STATIC partitioning              │ DYNAMIC partitioning             │
   │ (telephone circuits)             │ (internet, CPU threads, VMs)     │
   ├──────────────────────────────────┼──────────────────────────────────┤
   │ Even if you're the ONLY call on  │ Senders "PUSH AND JOSTLE with    │
   │ a 10,000-slot wire, you get the  │ each other to get their packets  │
   │ SAME FIXED BANDWIDTH as when it  │ over the wire as quickly as      │
   │ is fully utilized.               │ possible."                       │
   │                                  │                                  │
   │ ✅ LATENCY GUARANTEES             │ ❌ QUEUEING                       │
   │ ❌ REDUCED UTILIZATION            │ ✅ MAXIMIZES UTILIZATION          │
   │ ❌ MORE EXPENSIVE                 │ ✅ CHEAPER PER BYTE               │
   └──────────────────────────────────┴──────────────────────────────────┘

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "VARIABLE DELAYS IN NETWORKS ARE NOT A LAW OF NATURE, BUT SIMPLY THE ║
   ║   RESULT OF A COST-BENEFIT TRADE-OFF."                                ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ⚑ The same logic applies to CPUs: sharing a core between threads means a
     thread can be PAUSED FOR VARYING LENGTHS OF TIME — but it UTILIZES THE
     HARDWARE BETTER than a static allocation. "Better hardware utilization is
     also a significant motivation for using VIRTUAL MACHINES."
```

---
---

## 6. Unreliable clocks

```
   Applications use clocks for two DIFFERENT things:

   MEASURING A DURATION              DESCRIBING A POINT IN TIME
   ────────────────────              ──────────────────────────
   • Has this request timed out?     • When should the reminder be sent?
   • What's the p99 response time?   • When does this cache entry expire?
   • QPS over the last 5 minutes?    • What's the timestamp on this log line?

           ↓                                       ↓
     MONOTONIC CLOCK                       TIME-OF-DAY CLOCK
```

### The two kinds of clock

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  TIME-OF-DAY CLOCK                ║  MONOTONIC CLOCK                  ║
   ║  clock_gettime(CLOCK_REALTIME)    ║  clock_gettime(CLOCK_MONOTONIC)   ║
   ║  System.currentTimeMillis()       ║  System.nanoTime()                ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  Returns wall-clock time: seconds ║  GUARANTEED TO ALWAYS MOVE        ║
   ║  since the epoch (midnight UTC,   ║  FORWARDS. The ABSOLUTE VALUE IS  ║
   ║  1 Jan 1970), NOT COUNTING LEAP   ║  MEANINGLESS — maybe nanoseconds  ║
   ║  SECONDS.                         ║  since boot, or something else.   ║
   ║                                   ║                                   ║
   ║  Synchronized with NTP, so a      ║  ⚠️ "IT MAKES NO SENSE TO COMPARE  ║
   ║  timestamp on one machine         ║     MONOTONIC CLOCK VALUES FROM   ║
   ║  (ideally) means the same as on   ║     TWO DIFFERENT COMPUTERS."     ║
   ║  another.                         ║                                   ║
   ║                                   ║  NTP may SLEW it (adjust the      ║
   ║  ❌ "if the local clock is TOO FAR ║  rate, by default up to ±0.05%)   ║
   ║     AHEAD of the NTP server, IT   ║  but CANNOT make it JUMP.         ║
   ║     MAY BE FORCIBLY RESET and     ║                                   ║
   ║     APPEAR TO JUMP BACK IN TIME." ║  Resolution: microseconds or less.║
   ║  ❌ often ignores LEAP SECONDS     ║                                   ║
   ║  ➜ UNSUITABLE FOR MEASURING       ║  ➜ USE THIS FOR DURATIONS.        ║
   ║    ELAPSED TIME                   ║                                   ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝

   📖 "On a server with multiple CPU sockets, there may be A SEPARATE TIMER
      PER CPU… it is wise to take this guarantee of monotonicity WITH A PINCH
      OF SALT."
```

### The eight ways clock synchronization goes wrong

```
   ① QUARTZ DRIFT — Google assumes 200 PPM for their servers. Varies with
      TEMPERATURE. "This LIMITS THE BEST POSSIBLE ACCURACY you can achieve,
      EVEN IF EVERYTHING IS WORKING CORRECTLY."
   ② FORCIBLE RESET — if the clock differs too much from NTP, it may refuse
      to sync or be reset → "applications may see TIME GO BACKWARDS."
   ③ FIREWALLED OFF FROM NTP — "the misconfiguration may go UNNOTICED FOR
      SOME TIME. Anecdotal evidence suggests this does happen in practice."
   ④ NETWORK DELAY LIMITS ACCURACY — "a MINIMUM ERROR OF 35 ms is achievable
      over the internet, though occasional spikes lead to errors of AROUND
      A SECOND."
   ⑤ WRONG NTP SERVERS — "reporting time that is OFF BY HOURS… it's somewhat
      worrying to BET THE CORRECTNESS OF YOUR SYSTEMS ON THE TIME THAT YOU
      WERE TOLD BY A STRANGER ON THE INTERNET."
   ⑥ LEAP SECONDS — a minute of 59 or 61 seconds. "The fact that LEAP SECONDS
      HAVE CRASHED MANY LARGE SYSTEMS shows how easy it is for incorrect
      assumptions to sneak in." Best fix: make NTP servers "LIE" by SMEARING
      the adjustment over a day.
   ⑦ VIRTUAL MACHINES — the hardware clock is virtualized; a paused VM sees
      "the clock SUDDENLY JUMPING FORWARDS."
   ⑧ DEVICES YOU DON'T CONTROL — "Some users DELIBERATELY SET THEIR HARDWARE
      CLOCK TO AN INCORRECT DATE, for example to circumvent timing
      limitations in games."
```

### 💻 I computed the drift table, and it matches the book exactly

```
     time since sync     accumulated drift
   ────────────────────────────────────────
          30 seconds                6.0 ms      ← book says "6 ms"
            1 minute               12.0 ms
              1 hour              720.0 ms
               1 day               17.3 s       ← book says "17 seconds"
              1 week              121.0 s
```

**And the step-back that breaks duration measurement:**

```
   measured with TIME-OF-DAY clock : 3601.080 s   (real elapsed: 3600 s)
   NTP corrects → clock JUMPS BACKWARDS by 1080 ms
   any code doing  end - start  across that jump gets a NEGATIVE duration
```

### ⚠️ Why bad clocks are worse than bad hardware

> **"Part of the problem is that INCORRECT CLOCKS EASILY GO UNNOTICED. If a machine's CPU is defective or its network is misconfigured, it most likely WON'T WORK AT ALL, so it will quickly be noticed and fixed. On the other hand, if its quartz clock is defective or its NTP client is misconfigured, MOST THINGS SEEM TO WORK FINE, even though its clock gradually drifts further and further away from reality.**
>
> **"If some piece of software is relying on an accurately synchronized clock, the result is more likely to be SILENT AND SUBTLE DATA LOSS THAN A DRAMATIC CRASH."**

```
   ➜ THE OPERATIONAL MANDATE: "if you use software that requires synchronized
     clocks, it is ESSENTIAL that you also CAREFULLY MONITOR THE CLOCK OFFSETS
     between all the machines. Any nodes whose clock drifts too far from the
     others SHOULD BE DECLARED DEAD AND REMOVED FROM THE CLUSTER."
```

> 📖 **When accuracy really matters:** *MiFID II* requires high-frequency trading funds to synchronize **within 100 microseconds of UTC**, to help debug flash crashes and detect market manipulation. Achievable with **GPS receivers and the Precision Time Protocol** — *"however, it requires significant effort and expertise."*

---

## 7. Timestamps for ordering events — where LWW goes wrong

### 🔷 Figure 8-3 — The causally later write gets the earlier timestamp

```
                set x = 1
                    │                                        TIME ─────────►
   Client A ────────●───────────────────────────────────────────────────────►
                    │  ▲ ok
                    ▼  │
             42.003│42.004│42.005│42.006│42.007│42.008   ← NODE 1's CLOCK
   Node 1 ─────────●──●───────────────────────────────────────────────────►
                        ╲  x=1, ts=42.004
                         ╲
                          ▼
   Node 2 ──────────────────────●───────────────●──────────────────────────►
                            x=1, ts=42.004   x=2, ts=42.003
                                                  ▲
                                    💥 LOWER TIMESTAMP, LATER EVENT
                         ╱
                        ╱  x=2, ts=42.003
             42.001│42.002│42.003│42.004│42.005            ← NODE 3's CLOCK
   Node 3 ──────────────────●──────●──────────────────────────────────────►
                            │   ▲ ok
                            ▲   │
   Client B ────────────────●───●──────────────────────────────────────────►
                        increment x += 1

   THE CLOCK SKEW HERE IS UNDER 3 ms — "probably better than you can expect
   in practice." AND IT STILL BREAKS.

   ➜ Node 2 concludes x=1 (ts 42.004) is more recent than x=2 (ts 42.003),
     and DROPS THE INCREMENT. "In effect, CLIENT B'S INCREMENT OPERATION HAS
     BEEN LOST."
```

### The three fundamental problems with LWW

```
   ① "Database writes can MYSTERIOUSLY DISAPPEAR: a node with a LAGGING CLOCK
      is UNABLE TO OVERWRITE values previously written by a node with a FAST
      CLOCK until the clock skew has elapsed. This can cause ARBITRARY AMOUNTS
      OF DATA TO BE SILENTLY DROPPED WITHOUT ANY ERROR BEING REPORTED."

   ② "LWW CANNOT DISTINGUISH between writes that occurred SEQUENTIALLY in
      quick succession and writes that were TRULY CONCURRENT."
      → needs VERSION VECTORS (Chapter 5).

   ③ "Two nodes can INDEPENDENTLY GENERATE WRITES WITH THE SAME TIMESTAMP,
      especially when the clock only has millisecond resolution." A tiebreaker
      is needed, "but this CAN ALSO LEAD TO VIOLATIONS OF CAUSALITY."
```

> **The impossibility argument:** *"Even with tightly NTP-synchronized clocks, you could send a packet at timestamp 100 ms (according to the sender's clock), and have it ARRIVE AT TIMESTAMP 99 ms (according to the recipient's clock) — so it appears as though THE PACKET ARRIVED BEFORE IT WAS SENT, WHICH IS IMPOSSIBLE."*
>
> **Could better NTP fix it? "PROBABLY NOT, because NTP's synchronization accuracy is ITSELF LIMITED BY THE NETWORK ROUND-TRIP TIME. For correct ordering, you would need the clock source to be SIGNIFICANTLY MORE ACCURATE THAN THE THING YOU ARE MEASURING (namely network delay)."** — which is circular, and therefore hopeless.

### 💻 I reproduced Figure 8-3, then fixed it with a logical clock

```
PHYSICAL/CAUSAL order of events:
   1. Client A: set x = 1           on node1, timestamp 42.004
   2. Client B: increment x -> 2    on node3, timestamp 42.003

LWW keeps the HIGHEST timestamp -> 'Client A: set x = 1'
   💥 THE CAUSALLY LATER WRITE (x=2) WAS DISCARDED.

--- LOGICAL CLOCKS get it right, with no clock at all ---
   x=1 gets Lamport timestamp 1
   x=2 gets Lamport timestamp 2   (2 > 1 ✅ causality preserved)
```

> **Logical clocks** *"are based on INCREMENTING COUNTERS rather than an oscillating quartz crystal, and are A SAFER ALTERNATIVE for ordering events. Logical clocks DO NOT MEASURE THE TIME OF DAY, only the RELATIVE ORDERING of events."* (Regular clocks are called **physical clocks** by contrast.)

---

## 8. Clock readings have a confidence interval

> **"It doesn't make sense to think of a clock reading as A POINT IN TIME — it is more like A RANGE OF TIMES, within a CONFIDENCE INTERVAL."**
>
> **"If we only know the time +/− 100 ms, THE MICROSECOND DIGITS IN THE TIMESTAMP ARE ESSENTIALLY MEANINGLESS."**

```
   HOW THE UNCERTAINTY IS COMPUTED:
      GPS receiver / atomic clock attached → error range from the manufacturer
      time from a server → expected QUARTZ DRIFT SINCE LAST SYNC
                         + the NTP SERVER'S OWN UNCERTAINTY
                         + the NETWORK ROUND-TRIP TIME

   ❌ "Unfortunately, MOST SYSTEMS DON'T EXPOSE THIS UNCERTAINTY: when you call
      clock_gettime(), the return value DOESN'T TELL YOU THE EXPECTED ERROR,
      so YOU DON'T KNOW IF ITS CONFIDENCE INTERVAL IS 5 MILLISECONDS OR
      5 YEARS."

   ✅ THE EXCEPTION: Google's TRUETIME API in Spanner returns TWO VALUES:
         [earliest, latest]
      "the clock KNOWS that the actual current time is SOMEWHERE WITHIN THAT
       INTERVAL."
```

### 🔷 Synchronized clocks for global snapshots — Spanner's commit-wait

```
   THE OBSERVATION:
   ┌──────────────────────────────────────────────────────────────────────┐
   │  A = [A_earliest ═══════ A_latest]                                   │
   │                                     B = [B_earliest ═══ B_latest]    │
   │  DISJOINT → B DEFINITELY happened after A. NO DOUBT.                 │
   ├──────────────────────────────────────────────────────────────────────┤
   │  A = [A_earliest ═══════════ A_latest]                               │
   │                     B = [B_earliest ═════════ B_latest]              │
   │  OVERLAPPING → WE ARE UNSURE IN WHICH ORDER A AND B HAPPENED.        │
   └──────────────────────────────────────────────────────────────────────┘

   🔑 SPANNER'S TRICK: "DELIBERATELY WAITS FOR THE LENGTH OF THE CONFIDENCE
     INTERVAL before committing a read-write transaction. By doing so, it
     ensures that any transaction that may read the data is at a sufficiently
     later time, SO THEIR CONFIDENCE INTERVALS DO NOT OVERLAP."

   ➜ "In order to keep the wait time as short as possible, Spanner needs to
     keep the clock uncertainty AS SMALL AS POSSIBLE; for this purpose, Google
     DEPLOYS A GPS RECEIVER OR ATOMIC CLOCK IN EACH DATACENTER, allowing clocks
     to be synchronized to WITHIN ABOUT 7 ms."
```

### 💻 I implemented TrueTime intervals

```
   A = [99.9936, 100.0076]  (width 14.0 ms)
   B = [99.9991, 100.0131]  (width 14.0 ms)
   C = [100.0396, 100.0536] (width 14.0 ms)

   A before B?  intervals overlap        -> CANNOT TELL
   A before C?  A.latest < C.earliest    -> DEFINITELY YES
```

**Notice what commit-wait actually costs:** every read-write transaction pays ~14 ms of deliberate idling. Spanner buys correctness with latency, and buys the latency back by spending money on atomic clocks. **That's the cost-benefit trade-off from the latency sidebar, appearing as a product decision.**

---

## 9. Process pauses

### The lease code from the book, and its two bugs

```java
while (true) {
    request = getIncomingRequest();

    // Ensure that the lease always has at least 10 seconds remaining
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000) {
        lease = lease.renew();
    }

    if (lease.isValid()) {
        process(request);
    }
}
```

```
   ❌ BUG 1: IT RELIES ON SYNCHRONIZED CLOCKS. "The expiry time on the lease is
      set by A DIFFERENT MACHINE, and it's being compared to THE LOCAL SYSTEM
      CLOCK. If the clocks are out of sync by more than a few seconds, THIS
      CODE WILL START DOING STRANGE THINGS."

   ❌ BUG 2 (the deeper one): even using a monotonic clock, "THE CODE ASSUMES
      THAT VERY LITTLE TIME PASSES between the point that it CHECKS THE TIME
      and the time when THE REQUEST IS PROCESSED."

      ┌────────────────────────────────────────────────────────────────┐
      │  if (lease.isValid())  ←─── the CHECK                          │
      │                                                                │
      │       ⏸  ← A 15-SECOND PAUSE CAN HAPPEN RIGHT HERE  ⏸          │
      │                                                                │
      │       process(request);  ←─── the USE                          │
      └────────────────────────────────────────────────────────────────┘
      "there is NOTHING TO TELL THIS THREAD THAT IT WAS PAUSED FOR SO LONG."
```

### The seven ways a thread can freeze

```
   ① GARBAGE COLLECTION — "these 'STOP-THE-WORLD' GC PAUSES have sometimes
      been known to LAST FOR SEVERAL MINUTES! Even so-called 'concurrent'
      collectors like the HotSpot JVM's CMS CANNOT FULLY RUN IN PARALLEL."
   ② VM SUSPEND/RESUME — "can occur AT ANY TIME in a process execution, and
      can last for AN ARBITRARY LENGTH OF TIME." Used for LIVE MIGRATION.
   ③ END-USER DEVICES — "when the user CLOSES THE LID OF THEIR LAPTOP."
   ④ CONTEXT SWITCHES — OS or hypervisor. In a VM, CPU time spent in other
      VMs is called STEAL TIME.
   ⑤ SYNCHRONOUS DISK I/O — "in many languages, disk access can happen
      SURPRISINGLY, even if the code doesn't explicitly mention file access —
      the Java CLASSLOADER LAZILY LOADS CLASS FILES when first used, WHICH
      COULD HAPPEN AT ANY TIME." And if the disk is network-attached (EBS),
      you inherit network variability too.
   ⑥ SWAPPING / PAGE FAULTS — "a SIMPLE MEMORY ACCESS may result in a page
      fault." In extreme cases, THRASHING. (Often disabled on servers.)
   ⑦ SIGSTOP — "which you can do for example by PRESSING CTRL+Z IN A SHELL…
      you can imagine the signal being SENT ACCIDENTALLY by an operations
      engineer."
```

> **🔑 The framing that makes this click:** *"The problem is similar to making MULTI-THREADED CODE on a single machine THREAD-SAFE: you can't assume anything about timing, because arbitrary context switches and parallelism may occur."*
>
> **"Unfortunately, these tools DON'T DIRECTLY TRANSLATE to distributed systems, because A DISTRIBUTED SYSTEM HAS NO SHARED MEMORY — only messages sent over an unreliable network."**
>
> **"A node must assume that its execution can be paused for a significant length of time AT ANY POINT, EVEN IN THE MIDDLE OF A FUNCTION. During the pause, THE REST OF THE WORLD KEEPS MOVING, and may even DECLARE THE PAUSED NODE DEAD. Eventually the paused node may continue running, WITHOUT EVEN NOTICING THAT IT WAS ASLEEP."**

### Response time guarantees and real-time systems

```
   HARD REAL-TIME SYSTEMS: "computers that control AIRCRAFT, ROCKETS, ROBOTS,
   CARS… there is a SPECIFIED DEADLINE by which the software MUST respond."

   "If your car's onboard sensors detect that you are currently experiencing a
    crash, YOU WOULDN'T WANT THE RELEASE OF THE AIRBAG TO BE DELAYED DUE TO AN
    INOPPORTUNE GC PAUSE IN THE AIRBAG RELEASE SYSTEM."

   WHAT IT TAKES — support at EVERY level of the stack:
      • a REAL-TIME OPERATING SYSTEM (RTOS) with guaranteed CPU allocation
      • library functions must DOCUMENT WORST-CASE EXECUTION TIMES
      • dynamic memory allocation RESTRICTED OR DISALLOWED
      • "an ENORMOUS AMOUNT of testing and measurement"

   ⚠️ "REAL-TIME IS NOT THE SAME AS HIGH PERFORMANCE — in fact, real-time
      systems may have LOWER THROUGHPUT, since they have to prioritize timely
      responses above all else."

   ➜ "For most server-side data processing systems, real-time requirements are
     SIMPLY NOT ECONOMICAL OR APPROPRIATE."
```

### Limiting the impact of garbage collection — two pragmatic tricks

```
   ① TREAT GC PAUSES LIKE BRIEF PLANNED OUTAGES
      "If the runtime can WARN the application that a node soon requires a GC
       pause, the application can STOP SENDING NEW REQUESTS to that node, wait
       for it to finish outstanding requests, and THEN perform the GC while no
       requests are in progress."
      ➜ "This HIDES GC PAUSES FROM CLIENTS, and REDUCES THE HIGH PERCENTILES
        of response time." (Used in latency-sensitive financial trading.)

   ② USE THE COLLECTOR ONLY FOR SHORT-LIVED OBJECTS, and RESTART PROCESSES
      PERIODICALLY before they accumulate enough long-lived objects to require
      a full GC. "One node can be restarted at a time, and traffic can be
      shifted away before the planned restart, LIKE IN A ROLLING UPGRADE."
```

---
---

## 10. Knowledge, truth and lies

> **"A node in the network CANNOT KNOW ANYTHING FOR SURE — it can only MAKE GUESSES based on the messages it receives (or doesn't receive). If a remote node doesn't respond, there is NO WAY OF KNOWING WHAT STATE IT IS IN, because PROBLEMS IN THE NETWORK CANNOT RELIABLY BE DISTINGUISHED FROM PROBLEMS AT A NODE."**
>
> **"Discussions of these systems BORDER ON THE PHILOSOPHICAL: What do we know to be true or false in our system? How sure can we be of that knowledge, if the mechanisms for perception and measurement are unreliable?"**

### The truth is defined by the majority — three nightmares

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  NIGHTMARE 1 — THE ASYMMETRIC FAULT                                   ║
   ║  A node RECEIVES all messages but its OUTGOING messages are dropped.  ║
   ║  "the semi-disconnected node is DRAGGED TO THE GRAVEYARD, KICKING AND ║
   ║   SCREAMING 'I'M NOT DEAD!' — but since nobody can hear its           ║
   ║   screaming, THE FUNERAL PROCESSION CONTINUES WITH STOIC              ║
   ║   DETERMINATION."                                                     ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  NIGHTMARE 2 — the node NOTICES its messages aren't acknowledged and  ║
   ║  realizes there's a network fault. "Nevertheless, the node is WRONGLY ║
   ║  DECLARED DEAD by the other nodes, and it CANNOT DO ANYTHING ABOUT    ║
   ║  IT."                                                                 ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  NIGHTMARE 3 — THE GC PAUSE                                           ║
   ║  A one-minute stop-the-world pause. The others "declare the node dead ║
   ║  and LOAD IT ONTO THE HEARSE. Finally the GC finishes… the supposedly ║
   ║  dead node SUDDENLY RAISES ITS HEAD OUT OF THE COFFIN, IN FULL        ║
   ║  HEALTH, AND STARTS CHEERFULLY CHATTING WITH BYSTANDERS."             ║
   ║  "from ITS perspective, HARDLY ANY TIME PASSED."                      ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   🔑 THE MORAL: "A NODE CANNOT NECESSARILY TRUST ITS OWN JUDGMENT OF A
     SITUATION… If a MAJORITY of nodes declares another node dead, then IT
     MUST BE CONSIDERED DEAD, EVEN IF THAT NODE STILL VERY MUCH FEELS ALIVE.
     The individual node MUST ABIDE BY THE MAJORITY DECISION, AND STEP DOWN."
```

### 💻 Why a majority is safe — I verified it exhaustively

```
   n=3: majority=2, tolerates 1 failure,  two DISJOINT majorities possible? NO
   n=5: majority=3, tolerates 2 failures, two DISJOINT majorities possible? NO
   n=7: majority=4, tolerates 3 failures, two DISJOINT majorities possible? NO
```

I enumerated **every pair of majority subsets** for each n and checked for disjointness. There are none. **"There can only be ONE majority in the system — there cannot be two majorities with conflicting decisions at the same time."** That's not a heuristic, it's the pigeonhole principle, and it's the foundation everything in Chapter 9 is built on.

---

## 11. The leader and the lock — fencing tokens

```
   Systems frequently require THERE TO BE ONLY ONE of some thing:
      • only one node may be LEADER for a partition (avoid split brain)
      • only one client may HOLD THE LOCK for a resource
      • only one user may REGISTER A PARTICULAR USERNAME

   ⚠️ "even if a node BELIEVES that it is 'THE CHOSEN ONE', THAT DOESN'T
      NECESSARILY MEAN THE MAJORITY OF NODES AGREES!"
```

### 🔷 Figure 8-4 — Incorrect distributed lock: the file gets corrupted

```
   Lock       ├── lock held by client 1 ──────┤├── lock held by client 2 ──┤
   service    ●────●                    ●          ●────●           TIME ──►
             get   ok                 lease       get   ok
            lease                    expired     lease
              │                          │
   Client 1 ──●══════ STOP-THE-WORLD GC PAUSE ══════════●───────────────────►
                                                    write data
                                                        │
   Client 2 ──────────────────────────────●─────────────┼───────────────────►
                                      write data        │
                                          │   ▲ ok      │
                                          ▼   │         ▼
   Storage ───────────────────────────────●───●─────────●───────────────────►
                                                        💥 BOTH CLIENTS WROTE.
                                                           FILE CORRUPTED.

   ⚑ "The bug is NOT THEORETICAL: HBase USED TO HAVE THIS PROBLEM."
```

### 🔷 Figure 8-5 — Fencing tokens make it safe

```
   Lock       ├── lock held by client 1 ──────┤├── lock held by client 2 ──┤
   service    ●────●                    ●          ●────●           TIME ──►
             get   ok,                lease       get   ok,
            lease  TOKEN: 33         expired     lease  TOKEN: 34
              │                          │
   Client 1 ──●══════ STOP-THE-WORLD GC PAUSE ══════════●───────────────────►
                                                   write, token: 33
                                                        │
   Client 2 ──────────────────────────────●─────────────┼───────────────────►
                                     write, token: 34   │
                                          │   ▲ ok      ▼
   Storage ───────────────────────────────●───●─────────✗───────────────────►
                                                    REJECTED:
                                                    OLD TOKEN (33 < 34)

   The token is "A NUMBER THAT INCREASES EVERY TIME A LOCK IS GRANTED."
   If ZooKeeper is the lock service, the transaction ID `zxid` or node version
   `cversion` can be used — "they are guaranteed to be MONOTONICALLY
   INCREASING."
```

### 🔑 The crucial requirement, easy to get wrong

> **"Note this requires THE RESOURCE ITSELF to take an active role in checking tokens, and rejecting any writes on which the token has gone backwards — IT IS NOT SUFFICIENT TO RELY ON CLIENTS CHECKING THEIR LOCK STATUS THEMSELVES."**
>
> **"Checking a token on the server side may seem like a downside, but it is ARGUABLY A GOOD THING: it is UNWISE FOR A SERVICE TO ASSUME THAT ITS CLIENTS WILL ALWAYS BE WELL-BEHAVED, because the clients are often run by people whose priorities are VERY DIFFERENT from the priorities of the people running the service."**

*(The reference here is Caitie McCaffrey's talk, "Clients are jerks: aka how Halo 4 DoSed the services at launch & how we survived.")*

### 💻 I ran the book's lease loop with a 40-second pause, both ways

```
--- Figure 8-4: WITHOUT fencing tokens ---
   t=  5.0s  lease has 25s left -> looks valid
   t= 45.0s  *** STOP-THE-WORLD GC PAUSE of 40s -- thread frozen ***
   t= 45.0s  meanwhile: lease EXPIRED, client 2 took over with token 34
   t= 45.0s  thread resumes and writes, still holding token 33
   t= 45.0s  write ACCEPTED  💥 FILE CORRUPTED (two writers)

--- Figure 8-5: WITH fencing tokens ---
   t=  5.0s  lease has 25s left -> looks valid
   t= 45.0s  *** STOP-THE-WORLD GC PAUSE of 40s -- thread frozen ***
   t= 45.0s  meanwhile: lease EXPIRED, client 2 took over with token 34
   t= 45.0s  thread resumes and writes, still holding token 33
   t= 45.0s  STORAGE REJECTS: token 33 < highest seen 34  ✅ SAFE
```

**Note that client 1 behaves identically in both runs.** It checked its lease, it believed itself valid, it wrote. Nothing about the *client* changed — the safety came entirely from the *storage layer* refusing a stale token. That's the whole argument for server-side checking.

---

## 12. Byzantine faults

> **"In this book we assume that nodes are UNRELIABLE BUT HONEST: they may be slow or never respond, and their state may be outdated, but we assume that IF A NODE DOES RESPOND, IT IS TELLING THE 'TRUTH'."**
>
> **"If a node sends untrue messages to other nodes, that is known as a BYZANTINE FAULT."**

```
   ⚠️ FENCING TOKENS DO NOT PROTECT AGAINST THIS:
   "if the node DELIBERATELY wanted to subvert the system's guarantees, it
    could EASILY DO SO BY SENDING MESSAGES WITH A FAKE FENCING TOKEN."
```

### 📦 Sidebar: The Byzantine Generals Problem

```
   THE TWO GENERALS PROBLEM: two generals must agree on a battle plan, camped
   at different sites, communicating ONLY BY MESSENGER — and "the messengers
   sometimes get DELAYED OR CAPTURED (LIKE PACKETS IN A NETWORK)."

   THE BYZANTINE VERSION: n generals must agree, but "THERE ARE SOME TRAITORS
   IN THEIR MIDST… the traitors may try to DECEIVE AND CONFUSE the others by
   sending fake or untrue messages (WHILE TRYING TO REMAIN UNDISCOVERED). IT
   IS NOT KNOWN IN ADVANCE WHO THE TRAITORS ARE."

   📖 "There ISN'T ANY HISTORIC EVIDENCE that the generals of Byzantium were
      any more prone to intrigue and conspiracy than those elsewhere. Rather,
      the name is derived from BYZANTINE in the sense of EXCESSIVELY
      COMPLICATED, BUREAUCRATIC, DEVIOUS."
      Lamport "wanted to choose a nationality that would not offend any
      readers, and he was advised that calling it THE ALBANIAN GENERALS
      PROBLEM WAS NOT SUCH A GOOD IDEA."
```

### When Byzantine fault tolerance actually matters

```
   ✅ AEROSPACE — "data in a computer's memory or CPU register could become
      CORRUPTED BY RADIATION… a system failure would be VERY EXPENSIVE (an
      aircraft crashing, or a rocket colliding with the ISS)."
   ✅ MULTI-ORGANIZATION SYSTEMS — "some participants may attempt to CHEAT OR
      DEFRAUD others… systems like the BITCOIN BLOCKCHAIN can be considered a
      way of getting MUTUALLY UNTRUSTING PARTIES to agree whether a
      transaction happened, WITHOUT RELYING ON A CENTRAL AUTHORITY."

   ❌ AND WHY IT USUALLY DOESN'T:
      "In your datacenter, ALL THE NODES ARE CONTROLLED BY YOUR ORGANIZATION,
       and RADIATION LEVELS ARE LOW ENOUGH that memory corruption is not a
       major problem… In most server-side data systems, THE COST OF DEPLOYING
       BYZANTINE FAULT TOLERANT SOLUTIONS MAKES THEM IMPRACTICAL."
```

### 💻 The two things BFT cannot do for you

```
A node that is merely CONFUSED is stopped by fencing:
   client 1 (paused, honest): sends token 33 -> REJECTED ✅

A node that is MALICIOUS simply lies:
   attacker: sends token 9999 (fabricated) -> ACCEPTED 💥

BFT needs a SUPERMAJORITY > 2/3 correct:
   total nodes n     max Byzantine f
               4                   1
               7                   2
             100                  33
```

> ⚠️ **BFT does not save you from bugs:** *"A bug in the software could be regarded as a Byzantine fault, but IF YOU DEPLOY THE SAME SOFTWARE TO ALL NODES, THEN A BYZANTINE FAULT TOLERANT ALGORITHM CANNOT SAVE YOU… To use this approach against bugs, you would have to have FOUR INDEPENDENT IMPLEMENTATIONS of the same software, and hope that a bug only appears in one of the four."*
>
> ⚠️ **Nor from attackers:** *"in most systems, IF AN ATTACKER CAN COMPROMISE ONE NODE, THEY CAN PROBABLY COMPROMISE ALL OF THEM, because they are probably running the same software. Thus, TRADITIONAL MECHANISMS (authentication, access control, encryption, firewalls) continue to be the main protection."*

### Weak forms of lying — the pragmatic middle ground

```
   "Not full-blown Byzantine fault tolerance, as they would NOT WITHSTAND A
    DETERMINED ADVERSARY, but they are nevertheless SIMPLE AND PRAGMATIC STEPS
    towards better reliability."

   ① CHECKSUMS IN THE APPLICATION-LEVEL PROTOCOL — "corrupted packets are
      usually caught by the checksums built into TCP and UDP, BUT SOMETIMES
      THEY EVADE DETECTION."
   ② INPUT SANITIZATION — range checks, string-size limits to prevent DoS
      through large memory allocations.
   ③ MULTIPLE NTP SERVERS — "the client contacts all of them, estimates their
      errors, and CHECKS THAT A MAJORITY AGREE. A misconfigured NTP server is
      detected as AN OUTLIER and EXCLUDED."  ← a majority quorum, for clocks
```

---

## 13. System models and reality

> **"Algorithms need to be written in a way that DOES NOT DEPEND TOO HEAVILY ON THE DETAILS of the hardware and software configuration. This requires that we FORMALIZE the kinds of fault we expect. We do this by defining a SYSTEM MODEL, which is an ABSTRACTION THAT DESCRIBES WHAT THINGS AN ALGORITHM MAY ASSUME."**

```
   ╔═══════════════════ TIMING ASSUMPTIONS ════════════════════════════════╗
   ║  SYNCHRONOUS                                                          ║
   ║    bounded network delay, bounded pauses, bounded clock error.        ║
   ║    ⚠️ "NOT A REALISTIC MODEL of most practical systems."               ║
   ╟───────────────────────────────────────────────────────────────────────╢
   ║  PARTIALLY SYNCHRONOUS                                                ║
   ║    "behaves like a synchronous system MOST OF THE TIME, but it        ║
   ║     SOMETIMES EXCEEDS THE BOUNDS… When this happens, network delay,   ║
   ║     pauses and clock error may become ARBITRARILY LARGE."             ║
   ║    ✅ "This is A REALISTIC MODEL OF MANY SYSTEMS."                     ║
   ╟───────────────────────────────────────────────────────────────────────╢
   ║  ASYNCHRONOUS                                                         ║
   ║    "not allowed to make ANY timing assumptions whatsoever — in fact,  ║
   ║     IT DOES NOT EVEN HAVE A CLOCK (so it cannot use timeouts).        ║
   ║     …VERY RESTRICTIVE."                                               ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ╔═══════════════════ NODE FAILURE MODELS ═══════════════════════════════╗
   ║  CRASH-STOP      a node fails ONLY by crashing, and "thereafter that  ║
   ║                  node is GONE FOREVER — IT NEVER COMES BACK."         ║
   ║  CRASH-RECOVERY  nodes may crash and START RESPONDING AGAIN after an  ║
   ║                  unknown time. STABLE STORAGE survives; IN-MEMORY     ║
   ║                  STATE IS LOST.                                       ║
   ║  BYZANTINE       "nodes may do ABSOLUTELY ANYTHING."                  ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ➜ "For modeling real systems, THE PARTIALLY SYNCHRONOUS MODEL WITH
     CRASH-RECOVERY FAULTS IS GENERALLY THE MOST USEFUL MODEL."
```

### Correctness: the fencing-token example

```
   UNIQUENESS          No two requests for a fencing token return the same value.
   MONOTONIC SEQUENCE  If request x returned tx and y returned ty, and x
                       completed before y began, then tx < ty.
   AVAILABILITY        A node that requests a token and does not crash
                       EVENTUALLY receives a response.
```

### 🔑 Safety vs liveness

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  SAFETY                           ║  LIVENESS                         ║
   ║  "nothing bad happens"            ║  "something good EVENTUALLY       ║
   ║                                   ║   happens"                        ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  "If violated, WE CAN POINT AT A  ║  "It may NOT HOLD at some point   ║
   ║   PARTICULAR POINT IN TIME at     ║   in time, BUT THERE IS ALWAYS    ║
   ║   which it was broken."           ║   HOPE THAT IT MAY BE SATISFIED   ║
   ║                                   ║   IN FUTURE."                     ║
   ║  ⚠️ "After a safety property has   ║                                   ║
   ║     been violated, THE VIOLATION  ║  🔑 "A GIVEAWAY is that liveness  ║
   ║     CANNOT BE UNDONE — THE DAMAGE ║    properties often include the   ║
   ║     IS ALREADY DONE."             ║    word 'EVENTUALLY'."            ║
   ║                                   ║    (And yes — EVENTUAL CONSISTENCY║
   ║                                   ║     IS A LIVENESS PROPERTY.)      ║
   ╠═══════════════════════════════════╩═══════════════════════════════════╣
   ║  ➜ THE PRACTICAL PAYOFF:                                              ║
   ║  SAFETY must hold IN ALL POSSIBLE SITUATIONS — "even if all nodes     ║
   ║  crash, or the entire network fails, the algorithm must ensure IT     ║
   ║  DOES NOT RETURN A WRONG RESULT."                                     ║
   ║  LIVENESS is ALLOWED CAVEATS — "a request needs to receive a response ║
   ║  ONLY IF a majority of nodes is not crashed, and ONLY IF the network  ║
   ║  eventually recovers."                                                ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 💻 The test, applied

```
property                                                  kind
──────────────────────────────────────────────────────────────
uniqueness: no two requests get the same token          SAFETY
monotonic sequence: x before y implies tx < ty          SAFETY
availability: a non-crashed node EVENTUALLY responds    LIVENESS
eventual consistency: replicas EVENTUALLY converge      LIVENESS

CAN YOU POINT AT A SPECIFIC MOMENT IT WAS VIOLATED?
   SAFETY   yes — the exact call that returned a duplicate
   SAFETY   yes — the exact out-of-order pair
   LIVENESS no — it might still arrive
   LIVENESS no — they might still converge
```

### ⚠️ Mapping models to the real world — where the abstraction leaks

> **"When implementing an algorithm in practice, THE MESSY FACTS OF REALITY COME BACK TO BITE YOU AGAIN."**

```
   Crash-recovery assumes DATA IN STABLE STORAGE SURVIVES CRASHES. But:
   • what if the data on disk is CORRUPTED, or WIPED OUT due to hardware
     error or misconfiguration?
   • what if a server has a FIRMWARE BUG and FAILS TO RECOGNIZE ITS HARD
     DRIVES ON REBOOT, even though they're correctly attached?

   🔑 "Quorum algorithms RELY ON A NODE REMEMBERING THE DATA THAT IT CLAIMS TO
     HAVE STORED. IF A NODE MAY SUFFER FROM AMNESIA AND FORGET PREVIOUSLY
     STORED DATA, THAT BREAKS THE QUORUM CONDITION, AND THUS BREAKS THE
     CORRECTNESS OF THE ALGORITHM."
```

> **And the most human sentence in the book:** *"a real implementation may still have to include code to handle the case where something happens that was assumed to be impossible, even if that handling boils down to `printf("Sucks to be you")` and `exit(666)` — i.e. letting a human operator clean up the mess. **(This is arguably the difference between computer science and software engineering.)**"*
>
> **But the defence of theory:** *"That is NOT TO SAY that theoretical, abstract system models are worthless — QUITE THE OPPOSITE… theoretical analysis can UNCOVER PROBLEMS IN AN ALGORITHM THAT MIGHT REMAIN HIDDEN FOR A LONG TIME in a real system, and that only come to bite you when your assumptions are defeated due to unusual circumstances. **THEORETICAL ANALYSIS AND EMPIRICAL TESTING ARE EQUALLY IMPORTANT.**"*

---

## 14. Chapter Summary

```
   THE THREE PROBLEMS, all of them PARTIAL FAILURES:

   ① "Whenever you try to send a packet over the network, IT MAY BE LOST OR
      ARBITRARILY DELAYED. Likewise, THE REPLY may be lost or delayed, so if
      you don't get a reply, YOU HAVE NO IDEA WHETHER THE MESSAGE GOT THROUGH."

   ② "A node's clock may be SIGNIFICANTLY OUT OF SYNC with other nodes
      (DESPITE YOUR BEST EFFORTS TO SET UP NTP), it may SUDDENLY JUMP FORWARD
      OR BACK IN TIME, and relying on it is dangerous because YOU MOST LIKELY
      DON'T HAVE A GOOD MEASURE OF YOUR CLOCK'S ERROR INTERVAL."

   ③ "A process may PAUSE FOR A SUBSTANTIAL AMOUNT OF TIME AT ANY POINT in its
      execution, BE DECLARED DEAD by other nodes, and then COME BACK TO LIFE
      AGAIN WITHOUT REALIZING THAT IT WAS PAUSED."
```

> **On detection:** *"To tolerate faults, THE FIRST STEP IS TO DETECT THE FAULT, BUT EVEN THAT IS HARD… timeouts CAN'T DISTINGUISH between network and node failures. Moreover, sometimes a node can be in a DEGRADED STATE: a Gigabit network interface could suddenly drop to **1 KILOBIT/S** throughput due to a driver bug. Such a system that is **'LIMPING', BUT NOT DEAD, CAN BE EVEN MORE DIFFICULT TO DEAL WITH THAN A CLEANLY FAILED NODE.**"*

> **On what's left:** *"there is NO GLOBAL VARIABLE, NO SHARED MEMORY, NO COMMON KNOWLEDGE or any other kind of shared state between the machines. **Nodes can't even agree what time it is, let alone anything more profound.** The only way information can flow is by sending it over the unreliable network. Major decisions CANNOT BE SAFELY MADE BY A SINGLE NODE, so we require protocols that enlist the help from other nodes and try to get A MAJORITY QUORUM to agree."*

### 💡 The pragmatic advice

> **"Distributed systems engineers will often regard a problem as TRIVIAL if it can be solved on a single computer, and indeed A SINGLE COMPUTER CAN DO A LOT NOWADAYS. IF YOU CAN AVOID OPENING PANDORA'S BOX, AND SIMPLY KEEP THINGS ON A SINGLE MACHINE, IT IS GENERALLY WORTH DOING SO."**
>
> **But:** *"scalability is NOT THE ONLY REASON for wanting to use a distributed system. FAULT TOLERANCE and LOW LATENCY are equally important goals, and those things CANNOT BE ACHIEVED WITH A SINGLE NODE."*

### And the final tangent

> *"We saw that [unreliability] ISN'T [a law of nature]: it is possible to give hard real-time response guarantees and bounded delay in networks, BUT DOING SO IS VERY EXPENSIVE and results in LOWER UTILIZATION of hardware resources. **Most non-safety-critical systems choose CHEAP AND UNRELIABLE over EXPENSIVE AND RELIABLE.**"*
>
> *"By contrast, distributed systems can RUN FOREVER without being interrupted at the service level, because all faults and maintenance can be handled at the node level — **at least in theory. (In practice, if a bad configuration change is rolled out to all nodes, that will still bring a distributed system to its knees.)**"*

---

# 15. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.8 — THE TROUBLE WITH DISTRIBUTED SYSTEMS                             ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  SINGLE COMPUTER = deterministic, "fully functional OR entirely broken" —     ║
║  a DELIBERATE design choice: crash rather than return a wrong result.         ║
║  DISTRIBUTED = PARTIAL FAILURE, and it is NON-DETERMINISTIC. "You may not     ║
║  even know whether something succeeded or not." ← THIS is what makes it hard. ║
║  HPC handles faults by ESCALATING TO TOTAL FAILURE (checkpoint + restart      ║
║  the whole cluster). Cloud must TOLERATE them: online service, commodity      ║
║  hardware, "something is ALWAYS broken", rolling upgrades, the internet.      ║
║  You CAN build reliable from unreliable (ECC, TCP-over-IP) but there's always ║
║  a limit — "TCP cannot magically remove delays."                              ║
║  ➜ "SUSPICION, PESSIMISM AND PARANOIA PAY OFF."                               ║
║                                                                               ║
║  ── UNRELIABLE NETWORKS ────────────────────────────────────────────────────  ║
║  6 failure modes; 3 are INDISTINGUISHABLE (request lost / node down /         ║
║  response lost). MEASURED: 4 realities → 1 observation ("timeout"), and in    ║
║  2 of them THE OPERATION ACTUALLY RAN → retries need IDEMPOTENCE.             ║
║  PRACTICE: ~12 faults/month in one datacenter · redundant gear does NOT help  ║
║  much because HUMAN ERROR is a major cause · a NIC that drops all INBOUND     ║
║  but sends OUTBOUND fine · a switch upgrade delaying packets >1 MINUTE.       ║
║  ⚠️ "network partition"=netsplit ≠ partition(shard). Book says NETWORK FAULT. ║
║  DETECTION: RST/FIN, crash-notify scripts, switch mgmt, ICMP — none reliable. ║
║  "If you want to be sure a request succeeded you need a POSITIVE RESPONSE     ║
║   FROM THE APPLICATION ITSELF."                                               ║
║  TIMEOUTS: MEASURED trade-off — 50ms → 2.9% false deaths; 5s → 0% but 5s of   ║
║  downtime. NO CORRECT VALUE. Premature death SHIFTS LOAD → CASCADING FAILURE. ║
║  2d+r would work IF delays and handling times were bounded. THEY AREN'T.      ║
║  QUEUEING is the cause: switch queues · OS run queue · VM pauses · TCP flow   ║
║  control · TCP retransmit. MEASURED M/M/1: ρ=0.5→2x, ρ=0.95→20x, ρ=0.99→100x. ║
║  NOISY NEIGHBORS in multitenant clouds make it worse and invisible.           ║
║  ✅ PHI ACCRUAL (Akka, Cassandra): continuous SUSPICION LEVEL adapted to the   ║
║     observed jitter, not a binary threshold. MEASURED: 130ms silence → φ=4.5. ║
║  CIRCUIT-SWITCHED (telephone, ISDN 16 bits/250µs) = BOUNDED DELAY, no queue.  ║
║  PACKET-SWITCHED = chosen for BURSTY TRAFFIC: circuits would make you GUESS   ║
║  a bandwidth allocation, wasting capacity.                                    ║
║  🔑 "VARIABLE DELAYS ARE NOT A LAW OF NATURE, BUT THE RESULT OF A COST-BENEFIT ║
║     TRADE-OFF" — static partitioning buys latency guarantees with UTILIZATION.║
║                                                                               ║
║  ── UNRELIABLE CLOCKS ──────────────────────────────────────────────────────  ║
║  TIME-OF-DAY (currentTimeMillis): NTP-synced, comparable across machines, BUT ║
║     CAN JUMP BACKWARDS, ignores leap seconds → NEVER USE FOR DURATIONS.       ║
║  MONOTONIC (nanoTime): always forwards, NTP can only SLEW it (±0.05%), but    ║
║     the ABSOLUTE VALUE IS MEANINGLESS → never compare across machines.        ║
║  8 FAILURE MODES: 200ppm drift (MEASURED: 6ms/30s, 17.3s/day — matches book) ·║
║     forcible reset · firewalled NTP going unnoticed · ≥35ms error over the    ║
║     internet · misconfigured servers off by HOURS · LEAP SECONDS (crashed     ║
║     many systems; fix = SMEARING) · VM clock jumps · users faking their clock.║
║  ⚠️ A BROKEN CLOCK LOOKS FINE. "More likely SILENT AND SUBTLE DATA LOSS than a ║
║     dramatic crash." → MONITOR CLOCK OFFSETS; evict drifting nodes.           ║
║  LWW (Fig 8-3): with only 3ms skew, the CAUSALLY LATER write gets the LOWER   ║
║     timestamp and IS SILENTLY DISCARDED. Better NTP can't fix it — accuracy   ║
║     is bounded by the very network delay you're trying to measure.            ║
║     ➜ LOGICAL CLOCKS (counters) order events correctly WITH NO CLOCK AT ALL.  ║
║  CONFIDENCE INTERVALS: a reading is A RANGE, not a point. "If you only know   ║
║     the time ±100ms, the microsecond digits are MEANINGLESS." Most APIs don't ║
║     expose it; TRUETIME returns [earliest, latest]. SPANNER COMMIT-WAIT: idle ║
║     for the interval width so intervals can't overlap → GPS/atomic clocks per ║
║     DC, ~7ms uncertainty. Correctness bought with LATENCY and MONEY.          ║
║                                                                               ║
║  ── PROCESS PAUSES ─────────────────────────────────────────────────────────  ║
║  The lease loop has TWO bugs: it compares a REMOTE expiry to a LOCAL clock,   ║
║  and it assumes NO TIME PASSES between the CHECK and the USE.                 ║
║  7 CAUSES: stop-the-world GC (MINUTES) · VM suspend/live-migration · laptop   ║
║  lid · context switch / STEAL TIME · sync disk I/O (Java CLASSLOADER loads    ║
║  lazily!) · page faults & THRASHING · SIGSTOP (Ctrl+Z).                       ║
║  "A node must assume it can be paused AT ANY POINT, EVEN MID-FUNCTION.        ║
║   During the pause THE REST OF THE WORLD KEEPS MOVING."                       ║
║  HARD REAL-TIME exists (airbags) but needs an RTOS, documented WCETs, no      ║
║  dynamic allocation — expensive, and REAL-TIME ≠ HIGH PERFORMANCE.            ║
║  MITIGATE GC: treat pauses as PLANNED OUTAGES and drain the node first; or    ║
║  restart processes before long-lived objects accumulate (rolling-upgrade).    ║
║                                                                               ║
║  ── KNOWLEDGE, TRUTH AND LIES ──────────────────────────────────────────────  ║
║  A semi-disconnected node is "DRAGGED TO THE GRAVEYARD KICKING AND SCREAMING".║
║  A GC'd node "RAISES ITS HEAD OUT OF THE COFFIN" with no idea time passed.    ║
║  🔑 TRUTH IS DEFINED BY THE MAJORITY. VERIFIED EXHAUSTIVELY: for n=3,5,7 NO   ║
║     TWO MAJORITY SETS ARE EVER DISJOINT → only one majority can exist.        ║
║  FENCING TOKENS: monotonically increasing number issued with each lock;       ║
║     storage REJECTS any write with a lower token (Fig 8-4 corruption → 8-5    ║
║     safety; HBase really had this bug). ZooKeeper zxid/cversion work.         ║
║     ⚠️ THE RESOURCE must check — "not sufficient to rely on clients checking   ║
║        their lock status themselves." Clients are jerks.                      ║
║  BYZANTINE = a node LIES. Fencing does NOT stop it (just forge a token).      ║
║     Needed for aerospace (radiation) and mutually-untrusting parties          ║
║     (Bitcoin). Needs n > 3f. ❌ Does NOT protect against BUGS (same binary     ║
║     everywhere) or ATTACKERS (compromise one → compromise all).               ║
║     WEAK LYING defences worth having: app-level checksums, input sanitization,║
║     MULTIPLE NTP SERVERS with majority agreement.                             ║
║                                                                               ║
║  ── SYSTEM MODELS ──────────────────────────────────────────────────────────  ║
║  TIMING:  synchronous (unrealistic) · PARTIALLY SYNCHRONOUS (realistic) ·     ║
║           asynchronous (no clock at all; very restrictive)                    ║
║  NODES:   crash-stop · CRASH-RECOVERY (stable storage survives) · Byzantine   ║
║  ➜ USE: PARTIALLY SYNCHRONOUS + CRASH-RECOVERY.                               ║
║  SAFETY = "nothing bad happens"; you can POINT AT THE MOMENT it broke, and    ║
║     IT CANNOT BE UNDONE. Must hold IN ALL SITUATIONS.                         ║
║  LIVENESS = "something good EVENTUALLY happens" (the giveaway word). May be   ║
║     unsatisfied now but satisfied later. CAVEATS ALLOWED (majority up,        ║
║     network eventually recovers). Eventual consistency is LIVENESS.           ║
║  ⚠️ Models leak: what if stable storage suffers AMNESIA? That breaks quorums.  ║
║     Real code still needs printf("Sucks to be you"); exit(666).               ║
║     "THEORETICAL ANALYSIS AND EMPIRICAL TESTING ARE EQUALLY IMPORTANT."       ║
║                                                                               ║
║  ➜ A 'LIMPING' node (1 Gbit NIC dropping to 1 kbit/s) is HARDER to handle     ║
║    than a cleanly dead one. And: IF YOU CAN KEEP IT ON ONE MACHINE, DO.       ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 16. ✅ Test yourself

1. **You send a request and get no response. What do you know?**
   → Only that you haven't received a response. You cannot distinguish a lost request, a dead node, a lost response, or a slow node — and in two of those cases the operation *did* execute. This is why retries need idempotence.

2. **Why is a short failure-detection timeout potentially worse than a long one?**
   → Because wrongly declaring a live node dead transfers its load to other nodes, adding network and CPU pressure. On a system already under strain that can push more nodes over the threshold, and in the extreme every node declares every other dead.

3. **Your p50 latency looks fine but p99 is terrible. What's the most likely mechanism?**
   → Queueing near saturation. My M/M/1 measurements at ρ=0.95 showed p50 at 13.8 and p99 at 111 — the median degrades gently while the tail explodes. It's the same curve behind Chapter 1's percentiles and Chapter 7's 2PL latency.

4. **Why can't better NTP synchronization fix last-write-wins?**
   → NTP's accuracy is bounded by network round-trip time, which is the very quantity that creates the ordering ambiguity. You'd need a clock source significantly more accurate than the delay you're measuring, which is circular. Use logical clocks instead.

5. **When should you use a monotonic clock rather than a time-of-day clock?**
   → For any duration: timeouts, response times, rate measurements. Time-of-day clocks can be stepped backwards by NTP, so `end - start` can go negative. Time-of-day is only for points in time that must mean something across machines.

6. **What's wrong with `if (lease.isValid()) process(request);`?**
   → Two things. It compares a remote machine's expiry to the local clock, and more fundamentally it assumes no time passes between the check and the use. A GC pause between those two lines means the thread acts on a lease that expired long ago.

7. **How do fencing tokens fix that, and where must the check live?**
   → Each lock grant carries a monotonically increasing number, and the *resource* rejects any write bearing a token lower than the highest it has seen. The check must be server-side — the paused client behaves identically either way and cannot detect its own staleness.

8. **A node insists it's alive and healthy, but four of five peers say it's dead. Who's right?**
   → The majority, operationally speaking. The node must step down. It may genuinely be healthy behind an asymmetric network fault, but a system that lets individual nodes override the majority admits split brain.

9. **Would Byzantine fault tolerance have protected you from a bug in your database?**
   → No, if you deployed the same binary everywhere. BFT needs more than two-thirds of nodes behaving correctly; a shared bug fails all of them simultaneously. You'd need four genuinely independent implementations.

10. **Is eventual consistency a safety or liveness property, and why does the distinction matter?**
    → Liveness — the word "eventually" gives it away. It matters because safety properties must hold in every situation an algorithm claims to handle, while liveness properties are allowed caveats such as "provided a majority is up and the network eventually recovers." Safety violations are permanent; liveness can still be satisfied later.

---

*All quoted material, figures and examples are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 8. Diagrams have been redrawn in ASCII from the book's originals. The simulations and all measured outputs are supplementary material I added; every number quoted was produced by running the accompanying code.*
