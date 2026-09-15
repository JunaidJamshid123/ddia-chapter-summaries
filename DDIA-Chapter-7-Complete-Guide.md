# DDIA — Chapter 7: Transactions
### Complete study guide — theory, every diagram redrawn, and a working MVCC database

> *Some authors have claimed that general two-phase commit is too expensive to support, because of the performance or availability problems that it brings. We believe it is better to have application programmers deal with performance problems due to over-use of transactions as bottlenecks arise, rather than always coding around the lack of transactions.*
> — James Corbett et al., *Spanner: Google's Globally-Distributed Database* (2012)

---

## 0. The map of this chapter

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  PART A — WHAT IS A TRANSACTION?                                            │
│     ACID (and why the C doesn't belong) · single- vs multi-object ·         │
│     handling aborts and retries                                             │
│                                                                             │
│  PART B — WEAK ISOLATION LEVELS, and the race conditions they miss          │
│                                                                             │
│     READ COMMITTED ──────► SNAPSHOT ISOLATION ──────► SERIALIZABLE          │
│     no dirty reads         + no read skew             + no lost updates     │
│     no dirty writes        (MVCC)                     + no write skew       │
│                                                       + no phantoms         │
│                                                                             │
│  PART C — THREE WAYS TO IMPLEMENT SERIALIZABILITY                           │
│                                                                             │
│     ① ACTUAL SERIAL EXECUTION   ② TWO-PHASE LOCKING   ③ SSI                 │
│       (pessimistic extreme)       (pessimistic)         (OPTIMISTIC)        │
│       single thread + stored      30 years of           new, promising,     │
│       procedures, in-memory       standard practice     PostgreSQL 9.1+     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why transactions exist at all

> **In the harsh reality of data systems, many things can go wrong:**

```
   • the database software or hardware may fail AT ANY TIME (including in the
     middle of a write operation)
   • the application may crash AT ANY TIME (including halfway through a series
     of operations)
   • interruptions in the network can cut off the application from the database,
     or one database node from another
   • several clients may write at the same time, OVERWRITING EACH OTHERS' CHANGES
   • a client may read data that DOESN'T MAKE SENSE because it has only
     partially been updated
   • RACE CONDITIONS between clients can cause surprising bugs
```

> **A transaction is a way for an application to group several reads and writes together into a LOGICAL UNIT.** Conceptually, all the reads and writes execute as one operation: either the entire transaction **succeeds (commit)** or it **fails (abort, rollback).** If it fails, the application can safely retry. **This makes error handling much simpler, because it doesn't need to worry about PARTIAL FAILURE.**

### 🔑 The framing that matters most

> **"Transactions are NOT A LAW OF NATURE; they were created with a purpose, namely in order to SIMPLIFY THE PROGRAMMING MODEL for applications accessing a database."** By using transactions, the application is **free to ignore certain potential error scenarios and concurrency issues, because the database takes care of them instead** (we call these **safety guarantees**).

```
   ⚠️ AND THE BALANCED VIEW, stated unusually forcefully:

   "There emerged a popular belief that transactions were the ANTITHESIS OF
    SCALABILITY… On the other hand, transactional guarantees are sometimes
    presented by database vendors as an ESSENTIAL REQUIREMENT for 'serious
    applications' with 'valuable data'.

                  BOTH VIEWPOINTS ARE PURE HYPERBOLE."
```

---

# PART A — THE SLIPPERY CONCEPT OF A TRANSACTION

## 1. The meaning of ACID

> Coined in **1983 by Theo Härder and Andreas Reuter** to establish precise terminology. **"However, in practice, one database's implementation of ACID does not equal another's… Today, when a system claims to be 'ACID compliant', it's unclear what guarantees you can actually expect. ACID has unfortunately become MOSTLY A MARKETING TERM."**

> 📖 And on the alternative: systems that don't meet ACID are sometimes called **BASE** — *Basically Available, Soft state, Eventual consistency.* **"This is even more vague than the definition of ACID. It seems that the only sensible definition of BASE is 'not ACID', i.e. it can mean almost anything you want."**

### A — Atomicity

```
   ⚠️ FIRST, WHAT IT IS NOT.
      In MULTITHREADED PROGRAMMING, "atomic" means no other thread can see
      the half-finished result of an operation.
      In ACID, ATOMICITY IS NOT ABOUT CONCURRENCY AT ALL — that's the I.

   ✅ WHAT IT IS:
      "ACID atomicity describes what happens if a client wants to make several
       writes, but A FAULT OCCURS AFTER SOME OF THE WRITES HAVE BEEN PROCESSED"
       — a process crashes, a network connection is interrupted, a disk becomes
       full, an integrity constraint is violated.

      → the transaction is ABORTED and the database must DISCARD OR UNDO any
        writes it made so far.

   WITHOUT ATOMICITY:
      "it's difficult to know which changes have taken effect and which
       haven't. The application could try again, but that risks MAKING THE
       SAME CHANGE TWICE."

   ➜ "The ability to ABORT a transaction on error, and have all writes from
     that transaction DISCARDED, is the defining feature of ACID atomicity.
     PERHAPS **ABORTABILITY** WOULD HAVE BEEN A BETTER TERM."
```

### C — Consistency (the letter that doesn't belong)

```
   THE WORD IS "TERRIBLY OVERLOADED" — three meanings already in this book:

   ┌────────────────────────┬─────────────────────────────────────────────┐
   │ Chapter 5              │ REPLICA CONSISTENCY / eventual consistency  │
   │ CAP theorem (Ch.9)     │ LINEARIZABILITY                             │
   │ ACID                   │ an APPLICATION-SPECIFIC notion of the       │
   │                        │ database being in a "good state"            │
   └────────────────────────┴─────────────────────────────────────────────┘

   ACID consistency = INVARIANTS that must always be true. e.g. "in an
   accounting system, credits and debits across all accounts must always
   be balanced."

   ⚠️ BUT: "this idea of consistency depends on THE APPLICATION'S notion of
      invariants, and IT'S THE APPLICATION'S RESPONSIBILITY to define its
      transactions correctly. THIS IS NOT SOMETHING THAT THE DATABASE CAN
      GUARANTEE: if you write bad data that violates your invariants, THE
      DATABASE CAN'T STOP YOU."

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  A, I and D are PROPERTIES OF THE DATABASE.                           ║
   ║  C is a PROPERTY OF THE APPLICATION.                                  ║
   ║             "Thus the letter C doesn't really belong in ACID."        ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   📖 Footnote: Joe Hellerstein has remarked that the C was "TOSSED IN TO MAKE
      THE ACRONYM WORK" in Härder and Reuter's paper.
```

### I — Isolation

### 🔷 Figure 7-1 — A race condition between two clients incrementing a counter

```
                  get counter      [42 + 1 = 43]      set counter = 43
                        │                                   │      TIME ────►
   User 1  ─────────────●───────────────────────────────────●─────────────────►
                        │  ▲ 42                             │  ▲ ok
                        ▼  │                                ▼  │
   Database ────────────●──●──────────●──────────────●──────●──●───────●───────►
                                      │  ▲ 42               │         ▲ ok
                                      ▼  │                  ▼         │
   User 2  ─────────────────────────●────●──────────────────────●─────●────────►
                                 get counter   [42+1=43]    set counter = 43

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE COUNTER SHOULD HAVE GONE 42 → 44 (two increments).               ║
   ║  IT ONLY WENT TO 43.                                                  ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

> **Isolation** means concurrently executing transactions are isolated from each other: **"they cannot step on each others' toes."** The textbooks formalize it as **serializability**: each transaction can pretend **it is the only transaction running on the entire database.**

```
   ⚠️ "However, IN PRACTICE, SERIALIZABLE ISOLATION IS RARELY USED, because it
      carries a PERFORMANCE PENALTY. Some popular databases such as Oracle 11g
      DON'T EVEN IMPLEMENT IT. In Oracle there is an isolation level called
      'serializable', but it actually implements SNAPSHOT ISOLATION, which is
      a WEAKER GUARANTEE."
```

### 💻 I reproduced Figure 7-1 with real threads on SQLite

```
5 threads × 200 increments, expected 1000

   read-modify-write in app code :   218   LOST 782 UPDATES
   atomic UPDATE value=value+1   :  1000   correct
```

**782 of 1000 increments vanished.** Not a rare edge case — under real contention the read-modify-write pattern loses *most* of its work. The one-line fix is the atomic operation.

### D — Durability

> **The promise that once a transaction has committed successfully, any data it has written will not be forgotten, even if there is a hardware fault or the database crashes.**

```
   single-node  → written to NON-VOLATILE STORAGE, usually plus a WRITE-AHEAD
                  LOG for recovery if on-disk structures are corrupted
   replicated   → successfully COPIED TO SOME NUMBER OF NODES

   ⚑ "In order to provide a durability guarantee, a database must WAIT until
     these writes or replications are COMPLETE before reporting a transaction
     as successfully committed."
```

### 📦 Sidebar: Replication and durability — "nothing is perfect"

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │ • write to disk + the machine dies → data isn't lost but it is       │
   │   INACCESSIBLE until you fix the machine. Replicas stay available.   │
   │ • a CORRELATED FAULT (power outage, a bug that crashes every node on │
   │   a particular input) can knock out ALL REPLICAS AT ONCE.            │
   │   → writing to disk is still relevant for in-memory databases.       │
   │ • ASYNC replication → recent writes lost when the leader fails.      │
   │ • "when the power is suddenly cut, SSDs in particular have been      │
   │   shown to sometimes VIOLATE THE GUARANTEES they are supposed to     │
   │   provide: EVEN fsync ISN'T GUARANTEED TO WORK CORRECTLY."           │
   │ • subtle storage-engine/filesystem interactions corrupt files after  │
   │   a crash.                                                           │
   │ • data on disk can become GRADUALLY CORRUPTED WITHOUT BEING DETECTED │
   │   — and by the time you notice, replicas AND recent backups may also │
   │   be corrupted.                                                      │
   │ • "if an SSD is disconnected from power, it can START LOSING DATA    │
   │   WITHIN A FEW WEEKS, depending on the temperature."                 │
   └──────────────────────────────────────────────────────────────────────┘

   ➜ "There is NO ONE TECHNIQUE that can provide absolute guarantees. There
     are only various RISK-REDUCTION TECHNIQUES… it's wise to take any
     theoretical 'guarantees' with a HEALTHY GRAIN OF SALT."
```

---

## 2. Single-object and multi-object operations

### Why multi-object transactions are needed — Figures 7-2 and 7-3

The example: an email app that denormalizes the unread counter because `SELECT COUNT(*)` is too slow.

### 🔷 Figure 7-2 — Violating isolation: reading another transaction's uncommitted writes

```
        insert into emails                        update mailboxes
        (recipient_id, body, unread_flag)         set unread = unread + 1
        values(2, 'Hello', true)                  where recipient_id = 2
               │                                          │       TIME ──────►
   User 1 ─────●──────────────────────────────────────────●─────────────────►
               │   ▲ ok                                   │    ▲ ok
               ▼   │                                      ▼    │
   Database ───●───●────────●─────────────────●───────────●────●─────────────►
                            │ ('Hello',true)  │ 0
                            ▼                 ▼
   User 2 ──────────────────●─────────────────●─────────────────────────────►
                    select body, unread_flag   select unread
                    from emails                from mailboxes

   💥 USER 2 SEES: an unread message in the list, but a counter showing ZERO.
      The email insert was visible; the counter increment was not.
```

### 🔷 Figure 7-3 — Atomicity: undoing prior writes when an error occurs

```
        insert into emails …                      update mailboxes …
               │                                          │       TIME ──────►
   User 1 ─────●──────────────────────────────────────────●─────────────────►
               │   ▲ ok                                   │    ▲
               ▼   │                                      ▼    │
   Database ───●───●──────────────────────────────────────●────✗ ERROR/TIMEOUT
                                                               │
                              ┌────────────────────────────────┘
                              ▼
               "if the update to the counter fails, the transaction is
                ABORTED and THE INSERTED EMAIL IS ROLLED BACK."
```

> **How transactions are delimited:** in relational databases, typically **based on the client's TCP connection** — everything between `BEGIN TRANSACTION` and `COMMIT` on a connection.
>
> 📖 *Footnote:* **"This is not ideal. If the TCP connection is interrupted, the transaction must be aborted. If the interruption happens AFTER the client requested commit but BEFORE the server acknowledges it, THE CLIENT DOESN'T KNOW WHETHER IT WAS COMMITTED OR NOT."** A transaction manager can instead group operations by a unique transaction ID not bound to a TCP connection.

```
   ⚠️ Many non-relational databases have no grouping mechanism. "Even if there
     is a multi-object API (e.g. a MULTI-PUT that updates several keys), THAT
     DOESN'T NECESSARILY MEAN IT HAS TRANSACTION SEMANTICS: the command may
     SUCCEED FOR SOME KEYS AND FAIL FOR OTHERS, leaving the database in a
     PARTIALLY UPDATED STATE."
```

### Single-object writes

```
   Writing a 20 kB JSON document:
   • network interrupted after 10 kB → does the DB store an UNPARSEABLE FRAGMENT?
   • power fails mid-overwrite → do you get OLD AND NEW SPLICED TOGETHER?
   • another client reads mid-write → does it see a PARTIALLY UPDATED VALUE?

   ➜ "Those issues would be incredibly confusing, so storage engines ALMOST
     UNIVERSALLY aim to provide atomicity and isolation ON THE LEVEL OF A
     SINGLE OBJECT on one node."
        atomicity → a LOG for crash recovery
        isolation → a LOCK on each object
```

> ⚠️ **On marketing:** *"compare-and-set and other single-object operations have been dubbed 'lightweight transactions' or even 'ACID' for marketing purposes, but that terminology is MISLEADING. A transaction is usually understood as a mechanism for grouping MULTIPLE operations on MULTIPLE objects into one unit of execution."*

### The three cases where you genuinely need multi-object transactions

```
   ① FOREIGN KEYS / GRAPH EDGES
      "when inserting several records that refer to each other, the foreign
       keys have to be CORRECT AND UP-TO-DATE, otherwise the data becomes
       nonsensical."

   ② DENORMALIZED DATA in document models
      Fields updated together are usually in one document — but document DBs
      lacking joins ENCOURAGE DENORMALIZATION. "When denormalized information
      needs to be updated, you need to update SEVERAL DOCUMENTS IN ONE GO."

   ③ SECONDARY INDEXES  ← the one people forget
      "These indexes are DIFFERENT DATABASE OBJECTS from a transaction point
       of view: without transaction isolation, it's possible for A RECORD TO
       APPEAR IN ONE INDEX BUT NOT ANOTHER."
```

---

## 3. Handling errors and aborts

> ACID databases are based on this philosophy: **"if the database is in danger of violating its guarantee of atomicity, isolation or durability, it would rather ABANDON THE TRANSACTION ENTIRELY than allow it to continue."**
>
> Not all systems follow that: **leaderless replication datastores work on a "best effort" basis — "the database will do as much as it can, and if it runs into an error, IT WON'T UNDO SOMETHING IT HAS ALREADY DONE."**

### 😐 The ORM complaint, which is a good one

> Popular ORM frameworks such as **Rails' ActiveRecord and Django DON'T RETRY ABORTED TRANSACTIONS** — the error bubbles up as an exception, **"so any user input is thrown away and the user gets an error message. THIS IS A SHAME, BECAUSE THE WHOLE POINT OF ABORTS IS TO ENABLE SAFE RETRIES."**

### The five ways retrying can still go wrong

```
   ① THE TRANSACTION ACTUALLY SUCCEEDED, but the network failed while
      acknowledging → retrying PERFORMS IT TWICE, unless you have
      application-level DEDUPLICATION.

   ② IF THE ERROR IS DUE TO OVERLOAD, retrying MAKES THE PROBLEM WORSE.
      → limit retries, use EXPONENTIAL BACKOFF, handle overload errors
        differently.

   ③ ONLY TRANSIENT ERRORS ARE WORTH RETRYING (deadlock, isolation violation,
      network interruption, failover). Retrying a PERMANENT error
      (constraint violation) is POINTLESS.

   ④ SIDE EFFECTS OUTSIDE THE DATABASE may happen even if the transaction
      aborts. "If you're sending an email, you wouldn't want to send it again
      every time you retry." → 2-phase commit can help (Ch.9).

   ⑤ IF THE CLIENT PROCESS FAILS WHILE RETRYING, the data is lost.
```

---
---

# PART B — WEAK ISOLATION LEVELS

> **"Concurrency bugs are hard to find by testing, because bugs are ONLY TRIGGERED WHEN YOU GET UNLUCKY WITH THE TIMING."** Concurrency is also very difficult to reason about, **"especially in a large application where you don't necessarily know which other pieces of code are accessing the database."**

### ⚠️ These are not theoretical problems

> Concurrency bugs caused by weak isolation **"have caused SUBSTANTIAL LOSS OF MONEY, have led to INVESTIGATION BY FINANCIAL AUDITORS, and caused CUSTOMER DATA TO BE CORRUPTED."**
>
> **"A popular comment on revelations of such problems is 'use an ACID database if you're handling financial data!', BUT THAT MISSES THE POINT. Even many popular relational database systems (which are usually considered 'ACID') USE WEAK ISOLATION, so they wouldn't necessarily have prevented these bugs."**

---

## 4. Read committed

> **The most basic level of transaction isolation.** It makes two guarantees:

```
   ① NO DIRTY READS  — when reading, you only see data that has been COMMITTED
   ② NO DIRTY WRITES — when writing, you only overwrite data that has been
                       COMMITTED
```

> 📖 *Footnote:* some databases support an even weaker level called **read uncommitted**, which **prevents dirty writes but not dirty reads.**

### 🔷 Figure 7-4 — No dirty reads

```
                    set x = 3          set y = 3        commit
                        │                  │               │      TIME ──────►
   User 1  ─────────────●──────────────────●───────────────●──────────────────►
                        │  ▲ ok            │  ▲ ok         │
                        ▼  │               ▼  │            ▼
   Database ────────────●──●───────────────●──●────────────●─────────────────►
              │ 2                │ 2                            │ 3
              ▼                  ▼                              ▼
   User 2  ───●──────────────────●──────────────────────────────●─────────────►
            get x              get x                          get x
            → 2                → 2  ← STILL the old value!     → 3

   ⚑ "any writes by a transaction only become visible to others WHEN THAT
     TRANSACTION COMMITS — and then, ALL OF ITS WRITES BECOME VISIBLE AT ONCE."
```

**Two reasons dirty reads matter:**

```
   ① Another transaction may SEE SOME UPDATES BUT NOT OTHERS (Figure 7-2:
      the new unread email but not the updated counter). "Seeing the database
      in a partially updated state is confusing to users, and MAY CAUSE OTHER
      TRANSACTIONS TO TAKE INCORRECT DECISIONS."

   ② If a transaction ABORTS, its writes are rolled back. With dirty reads,
      "a transaction may see data that WAS LATER ROLLED BACK, i.e. WHICH WAS
      NEVER ACTUALLY COMMITTED. Reasoning about the consequences quickly
      becomes MIND-BENDING."
```

### 🔷 Figure 7-5 — Dirty writes: the car sold to Bob but invoiced to Alice

```
          update listings                    update invoices
          set buyer='Alice'                  set recipient='Alice'
          where id=1234                      where listing_id=1234    commit
               │                                    │                    │
   Alice ──────●────────────────────────────────────●────────────────────●────►
               │  ▲ ok                              │  ▲ ok
               ▼  │                                 ▼  │
   LISTINGS ───●──●──────●────────────────────────────────────────────────────►
                         │  ▲ ok
                         ▼  │      final buyer = 'Bob'   ◄────┐
   INVOICES ─────────────────────────●──●──────────────────────┼─────────────►
                                     │  ▲ ok                   │
                          final recipient = 'Alice' ◄──────────┼──┐
                                     ▼                         │  │
   Bob   ────────────────●───────────●───────────────●─────────┘  │
               update listings   update invoices   commit         │
               set buyer='Bob'   set recipient='Bob'              │
                                                                  │
   💥 THE SALE GOES TO BOB (he wins the listings write) BUT THE INVOICE
      GOES TO ALICE (she wins the invoices write). ─────────────────┘
      "Read committed prevents such mishaps."
```

> ⚠️ **But note what read committed does NOT prevent:** the counter race of Figure 7-1. *"In this case the second write happens AFTER the first transaction has committed, so it's NOT A DIRTY WRITE. It's still incorrect, but for a different reason."*

### Implementing read committed

**The default in Oracle 11g, PostgreSQL, SQL Server 2012, MemSQL and many others.**

```
   DIRTY WRITES → prevented with ROW-LEVEL LOCKS. A transaction wanting to
   modify an object must acquire a lock and HOLD IT UNTIL COMMIT OR ABORT.

   DIRTY READS → you COULD use the same lock briefly on read…
   ❌ BUT: "the approach of requiring read locks DOES NOT WORK WELL IN
      PRACTICE, because ONE LONG-RUNNING WRITE TRANSACTION CAN FORCE MANY
      READ-ONLY TRANSACTIONS TO WAIT… a slowdown in one part of an
      application can have a KNOCK-ON EFFECT in a completely different part."

   ✅ INSTEAD (Figure 7-4): "for every object that is written, the database
      remembers BOTH THE OLD COMMITTED VALUE AND THE NEW VALUE set by the
      transaction that holds the write lock. Any other transaction reading
      the object is simply GIVEN THE OLD VALUE."
```

> 📖 *Footnote:* at time of writing, the only mainstream databases using **locks** for read committed are **IBM DB2** and **SQL Server with `read_committed_snapshot=off`**.

### 💻 My implementation demonstrating both

```
txn2 has set x=3 but NOT committed.
   txn3 reads x = 2   <- sees the OLD committed value 2
   after commit, a new txn reads x = 3

DIRTY WRITE prevention:
   txn3: dirty write blocked on 'listing'
```

---

## 5. Snapshot isolation and repeatable read

### 🔷 Figure 7-6 — Read skew: $100 vanishes

> Alice has $1,000 split across two accounts with $500 each. A transaction transfers $100 from one to the other.

```
        select balance from accounts where id=1     select balance from
                       │                            accounts where id=2
                       │                                   │       TIME ─────►
   Alice ──────────────●───────────────────────────────────●──────────────────►
                       │  ▲ 500                            │  ▲ 400
                       ▼  │                                ▼  │
   ACCOUNT 1 ──────────●──●────────●──────────────────────────────────────────►
        balance = 500           now balance = 600
                                         ↑
   ACCOUNT 2 ────────────────────────────┼────────────●──────────────────────►
        balance = 500                                 now balance = 400
                                         │                 │
   Transfer ─────────────────────────────●─────────────────●─────────●───────►
                          update accounts set    update accounts set   commit
                          balance=balance+100    balance=balance-100
                          where id=1             where id=2

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  ALICE SEES:  account1 = $500  (read BEFORE the +100 arrived)         ║
   ║               account2 = $400  (read AFTER  the -100 left)            ║
   ║               TOTAL   = $900   💥 $100 HAS VANISHED INTO THIN AIR     ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

> This anomaly is called **non-repeatable read** or **read skew**. *"Read skew is considered ACCEPTABLE under read committed isolation: the account balances that Alice saw WERE INDEED COMMITTED at the time when she read them."*

### Two situations that genuinely cannot tolerate it

```
   ① BACKUPS
      "Taking a backup requires making a copy of the ENTIRE database, which
       may take HOURS. During that time, writes continue. Thus you could end
       up with SOME PARTS OF THE BACKUP containing an OLDER version and other
       parts a NEWER version. If you need to RESTORE from such a backup, the
       inconsistencies (such as disappearing money) BECOME PERMANENT."
                                                   ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
                       ← this is the word that makes it serious

   ② ANALYTIC QUERIES AND INTEGRITY CHECKS
      Queries scanning large parts of the database "are likely to return
      NONSENSICAL RESULTS if they observe parts of the database at DIFFERENT
      POINTS IN TIME."
```

### 🔑 The solution: snapshot isolation (MVCC)

> **"Each transaction reads from a CONSISTENT SNAPSHOT of the database — all the data that was committed at a particular point in time.** Even if the data is subsequently changed by another transaction, **each transaction sees the old data from the time when that transaction started."**

**Supported by:** PostgreSQL, MySQL/InnoDB, Oracle, SQL Server, and more.

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║          THE PERFORMANCE MANTRA OF SNAPSHOT ISOLATION:                ║
   ║                                                                       ║
   ║          READERS NEVER BLOCK WRITERS,                                 ║
   ║          AND WRITERS NEVER BLOCK READERS.                             ║
   ║                                                                       ║
   ║  Write locks still prevent dirty writes, but LOCKS ARE NOT REQUIRED   ║
   ║  FOR READS. "This allows a database to handle long-running read       ║
   ║  queries on a consistent snapshot AT THE SAME TIME as processing      ║
   ║  writes normally, WITHOUT ANY LOCK CONTENTION between the two."       ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   Read committed needs AT MOST TWO versions of an object.
   Snapshot isolation needs SEVERAL committed versions, because in-progress
   transactions may need the state at DIFFERENT POINTS IN TIME.
   ➜ hence: a MULTIVERSION technique.
```

### 🔷 Figure 7-7 — Implementing snapshot isolation with multiversion objects

```
   Every row carries two hidden fields:
      created_by  — the txid of the transaction that INSERTED it
      deleted_by  — initially empty; set to the txid that DELETED it

   ⚑ AN UPDATE IS INTERNALLY TRANSLATED INTO A DELETE AND A CREATE.

        select balance                          select balance
        from accounts where id=1                from accounts where id=2
              │                                        │        TIME ────────►
   txid 12 ───●────────────────────────────────────────●──────────●──────────►
              │  ▲ 500                                 │  ▲ 500      commit
              ▼  │                                     ▼  │
   ACCOUNT 1  ┌─────────────────┐    ┌─────────────────┐
              │ created_by = 3  │    │ created_by = 3  │
              │ deleted_by = nil│ →  │ deleted_by = 13 │  ← marked deleted
              │ balance = 500   │    │ balance = 500   │
              └─────────────────┘    └─────────────────┘
                                     ┌─────────────────┐
                                     │ created_by = 13 │  ← NEW VERSION
                                     │ deleted_by = nil│
                                     │ balance = 600   │
                                     └─────────────────┘
   ACCOUNT 2  ┌─────────────────┐                       ┌─────────────────┐
              │ created_by = 5  │                       │ created_by = 5  │
              │ deleted_by = nil│  ──────────────────►  │ deleted_by = 13 │
              │ balance = 500   │                       │ balance = 500   │
              └─────────────────┘                       └─────────────────┘
                                                        ┌─────────────────┐
                                                        │ created_by = 13 │
                                                        │ balance = 400   │
                                                        └─────────────────┘
   txid 13 ────────────●───────────────────●──────────────────●─────────────►
              update accounts set     update accounts set    commit
              balance=balance+100     balance=balance-100
              where id=1              where id=2
```

> A deleted row **isn't actually removed** — later, **"when it is certain that no transaction can any longer access the deleted data, a GARBAGE COLLECTION process removes rows marked for deletion and frees their space."**
>
> 📖 *Footnote:* transaction IDs are **32-bit integers, so they overflow after ~4 billion transactions.** PostgreSQL's **vacuum** process ensures overflow doesn't affect the data.

### 🔑 The four visibility rules

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  ① At the START of each transaction, the database makes a LIST OF ALL ║
   ║     OTHER TRANSACTIONS IN PROGRESS at that time. Any writes made by    ║
   ║     one of those are IGNORED — EVEN IF IT SUBSEQUENTLY COMMITS.        ║
   ║  ② Any writes made by ABORTED transactions are ignored.                ║
   ║  ③ Any writes made by transactions with a LATER TRANSACTION ID are     ║
   ║     ignored, REGARDLESS of whether that transaction has committed.     ║
   ║  ④ All other writes are VISIBLE.                                       ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   PUT ANOTHER WAY — an object is VISIBLE if:
      • at the time the reader's transaction started, the transaction which
        CREATED it had ALREADY COMMITTED, and
      • the object is NOT MARKED FOR DELETION — or if it is, the transaction
        that requested deletion HAD NOT YET COMMITTED at the time the
        reader's transaction started.

   ➜ "By NEVER UPDATING VALUES IN PLACE, but instead creating a new version
     every time a value is changed, the database can provide a consistent
     snapshot while incurring ONLY A SMALL OVERHEAD."
```

### 💻 I implemented these four rules verbatim, and the version chain matches Figure 7-7

```
Alice's read transaction starts (txid=2)
   reads account1 = 500
   ...meanwhile txn3 transfers $100 and COMMITS
   Alice reads account2 = 500  <- still the SNAPSHOT value
   total Alice sees = 1000  ✓ $1000

version chain on disk:
   account1=500   created_by=1   deleted_by=3
   account2=500   created_by=1   deleted_by=3
   account1=600   created_by=3   deleted_by=—
   account2=400   created_by=3   deleted_by=—
```

**Four rows for two accounts** — exactly the structure of Figure 7-7. Alice's txid (2) is lower than the transfer's (3), so rule ③ makes both new versions invisible to her, and she correctly sees $1000.

### 💻 And the direct comparison

```
   read committed  : acct1=500  acct2=400  total=$900   ❌ $100 VANISHED
   snapshot        : acct1=500  acct2=500  total=$1000  ✅ consistent
```

### Indexes and snapshot isolation

```
   OPTION 1: the index points to ALL VERSIONS of an object, and an index query
             FILTERS OUT versions not visible to the current transaction.
             Garbage collection removes index entries along with the versions.
             (PostgreSQL optimizes by avoiding index updates if different
              versions fit on the SAME PAGE.)

   OPTION 2 — CouchDB, Datomic, LMDB: APPEND-ONLY / COPY-ON-WRITE B-TREES.
             A modified page is NOT OVERWRITTEN; a NEW COPY is created, and
             parent pages up to the root are copied and updated to point at it.
             Unaffected pages are NOT COPIED and remain IMMUTABLE.

                  root'  ← new root = a consistent SNAPSHOT
                 ╱    ╲
             new p    OLD p  (shared, immutable)
             ╱   ╲      ╲
         new     OLD     OLD

             "every write transaction creates a NEW B-TREE ROOT, and a
              particular root IS a consistent snapshot at the point in time
              when it was created. THERE IS NO NEED TO FILTER OUT OBJECTS
              BASED ON TRANSACTION IDS."
             ⚠️ still needs a background process for compaction and GC.
```

### 😵 Repeatable read and naming confusion

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE SAME THING, FOUR DIFFERENT NAMES:                                ║
   ║     Oracle              calls snapshot isolation  "SERIALIZABLE"      ║
   ║     PostgreSQL & MySQL  call it                   "REPEATABLE READ"   ║
   ║     IBM DB2             uses "repeatable read" to mean SERIALIZABILITY║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  WHY: the SQL standard has NO CONCEPT of snapshot isolation, because  ║
   ║  it is based on System R's 1975 definitions AND SNAPSHOT ISOLATION    ║
   ║  HADN'T YET BEEN INVENTED. Postgres and MySQL call it "repeatable     ║
   ║  read" because it MEETS THE REQUIREMENTS OF THE STANDARD, so they can ║
   ║  CLAIM STANDARDS COMPLIANCE.                                          ║
   ║                                                                       ║
   ║  "the SQL standard's definition of isolation levels is FLAWED — it is ║
   ║   AMBIGUOUS, IMPRECISE and NOT AS IMPLEMENTATION-INDEPENDENT as a     ║
   ║   standard should be."                                                ║
   ║                                                                       ║
   ║          ➜ "AS A RESULT, NOBODY REALLY KNOWS WHAT                     ║
   ║               REPEATABLE READ MEANS."                                 ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 6. Preventing lost updates

> **"The most well-known of these is the LOST UPDATE problem"** (Figure 7-1). It occurs with a **read-modify-write cycle**: two transactions do it concurrently, and **"the second write does not include the first modification. We sometimes say that the later write CLOBBERS the earlier write."**

```
   THREE COMMON SCENARIOS:
   • incrementing a counter or updating an account balance
   • making a local change to a complex value, e.g. adding an element to a
     list within a JSON document (parse → change → write back)
   • two users editing a WIKI PAGE at the same time, each saving by sending
     THE ENTIRE PAGE CONTENTS to the server
```

### The five solutions

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │ ① ATOMIC WRITE OPERATIONS                                            │
   │      UPDATE counters SET value = value + 1 WHERE key = 'foo';        │
   │    "usually the BEST SOLUTION if your code can be expressed in terms │
   │     of those operations." MongoDB has atomic JSON-part operations;   │
   │     Redis has them for priority queues.                              │
   │    Implemented via an EXCLUSIVE LOCK on read (CURSOR STABILITY), or  │
   │    by forcing all atomic ops onto a single thread.                   │
   │    ⚠️ "ORM frameworks make it EASY TO ACCIDENTALLY WRITE CODE which   │
   │       performs UNSAFE READ-MODIFY-WRITE CYCLES instead."             │
   ├──────────────────────────────────────────────────────────────────────┤
   │ ② EXPLICIT LOCKING —  SELECT … FOR UPDATE                            │
   │      BEGIN TRANSACTION;                                              │
   │      SELECT * FROM figures                                           │
   │        WHERE name='robot' AND game_id=222  FOR UPDATE;  ← takes locks│
   │      -- check whether the move is valid, then:                       │
   │      UPDATE figures SET position='c4' WHERE id=1234;                 │
   │      COMMIT;                                                         │
   │    Needed when validity depends on application logic you can't       │
   │    express as a query (e.g. the rules of a game).                    │
   │    ⚠️ "It's EASY TO FORGET to add a necessary lock somewhere."        │
   ├──────────────────────────────────────────────────────────────────────┤
   │ ③ AUTOMATICALLY DETECTING LOST UPDATES                               │
   │    Let them run in parallel; if the transaction manager detects a    │
   │    lost update, ABORT and retry.                                     │
   │    ✅ PostgreSQL repeatable read, Oracle serializable, SQL Server     │
   │       snapshot isolation ALL DETECT THIS AUTOMATICALLY.              │
   │    ❌ MySQL/InnoDB's repeatable read DOES NOT.                        │
   │       "Some authors argue that a database MUST prevent lost updates  │
   │        in order to qualify as providing snapshot isolation, SO MYSQL │
   │        DOES NOT PROVIDE SNAPSHOT ISOLATION UNDER THIS DEFINITION."   │
   │    🔑 "a GREAT FEATURE, because it doesn't require application code  │
   │       to use any special database features… LESS ERROR-PRONE."       │
   ├──────────────────────────────────────────────────────────────────────┤
   │ ④ COMPARE-AND-SET                                                    │
   │      UPDATE wiki_pages SET content='new content'                     │
   │        WHERE id=1234 AND content='old content';                      │
   │    ⚠️ "IF THE DATABASE ALLOWS THE WHERE CLAUSE TO READ FROM AN OLD    │
   │       SNAPSHOT, THIS STATEMENT MAY NOT PREVENT LOST UPDATES. CHECK   │
   │       whether your database's compare-and-set operation is SAFE."    │
   ├──────────────────────────────────────────────────────────────────────┤
   │ ⑤ IN REPLICATED DATABASES — a different world entirely               │
   │    "Locks and compare-and-set assume there is a SINGLE UP-TO-DATE    │
   │     COPY of the data." Multi-leader and leaderless replication can't │
   │     guarantee that, "so techniques based on locks or compare-and-set │
   │     DO NOT APPLY."                                                   │
   │    ✅ instead: SIBLINGS + merge (Ch.5), or COMMUTATIVE atomic ops     │
   │       (Riak 2.0 datatypes / CRDTs).                                  │
   │    ❌ "LWW is prone to lost updates… UNFORTUNATELY, LWW IS THE        │
   │       DEFAULT IN MANY REPLICATED DATABASES."                         │
   └──────────────────────────────────────────────────────────────────────┘
```

### 💻 All five, measured

```
   no protection              42 + 1 + 1 = 43   ❌ LOST UPDATE
   atomic UPDATE x=x+1        42 + 1 + 1 = 44   ✅
   SELECT … FOR UPDATE        42 + 1 + 1 = 44   ✅
   compare-and-set + retry    42 + 1 + 1 = 44   ✅
   SSI detects + retry        42 + 1 + 1 = 44   ✅
```

---

## 7. Write skew and phantoms

### 🔷 Figure 7-8 — Write skew: the on-call doctors

> The hospital **absolutely must have at least one doctor on call.** Alice and Bob are both on call, both feel unwell, and both click "go off call" at approximately the same time.

```
   ALICE'S TRANSACTION                 ┌──────┬─────────┐   BOB'S TRANSACTION
   begin transaction                   │ name │ on_call │   begin transaction
                                       ├──────┼─────────┤
   currently_on_call = (               │Alice │  true   │   currently_on_call = (
     select count(*) from doctors      │Bob   │  true   │     select count(*) …
     where on_call = true              │Carol │  false  │     where on_call=true
     and shift_id = 1234               └──────┴─────────┘     and shift_id=1234
   )                                                        )
   ➜ currently_on_call = 2                                  ➜ currently_on_call = 2

   if (currently_on_call >= 2) {  ✅ passes
      update doctors                   ┌──────┬─────────┐
      set on_call = false      ──────► │Alice │  FALSE  │
      where name = 'Alice'             └──────┴─────────┘
   }                                                        if (currently_on_call
                                       ┌──────┬─────────┐        >= 2) {  ✅ passes
                                       │Bob   │  FALSE  │ ◄──── update doctors
   commit transaction                  └──────┴─────────┘        set on_call=false
                                                                 where name='Bob'
                             ┌──────┬─────────┐               }
                             │Alice │  false  │
                             │Bob   │  false  │               commit transaction
                             │Carol │  false  │
                             └──────┴─────────┘
                 💥 NO DOCTOR IS ON CALL. INVARIANT VIOLATED.
```

### Characterizing write skew

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  It is NEITHER a dirty write NOR a lost update, "because the two      ║
   ║  transactions are updating TWO DIFFERENT OBJECTS."                    ║
   ║                                                                       ║
   ║  "You can think of write skew as a GENERALIZATION OF LOST UPDATE.     ║
   ║   Write skew can occur if two transactions READ THE SAME OBJECTS, and ║
   ║   then UPDATE SOME OF THOSE OBJECTS (different transactions may       ║
   ║   update different objects). In the SPECIAL CASE where different      ║
   ║   transactions update THE SAME object, you get a dirty write or lost  ║
   ║   update anomaly."                                                    ║
   ╚═══════════════════════════════════════════════════════════════════════╝

         READ the same objects
                  │
         ┌────────┴────────┐
         ▼                 ▼
   update the SAME    update DIFFERENT
   object             objects
         │                 │
         ▼                 ▼
   dirty write /      WRITE SKEW
   lost update
```

### ⚠️ Your options are much more restricted than for lost updates

```
   ❌ ATOMIC SINGLE-OBJECT OPERATIONS don't help — MULTIPLE OBJECTS involved.
   ❌ AUTOMATIC LOST-UPDATE DETECTION doesn't help either. Write skew is NOT
      automatically detected in PostgreSQL repeatable read, MySQL/InnoDB
      repeatable read, Oracle serializable, or SQL Server snapshot isolation.
      ➜ "AUTOMATICALLY PREVENTING WRITE SKEW REQUIRES TRUE SERIALIZABLE
        ISOLATION."
   ⚠️ CONSTRAINTS could work — but "in order to specify that at least one
      doctor must be on call, you would need a constraint that INVOLVES
      MULTIPLE OBJECTS. Most databases do not have built-in support for such
      constraints. You may be able to implement them with triggers or
      materialized views, but THE RESULT CAN END UP QUITE HACKY."
   ✅ SECOND-BEST: explicitly lock the rows the transaction depends on:

        BEGIN TRANSACTION;
        SELECT * FROM doctors
          WHERE on_call = true AND shift_id = 1234  FOR UPDATE;   ← locks them
        UPDATE doctors SET on_call = false
          WHERE name = 'Alice' AND shift_id = 1234;
        COMMIT;
```

### Four more examples of write skew

| Scenario | The check-then-act |
|---|---|
| **Meeting room booking** | Check for overlapping bookings → if none, insert. **"Snapshot isolation does not prevent another user concurrently inserting a conflicting meeting."** |
| **Multiplayer game** | A lock stops two players moving the *same* figure, **"but the lock doesn't prevent two DIFFERENT figures from being moved to the same position"** |
| **Claiming a username** | Check if taken → if not, create. **✅ Fortunately a UNIQUE CONSTRAINT is a simple solution here** |
| **Preventing double-spending** | Insert a tentative spending item, list all items, check the sum is positive. **"Two spending items inserted concurrently could together make the balance negative, but NEITHER TRANSACTION NOTICES THE OTHER."** |

### 🔑 The universal four-step pattern

```
   ① A SELECT query CHECKS WHETHER SOME REQUIREMENT IS SATISFIED by searching
      for rows matching a condition.
                             │
                             ▼
   ② The application code DECIDES HOW TO CONTINUE based on the result.
                             │
                             ▼
   ③ If it decides to go ahead, it makes a WRITE and COMMITS.
                             │
                             ▼
   ④ ⚠️ IF YOU REPEATED THE SELECT FROM STEP 1 NOW, YOU WOULD GET A DIFFERENT
        RESULT — because the write in step 3 changed the set of rows matching
        the search condition.
```

### Phantoms

```
   In the DOCTORS example, the row modified in step 3 WAS ONE OF THE ROWS
   RETURNED IN STEP 1 → so SELECT FOR UPDATE can lock it. ✅

   ⚠️ BUT THE OTHER FOUR EXAMPLES ARE DIFFERENT: "they check for the ABSENCE
      of rows matching some search condition, and the write ADDS A ROW
      matching the same condition. If the query in step 1 doesn't return any
      rows, SELECT FOR UPDATE CAN'T ATTACH LOCKS TO ANYTHING."

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  PHANTOM: "where a WRITE in one transaction CHANGES THE RESULT OF A   ║
   ║  SEARCH QUERY in another transaction."                                ║
   ║                                                                       ║
   ║  "Snapshot isolation AVOIDS PHANTOMS IN READ-ONLY QUERIES, but in     ║
   ║   READ-WRITE transactions, phantoms can lead to PARTICULARLY TRICKY   ║
   ║   CASES OF WRITE SKEW."                                               ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### Materializing conflicts — the last resort

```
   "If the problem is that THERE IS NO OBJECT TO WHICH WE CAN ATTACH THE
    LOCKS, perhaps we can ARTIFICIALLY INTRODUCE A LOCK OBJECT?"

   For meeting rooms: create a table of (room, 15-minute time slot) rows for
   ALL POSSIBLE COMBINATIONS ahead of time, e.g. for the next 6 months.
   A booking transaction SELECT … FOR UPDATEs the rows for the desired room
   and period, THEN checks for overlaps and inserts.

   ⚑ "the additional table ISN'T USED TO STORE INFORMATION ABOUT THE BOOKING
     — it's PURELY A COLLECTION OF LOCKS."

   ⚠️ "It can be HARD AND ERROR-PRONE to figure out how to materialize
      conflicts, and IT'S UGLY TO LET A CONCURRENCY CONTROL MECHANISM LEAK
      INTO THE APPLICATION DATA MODEL. For those reasons, materializing
      conflicts should be considered A LAST RESORT if no alternative is
      possible. A SERIALIZABLE ISOLATION LEVEL IS MUCH PREFERABLE."
```

### 💻 The doctors scenario, run at two isolation levels

```
   --- isolation = snapshot ---
      Alice's txn counts 2 on call
      Bob's txn counts 2 on call
      Alice's txn COMMITTED
      Bob's txn COMMITTED
      doctors on call afterwards: 0   ❌ INVARIANT VIOLATED

   --- isolation = serializable ---
      Alice's txn counts 2 on call
      Bob's txn counts 2 on call
      Alice's txn COMMITTED
      Bob's txn ABORTED -- concurrent txn2 wrote ['Alice'], which we had read
                           (our premise is stale)
      doctors on call afterwards: 1   ✅ invariant held
```

**Note what SSI did:** Alice committed fine. Bob's transaction had *read* Alice's row while deciding, and Alice then changed it — so Bob's premise was stale and he was aborted. **Exactly one doctor remains on call.**

---
---

# PART C — SERIALIZABILITY

> **"It's a sad situation:"**

```
   • "Isolation levels are HARD TO UNDERSTAND, and INCONSISTENTLY IMPLEMENTED
      in different databases."
   • "If you look at your application code, IT'S DIFFICULT TO TELL WHETHER IT
      IS SAFE to run at a particular isolation level — especially in a large
      application, where you might not be aware of all the things that may be
      happening concurrently."
   • "THERE ARE NO GOOD TOOLS to help us detect race conditions… Automated
      testing for concurrency issues is hard, because they are usually
      NON-DETERMINISTIC — problems only occur if you get unlucky with timing."

   ➜ "This is NOT A NEW PROBLEM — it has been like this SINCE THE 1970s. All
     along, the answer from researchers has been simple: USE SERIALIZABLE
     ISOLATION!"
```

> **Serializable isolation guarantees that even though transactions may execute in parallel, the end result is the same as if they had executed ONE AT A TIME, SERIALLY.** Thus **"if the transactions behave correctly when run individually, they continue to be correct when run concurrently — the database prevents ALL POSSIBLE RACE CONDITIONS."**

---

## 8. Technique 1 — Actual serial execution

> **"The simplest way of avoiding any concurrency problems is to REMOVE THE CONCURRENCY ENTIRELY."** Execute one transaction at a time, in serial order, **on a single thread.** The resulting isolation is **by definition serializable.**

### Why this only became feasible around 2007

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │ ① RAM BECAME CHEAP ENOUGH to keep the entire active dataset in       │
   │    memory. "When all data a transaction needs is in memory,          │
   │    transactions can execute MUCH FASTER than if they have to wait    │
   │    for data to be loaded from disk."                                 │
   │                                                                      │
   │ ② DESIGNERS REALIZED OLTP TRANSACTIONS ARE USUALLY SHORT and make    │
   │    only a small number of reads and writes. Long-running ANALYTICS   │
   │    queries are typically READ-ONLY, so they can run on a consistent  │
   │    snapshot OUTSIDE the serial execution loop.                       │
   └──────────────────────────────────────────────────────────────────────┘

   Implemented in VoltDB/H-Store, Redis and Datomic.
   "A system designed for single-threaded execution can SOMETIMES PERFORM
    BETTER than a system that supports concurrency, because it can AVOID THE
    COORDINATION OVERHEAD OF LOCKING. However, its throughput is LIMITED TO
    THAT OF A SINGLE CPU CORE."
```

### 🔷 Figure 7-9 — Interactive transaction vs stored procedure

```
   ═══ INTERACTIVE TRANSACTION ══════════════════════════════════════════════
                 select count(*)        if (currently_      update doctors
                 from doctors …              on_call ≥ 2) { set on_call=false
                       │                     …             where name='Bob'
                       │                 }                       │
   Application ────────●─────────────────────●─────────────────────●─────────►
                  ╲    ▲ 2               ╱                    ╲    ▲ ok
        network    ╲   │                ╱  ← application      ╱     │
          hop       ╲  │               ╱     THINKS here     ╱      │
                     ▼ │              ▼                     ▼       │
   Query processor ───●─●──────────────────────────────────────●────●────────►
   Storage
        ⚠️ FOUR NETWORK HOPS, and the DB is IDLE between them.

   ═══ STORED PROCEDURE ═════════════════════════════════════════════════════
                          execute stored procedure
                          take_doctor_off_call_if_safe
                          with name='Bob', shift_id=1234
                                    │
   Application ─────────────────────●───────────────────────────●────────────►
                            ╲                                  ╱  ▲ ok
                  network    ╲                                ╱   │
                    hop       ╲   select… → 2                ╱    │
                               ╲  if (…)…                   ╱     │
                                ╲ update… → ok             ╱      │
                                 ▼                        ▼       │
   Query processor ───────────────●──────────────────────●────────●──────────►
   Storage
        ✅ ONE NETWORK HOP. Everything runs inside the database.
```

> **The reason this matters:** *"In this interactive style, A LOT OF TIME IS SPENT IN NETWORK COMMUNICATION. If you were to disallow concurrency and only process one transaction at a time, THE THROUGHPUT WOULD BE DREADFUL, because the database would spend most of its time WAITING FOR THE APPLICATION to issue the next query."*

### 📖 A nice historical aside

> In the early days, **"the intention was that a database transaction could encompass an ENTIRE FLOW OF USER ACTIVITY"** — booking an airline ticket, from searching routes through to payment, as one atomic transaction.
>
> **"Unfortunately, HUMANS ARE VERY SLOW TO MAKE UP THEIR MIND AND RESPOND."** So almost all OLTP applications keep transactions short by **avoiding interactively waiting for a user within a transaction.** On the web, a transaction is committed **within the same HTTP request.**

### Pros and cons of stored procedures

```
   ❌ THE BAD REPUTATION, and it's largely deserved:
      • "Each vendor has their OWN LANGUAGE (Oracle PL/SQL, SQL Server T-SQL,
        PostgreSQL PL/pgSQL). These languages HAVEN'T KEPT UP with developments
        in general-purpose languages, so they look QUITE UGLY AND ARCHAIC, and
        they LACK THE ECOSYSTEM OF LIBRARIES."
      • "Code running in a database is DIFFICULT TO MANAGE: harder to debug,
        more awkward to keep in VERSION CONTROL and deploy, trickier to test,
        and difficult to integrate with a METRICS COLLECTION system."
      • "A database is often MUCH MORE PERFORMANCE-SENSITIVE than an
        application server, because a single instance is shared by many app
        servers. A BADLY WRITTEN STORED PROCEDURE can cause MUCH MORE TROUBLE."

   ✅ "However, those issues CAN BE OVERCOME. Modern implementations have
      ABANDONED PL/SQL and use existing general-purpose languages:"
         VoltDB  → Java or Groovy
         Datomic → Java or Clojure
         Redis   → Lua

   ⚑ VoltDB also uses stored procedures FOR REPLICATION: instead of copying
     writes, it EXECUTES THE SAME STORED PROCEDURE ON EACH REPLICA. It
     therefore requires procedures to be DETERMINISTIC (a transaction needing
     the current time must use special deterministic APIs).
     ← this is statement-based replication from Ch.5, made safe.
```

### Partitioning, and the hard ceiling

```
   To scale beyond one core: PARTITION the data (Ch.6), supported in VoltDB.
   "If each transaction only needs to read and write data within A SINGLE
    PARTITION, then each partition can have ITS OWN TRANSACTION PROCESSING
    THREAD… throughput SCALES LINEARLY WITH THE NUMBER OF CPU CORES."

   ⚠️ BUT CROSS-PARTITION TRANSACTIONS must be "performed in LOCK-STEP across
      all partitions."

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "VoltDB reports a throughput of about 1,000 CROSS-PARTITION WRITES   ║
   ║   PER SECOND. This is ORDERS OF MAGNITUDE BELOW its single-partition  ║
   ║   throughput, AND CANNOT BE INCREASED BY ADDING MORE MACHINES."       ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   And whether transactions CAN be single-partition "depends very much on the
   structure of the data: simple key-value data partitions easily, but DATA
   WITH MULTIPLE SECONDARY INDEXES is likely to require A LOT OF
   CROSS-PARTITION COORDINATION." ← Chapter 6's scatter/gather, again
```

### The four constraints, summarized

```
   ① EVERY TRANSACTION MUST BE SMALL AND FAST — "it takes only ONE SLOW
      TRANSACTION to STALL ALL TRANSACTION PROCESSING."
   ② The ACTIVE DATASET MUST FIT IN MEMORY.
      📖 Footnote: if a transaction needs data not in memory, the best
         solution may be to ABORT it, asynchronously fetch the data while
         processing others, and RESTART it — ANTI-CACHING (Ch.3).
   ③ WRITE THROUGHPUT must be handleable on a single core, OR transactions
      must be partitionable without cross-partition coordination.
   ④ CROSS-PARTITION TRANSACTIONS are possible but "there is A HARD LIMIT."
```

---

## 9. Technique 2 — Two-phase locking (2PL)

> **"For around 30 years, there was only ONE widely used algorithm for serializability in databases, and that is two-phase locking."**

```
   ⚠️ 2PL IS NOT 2PC. "Two-phase locking sounds very similar to TWO-PHASE
      COMMIT, but they are COMPLETELY DIFFERENT THINGS." (2PC is Chapter 9.)
```

### 🔑 The rule that makes 2PL different from read committed

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  Several transactions may CONCURRENTLY READ the same object as long   ║
   ║  as nobody is writing to it. But AS SOON AS ANYONE WANTS TO WRITE,    ║
   ║  EXCLUSIVE ACCESS IS REQUIRED.                                        ║
   ║                                                                       ║
   ║  • A has READ an object, B wants to WRITE → B WAITS for A to finish.  ║
   ║    (ensures B can't change the object unexpectedly BEHIND A'S BACK)   ║
   ║  • A has WRITTEN an object, B wants to READ → B WAITS for A.          ║
   ║    (reading an OLD VERSION, as in snapshot isolation, is NOT          ║
   ║     ACCEPTABLE under 2PL)                                             ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  SNAPSHOT ISOLATION  │  readers never block writers, and vice versa   ║
   ║  TWO-PHASE LOCKING   │  READERS BLOCK WRITERS, AND WRITERS BLOCK      ║
   ║                      │  READERS                                       ║
   ║                      │                                                ║
   ║        ← that one line is the whole difference →                      ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

**Used by:** the serializable isolation level in **MySQL (InnoDB)** and **SQL Server**, and the **repeatable read** level in **DB2**.

### Where the name comes from

```
   • READ  → acquire the lock in SHARED mode. Many transactions may hold it
             in shared mode simultaneously; but if another holds it in
             EXCLUSIVE mode, WAIT.
   • WRITE → acquire the lock in EXCLUSIVE mode. NO other transaction may
             hold it at the same time (shared OR exclusive), so WAIT.
   • READ THEN WRITE → UPGRADE the shared lock to an exclusive lock.

   ┌─────────────────────────────────────────────────────────────────────┐
   │  PHASE 1 (while the transaction executes) — LOCKS ARE ACQUIRED      │
   │  PHASE 2 (at the end of the transaction)  — ALL LOCKS ARE RELEASED  │
   │                                                                     │
   │  locks │      ╱▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔│                          │
   │  held  │    ╱                            │  ← all released at once  │
   │        │  ╱                              │     on commit/abort      │
   │        │╱                                │                          │
   │        └──────────────────────────────────────► time                │
   │          ◄─── phase 1 ────────────────►  ◄ phase 2                  │
   └─────────────────────────────────────────────────────────────────────┘
   📖 Footnote: sometimes called STRONG STRICT two-phase locking (SS2PL).
```

> **Deadlock:** *"Since so many locks are in use, it can happen QUITE EASILY that transaction A is stuck waiting for B to release its lock, and vice versa. The database automatically DETECTS deadlock and ABORTS one of them. The aborted transaction NEEDS TO BE RETRIED BY THE APPLICATION."*

### ⚠️ Performance — why it hasn't been used by everybody since the 1970s

```
   "Transaction throughput and response times are SIGNIFICANTLY WORSE under
    2PL than under weak isolation."

   • partly the OVERHEAD of acquiring/releasing locks
   • but MORE IMPORTANTLY, REDUCED CONCURRENCY: "by design, if two concurrent
     transactions try to do anything which may IN ANY WAY result in a race
     condition, ONE HAS TO WAIT."
   • traditional relational DBs DON'T LIMIT TRANSACTION DURATION → "when one
     transaction has to wait on another, THERE IS NO LIMIT ON HOW LONG IT MAY
     HAVE TO WAIT." Queues form.

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "Databases running 2PL can have QUITE UNSTABLE LATENCIES, and can be ║
   ║   VERY SLOW AT HIGH PERCENTILES if there is contention. IT MAY TAKE   ║
   ║   JUST ONE SLOW TRANSACTION, or one transaction that accesses a lot   ║
   ║   of data and acquires many locks, TO CAUSE THE REST OF THE SYSTEM TO ║
   ║   GRIND TO A HALT."                        ← Chapter 1's p99, again   ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   And deadlocks are MUCH MORE FREQUENT under 2PL serializable. "When a
   transaction is aborted due to deadlock and is retried, it needs to DO ITS
   WORK ALL OVER AGAIN → SIGNIFICANT WASTED EFFORT."
```

### 💻 I implemented 2PL locking and produced a deadlock

```
2PL rules: readers block writers AND writers block readers.
   txnA acquires SHARED lock on 'x'    : True
   txnB acquires SHARED lock on 'x'    : True   (both readers OK)
   txnA upgrades to EXCLUSIVE on 'x'   : False  <- BLOCKED, B still reading
   ...B commits and releases. A retries: True

DEADLOCK -- two transactions grabbing locks in opposite order:
   A holds x, wants y -> granted? False
   B holds y, wants x -> granted? False
   => WAIT-FOR CYCLE: A→B→A. The database detects it and ABORTS one.
```

### Predicate locks — the phantom problem, solved properly

```
   A PREDICATE LOCK "belongs to ALL OBJECTS THAT MATCH SOME SEARCH CONDITION":

        SELECT * FROM bookings
          WHERE room_id = 123
            AND end_time   > '2015-01-01 12:00'
            AND start_time < '2015-01-01 13:00';

   • A wants to READ objects matching a condition → acquire a SHARED-MODE
     PREDICATE LOCK on the conditions. If B holds an exclusive lock on ANY
     matching object, A waits.
   • A wants to INSERT/UPDATE/DELETE → first check whether the OLD OR NEW
     VALUE matches any existing predicate lock. If B holds one, A waits.

   🔑 "The KEY IDEA is that a predicate lock APPLIES EVEN TO OBJECTS WHICH DO
     NOT YET EXIST IN THE DATABASE, BUT MIGHT BE ADDED IN FUTURE (PHANTOMS).
     If 2PL includes predicate locks, the database prevents ALL FORMS OF
     WRITE SKEW and other race conditions."
```

### Index-range locks — the practical approximation

```
   ⚠️ "Predicate locks DO NOT PERFORM WELL: if there are many locks by active
      transactions, CHECKING FOR MATCHING LOCKS BECOMES TIME-CONSUMING."

   ➜ Most 2PL databases implement INDEX-RANGE LOCKING (a.k.a. NEXT-KEY LOCKING).

   🔑 "IT'S SAFE TO SIMPLIFY A PREDICATE BY MAKING IT MATCH A GREATER SET OF
     OBJECTS."

     original : room 123, noon–1pm
     approx A : room 123, ANY TIME            ← lock the room_id index entry
     approx B : ALL ROOMS, noon–1pm           ← lock a range of the time index

     "This is safe, because ANY WRITE THAT MATCHES THE ORIGINAL PREDICATE WILL
      DEFINITELY ALSO MATCH THE APPROXIMATIONS."

   An approximation is attached to ONE OF THE INDEXES. Another transaction
   inserting a conflicting booking must UPDATE THE SAME PART OF THE INDEX,
   encounters the shared lock, and WAITS.

   ⚠️ FALLBACK: "if there is NO SUITABLE INDEX where a range lock can be
      attached, the database can fall back to A SHARED LOCK ON THE ENTIRE
      TABLE. This would not be good for performance… but it's A SAFE FALLBACK."
```

---

## 10. Technique 3 — Serializable snapshot isolation (SSI)

> **"This chapter has painted a bleak picture… Are serializable isolation and good performance FUNDAMENTALLY AT ODDS with each other? PERHAPS NOT."**

```
   SSI provides FULL SERIALIZABILITY with "only a SMALL PERFORMANCE PENALTY
   compared to snapshot isolation."

   First described in 2008 (Michael Cahill's PhD thesis).
      • PostgreSQL's serializable isolation level SINCE VERSION 9.1
      • FoundationDB uses a similar algorithm

   "As SSI is so young compared to other concurrency control mechanisms, it is
    still PROVING ITS PERFORMANCE IN PRACTICE, but it has the chance of
    BECOMING THE NEW DEFAULT IN FUTURE."
```

### Pessimistic vs optimistic concurrency control

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  PESSIMISTIC                      ║  OPTIMISTIC                       ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  "if anything MIGHT POSSIBLY go   ║  "instead of blocking, transactions║
   ║   wrong, it's better to WAIT      ║   CONTINUE ANYWAY, in the hope     ║
   ║   until the situation is safe."   ║   that everything will turn out    ║
   ║   Like MUTUAL EXCLUSION.          ║   alright. When a transaction wants║
   ║                                   ║   to commit, the database CHECKS   ║
   ║  = TWO-PHASE LOCKING              ║   whether anything bad happened;   ║
   ║                                   ║   if so, ABORT AND RETRY."         ║
   ║  SERIAL EXECUTION is "pessimistic ║                                    ║
   ║  TO THE EXTREME: essentially      ║  = SSI                             ║
   ║  equivalent to each transaction   ║                                    ║
   ║  having an EXCLUSIVE LOCK ON THE  ║  ❌ "performs BADLY if there is     ║
   ║  ENTIRE DATABASE. We compensate   ║     HIGH CONTENTION… if the system ║
   ║  by making each transaction VERY  ║     is already close to maximum    ║
   ║  FAST."                           ║     throughput, the additional     ║
   ║                                   ║     load from RETRIED transactions ║
   ║                                   ║     CAN MAKE PERFORMANCE WORSE."   ║
   ║                                   ║  ✅ "if there is ENOUGH SPARE       ║
   ║                                   ║     CAPACITY and contention is not ║
   ║                                   ║     too high, optimistic tends to  ║
   ║                                   ║     PERFORM BETTER."               ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝

   ⚑ "Contention can be REDUCED WITH COMMUTATIVE ATOMIC OPERATIONS: if several
     transactions want to increment a counter, IT DOESN'T MATTER IN WHICH
     ORDER the increments are applied, so they can all be applied without
     conflicting."
```

### 🔑 Decisions based on an outdated premise

> **"The transaction is taking an action based on a PREMISE (a fact that was true at the beginning of the transaction, e.g. 'there are currently two doctors on call'). Later, when the transaction wants to commit, the original data may have changed — I.E. THE PREMISE MAY NO LONGER BE TRUE."**
>
> **"When the application makes a query, the database DOESN'T KNOW HOW THE APPLICATION LOGIC USES THE RESULT. To be safe, the database needs to ASSUME THAT ANY CHANGE IN THE QUERY RESULT MEANS THAT WRITES IN THAT TRANSACTION MAY BE INVALID."**

```
   TWO CASES THE DATABASE MUST DETECT:
      ① STALE MVCC READS  — the uncommitted write occurred BEFORE the read
      ② WRITES THAT AFFECT PRIOR READS — the write occurs AFTER the read
```

### 🔷 Figure 7-10 — Detecting stale MVCC reads

```
         select count(*)       update doctors
         from doctors          set on_call=false
         where on_call=true    where name='Alice'          commit
                │                     │                       │   TIME ─────►
   txn 42 ──────●─────────────────────●───────────────────────●─────────────►
                │  ▲ 2                │  ▲ ok                 │  ▲ ok
                ▼  │                  ▼  │                    ▼  │
   Database ────●──●──────●──●─────────●──●───────●──●─────────●──●────✗─────►
                       │  ▲ 2              │  ▲ ok      │  ▲          ABORT
                       ▼  │                ▼  │         ▼  │            │
   txn 43 ─────────────●──●────────────────●──●─────────●──●────────────●────►
                  select count(*)     update doctors   commit        retry…
                  where on_call=true  set on_call=false
                                      where name='Bob'

   ┌────────┬───────┬─────────┬────────────┬────────────┐
   │shift_id│ name  │ on_call │ created_by │ deleted_by │
   ├────────┼───────┼─────────┼────────────┼────────────┤
   │  1234  │ Alice │  true   │     1      │     42     │ ← txn43 SEES this,
   │  1234  │ Alice │  false  │    42      │     —      │   because txn42 hadn't
   │  1234  │ Bob   │  true   │     1      │     —      │   committed yet.
   │  1234  │ Carol │  false  │     1      │     —      │   The manager NOTES
   └────────┴───────┴─────────┴────────────┴────────────┘   it may go stale.
```

> **"The database needs to TRACK WHEN A TRANSACTION IGNORES ANOTHER TRANSACTION'S WRITES due to MVCC visibility rules. When the transaction wants to commit, the database checks whether ANY OF THE IGNORED WRITES HAVE NOW BEEN COMMITTED. If yes, THE TRANSACTION MUST BE ABORTED."**

### 💡 Why wait until commit?

> **"If transaction 43 was a READ-ONLY transaction, IT WOULDN'T NEED TO BE ABORTED, because there is no risk of write skew. At the time when transaction 43 makes its read, THE DATABASE DOESN'T YET KNOW WHETHER THAT TRANSACTION IS GOING TO LATER PERFORM A WRITE. SSI needs to support LONG-RUNNING READS from a consistent snapshot, just like regular snapshot isolation, WITHOUT UNNECESSARY ABORTS."**

*(This detail mattered in my implementation — my first version aborted read-only transactions and broke the read-skew test. Adding the "only if it wrote something" condition fixed it, exactly as the book describes.)*

### 🔷 Figure 7-11 — Detecting writes that affect prior reads

```
         select count(*)         update doctors
         from doctors            set on_call=false
         where on_call=true      where name='Alice'        commit
                │                       │                     │    TIME ────►
   txn 42 ──────●───────────────────────●─────────────────────●──────────────►
                │  ▲ 2                  │  ▲ ok               │  ▲ ok
                ▼  │                    ▼  │                  ▼  │
   Database ────●──●───●──●──────────────●──●─────●──●──────────●──●───✗──────►
                    │  ▲ 2                     │  ▲ ok    │  ▲       ABORT
                    ▼  │                       ▼  │       ▼  │         │
   txn 43 ──────────●──●───────────────────────●──●───────●──●─────────●──────►
              select count(*)        update doctors      commit     retry…
              where on_call=true     set on_call=false
                                     where name='Bob'

   KEY-RANGE INFORMATION (index-range locks on doctors.shift_id)
   ┌──────┬────────────────────────┐    ┌─────┬────────┬─────────┐
   │ 1234 │ read by transaction 42 │    │ old │ Alice  │  true   │
   │ 1234 │ read by transaction 43 │    │ new │ Alice  │  false  │
   └──────┴────────────────────────┘    └─────┴────────┴─────────┘
                                        txn42's update AFFECTS txn43's read
```

### 🔑 The tripwire — the single best image in the chapter

> When a transaction writes, **"it must look in the indexes for any other transactions that have RECENTLY READ the affected data. This is similar to acquiring a write lock on the affected key range, BUT RATHER THAN BLOCKING UNTIL THE READERS HAVE COMMITTED, THE LOCK ACTS AS A TRIPWIRE: it simply NOTIFIES the transactions that the data they read MAY NO LONGER BE UP TO DATE."**

```
        2PL LOCK                            SSI TRIPWIRE
        ────────                            ────────────
        ┌──────────┐                        ┌ ─ ─ ─ ─ ─┐
        │  BLOCKS  │  you stop and wait      ┆  NOTIFIES ┆  you carry on;
        │          │                        ┆           ┆  the check happens
        └──────────┘                        └ ─ ─ ─ ─ ─┘  at COMMIT time
```

> The information **"only needs to be kept for a while: after a transaction has finished, and all concurrent transactions have finished, THE DATABASE CAN FORGET WHAT DATA IT READ."**

**And note the outcome in Figure 7-11:** transaction 42 commits successfully — *"although 43's write affected 42, 43 hasn't yet committed, so the write has not yet taken effect."* But when 43 wants to commit, **42's conflicting write has already been committed, so 43 must abort.** First to commit wins.

### Performance of SSI

```
   ⚖️ THE GRANULARITY TRADE-OFF
      "If the database keeps track of each transaction's activity at GREAT
       DETAIL, it can be PRECISE about which transactions need to abort, but
       THE BOOKKEEPING OVERHEAD CAN BECOME SIGNIFICANT. LESS DETAILED TRACKING
       IS FASTER, but may lead to MORE TRANSACTIONS BEING ABORTED THAN
       STRICTLY NECESSARY."
      (PostgreSQL uses theory to reduce unnecessary aborts.)

   ✅ VS TWO-PHASE LOCKING: "one transaction DOESN'T NEED TO BLOCK waiting for
      locks held by another. Writers don't block readers, and vice versa. THIS
      MAKES QUERY LATENCY MUCH MORE PREDICTABLE AND LESS VARIABLE. In
      particular, READ-ONLY QUERIES CAN RUN ON A CONSISTENT SNAPSHOT WITHOUT
      REQUIRING ANY LOCKS."

   ✅ VS SERIAL EXECUTION: "NOT LIMITED TO THE THROUGHPUT OF A SINGLE CPU CORE.
      FoundationDB DISTRIBUTES THE DETECTION of serialization conflicts across
      multiple machines… transactions can read and write data in MULTIPLE
      PARTITIONS while still preserving serializable isolation."

   ⚠️ "The RATE OF ABORTS significantly affects overall performance. A
      transaction that reads and writes over a LONG PERIOD is likely to run
      into conflicts and abort, so SSI REQUIRES THAT READ-WRITE TRANSACTIONS
      BE FAIRLY SHORT (long-running READ-ONLY transactions may be ok).
      However, SSI is probably LESS SENSITIVE TO SLOW TRANSACTIONS than 2PL
      or serial execution."
```

---

## 11. 💻 THE PREVENTION MATRIX — measured, not quoted

I built a miniature MVCC database implementing read committed, snapshot isolation, and SSI, then ran every anomaly from the chapter against every level:

```
anomaly                         read committed          snapshot      serializable
──────────────────────────────────────────────────────────────────────────────────
dirty read                         ✅ prevented       ✅ prevented       ✅ prevented
dirty write                        ✅ prevented       ✅ prevented       ✅ prevented
read skew (non-repeatable)            ❌ OCCURS       ✅ prevented       ✅ prevented
lost update                           ❌ OCCURS          ❌ OCCURS       ✅ prevented
write skew                            ❌ OCCURS          ❌ OCCURS       ✅ prevented
```

**The staircase shape is the whole chapter in one picture.** Each level adds exactly one row of protection, and only serializable covers everything.

> ⚠️ **One honest caveat about my "snapshot" column.** I implemented *basic* snapshot isolation. As the book notes, **PostgreSQL's repeatable read, Oracle's serializable, and SQL Server's snapshot isolation DO automatically detect lost updates** — so on those systems that cell would be ✅. **MySQL/InnoDB's repeatable read does not**, so there the cell is ❌ as shown. This is precisely the kind of implementation divergence that makes "nobody really knows what repeatable read means" true.

---

## 12. Chapter Summary

> **"Transactions are an ABSTRACTION LAYER that allow an application to PRETEND that certain concurrency problems and certain kinds of hardware and software fault DON'T EXIST. A large class of errors is reduced down to a SIMPLE TRANSACTION ABORT, and the application just needs to TRY AGAIN."**

```
   ⚠️ THE WARNING TO NoSQL USERS:
   "Many NoSQL systems abandoned transactions in the name of scalability,
    availability and performance. Unfortunately this means that applications
    using such data systems either need to IMPLEMENT THEIR OWN TRANSACTION
    MANAGEMENT — WHICH IS UNLIKELY, BECAUSE IT'S HARD TO IMPLEMENT CORRECTLY —
    OR ACCEPT THAT THEIR DATA IS APPROXIMATE."
```

### The six race conditions

| Anomaly | What happens | Prevented by |
|---|---|---|
| **Dirty reads** | One client reads another's writes **before they are committed** | **Read committed** and stronger |
| **Dirty writes** | One client **overwrites data another has written but not committed** | **Almost all** transaction implementations |
| **Read skew** (non-repeatable reads) | A client sees **different parts of the database at different points in time** | **Snapshot isolation**, usually via **MVCC** |
| **Lost updates** | Two clients do a read-modify-write; one **overwrites the other without incorporating its changes** | **Some** implementations of snapshot isolation |
| **Write skew** | A transaction reads, decides, and writes — but **by the time the write is made, the premise is no longer true** | **Only serializable isolation** |
| **Phantom reads** | A transaction reads objects matching a condition; another client **makes a write that affects the results of that search** | Snapshot isolation prevents straightforward ones; **phantoms in write skew need index-range locks** |

### The three implementations

```
   ① LITERALLY EXECUTING TRANSACTIONS IN A SERIAL ORDER
      "If you can make each transaction VERY FAST, and the throughput is low
       enough to process on a SINGLE CPU CORE, this is a SIMPLE AND EFFECTIVE
       option."

   ② TWO-PHASE LOCKING
      "For decades this has been THE STANDARD WAY of implementing
       serializability, but MANY APPLICATIONS AVOID USING IT because of its
       performance characteristics."

   ③ SERIALIZABLE SNAPSHOT ISOLATION
      "A fairly new algorithm that AVOIDS MOST OF THE DOWNSIDES of the
       previous approaches. It uses an OPTIMISTIC approach, allowing
       transactions to PROCEED WITHOUT BLOCKING. When a transaction wants to
       commit, IT IS CHECKED, AND ABORTED IF THE EXECUTION WAS NOT
       SERIALIZABLE."
```

> And the hook into the next chapters: *"Most of the ideas and algorithms in this chapter apply no matter whether the database is running on a single machine, or replicated and partitioned across multiple machines. However, there is an ADDITIONAL SET OF DIFFICULT CHALLENGES that arises if you try to implement transactions in DISTRIBUTED databases."*

---

# 13. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.7 — TRANSACTIONS                                                     ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  A transaction groups reads+writes into ONE LOGICAL UNIT: all-or-nothing.     ║
║  "Transactions are NOT A LAW OF NATURE — they exist to SIMPLIFY THE           ║
║   PROGRAMMING MODEL." Both "transactions don't scale" and "you need ACID for  ║
║   serious data" are PURE HYPERBOLE.                                           ║
║                                                                               ║
║  ── ACID (1983, Härder & Reuter; now "mostly a MARKETING TERM") ────────────  ║
║  A ATOMICITY   = ABORTABILITY. NOT about concurrency. If a fault hits         ║
║      mid-transaction, DISCARD everything → safe to retry.                     ║
║  C CONSISTENCY = application invariants. A PROPERTY OF THE APPLICATION, not   ║
║      the database. "The letter C doesn't really belong in ACID" — it was      ║
║      "tossed in to make the acronym work."                                    ║
║  I ISOLATION   = concurrent txns don't step on each other. Formalized as      ║
║      SERIALIZABILITY. Rarely used in practice (Oracle 11g doesn't even        ║
║      implement it — its "serializable" is SNAPSHOT ISOLATION).                ║
║  D DURABILITY  = committed data survives crashes. NOTHING IS PERFECT: fsync   ║
║      can lie, SSDs lose data unplugged for weeks, corruption spreads to       ║
║      backups. Only RISK-REDUCTION, no guarantees.                             ║
║  (BASE = "not ACID", i.e. almost anything you want.)                          ║
║                                                                               ║
║  MULTI-OBJECT TXNS NEEDED FOR: foreign keys/edges · denormalized data ·       ║
║     SECONDARY INDEXES (a record can appear in one index but not another).     ║
║  Single-object atomicity is near-universal (log + per-object lock) but CAS    ║
║     marketed as "lightweight transactions" is MISLEADING.                     ║
║  RETRY PITFALLS: it may have SUCCEEDED and the ack was lost · overload gets   ║
║     WORSE · only TRANSIENT errors are worth retrying · EXTERNAL SIDE EFFECTS  ║
║     still happen · the client may die mid-retry. ORMs don't retry at all.     ║
║                                                                               ║
║  ── READ COMMITTED (default in Oracle/Postgres/SQL Server/MemSQL) ──────────  ║
║  ① no DIRTY READS  ② no DIRTY WRITES                                          ║
║  dirty write example (Fig 7-5): car SOLD TO BOB but INVOICED TO ALICE.        ║
║  Impl: row-level WRITE LOCKS for writes; for reads DON'T use locks (one long  ║
║     writer would stall every reader) — keep OLD + NEW value and serve the old.║
║                                                                               ║
║  ── SNAPSHOT ISOLATION / MVCC ──────────────────────────────────────────────  ║
║  Fixes READ SKEW (Fig 7-6: Alice sees $900 of her $1000). Matters for         ║
║     BACKUPS (inconsistency becomes PERMANENT on restore) and ANALYTICS.       ║
║  🔑 MANTRA: READERS NEVER BLOCK WRITERS, WRITERS NEVER BLOCK READERS.          ║
║  Impl: every row has created_by / deleted_by txids. UPDATE = DELETE + CREATE. ║
║  VISIBILITY: ignore writes by txns (1) in-flight at snapshot time — even if   ║
║     they later commit — (2) aborted, (3) with a HIGHER txid. Else visible.    ║
║  Indexes: point at all versions + filter, OR append-only/COPY-ON-WRITE        ║
║     B-trees (CouchDB/Datomic/LMDB) where each new ROOT IS a snapshot.         ║
║  😵 NAMING: Oracle calls it "serializable"; Postgres/MySQL call it            ║
║     "repeatable read"; DB2 uses "repeatable read" for SERIALIZABILITY.        ║
║     The SQL standard predates snapshot isolation. "NOBODY REALLY KNOWS WHAT   ║
║     REPEATABLE READ MEANS."                                                   ║
║                                                                               ║
║  ── LOST UPDATE (read-modify-write; the later write CLOBBERS the earlier) ──  ║
║  MEASURED: 5 threads × 200 increments → 218. 782 UPDATES LOST.                ║
║  FIXES: ① ATOMIC OPS (UPDATE x=x+1) — usually best ② SELECT…FOR UPDATE        ║
║     ③ AUTOMATIC DETECTION (Postgres RR ✅, Oracle ✅, SQL Server ✅,             ║
║        MySQL InnoDB ❌) — best because it needs NO application discipline     ║
║     ④ COMPARE-AND-SET — ⚠️ UNSAFE if the WHERE clause reads an old snapshot   ║
║     ⑤ REPLICATED DBs: locks/CAS DON'T APPLY → siblings+merge or CRDTs.        ║
║        LWW loses updates AND IS THE DEFAULT IN MANY SYSTEMS.                  ║
║                                                                               ║
║  ── WRITE SKEW (Fig 7-8: both doctors go off call → ZERO on call) ──────────  ║
║  = read the same objects, then update DIFFERENT objects. A GENERALIZATION OF  ║
║     LOST UPDATE (same object → lost update; different objects → write skew).  ║
║  ❌ atomic ops don't help ❌ lost-update detection doesn't help                 ║
║  ❌ multi-object constraints mostly unsupported (triggers = "quite hacky")     ║
║  ➜ NEEDS TRUE SERIALIZABILITY. Second best: SELECT … FOR UPDATE.              ║
║  MORE CASES: meeting rooms · game moves · usernames (✅ unique constraint!) ·  ║
║     double-spending. ALL share the CHECK-THEN-ACT pattern.                    ║
║  PHANTOM = a write in one txn changes the RESULT OF A SEARCH in another. When ║
║     the check is for ABSENCE of rows, FOR UPDATE has nothing to lock.         ║
║     MATERIALIZING CONFLICTS (pre-create lock rows) = UGLY LAST RESORT.        ║
║                                                                               ║
║  ── SERIALIZABILITY: THREE IMPLEMENTATIONS ─────────────────────────────────  ║
║  ① ACTUAL SERIAL EXECUTION (VoltDB/H-Store, Redis, Datomic). Feasible since   ║
║     ~2007: cheap RAM + OLTP txns are short. REQUIRES STORED PROCEDURES        ║
║     (one network hop, not four) in real languages (Java/Groovy/Clojure/Lua).  ║
║     VoltDB replicates BY RE-RUNNING THE PROCEDURE → must be DETERMINISTIC.    ║
║     Limits: one slow txn STALLS EVERYTHING · dataset must fit in RAM ·        ║
║     single core unless partitioned · CROSS-PARTITION ≈ 1,000 writes/sec and   ║
║     CANNOT BE IMPROVED BY ADDING MACHINES.                                    ║
║  ② TWO-PHASE LOCKING (MySQL InnoDB, SQL Server serializable; DB2 RR).         ║
║     ⚠️ 2PL ≠ 2PC. Shared/exclusive locks, held to END of txn (hence "two      ║
║     phase": acquire, then release all at once). READERS BLOCK WRITERS AND     ║
║     WRITERS BLOCK READERS — the exact opposite of snapshot isolation.         ║
║     ❌ unstable latency, terrible at high percentiles, ONE slow txn can grind  ║
║        the system to a halt; DEADLOCKS are frequent → abort + redo all work.  ║
║     PHANTOMS: PREDICATE LOCKS are correct but SLOW → INDEX-RANGE (next-key)   ║
║     LOCKS approximate them by locking a BIGGER set (always safe). No suitable ║
║     index → fall back to a WHOLE-TABLE shared lock.                           ║
║  ③ SSI (Cahill 2008; PostgreSQL 9.1+, FoundationDB). OPTIMISTIC.              ║
║     Snapshot isolation + detection of SERIALIZATION CONFLICTS at commit.      ║
║     Core idea: a txn acts on a PREMISE that may go STALE. Two detections:     ║
║        (a) STALE MVCC READ — we ignored a write that has since committed      ║
║        (b) WRITE AFFECTING A PRIOR READ — index-range info as a TRIPWIRE      ║
║            that NOTIFIES rather than BLOCKS                                   ║
║     Checks are deferred to COMMIT so READ-ONLY TXNS NEVER ABORT.              ║
║     ✅ predictable latency, scales across machines  ❌ bad under HIGH          ║
║        CONTENTION (retry storms); read-write txns must be SHORT.              ║
║                                                                               ║
║  ── MEASURED PREVENTION MATRIX ─────────────────────────────────────────────  ║
║                          read committed   snapshot   serializable            ║
║     dirty read                  ✅            ✅            ✅                  ║
║     dirty write                 ✅            ✅            ✅                  ║
║     read skew                   ❌            ✅            ✅                  ║
║     lost update                 ❌         ❌ (varies!)     ✅                  ║
║     write skew                  ❌            ❌            ✅                  ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 14. ✅ Test yourself

1. **Why does Kleppmann say the C doesn't belong in ACID?**
   → Atomicity, isolation and durability are properties the *database* enforces. Consistency is an application-defined invariant — the database can't stop you writing data that violates it. Hellerstein's remark is that the C was tossed in to make the acronym pronounceable.

2. **Atomicity would be better named what, and why?**
   → Abortability. ACID atomicity isn't about concurrency (that's isolation); it's about discarding all writes when a fault interrupts a transaction partway, so the application can safely retry without risking duplicate effects.

3. **Read committed prevented the dirty-write case in Figure 7-5 but not the counter race in Figure 7-1. What's the difference?**
   → In Figure 7-5 the second write lands on an *uncommitted* value — a dirty write. In Figure 7-1 the second write happens after the first transaction committed, so nothing dirty occurred; the problem is that the second transaction's *read* was stale. That's a lost update, which needs different machinery.

4. **Why is read skew merely annoying for Alice's banking page but serious for a backup?**
   → For Alice it's transient — refreshing shows the right total. A backup is a durable artifact: if it captures different parts of the database at different times, the inconsistency is written into the copy, and restoring makes the missing $100 **permanent**.

5. **Under snapshot isolation, which writes does a transaction ignore?**
   → Three kinds: writes by transactions that were in flight when its snapshot was taken (even if they later commit), writes by aborted transactions, and writes by transactions with a higher transaction ID. Everything else is visible.

6. **Write skew is a "generalization of lost update." Explain.**
   → Both start with two transactions reading the same objects and then writing. If they write the *same* object you get a lost update or dirty write. If they write *different* objects — Alice's row and Bob's row — you get write skew, which is why single-object protections and lost-update detection both fail on it.

7. **`SELECT ... FOR UPDATE` fixes the doctors case but not the meeting-room case. Why?**
   → In the doctors case the rows being updated were returned by the initial SELECT, so there's something to lock. The meeting-room case checks for the *absence* of rows and then inserts one; if the query returns nothing, there's no row to attach a lock to. That's a phantom.

8. **What does 2PL's "readers block writers and writers block readers" cost you?**
   → Concurrency, and specifically tail latency. Any pair of transactions that could conceivably race now serialize, queues form with no bound on wait time, and one slow lock-heavy transaction can stall the system. Deadlocks also become frequent, and each abort throws away completed work.

9. **Why does SSI defer its conflict check to commit time instead of aborting the moment a stale read is detected?**
   → Because at read time the database doesn't yet know whether the transaction will write. A read-only transaction can never cause write skew and should never be aborted — SSI needs to support long-running consistent-snapshot reads, just like plain snapshot isolation.

10. **Index-range locks lock more rows than strictly necessary. Why is that safe?**
    → Because widening a predicate is always conservative: any write matching the original condition also matches the broader approximation. You get false conflicts (unnecessary waiting) but never missed ones, and the cost of checking is far lower than for exact predicate locks.

---

*All quoted material, figures and examples are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 7. Diagrams have been redrawn in ASCII from the book's originals. The miniature MVCC database, the measured outputs, and the prevention matrix in §11 are supplementary material I added; every number quoted was produced by running the accompanying code.*
