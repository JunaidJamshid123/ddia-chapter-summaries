# DDIA — Chapter 3: Storage and Retrieval
### Complete study guide — theory, every diagram redrawn, working implementations, and added material

> *Wer Ordnung hält, ist nur zu faul zum Suchen.*
> (If you keep things tidily ordered, you're just too lazy to go searching.)
> — German proverb

That proverb is a joke about the central trade-off of the chapter. Sorting things costs you effort **up front**; searching costs you effort **later**. Every storage engine in this chapter is a different answer to *when do you want to pay?*

---

## 0. The map of this chapter

> On the most fundamental level, a database needs to do two things: **when you give it some data, it should store the data — and when you ask it again later, it should give the data back to you.**

Chapter 2 was the *developer's* view: the format in which you hand data over. Chapter 3 is **the database's own view**: how it stores what you gave it, and how it finds it again.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│                          STORAGE ENGINES                                   │
│                                 │                                          │
│           ┌─────────────────────┴──────────────────────┐                   │
│           ▼                                            ▼                   │
│   ╔═══════════════════╗                    ╔═══════════════════════╗       │
│   ║ OLTP              ║                    ║ OLAP / ANALYTICS      ║       │
│   ║ transaction       ║                    ║ data warehouse        ║       │
│   ║ processing        ║                    ║                       ║       │
│   ╠═══════════════════╣                    ╠═══════════════════════╣       │
│   ║ bottleneck:       ║                    ║ bottleneck:           ║       │
│   ║   DISK SEEK TIME  ║                    ║   DISK BANDWIDTH      ║       │
│   ╚═════════╤═════════╝                    ╚═══════════╤═══════════╝       │
│             │                                          │                   │
│      ┌──────┴───────┐                                  ▼                   │
│      ▼              ▼                          COLUMN-ORIENTED             │
│  LOG-STRUCTURED  UPDATE-IN-PLACE               storage                     │
│  ─────────────   ───────────────               • compression               │
│  append only,    fixed-size pages,             • bitmap + RLE              │
│  never modify    overwrite them                • sort orders               │
│                                                • data cubes                │
│  Bitcask         B-TREES                                                   │
│  SSTables        (relational DBs,                                          │
│  LSM-trees        many NoSQL too)                                          │
│  LevelDB                                                                   │
│  Cassandra                                                                 │
│  HBase, Lucene                                                             │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Why should you care if you'll never write a storage engine?

> You're probably not going to implement your own storage engine from scratch, **but you do need to select a storage engine that is appropriate for your application**, from the many available. In order to **tune** a storage engine to perform well on your kind of workload, you need a rough idea of what it is doing under the hood.

---

# PART A — DATA STRUCTURES THAT POWER YOUR DATABASE

## 1. The world's simplest database

Two bash functions. That's the whole database:

```bash
#!/bin/bash

db_set () {
    echo "$1,$2" >> database
}

db_get () {
    grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

**It works:**

```bash
$ db_set 123456 '{"name":"London","attractions":["Big Ben","London Eye"]}'
$ db_set 42     '{"name":"San Francisco","attractions":["Golden Gate Bridge"]}'

$ db_get 42
{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
```

**And updates work too — but look at what actually happens on disk:**

```bash
$ db_set 42 '{"name":"San Francisco","attractions":["Exploratorium"]}'

$ db_get 42
{"name":"San Francisco","attractions":["Exploratorium"]}

$ cat database
123456,{"name":"London","attractions":["Big Ben","London Eye"]}
42,{"name":"San Francisco","attractions":["Golden Gate Bridge"]}     ← OLD VALUE STILL HERE
42,{"name":"San Francisco","attractions":["Exploratorium"]}          ← new value appended
```

```
   ┌───────────────────────────────────────────────────────────────────────┐
   │  Every db_set APPENDS. Old versions are never overwritten.            │
   │  db_get must take the LAST occurrence — hence  tail -n 1              │
   └───────────────────────────────────────────────────────────────────────┘
```

### The verdict: brilliant writes, catastrophic reads

```
   ┌──────────────────────────────┬──────────────────────────────────────┐
   │  db_set  ✅  SURPRISINGLY GOOD│  db_get  ❌  TERRIBLE                 │
   ├──────────────────────────────┼──────────────────────────────────────┤
   │  Appending to a file is       │  Scans the ENTIRE file from          │
   │  generally VERY EFFICIENT.    │  beginning to end, every time.       │
   │                               │                                      │
   │  Many databases internally    │  Cost of a lookup = O(n).            │
   │  use a LOG — an append-only   │  Double the records → double the     │
   │  data file — quite similar    │  lookup time.                        │
   │  to this.                     │                                      │
   └──────────────────────────────┴──────────────────────────────────────┘
```

**I ran exactly this to confirm the O(n) claim:**

```
   n =   1,000   lookup =   0.40 ms
   n =  10,000   lookup =   2.57 ms      ← 10x data,  6.4x slower
   n = 100,000   lookup =  15.57 ms      ← 10x data,  6.1x slower
   n = 400,000   lookup =  63.47 ms      ←  4x data,  4.1x slower
```

Textbook linear. Now extrapolate: at 100 million records that's a **4-second** lookup, for a single key.

> 📖 **Terminology note from the book:** "log" here does **not** mean application logs (human-readable text describing what's happening). It means **an append-only sequence of records**. It doesn't have to be human-readable — it might be binary, intended only for other programs.

**What real databases add on top of this basic principle:** concurrency control, reclaiming disk space so the log doesn't grow forever, error handling, partially-written records. But the core idea is the same.

---

## 2. What an index is, and what it costs

> The general idea: **keep some additional metadata on the side, which acts as a signpost and helps you locate the data you want.** If you want to search the same data in several different ways, you may need several different indexes on different parts of the data.

```
   ╔═════════════════════════════════════════════════════════════════════════╗
   ║                THE CENTRAL TRADE-OFF OF ALL STORAGE SYSTEMS             ║
   ╠═════════════════════════════════════════════════════════════════════════╣
   ║                                                                         ║
   ║      WELL-CHOSEN INDEXES              BUT EVERY INDEX                   ║
   ║      SPEED UP READ QUERIES     ◄────► SLOWS DOWN WRITES                 ║
   ║                                                                         ║
   ║   Because: for writes, it's hard to beat simply appending to a file —   ║
   ║   the simplest possible write operation. Any index must ALSO be         ║
   ║   updated every single time data is written.                            ║
   ║                                                                         ║
   ║   ➜ CONSEQUENCE: databases don't index everything by default. They      ║
   ║     require YOU to choose indexes manually, using your knowledge of     ║
   ║     the application's typical query patterns.                           ║
   ╚═════════════════════════════════════════════════════════════════════════╝
```

Also important: **an index is derived from the primary data.** Adding or removing one **does not affect the contents** of the database — only the performance of queries.

---

## 3. Hash indexes

The simplest possible strategy: **keep an in-memory hash map where every key maps to a byte offset in the data file.**

### 🔷 Figure 3-1 — Log of key-value pairs with an in-memory hash index

```
    IN-MEMORY HASH MAP                    (kept entirely in RAM)
    ┌──────────┬─────────────┐
    │   key    │ byte offset │
    ├──────────┼─────────────┤
    │  123456  │      0      │────────────┐
    │  42      │     64      │──────────┐ │
    └──────────┴─────────────┘          │ │
                                        │ │
    LOG-STRUCTURED FILE ON DISK         │ │      (each box = one byte)
    ════════════════════════════════════│═│════════════════════════════════
                                        │ │
    offset 0 ◄──────────────────────────│─┘
    ┌───────────────────────────────────▼──────────────────────────────────┐
  0 │ 1 2 3 4 5 6 , { " n a m e " : " L o n d o n " , " a t t r a          │
 30 │ c t i o n s " : [ " B i g   B e n " , " L o n d o n   E y e          │
 60 │ " ] } \n │ 4 2 , { " n a m e " : " S a n   F r a n c i s c o "       │
    └──────────┼───────────────────────────────────────────────────────────┘
    offset 64 ◄┘
 90 │ , " a t t r a c t i o n s " : [ " G o l d e n   G a t e   B          │
120 │ r i d g e " ] } \n                                                   │

    WRITE:  append to file, then update hash map with the new offset
            (works for both new keys AND updates)
    READ:   hash map → offset → seek to that location → read the value
                                 └── exactly ONE disk seek ──┘
```

### This is real: Bitcask (Riak's default storage engine)

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │  ✅ High-performance reads AND writes                                │
   │                                                                     │
   │  ⚠️  REQUIREMENT: all the KEYS must fit in available RAM             │
   │      (the hash map is kept completely in memory)                    │
   │                                                                     │
   │  ✅ VALUES can exceed memory — loaded from disk with ONE seek.       │
   │      And if that part of the file is already in the filesystem      │
   │      cache, a read requires NO disk I/O at all.                     │
   └─────────────────────────────────────────────────────────────────────┘
```

**The ideal workload — and it's very specific:**

> The key might be the URL of a cat video, and the value the number of times it has been played (incremented on every play). **Lots of writes, but not too many distinct keys** — a large number of **writes per key**.

### Compaction: how you avoid running out of disk

Break the log into **segments** of a certain size. **Compaction** = throw away duplicate keys, keep only the most recent update for each key.

### 🔷 Figure 3-2 — Compaction of a key-value update log

```
   DATA FILE SEGMENT (12 records)
   ┌───────────┬───────────┬───────────┬───────────┬───────────┬───────────┐
   │ mew: 1078 │purr: 2103 │purr: 2104 │ mew: 1079 │ mew: 1080 │ mew: 1081 │
   ├───────────┼───────────┼───────────┼───────────┼───────────┼───────────┤
   │purr: 2105 │purr: 2106 │purr: 2107 │yawn:  511 │purr: 2108 │ mew: 1082 │
   └───────────┴───────────┴───────────┴───────────┴───────────┴───────────┘
                                    │
                                    │  COMPACTION PROCESS
                                    │  (keep only the LAST value per key)
                                    ▼
   COMPACTED SEGMENT (3 records)
   ┌───────────┬───────────┬───────────┐
   │yawn:  511 │ mew: 1082 │purr: 2108 │
   └───────────┴───────────┴───────────┘

   12 records ──────► 3 records            (counting cat-video plays)
```

### 🔷 Figure 3-3 — Compaction AND segment merging, simultaneously

Since compaction often makes segments much smaller, you can **merge several segments at the same time**:

```
   DATA FILE SEGMENT 1
   ┌───────────┬───────────┬───────────┬───────────┬───────────┬───────────┐
   │ mew: 1078 │purr: 2103 │purr: 2104 │ mew: 1079 │ mew: 1080 │ mew: 1081 │
   ├───────────┼───────────┼───────────┼───────────┼───────────┼───────────┤
   │purr: 2105 │purr: 2106 │purr: 2107 │yawn:  511 │purr: 2108 │ mew: 1082 │
   └───────────┴───────────┴───────────┴───────────┴───────────┴───────────┘

   DATA FILE SEGMENT 2                                          ← more recent
   ┌───────────┬───────────┬───────────┬────────────┬──────────┬───────────┐
   │purr: 2109 │purr: 2110 │ mew: 1083 │scratch: 252│ mew: 1084│ mew: 1085 │
   ├───────────┼───────────┼───────────┼────────────┼──────────┼───────────┤
   │purr: 2111 │ mew: 1086 │purr: 2112 │purr: 2113  │ mew: 1087│purr: 2114 │
   └───────────┴───────────┴───────────┴────────────┴──────────┴───────────┘
                                    │
                                    │  COMPACTION AND MERGING PROCESS
                                    ▼
   MERGED SEGMENTS 1 AND 2
   ┌───────────┬────────────┬───────────┬───────────┐
   │yawn:  511 │scratch: 252│ mew: 1087 │purr: 2114 │
   └───────────┴────────────┴───────────┴───────────┘

   24 records ──────► 4 records          (newer segment WINS on conflicts)
```

**I implemented this with real byte offsets and file seeks. Actual output:**

```
segment 1 (12 records)  -> compacted: {'mew':'1082', 'purr':'2108', 'yawn':'511'}
segment 2 (12 records)  -> compacted: {'purr':'2114', 'mew':'1087', 'scratch':'252'}
merged 1+2              : {'mew':'1087', 'purr':'2114', 'yawn':'511', 'scratch':'252'}
=> 24 records collapse to 4 -- a 6x reduction
```

### 🔑 Why merging can happen without downtime

```
   Segments are NEVER MODIFIED after being written (immutable).
   The merged segment is written to a NEW FILE.

   TIMELINE:
   ─────────────────────────────────────────────────────────────────────►
   [ merging runs in a BACKGROUND THREAD ]
              │
              │  meanwhile: reads and writes continue as normal,
              │             served from the OLD segment files
              ▼
   [ merge complete ] ──► switch reads to the NEW merged segment
                          ──► delete the old segment files

   ➜ Immutability is what makes this safe. Nothing is being changed
     out from under a concurrent reader.
```

### The five implementation details that matter in practice

| Issue | Solution |
|---|---|
| **File format** | CSV is not the best format for a log. Faster and simpler: **a binary format that encodes the length of a string in bytes, followed by the raw string** — no escaping needed |
| **Deleting records** | Append a special deletion record called a **tombstone**. When segments merge, the tombstone tells the merge process to discard all previous values for that key |
| **Crash recovery** | On restart, the in-memory hash maps are lost. You *could* rebuild by reading every segment end-to-end — but that makes restarts painful. **Bitcask stores a snapshot of each segment's hash map on disk** to load faster |
| **Partially written records** | The DB may crash halfway through appending. **Bitcask files include checksums** so corrupted parts of the log are detected and ignored |
| **Concurrency control** | Writes are appended in strictly sequential order → **one writer thread** is a common choice. Segments are append-only and immutable, so **many threads can read concurrently** |

### Why append-only, when overwriting seems less wasteful?

```
   ① SEQUENTIAL WRITES ARE MUCH FASTER THAN RANDOM WRITES
      Appending and segment merging are both sequential.
      This holds for BOTH spinning disks AND SSDs.

   ② CONCURRENCY AND CRASH RECOVERY ARE MUCH SIMPLER
      You never have to handle the case where a crash happened
      mid-overwrite, leaving a file with PART OF THE OLD VALUE
      AND PART OF THE NEW VALUE SPLICED TOGETHER.
                    ┌──────────────────────────┐
                    │ old old old NEW NEW new  │ ← corrupted, unrecoverable
                    └──────────────────────────┘

   ③ MERGING AVOIDS FRAGMENTATION of data files over time
```

### ❌ But hash indexes have two hard limits

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │ LIMIT 1 — THE HASH TABLE MUST FIT IN MEMORY                         │
   │                                                                     │
   │   Very large number of keys → you're out of luck.                   │
   │   Could you keep the hash map on disk? In principle — but it's      │
   │   hard to make an on-disk hash map perform well:                    │
   │     • lots of random-access I/O                                     │
   │     • expensive to grow when full                                   │
   │     • hash collisions require fiddly logic                          │
   ├─────────────────────────────────────────────────────────────────────┤
   │ LIMIT 2 — RANGE QUERIES ARE NOT EFFICIENT                           │
   │                                                                     │
   │   You cannot easily fetch all keys between kitty00000 and           │
   │   kitty99999 — you'd have to look up EACH KEY INDIVIDUALLY.         │
   │   (Hashing deliberately destroys ordering. That's the point of      │
   │    a hash — and here it's the problem.)                             │
   └─────────────────────────────────────────────────────────────────────┘
```

---

## 4. SSTables and LSM-trees

**One small change fixes both limits:** require that the sequence of key-value pairs in each segment file **is sorted by key.**

That's a **Sorted String Table**, or **SSTable**. (Also required: each key appears only once per merged segment — the merging process already guarantees this.)

> At first glance, that requirement seems to break our ability to use sequential writes. We'll get to that in a moment.

### The three advantages of sorting

#### ① Merging becomes simple and efficient — even for files bigger than memory

### 🔷 Figure 3-4 — Merging several SSTable segments

```
   SEGMENT 1 (oldest)
   ┌──────────────┬──────────────┬──────────────┬────────────────────┐
   │ handbag:8786 │handful:40308 │handicap:65995│handkerchief:16324  │
   ├──────────────┼──────────────┴──────────────┴────────────────────┤
   │handlebars:3869│handprinted:11150                                │
   └──────────────┴──────────────────────────────────────────────────┘

   SEGMENT 2
   ┌──────────────┬──────────────┬──────────────┬────────────────────┐
   │handcuffs:2729│handful:42307 │handicap:67884│handiwork:16912     │
   ├──────────────┴───┬──────────┴──────────────┴────────────────────┤
   │handkerchief:20952│handprinted:15725                             │
   └──────────────────┴──────────────────────────────────────────────┘

   SEGMENT 3 (newest)
   ┌──────────────┬──────────────┬──────────────┬────────────────────┐
   │handful:44662 │handicap:70836│handiwork:45521│handlebars:3869    │
   ├──────────────┼──────────────┴───────────────┴───────────────────┤
   │handoff:5741  │handprinted:33632                                 │
   └──────────────┴──────────────────────────────────────────────────┘
                            │
                            │   COMPACTION AND MERGING PROCESS
                            │   (like mergesort: read all inputs side by side,
                            │    copy the LOWEST key to output, repeat)
                            ▼
   MERGED 1, 2, 3
   ┌──────────────┬──────────────┬──────────────┬────────────────────┐
   │ handbag:8786 │handcuffs:2729│handful:44662 │handicap:70836      │
   ├──────────────┼──────────────┼──────────────┼────────────────────┤
   │handiwork:45521│handkerchief:20952│handlebars:3869│handoff:5741  │
   ├──────────────┴──────────────┴──────────────┴────────────────────┤
   │handprinted:33632                                                │
   └─────────────────────────────────────────────────────────────────┘
```

**Why memory size doesn't matter:** you only ever hold **one key from each input file** at a time. The files stream past.

**And the conflict rule falls out for free:**

> Each segment contains all the values written during **some period of time**. So all the values in one input segment must be **more recent** than all the values in another (assuming you always merge adjacent segments). When multiple segments contain the same key, **keep the value from the most recent segment.**

**My streaming implementation produced exactly the book's merged output:**

```
   handbag        8786          handkerchief   20952
   handcuffs      2729          handlebars     3869
   handful        44662         handoff        5741
   handicap       70836         handprinted    33632
   handiwork      45521
```

#### ② You no longer need every key in memory — the index can be SPARSE

### 🔷 Figure 3-5 — An SSTable with a sparse in-memory index

```
   SPARSE INDEX (in memory)          SORTED SEGMENT FILE (SSTable) on disk
   ┌──────────┬─────────────┐        ┌──────────────────────────────────────┐
   │   key    │ byte offset │        │ ……… hand: 91541                      │
   ├──────────┼─────────────┤        ├──────────────────────────────────────┤
   │    …     │      …      │        │ handbag:8786  handcuffs:2729         │┐
   │ hammock  │   100491    │───────►│ handful:44662                        ││
   ├──────────┼─────────────┤        ├──────────────────────────────────────┤│ ONE
   │ handbag  │   102134    │───────►│ handicap:70836  handiwork:45521      ││ COMPRESSIBLE
   ├──────────┼─────────────┤        │ handkerchief:20952                   ││ BLOCK
   │ handsome │   104667    │──┐     ├──────────────────────────────────────┤│
   ├──────────┼─────────────┤  │     │ handlebars:3869  handoff:5741        ││
   │ hangout  │   106812    │  │     │ handprinted:33632                    │┘
   ├──────────┼─────────────┤  │     ├──────────────────────────────────────┤
   │    …     │      …      │  └────►│ handsome:86478  handwaving:44005     │
   └──────────┴─────────────┘        │ handwriting:22846                    │
                                     ├──────────────────────────────────────┤
                                     │ ………                                  │
                                     └──────────────────────────────────────┘

   LOOKING FOR "handiwork" — you don't know its offset. But:
      • you know the offset of handbag  (102134)
      • you know the offset of handsome (104667)
      • SORTING guarantees handiwork lies BETWEEN them
   ➜ jump to 102134, scan forward until found (or prove it's absent)

   One index key per FEW KILOBYTES is enough — a few KB scans very fast.
```

> 📖 **Footnote worth knowing:** if all keys and values had a **fixed size**, you could binary-search the segment file and skip the in-memory index entirely. In practice they're variable-length, which makes it hard to tell where one record ends and the next begins without an index.

#### ③ Blocks can be compressed before writing

Since reads scan over several key-value pairs in the range anyway, **group records into a block and compress it.** Each sparse index entry points at the start of a compressed block.

> **Nowadays, disk bandwidth is usually a worse bottleneck than CPU**, so it is worth spending a few additional CPU cycles to reduce the amount of data you read and write.

### But how do you get sorted data from unsorted writes?

**Answer: sort in memory, where it's easy.** Red-Black trees or AVL trees let you insert in any order and read back in sorted order.

### 🔷 The complete LSM-tree architecture

```
                          ┌──────────────┐
       WRITE ────────────►│              │
         │                │   MEMTABLE   │  in-memory balanced tree
         │                │  (Red-Black) │  (a few MB)
         │                └──────┬───────┘
         │                       │
         ▼                       │ when it exceeds the threshold,
   ┌───────────┐                 │ write out as an SSTable
   │    WAL    │                 │ (efficient — already sorted!)
   │ (on disk) │                 ▼
   │ append-   │        ┌─────────────────┐  ◄── newest segment
   │ only, un- │        │   SSTable  L0   │
   │ sorted    │        ├─────────────────┤
   └───────────┘        │   SSTable  L1   │
   purpose: restore     ├─────────────────┤
   the memtable after   │   SSTable  L2   │
   a crash. Discarded   ├─────────────────┤
   whenever memtable    │   SSTable  L3   │  ◄── oldest segment
   is flushed.          └────────┬────────┘
                                 │
                                 │  BACKGROUND merging & compaction
                                 ▼
                        combine segments, discard
                        overwritten & deleted values


   READ PATH:  memtable → newest SSTable → next-older → … → oldest
               └──────────── stop at the first hit ────────────┘
```

**The four rules:**

```
   1. WRITE → add to the in-memory balanced tree (the MEMTABLE)
   2. MEMTABLE EXCEEDS THRESHOLD (a few MB) → write out to disk as an
      SSTable file. Efficient, because the tree already keeps things
      sorted. It becomes the most recent segment. Memtable is emptied.
   3. READ → try memtable, then most recent on-disk segment, then the
      next-older, etc.
   4. FROM TIME TO TIME → run merging + compaction in the background.
```

**The one problem, and its fix:**

> If the database crashes, the most recent writes (in the memtable, not yet on disk) **are lost.** So keep **a separate log on disk to which every write is immediately appended**. That log is **not** in sorted order — but that doesn't matter, because **its only purpose is to restore the memtable after a crash.** Every time the memtable is written out, the corresponding log can be discarded.

### Who uses this

| System | Note |
|---|---|
| **LevelDB, RocksDB** | Embeddable key-value storage engine libraries. LevelDB can be used in Riak as an alternative to Bitcask |
| **Cassandra, HBase** | Both inspired by Google's **Bigtable** paper, which introduced the terms *SSTable* and *memtable* |
| **Original name** | Described by Patrick O'Neil et al. as the **Log-Structured Merge-Tree (LSM-Tree)**, building on log-structured file systems |
| **Lucene** (Elasticsearch, Solr) | Uses a similar method for its **term dictionary** |

**How Lucene applies it — worth understanding, since full-text search feels unrelated:**

```
   A full-text index is more complex than a key-value index, but at its
   core is the same idea:

       KEY   = a word (a TERM)
       VALUE = the IDs of all documents containing that word
               (the POSTINGS LIST)

   term → postings list is kept in SSTable-like SORTED FILES,
   merged in the background as needed.
```

### 🎈 Bloom filters — the fix for the LSM-tree's worst case

> The LSM-tree algorithm can be **slow when looking up keys that do not exist**: you have to check the memtable, then the segments **all the way back to the oldest** (possibly reading from disk each time) before you can be sure the key doesn't exist.

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  A BLOOM FILTER is a memory-efficient data structure for             │
   │  APPROXIMATING the contents of a set.                                │
   │                                                                      │
   │      "is key X in this SSTable?"                                     │
   │                                                                      │
   │        NO   ──► DEFINITELY not present.  Skip the disk read. ✅       │
   │        YES  ──► PROBABLY present. Go and check. (may be a false      │
   │                 positive — but never a false NEGATIVE)               │
   │                                                                      │
   │  ➜ saves many unnecessary disk reads for non-existent keys           │
   └──────────────────────────────────────────────────────────────────────┘
```

**I built one and measured it — 10,000 keys, 10 bits per key, 7 hash functions:**

```
   false positives on 100,000 absent keys: 837  (0.84%)
   => 99.16% of misses avoid touching the disk entirely
```

For **10 bits per key** — about 12 KB of RAM for 10,000 keys — you eliminate 99% of pointless disk reads. That's an extraordinary return.

### And here is my LSM-tree actually running, with a trace of each lookup

```
get('handiwork'):
   sstable 2: MISS_BLOOM          ← bloom filter said no. Zero disk I/O.
   sstable 1: MISS_BLOOM          ← again, zero disk I/O
   sstable 0: scanned 1 recs      ← sparse index landed us right on it
   => 4000

get('handful'):
   sstable 2: scanned 2 recs      ← found in newest segment, stop immediately
   => 99999                       ← the UPDATED value, not the original

get('handbag'):
   sstable 2: scanned 1 recs
   => None                        ← TOMBSTONE found. Correctly reports deleted.

get('zzz_nonexistent'):
   sstable 2: MISS_BLOOM
   sstable 1: MISS_BLOOM
   sstable 0: MISS_BLOOM
   => None                        ← the worst case, made cheap by bloom filters
```

**Every mechanism in the section is visible in that trace:** newest-first search order, tombstones, sparse-index scanning, and bloom filters turning the worst case into no disk I/O at all.

### Why LSM-trees work well

> Even when the dataset is **much bigger than memory** it continues to work well. Since data is stored in sorted order, you can **efficiently perform range queries**. And because the disk writes are sequential, the LSM-tree can support **remarkably high write throughput.**

---

## 5. B-trees

> The log-structured indexes we have discussed are gaining acceptance, but **they are not the most common type of index.** The most widely used indexing structure is quite different: **the B-tree.**

**Introduced in 1970. Called "ubiquitous" less than 10 years later.** Still the standard index implementation in almost all relational databases, and many non-relational ones.

### The fundamental difference from LSM-trees

```
   ┌────────────────────────────────┬────────────────────────────────────┐
   │  LOG-STRUCTURED                │  B-TREE                            │
   ├────────────────────────────────┼────────────────────────────────────┤
   │  VARIABLE-SIZE segments        │  FIXED-SIZE blocks or PAGES        │
   │  (several megabytes or more)   │  (traditionally 4 kB)              │
   │                                │                                    │
   │  always write a segment        │  read or write ONE PAGE at a time  │
   │  SEQUENTIALLY                  │                                    │
   │                                │  ➜ corresponds more closely to the │
   │                                │    underlying hardware — disks are │
   │                                │    also arranged in fixed-size     │
   │                                │    blocks                          │
   └────────────────────────────────┴────────────────────────────────────┘
```

Each page has an **address**, which lets one page refer to another — **like a pointer, but on disk instead of in memory.**

### 🔷 Figure 3-6 — Looking up a key in a B-tree

```
                          "Look up user_id = 251"
                                    │
                                    ▼
   ROOT PAGE
   ┌────┬─────┬────┬─────┬────┬─────┬────┬─────┬────┬─────┬────┬─────┬────┐
   │ref │ 100 │ref │ 200 │ref │ 300 │ref │ 400 │ref │ 500 │ref │     │    │
   └─┬──┴─────┴─┬──┴─────┴─┬──┴─────┴─┬──┴─────┴─┬──┴─────┴─┬──┴─────┴────┘
     │          │          │          │          │          │
  key<100  100≤key<200  200≤key<300  300≤key<400  400≤key<500  key≥500
     │          │          │
     │          │          │  251 falls between 200 and 300 ──┐
     ▼          ▼          │                                  │
   ┌───────────────────────▼──────────────────────────────────▼─────────┐
   │ ref │ 210 │ ref │ 230 │ ref │ 250 │ ref │ 270 │ ref │ 290 │ ref     │
   └─────┴─────┴─────┴─────┴──┬──┴─────┴─────┴─────┴─────┴─────┴─────────┘
                              │
                      250 ≤ key < 270
                              │
                              ▼
   LEAF PAGE
   ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
   │ 250 │ val │ 251 │ val │ 252 │ val │ 253 │ val │ 254 │ val │  ◄── FOUND
   └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
```

**The structure:**

```
   • ONE page is designated the ROOT. Every lookup starts here.
   • A page contains k keys and k+1 REFERENCES to child pages.
     (In the figure k = 5. In reality k is typically in the HUNDREDS.)
   • Each child is responsible for a CONTINUOUS RANGE of keys; the keys
     in the parent indicate where the boundaries lie.
   • A LEAF PAGE either contains the value for each key INLINE, or
     references to pages where each value can be found.

   The number of references to child pages is the BRANCHING FACTOR.
```

### 🔷 Figure 3-7 — Growing a B-tree by splitting a page

**Update an existing key:** find the leaf page, change the value, write the page back. Any references to that page remain valid.

**Add a new key:** find the page whose range encompasses it, add it. **If there isn't enough free space, the page is split into two half-full pages, and the parent is updated.**

```
   BEFORE — inserting 334 into a full page
   ┌─────┬─────┬─────┬─────┬─────┬─────┬────────────────────┐
   │ ref │ 310 │ ref │ 333 │ ref │ 345 │   (spare space)    │  ← parent
   └─────┴─────┴─────┴──┬──┴─────┴─────┴────────────────────┘
                        │  333 ≤ key < 345
                        ▼
   ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
   │ 333 │ val │ 335 │ val │ 337 │ val │ 340 │ val │ 342 │ val │  ← FULL
   └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                                       no room for 334!

   ═══════════════════════════════════════════════════════════════════

   AFTER adding key 334 — the page SPLIT, and the parent gained a key
   ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬──────────┐
   │ ref │ 310 │ ref │ 333 │ ref │ 337 │ ref │ 345 │  (spare) │  ← parent
   └─────┴─────┴─────┴──┬──┴─────┴──┬──┴─────┴─────┴──────────┘
                        │           │
              333≤key<337     337≤key<345
                        ▼           ▼
   ┌─────┬─────┬─────┬─────┬─────┬─────┐   ┌─────┬─────┬─────┬─────┬─────┬─────┐
   │ 333 │ val │ 334 │ val │ 335 │ val │   │ 337 │ val │ 340 │ val │ 342 │ val │
   └─────┴─────┴─────┴─────┴─────┴─────┘   └─────┴─────┴─────┴─────┴─────┴─────┘
        (spare space)                              (spare space)
   └────────── two HALF-FULL pages, both with room to grow ──────────┘
```

**I implemented the split and ran the book's exact example:**

```
Inserting 333, 335, 337, 340, 342 then 334:
   before 334 -> root keys: [337] | height 2
   after  334 -> root keys: [337] | height 2 | splits so far: 1
```

### 🔑 Why the height stays tiny — the reason B-trees win

> This algorithm ensures the tree **remains balanced**: a B-tree with n keys always has a height of **O(log n)**. Even if the tree is very large, you don't need to follow many page references.

**Computed with a realistic branching factor of 500:**

```
   n =             1,000 keys  ->  2 levels  =  at most 2 page reads
   n =         1,000,000 keys  ->  3 levels  =  at most 3 page reads
   n =     1,000,000,000 keys  ->  4 levels  =  at most 4 page reads
   n = 1,000,000,000,000 keys  ->  5 levels  =  at most 5 page reads
```

**A trillion keys, five disk reads.** That is why B-trees have survived 55 years. The high branching factor — hundreds of children per page, not two — is doing all the work. (If you know Red-Black or 2-3 trees, B-trees are very similar, **except for the larger branching factor.**)

> 📖 *Footnote:* inserting into a B-tree is reasonably intuitive. **Deleting** (while keeping the tree balanced) is somewhat more involved.

---

## 6. Update-in-place vs append-only logging

> The basic underlying write operation of a B-tree is to **overwrite a page on disk with new data.** It is assumed that the overwrite **does not change the location of the page**, so all references remain intact. **This is in stark contrast to LSM-trees, which only append to files and never modify them in place.**

**What overwriting actually means in hardware:**

```
   MAGNETIC HARD DRIVE:  move the disk head to the right place,
                         wait for the right position on the spinning
                         platter to come around, overwrite the sector.

   SSD:                  somewhat more complicated, but SIMILARLY SLOW.
```

### ⚠️ The danger: multi-page operations can corrupt the index

```
   A page split requires writing THREE pages:

        ┌─────────────┐
        │   PARENT    │  ← must be updated to point at both children
        └──┬───────┬──┘
           ▼       ▼
     ┌────────┐ ┌────────┐
     │ LEFT   │ │ RIGHT  │  ← both must be written
     └────────┘ └────────┘

   💥 IF THE DATABASE CRASHES AFTER WRITING ONLY SOME OF THEM:
      you get a CORRUPTED INDEX — e.g. an ORPHAN PAGE that is not
      a child of any parent.
```

### The fix: the write-ahead log (WAL)

```
   ┌───────────────────────────────────────────────────────────────────┐
   │  WRITE-AHEAD LOG (WAL), also known as the REDO LOG                │
   │                                                                   │
   │  An append-only file to which EVERY B-tree modification must be   │
   │  written BEFORE it can be applied to the pages of the tree.       │
   │                                                                   │
   │  On restart after a crash, this log is used to restore the        │
   │  B-tree to a consistent state.                                    │
   └───────────────────────────────────────────────────────────────────┘

   ➜ CONSEQUENCE: a B-tree index must write every piece of data
     AT LEAST TWICE — once to the log, once to the tree page itself
     (and perhaps again as pages are split).
```

> 📖 **Write amplification** — one write to the database resulting in multiple writes to disk. Of particular concern on **SSDs, which can only overwrite blocks a limited number of times before wearing out.**

**Kleppmann's honest verdict:** LSM-trees also re-write data multiple times due to repeated background merging. **"It's not clear whether B-trees or LSM-trees are better in this regard — it depends on the workload and the tuning. In the end, there is no alternative to benchmarking systems with your particular workload."**

**I worked out the arithmetic to make the trade-off concrete:**

```
   One logical 100-byte write, in bytes actually written to disk:

   B-TREE                                      LSM-TREE
   ──────                                      ────────
   WAL record          ~100 B                  WAL record       ~100 B
   leaf page rewrite    4096 B                 memtable flush   ~100 B
   (page split:        +8192 B)                L0→L1 merge      ~100 B
                                               L1→L2 merge      ~100 B

   1 page touched  →  4,196 B  =   42x         4 levels  →   500 B  =  5x
   2 pages touched →  8,292 B  =   83x         7 levels  →   800 B  =  8x
   3 pages touched → 12,388 B  =  124x        10 levels  → 1,100 B  = 11x
```

**But that table is misleading on its own, and it's important to see why:** the B-tree's writes are **random** (seek to a page, overwrite it) while the LSM-tree's are **sequential** (stream out a whole segment). On a spinning disk a sequential write can be 100× faster per byte. Raw amplification numbers don't settle the question — which is exactly why the book refuses to declare a winner.

### Concurrency

```
   B-TREE                              LOG-STRUCTURED
   ──────                              ──────────────
   Updating pages IN PLACE means       SIMPLER: all merging is done
   careful concurrency control is      in the BACKGROUND without
   required, or a thread may see       interfering with incoming
   the tree in an INCONSISTENT state.  queries, then old segments are
                                       ATOMICALLY SWAPPED for new ones
   Typically done with LATCHES         from time to time.
   (lightweight locks) protecting
   the tree's data structures.
```

---

## 7. B-tree optimizations

```
   ① COPY-ON-WRITE instead of overwrite+WAL
      Used by LMDB. A modified page is written to a DIFFERENT LOCATION,
      and a new version of the parent pages is created pointing at the
      new location. Also useful for concurrency control (snapshot
      isolation, Chapter 7).

   ② KEY ABBREVIATION
      Don't store the entire key. Especially in INTERIOR pages, keys only
      need enough information to act as BOUNDARIES between ranges.
      ➜ more keys per page ➜ HIGHER BRANCHING FACTOR ➜ FEWER LEVELS
      (This is roughly what's called a B+ tree — so common it often
       isn't distinguished from other B-tree variants.)

   ③ SEQUENTIAL LEAF LAYOUT
      Pages can sit anywhere on disk; nothing requires nearby key ranges
      to be nearby on disk. That's bad for range scans — a disk seek per
      page. Many implementations TRY to lay leaf pages out sequentially,
      but it's DIFFICULT TO MAINTAIN as the tree grows.
      ⚑ By contrast, LSM-trees rewrite large segments in one go during
        merging, so it's EASIER for them to keep sequential keys nearby.

   ④ SIBLING POINTERS
      Each leaf page holds references to its LEFT and RIGHT siblings,
      allowing scans in order without jumping back up to parents.

        ┌──────┐◄──►┌──────┐◄──►┌──────┐◄──►┌──────┐
        │ leaf │    │ leaf │    │ leaf │    │ leaf │
        └──────┘    └──────┘    └──────┘    └──────┘

   ⑤ FRACTAL TREES
      Borrow log-structured ideas to reduce disk seeks.
      (They have nothing to do with fractals.)
```

---

## 8. Comparing B-trees to LSM-trees

```
   ╔══════════════════════════════════════════════════════════════════════╗
   ║  RULE OF THUMB                                                       ║
   ║     LSM-trees are typically FASTER FOR WRITES                        ║
   ║     B-trees are thought to be FASTER FOR READS                       ║
   ║                                                                      ║
   ║  ⚠️  "Actual benchmarks are often INCONCLUSIVE and SENSITIVE to the   ║
   ║      details of the workload."                                       ║
   ╚══════════════════════════════════════════════════════════════════════╝
```

| | **LSM-trees** ✅ | **B-trees** ✅ |
|---|---|---|
| **Write throughput** | Much higher for random writes — **they turn all random writes into sequential writes** on the device | Random page overwrites |
| **Compression / space** | Rewrite large segments; less fragmentation | Fragmentation; pages left partially full |
| **Sequential layout** | Easier to keep sequential keys near each other on disk | Hard to maintain as the tree grows |
| **Predictable latency** | ❌ **Compaction can interfere with ongoing reads and writes.** Disks have limited resources, so a request can wait while an expensive compaction finishes. Impact on throughput and *average* response time is usually small — **but at higher percentiles the response time can be quite high** | ✅ **More predictable** |
| **Key uniqueness** | ❌ May hold **multiple copies of the same key** in different segments | ✅ **Each key exists in exactly one place in the index** |
| **Transactions** | Harder | ✅ Attractive for **strong transactional semantics**: transaction isolation is often implemented with **locks on ranges of keys**, and in a B-tree those locks can be **attached directly to the tree** |
| **Maturity** | Newer, "very promising" | Generally more mature; "very ingrained in the architecture of databases" |

> ⚑ **Notice the callback to Chapter 1.** The compaction problem is described in percentile terms — average impact small, **tail impact large**. This is exactly the p99 argument from Chapter 1, appearing as a concrete engineering consequence.

**The verdict:** *"There is no quick and easy rule for determining which type of storage engine is better for your use case, so it is worth testing empirically."*

---

## 9. Other indexing structures

### Primary vs secondary indexes

```
   PRIMARY KEY INDEX                    SECONDARY INDEX
   ─────────────────                    ───────────────
   Uniquely identifies ONE:             Created with CREATE INDEX.
     • row (relational)                 Often CRUCIAL for performing
     • document (document DB)           JOINS efficiently.
     • vertex (graph DB)
                                        ⚠️ KEYS ARE NOT UNIQUE — many
   Other records refer to it by           rows may share the same key.
   this ID; the index resolves
   those references.                    Two ways to handle that:
                                          ① make each value a LIST of
                                             matching row IDs (like a
                                             postings list)
                                          ② make each key unique by
                                             APPENDING a row identifier

   ➜ Either way, BOTH B-trees AND log-structured indexes work.
```

*Example from the book:* in Figure 2-1 (the LinkedIn schema), you'd want a secondary index on the `user_id` columns so you can find all the rows belonging to one user in each table.

### Storing values within the index: heap file, clustered, covering

```
   ┌─────────────────────────────────────────────────────────────────────────┐
   │  OPTION 1 — HEAP FILE (nonclustered)                                    │
   │                                                                         │
   │   index ──► reference ──► HEAP FILE (data in no particular order)       │
   │                                                                         │
   │   index_A ─┐                                                            │
   │   index_B ─┼──► ┌──────────────────────┐                                │
   │   index_C ─┘    │  the actual row      │  ← stored ONCE                 │
   │                 └──────────────────────┘                                │
   │                                                                         │
   │   ✅ AVOIDS DUPLICATING DATA when multiple secondary indexes exist.      │
   │   ✅ Updating a value without changing the key is efficient — overwrite  │
   │      in place, PROVIDED the new value isn't larger than the old.        │
   │   ⚠️ If the new value IS larger, the record probably must MOVE. Then     │
   │      either ALL indexes are updated to point at the new location,       │
   │      or a FORWARDING POINTER is left behind at the old one.             │
   ├─────────────────────────────────────────────────────────────────────────┤
   │  OPTION 2 — CLUSTERED INDEX                                             │
   │                                                                         │
   │   Store the indexed ROW DIRECTLY WITHIN THE INDEX.                      │
   │   Removes the extra hop from index to heap file.                        │
   │                                                                         │
   │   • MySQL InnoDB: the PRIMARY KEY is ALWAYS a clustered index, and      │
   │     secondary indexes refer to the primary key (not a heap location)    │
   │   • SQL Server: you can specify ONE clustered index per table           │
   ├─────────────────────────────────────────────────────────────────────────┤
   │  OPTION 3 — COVERING INDEX (index with included columns) — a compromise │
   │                                                                         │
   │   Stores SOME of a table's columns within the index.                    │
   │   Some queries can then be answered BY THE INDEX ALONE                  │
   │   — the index is said to COVER the query.                               │
   └─────────────────────────────────────────────────────────────────────────┘

   ⚠️ As with ANY duplication of data: clustered and covering indexes
      speed up reads, but require additional storage and add overhead
      on writes. Databases must also work harder to enforce transactional
      guarantees, so applications don't see inconsistencies.
```

### Multi-column indexes

**Concatenated index** — combine several fields into one key by appending one column to another:

```
   Like an old-fashioned PAPER PHONE BOOK: an index from
   (lastname, firstname) → phone number.

   ✅ find all people with a particular LASTNAME
   ✅ find a particular LASTNAME + FIRSTNAME combination
   ❌ USELESS for finding all people with a particular FIRSTNAME

   ┌───────────────────────────────────────────────────────────┐
   │  (lastname, firstname)   ← the ORDER in the index         │
   │   ▲          ▲              definition is everything      │
   │   │          │                                            │
   │   │          └── can only be used if lastname is fixed    │
   │   └───────────── can always be used                       │
   └───────────────────────────────────────────────────────────┘
```

**Multi-dimensional indexes** — the general solution, especially for geospatial data:

```sql
SELECT * FROM restaurants WHERE latitude  >  51.4946 AND latitude  <  51.5079
                            AND longitude > -0.1162 AND longitude < -0.1004;
```

```
   ❌ A standard B-tree or LSM-tree CANNOT answer this efficiently:

      it can give you       ALL restaurants in a LATITUDE range
                            (but at ANY longitude)
      or                    ALL restaurants in a LONGITUDE range
                            (but anywhere from north to south pole)
      but NOT BOTH SIMULTANEOUSLY.

      ┌─────────────────────────────┐         ┌─────────────────────────────┐
      │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│         │       ░░░░░░░░              │
      │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│   vs    │       ░░░░░░░░              │
      │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│         │       ░░░░░░░░              │
      └─────────────────────────────┘         └─────────────────────────────┘
        latitude index only                     longitude index only
                          you want:  ┌──────┐
                                     │▓▓▓▓▓▓│  ← the intersection
                                     └──────┘

   SOLUTIONS:
     • Translate 2D location into ONE number with a SPACE-FILLING CURVE,
       then use a regular B-tree
     • Specialized SPATIAL INDEXES: R-TREES. PostGIS implements geospatial
       indexes as R-trees via PostgreSQL's Generalized Search Tree facility
```

**The idea generalizes beyond geography — this is the interesting part:**

| Use case | Dimensions |
|---|---|
| E-commerce colour search | 3D index on **(red, green, blue)** → find products in a colour range |
| Weather observations | 2D index on **(date, temperature)** → all 2013 observations where temp was 25–30 °C |

> With a one-dimensional index you'd have to **scan all records from 2013 regardless of temperature and then filter**, or vice versa. A 2D index narrows by **both simultaneously.** (Used by HyperDex.)

### Fuzzy indexes

Everything so far assumes **exact data** and exact queries. What about **misspelled words**?

```
   LUCENE can search text for words within a certain EDIT DISTANCE.
   (Edit distance 1 = one letter added, removed, or replaced.)

   HOW IT WORKS — and note how it differs from LevelDB:

   ┌────────────────────────┬────────────────────────────────────────────┐
   │  LevelDB               │  Lucene                                    │
   ├────────────────────────┼────────────────────────────────────────────┤
   │  in-memory index is a  │  in-memory index is a FINITE STATE         │
   │  SPARSE COLLECTION of  │  AUTOMATON over the characters in the      │
   │  some of the keys      │  keys, similar to a TRIE                   │
   │                        │                                            │
   │                        │  That automaton can be transformed into a  │
   │                        │  LEVENSHTEIN AUTOMATON, which supports     │
   │                        │  efficient search within a given edit      │
   │                        │  distance.                                 │
   └────────────────────────┴────────────────────────────────────────────┘

   Other fuzzy techniques head toward DOCUMENT CLASSIFICATION and
   MACHINE LEARNING — see an information retrieval textbook.
```

---

## 10. Keeping everything in memory

> The data structures discussed so far have all been **answers to the limitations of disks.** Compared to main memory, disks are awkward to deal with — data must be laid out carefully for good performance. We tolerate this because disks have two significant advantages: **they are durable, and they have a lower cost per gigabyte than RAM.**

**But as RAM gets cheaper, the cost-per-gigabyte argument erodes.** Many datasets are simply not that big.

### How in-memory databases achieve durability

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  ① Special hardware — battery-powered RAM (still unusual)            │
   │  ② Writing a LOG OF CHANGES to disk                                  │
   │  ③ Writing PERIODIC SNAPSHOTS to disk                                │
   │  ④ REPLICATING the in-memory state to other machines                 │
   └──────────────────────────────────────────────────────────────────────┘

   On restart, state is reloaded from disk or over the network from
   a replica.

   ⚑ IT'S STILL AN IN-MEMORY DATABASE, because the disk is merely an
     APPEND-ONLY LOG FOR DURABILITY — reads are served ENTIRELY from
     memory.

   Operational bonus: files on disk can easily be backed up, inspected
   and analyzed by external utilities.
```

| System | Note |
|---|---|
| **Memcached** | Caching only — acceptable for data to be lost on restart |
| **VoltDB, MemSQL, Oracle TimesTen** | In-memory with a **relational** model |
| **RAMCloud** | Open source in-memory KV store with durability (log-structured for both memory and disk) |
| **Redis, Couchbase** | **Weak** durability — write to disk asynchronously |

### 🤔 The counter-intuitive part — why in-memory databases are actually fast

```
   ❌ WRONG REASON: "because they don't need to read from disk"

      Even a DISK-BASED engine may never read from disk if you have
      enough memory — the OPERATING SYSTEM CACHES recently used disk
      blocks in memory anyway.

   ✅ RIGHT REASON: they avoid the OVERHEADS OF ENCODING in-memory data
      structures into a form that can be WRITTEN TO DISK.

      ┌──────────────────────────────────────────────────────────┐
      │  in-memory object  ──[ serialize ]──►  disk-page format  │
      │                          ▲                               │
      │                    THIS is the cost you're avoiding      │
      └──────────────────────────────────────────────────────────┘
```

**A second benefit that's easy to overlook:** in-memory databases can provide **data models that are difficult to implement with disk-based indexes.** Redis offers priority queues and sets — comparatively simple to implement precisely because everything is in memory.

**Anti-caching** — extending in-memory architecture beyond memory size:

```
   Evict the LEAST-RECENTLY-USED data from memory to disk when memory
   runs short; load it back when accessed again.

   Similar to OS virtual memory and swap files — BUT the database can
   manage memory MORE EFFICIENTLY than the OS, because it works at the
   granularity of INDIVIDUAL RECORDS rather than entire memory pages.

   ⚠️ Still requires INDEXES TO FIT ENTIRELY IN MEMORY
      (like the Bitcask example at the start of the chapter).
```

---
---

# PART B — TRANSACTION PROCESSING OR ANALYTICS?

## 11. OLTP vs OLAP

> In the early days, a write to the database typically corresponded to **a commercial transaction**: making a sale, placing an order, paying a salary. As databases expanded into areas that didn't involve money changing hands, **the term transaction nevertheless stuck**, referring to **a group of reads and writes that form a logical unit.**

> ⚠️ **Important clarification:** a transaction **needn't necessarily have ACID properties**. Transaction processing just means **allowing clients to make low-latency reads and writes** — as opposed to batch processing jobs, which run periodically.

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  OLTP — ONLINE TRANSACTION PROCESSING                              │
   │                                                                    │
   │  An application looks up a SMALL NUMBER of records by some key,    │
   │  using an index. Records are inserted or updated based on user     │
   │  input. Because these applications are INTERACTIVE, the access     │
   │  pattern became known as OLTP.                                     │
   ├────────────────────────────────────────────────────────────────────┤
   │  OLAP — ONLINE ANALYTIC PROCESSING                                 │
   │                                                                    │
   │  An analytic query SCANS OVER A HUGE NUMBER of records and         │
   │  calculates AGGREGATE STATISTICS (count, sum, average) rather      │
   │  than returning raw data to the user.                              │
   │                                                                    │
   │  Written by BUSINESS ANALYSTS, feeding reports that help           │
   │  management make better decisions (BUSINESS INTELLIGENCE).         │
   └────────────────────────────────────────────────────────────────────┘
```

**The three example analytic queries from the book** — note how different they feel from "fetch user 251":

```
   • What was the total revenue of each of our stores in January?
   • How much more bananas than usual did we sell during our latest promotion?
   • Which brand of baby food is most often purchased together with
     brand X diapers?
```

> 📖 *Footnote:* "The meaning of *online* in OLAP is unclear; it probably refers to the fact that queries are not just for predefined reports, but that analysts use the system **interactively for explorative queries.**"

### Table 3-1 — Comparing OLTP and OLAP

| Property | **OLTP** | **OLAP** |
|---|---|---|
| **Main read pattern** | Small number of records per query, fetched **by key** | **Aggregate** over large number of records |
| **Main write pattern** | Random-access, low-latency writes from user input | **Bulk import (ETL)** or event stream |
| **Primarily used by** | End user/customer, via web application | Internal **analyst**, for decision support |
| **What data represents** | **Latest state** of data (current point in time) | **History of events** that happened over time |
| **Dataset size** | Gigabytes to terabytes | **Terabytes to petabytes** |

**The row that explains everything else is "what data represents."** OLTP stores *what is true now*; OLAP stores *what happened*. One is a mutable snapshot, the other an immutable accumulation — and that difference drives every storage decision below.

---

## 12. Data warehousing

> At first, the same databases were used for both. **SQL turned out to be quite flexible** — it works well for both. Nevertheless, in the late 1980s and early 1990s, there was a trend for companies to **stop using their OLTP systems for analytics** and run analytics on a separate database: **a data warehouse.**

### Why DBAs won't let analysts near the OLTP database

```
   OLTP systems are usually expected to be HIGHLY AVAILABLE and to
   process transactions with LOW LATENCY — they are often CRITICAL to
   the operation of the business.

   Analytic queries are EXPENSIVE, scanning large parts of the dataset.

   ➜ Running them on OLTP would HARM THE PERFORMANCE of concurrently
     executing transactions.

   ➜ So: a data warehouse is a SEPARATE database that analysts can
     query TO THEIR HEART'S CONTENT, without affecting OLTP operations.
     It contains a READ-ONLY COPY of the data from all the OLTP systems.
```

### 🔷 Figure 3-8 — ETL into a data warehouse

```
   USERS
   ┌──────────┐        ┌──────────┐        ┌──────────┐
   │ Customer │        │ Warehouse│        │  Truck   │
   │          │        │  worker  │        │  driver  │
   └────┬─────┘        └────┬─────┘        └────┬─────┘
        │                   │                   │
        ▼                   ▼                   ▼
   ╔═══════════════════════════════════════════════════════════════════╗
   ║  OLTP SYSTEMS                                                     ║
   ║  ┌──────────────┐  ┌──────────────────┐  ┌────────────────────┐   ║
   ║  │E-commerce    │  │Stock-keeping app │  │Vehicle route       │   ║
   ║  │site          │  │                  │  │planner             │   ║
   ║  └──────┬───────┘  └────────┬─────────┘  └─────────┬──────────┘   ║
   ║         ▼                   ▼                      ▼              ║
   ║     ┌────────┐          ┌──────────┐          ┌────────┐          ║
   ║     │ Sales  │          │Inventory │          │  Geo   │          ║
   ║     │   DB   │          │    DB    │          │   DB   │          ║
   ║     └────┬───┘          └─────┬────┘          └────┬───┘          ║
   ╚══════════│════════════════════│════════════════════│══════════════╝
              │ extract            │ extract            │ extract
              ▼                    ▼                    ▼
          transform            transform            transform
              │                    │                    │
              └──────────┬─────────┴──────────┬─────────┘
                      load                  load
   ╔═════════════════════▼════════════════════▼════════════════════════╗
   ║  OLAP SYSTEM                                                      ║
   ║                    ┌──────────────────────┐                       ║
   ║   Business ───────►│   DATA WAREHOUSE     │                       ║
   ║   analyst   query  │  (read-only copy of  │                       ║
   ║                    │   ALL the OLTP data) │                       ║
   ║                    └──────────────────────┘                       ║
   ╚═══════════════════════════════════════════════════════════════════╝

   E-T-L = EXTRACT (periodic dump or continuous stream)
           TRANSFORM (into an analysis-friendly schema, cleaned up)
           LOAD (into the warehouse)
```

> 💡 **A candid observation from the book:** "Data warehouses now exist in almost all large enterprises, but in small companies they are almost unheard of… **In a large company, a lot of heavy lifting is required to do something that is simple in a small company.**"

### The divergence

> On the surface, a data warehouse and a relational OLTP database look similar, **because they both have a SQL query interface. However, the internals can look quite different**, because they're optimized for very different query patterns.

```
   Some products support BOTH (Microsoft SQL Server, SAP HANA) — but
   "they are increasingly becoming TWO SEPARATE STORAGE AND QUERY
   ENGINES, which happen to be accessible through a common SQL interface."

   COMMERCIAL:  Teradata, Vertica, SAP HANA, ParAccel
                (expensive licenses; Amazon RedShift = hosted ParAccel)

   SQL-ON-HADOOP (open source, young, aiming to compete):
                Apache Hive, AMPLab's Shark, Cloudera Impala,
                Hortonworks Stinger, Facebook Presto, Apache Tajo,
                Apache Drill — some based on Google's Dremel
```

---

## 13. Stars and snowflakes: schemas for analytics

> In transaction processing there's a **wide range** of data models. In analytics, there is **much less diversity**. Many warehouses use a formulaic style: the **star schema** (also known as **dimensional modeling**).

### 🔷 Figure 3-9 — A star schema for a grocery retailer

```
                       ┌────────────────────────┐
                       │    dim_date table      │
                       ├────────┬──────┬────────┤
                       │date_key│ year │ month  │  … weekday, is_holiday
                       ├────────┼──────┼────────┤
                       │ 140101 │ 2014 │  jan   │  wed,  yes
                       │ 140102 │ 2014 │  jan   │  thu,  no
                       └────┬───┴──────┴────────┘
                            │
   ┌──────────────────┐     │      ┌────────────────────────────┐
   │ dim_product      │     │      │    dim_store table         │
   ├──────┬───────────┤     │      ├────────┬───────┬───────────┤
   │prod- │ sku       │     │      │store_sk│ state │   city    │
   │uct_sk│ descrip.  │     │      ├────────┼───────┼───────────┤
   │      │ brand     │     │      │   1    │  WA   │  Seattle  │
   │      │ category  │     │      │   2    │  CA   │ San Fran. │
   ├──────┼───────────┤     │      │   3    │  CA   │ Palo Alto │
   │  30  │ Bananas   │     │      └────┬───┴───────┴───────────┘
   │  31  │ Fish food │     │           │
   │  32  │ Croissant │     │           │
   └───┬──┴───────────┘     │           │
       │                    │           │
       │      ╔═════════════▼═══════════▼══════════════════════════════╗
       └─────►║              FACT_SALES TABLE  (the CENTRE)            ║
              ╠════════╤══════════╤════════╤═════════╤════════╤════════╣
              ║date_key│product_sk│store_sk│promo_sk │cust_sk │quantity║
              ║        │          │        │         │        │net_pric║
              ╠════════╪══════════╪════════╪═════════╪════════╪════════╣
              ║ 140102 │    31    │   3    │  NULL   │  NULL  │ 1 2.49 ║
              ║ 140102 │    69    │   5    │   19    │  NULL  │ 3 14.99║
              ║ 140102 │    74    │   3    │   23    │  191   │ 1 4.49 ║
              ║ 140102 │    33    │   8    │  NULL   │  235   │ 4 0.99 ║
              ╚════════╧═════╤════╧════════╧════╤════╧═══╤════╧════════╝
                             │                  │        │
       ┌─────────────────────┘                  │        │
       │                                        │        │
   ┌───▼────────────────────┐      ┌────────────▼───┐ ┌──▼──────────────────┐
   │  dim_promotion table   │      │ dim_customer   │ │  each row of the    │
   ├──────┬─────────────────┤      ├───────┬────────┤ │  FACT table = ONE   │
   │promo │ name            │      │cust_sk│ name   │ │  EVENT that         │
   │_sk   │ ad_type         │      ├───────┼────────┤ │  occurred at a      │
   ├──────┼─────────────────┤      │  190  │ Alice  │ │  particular time    │
   │  18  │ New Year sale   │      │  191  │  Bob   │ └─────────────────────┘
   │  19  │ Aquarium deal   │      │  192  │ Cecil  │
   │  20  │ Coffee & cake   │      └───────┴────────┘
   └──────┴─────────────────┘

   ★ THE NAME: visualize the relationships — the FACT TABLE is in the
     middle, surrounded by its foreign keys to DIMENSION TABLES like
     THE RAYS OF A STAR.
```

### How to read a star schema

```
   FACT TABLE                          DIMENSION TABLES
   ──────────                          ────────────────
   Each row = ONE EVENT that           The WHO, WHAT, WHERE, WHEN,
   occurred at a particular time       HOW and WHY of the event
   (a product purchased by a
   customer; a page view; a click)     Each row in dim_product = one
                                       type of product for sale, with
   Columns are either:                 SKU, description, brand,
     • ATTRIBUTES — the price sold     category, fat content, package
       at, the cost from the             size, …
       supplier (→ profit margin)
     • FOREIGN KEYS to dimensions      ⚑ EVEN DATE AND TIME get a
                                         dimension table — so extra
   ⚠️ Facts are captured as INDIVIDUAL   info like PUBLIC HOLIDAYS can
      EVENTS for maximum analysis        be encoded, letting queries
      flexibility. This means the        distinguish holiday from
      fact table becomes EXTREMELY       non-holiday sales.
      LARGE — Apple, Walmart or eBay
      may have TENS OF PETABYTES,
      most of it in fact tables.
```

### Star vs snowflake

```
   STAR SCHEMA                          SNOWFLAKE SCHEMA
   ───────────                          ────────────────
        dim_date                        dimensions are FURTHER BROKEN
            │                           DOWN into SUB-DIMENSIONS
   dim_prod─┼─dim_store
            │                           dim_product ──► dim_brand
      fact_sales                                    └─► dim_category
            │
   dim_cust─┴─dim_promo                 ✅ MORE NORMALIZED
                                        ❌ but STAR SCHEMAS ARE OFTEN
   brand & category stored as              PREFERRED because they are
   strings directly in dim_product         SIMPLER FOR ANALYSTS TO
                                           WORK WITH
```

**And note how wide these tables get:**

> Fact tables often have **over 100 columns, sometimes several hundred.** Dimension tables can also be very wide — `dim_store` may include which services are offered at each store, whether it has an in-store bakery, the square footage, when it opened, when it was last remodeled, how far it is from the nearest highway…

---

# PART C — COLUMN-ORIENTED STORAGE

## 14. The core insight

**The setup — Example 3-1**, analyzing whether people buy more fresh fruit or candy depending on the day of the week:

```sql
SELECT
  dim_date.weekday, dim_product.category,
  SUM(fact_sales.quantity) AS quantity_sold
FROM fact_sales
  JOIN dim_date    ON fact_sales.date_key   = dim_date.date_key
  JOIN dim_product ON fact_sales.product_sk = dim_product.product_sk
WHERE
  dim_date.year = 2013 AND
  dim_product.category IN ('Fresh fruit', 'Candy')
GROUP BY
  dim_date.weekday, dim_product.category;
```

```
   ┌───────────────────────────────────────────────────────────────────────┐
   │  THE FACT TABLE HAS 100+ COLUMNS.                                     │
   │  THIS QUERY TOUCHES EXACTLY THREE:                                    │
   │       date_key,  product_sk,  quantity                                │
   │                                                                       │
   │  ("SELECT *" queries are rarely needed for analytics.)                │
   │                                                                       │
   │  In a ROW-ORIENTED engine you may have indexes on date_key and        │
   │  product_sk — but the engine STILL HAS TO LOAD ALL THOSE ROWS         │
   │  (each with 100+ attributes) from disk into memory, PARSE them,       │
   │  and FILTER OUT the ones that don't qualify. That takes a long time.  │
   └───────────────────────────────────────────────────────────────────────┘
```

> **The idea behind column-oriented storage is simple: don't store all the values from one row together — store all the values from each column together instead.**

### 🔷 Figure 3-10 — Storing relational data by column

```
   fact_sales table (LOGICAL VIEW)
   ┌────────┬──────────┬────────┬─────────┬────────┬────┬───────┬────────┐
   │date_key│product_sk│store_sk│promo_sk │cust_sk │qty │net_pr │disc_pr │
   ├────────┼──────────┼────────┼─────────┼────────┼────┼───────┼────────┤
   │ 140102 │    69    │   4    │  NULL   │  NULL  │ 1  │ 13.99 │ 13.99  │
   │ 140102 │    69    │   5    │   19    │  NULL  │ 3  │ 14.99 │  9.99  │
   │ 140102 │    69    │   5    │  NULL   │  191   │ 1  │ 14.99 │ 14.99  │
   │ 140102 │    74    │   3    │   23    │  202   │ 5  │  0.99 │  0.89  │
   │ 140103 │    31    │   2    │  NULL   │  NULL  │ 1  │  2.49 │  2.49  │
   │ 140103 │    31    │   3    │  NULL   │  NULL  │ 3  │ 14.99 │  9.99  │
   │ 140103 │    31    │   3    │   21    │  123   │ 1  │ 49.99 │ 39.99  │
   │ 140103 │    31    │   8    │  NULL   │  233   │ 1  │  0.99 │  0.99  │
   └────────┴──────────┴────────┴─────────┴────────┴────┴───────┴────────┘

   COLUMNAR STORAGE LAYOUT (PHYSICAL — one file per column)
   ┌──────────────────────────────────────────────────────────────────────┐
   │ date_key       : 140102,140102,140102,140102,140103,140103,140103,…  │
   │ product_sk     : 69, 69, 69, 74, 31, 31, 31, 31                      │
   │ store_sk       : 4, 5, 5, 3, 2, 3, 3, 8                              │
   │ promotion_sk   : NULL, 19, NULL, 23, NULL, NULL, 21, NULL            │
   │ customer_sk    : NULL, NULL, 191, 202, NULL, NULL, 123, 233          │
   │ quantity       : 1, 3, 1, 5, 1, 3, 1, 1                              │
   │ net_price      : 13.99, 14.99, 14.99, 0.99, 2.49, 14.99, 49.99, 0.99 │
   │ discount_price : 13.99, 9.99, 14.99, 0.89, 2.49, 9.99, 39.99, 0.99   │
   └──────────────────────────────────────────────────────────────────────┘

   🔑 THE CRITICAL INVARIANT
      Each column file contains the rows IN THE SAME ORDER.
      To reassemble row 23, take the 23rd entry from each column file.

      column files:  [a₁ a₂ a₃ … a₂₃ …]
                     [b₁ b₂ b₃ … b₂₃ …]     row 23 = (a₂₃, b₂₃, c₂₃)
                     [c₁ c₂ c₃ … c₂₃ …]
                                  ▲
                       position IS the row identity — there are no
                       row IDs stored anywhere
```

> **Column storage is easiest to understand in a relational data model, but it applies equally to non-relational data.** **Parquet** is a columnar storage format for a *document* data model, based on Google's Dremel.

### 📊 I measured this on 200,000 synthetic fact rows

```
   Query needs 3 of 8 columns:  date_key, product_sk, quantity

   ROW store    : must read ALL   6,623,127 bytes
   COLUMN store : reads only      2,399,997 bytes
   ─────────────────────────────────────────────────
   => 2.8x less I/O  —  before any compression

   scan + aggregate timing:  row 57.9 ms   column 33.1 ms   (1.8x faster)
   (both computed the same answer, 1523 — correctness check)
```

And that's with only 8 columns. **Scale it to a realistic 100-column fact table** and the same 3-column query reads roughly **33× less data**.

---

## 15. Column compression

> In Figure 3-10, look at the sequences of values for each column: **they often look quite repetitive, which is a good sign for compression.**

### 🔷 Figure 3-11 — Bitmap encoding with run-length encoding

```
   COLUMN VALUES
   product_sk: 69 69 69 69 74 31 31 31 31 29 30 30 31 31 31 68 69 69
   position:    1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18

   ONE BITMAP PER DISTINCT VALUE  (bit = 1 if that row has that value)
   ┌───────────────┬────────────────────────────────────────────────────┐
   │ product_sk=29 │ 0  0  0  0  0  0  0  0  0  1  0  0  0  0  0  0  0  0│
   │ product_sk=30 │ 0  0  0  0  0  0  0  0  0  0  1  1  0  0  0  0  0  0│
   │ product_sk=31 │ 0  0  0  0  0  1  1  1  1  0  0  0  1  1  1  0  0  0│
   │ product_sk=68 │ 0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  1  0  0│
   │ product_sk=69 │ 1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1│
   │ product_sk=74 │ 0  0  0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  0│
   └───────────────┴────────────────────────────────────────────────────┘
                     └─ mostly zeros → the bitmaps are SPARSE ─┘
                                        │
                                        ▼
   RUN-LENGTH ENCODING (count alternating runs, starting with zeros)
   ┌───────────────┬──────────────┬───────────────────────────────────┐
   │ product_sk=29 │  9, 1        │ 9 zeros, 1 one, rest zeros        │
   │ product_sk=30 │ 10, 2        │ 10 zeros, 2 ones, rest zeros      │
   │ product_sk=31 │  5, 4, 3, 3  │ 5 zeros, 4 ones, 3 zeros, 3 ones  │
   │ product_sk=68 │ 15, 1        │ 15 zeros, 1 one, rest zeros       │
   │ product_sk=69 │  0, 4, 12, 2 │ 0 zeros, 4 ones, 12 zeros, 2 ones │
   │ product_sk=74 │  4, 1        │ 4 zeros, 1 one, rest zeros        │
   └───────────────┴──────────────┴───────────────────────────────────┘
```

**Why this works:** the number of distinct values in a column is often **small compared to the number of rows** — a retailer may have **billions of sales but only 100,000 distinct products.**

**I reproduced Figure 3-11 exactly with code.** The RLE output matches the book line for line.

### 🔑 Why bitmaps make warehouse queries fast

```
   WHERE product_sk IN (30, 68, 69)
   ─────────────────────────────────
   Load 3 bitmaps, compute the bitwise OR.  Extremely efficient.

      29: 000000000100000000
      30: 000000000011000000  ┐
      68: 000000000000000100  ├─ OR ─► 000000000011000111
      69: 111100000000000011  ┘

   WHERE product_sk = 31 AND store_sk = 3
   ───────────────────────────────────────
   Load both bitmaps, compute the bitwise AND.

   ⚑ WHY THIS IS VALID: the columns contain the rows IN THE SAME ORDER,
     so the kth bit in one column's bitmap corresponds to THE SAME ROW
     as the kth bit in another column's bitmap.
```

**My measured result on the real 18-value column:** `WHERE product_sk IN (31,69)` → OR → `111101111000111011` → **13 rows matched.** One CPU instruction per 64 rows.

### Memory bandwidth and vectorized processing

> For queries that scan millions of rows, **a big bottleneck is the bandwidth for getting data from disk into memory. However, that is not the only bottleneck.** Developers of analytical databases also worry about **using CPU cycles efficiently.**

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  THE CPU-LEVEL CONCERNS                                            │
   │    • bandwidth from MAIN MEMORY into the CPU CACHE                 │
   │    • avoiding BRANCH MISPREDICTIONS and bubbles in the CPU         │
   │      instruction pipeline                                          │
   │    • making use of SIMD (single-instruction-multi-data)            │
   ├────────────────────────────────────────────────────────────────────┤
   │  HOW COLUMNAR STORAGE HELPS                                        │
   │                                                                    │
   │  Take a chunk of COMPRESSED column data that fits comfortably in   │
   │  the CPU's L1 CACHE and iterate through it in a TIGHT LOOP.        │
   │                                                                    │
   │     ┌──────────────┐                                               │
   │     │  L1 CACHE    │ ← compressed column chunk                     │
   │     │ ▓▓▓▓▓▓▓▓▓▓▓▓ │   tight loop, no function calls,              │
   │     └──────────────┘   no per-record conditionals                  │
   │                                                                    │
   │  ➜ Much faster than code with lots of function calls and           │
   │    conditions for each record processed.                           │
   │  ➜ COMPRESSION ALSO MEANS MORE ROWS FIT IN THE SAME L1 CACHE.      │
   │  ➜ Operators like bitwise AND/OR can be designed to operate on     │
   │    compressed chunks DIRECTLY.                                     │
   │                                                                    │
   │            This technique is called VECTORIZED PROCESSING.         │
   └────────────────────────────────────────────────────────────────────┘
```

Note the double win of compression: **less disk I/O *and* better cache utilization.** It isn't just about saving space.

---

## 16. Sort order in column storage

> In a column store, **it doesn't necessarily matter in which order rows are stored.** Easiest is insertion order — appending to each column file. **However, we can choose to impose an order, like we did with SSTables, and use that as an indexing mechanism.**

### ⚠️ The rule you must not break

```
   ❌ IT WOULD NOT MAKE SENSE TO SORT EACH COLUMN INDEPENDENTLY
      — you'd no longer know which items belong to the same row!

      date_key  : [140103, 140102, 140102, …]  ← sorted
      product_sk: [31, 31, 69, …]              ← sorted separately
                     ✗ ROW IDENTITY DESTROYED

   ✅ THE DATA MUST BE SORTED AN ENTIRE ROW AT A TIME, even though
      it is STORED by column.
```

### Choosing sort keys

```
   The DB administrator picks the sort columns using knowledge of
   common queries.

   FIRST SORT KEY — e.g. date_key, if queries often target date ranges
      ➜ the optimizer can scan only the rows from last month

   SECOND SORT KEY — determines order among rows tied on the first
      e.g. product_sk, so all sales for the same product on the same
      day are grouped together in storage
      ➜ helps queries that group or filter by product within a date range
```

### 🔑 Sorting supercharges compression — and this is a bigger effect than it sounds

> If the primary sort column does not have many distinct values, then after sorting **it will have long sequences where the same value is repeated.** A simple run-length encoding could compress that column **down to a few kilobytes — even if the table has billions of rows.**

```
   UNSORTED           69 74 31 69 30 31 29 68 31 69 …  ← runs of length 1
                      └── RLE barely helps, may even HURT ──┘

   SORTED             29 30 30 31 31 31 31 31 68 69 69 69 …
                      └────── long runs → RLE is devastating ──────┘

   ⚠️ THE EFFECT DECAYS DOWN THE SORT PRIORITY:
      1st sort key  ████████████████████  strongest compression
      2nd sort key  ██████████            more jumbled
      3rd sort key  ████                  more jumbled still
      rest          ▌                     essentially random order

      "But having the first few columns sorted is still a win overall."
```

### 😲 I measured this, and the result genuinely surprised me

```
   200,000-row product_sk column, 6 distinct values:

   raw text                      : 599,999 bytes
   bitmap + RLE, UNSORTED        : 666,814 bytes   (0.9x — WORSE than raw!)
   bitmap + RLE, SORTED          :      34 bytes   (17,647x)
```

**Read those last two lines again.** On randomly-ordered data, bitmap+RLE actually made the column **bigger** — because with values interleaved randomly, almost every run has length 1, and you pay the encoding overhead for nothing. Sort the identical data and it collapses to **34 bytes.**

This is a far more dramatic demonstration than "compression works well on columns." **Sort order is not a minor tuning knob — it is the single biggest lever in columnar storage**, and getting it wrong can make compression counterproductive.

### Several different sort orders

> A clever extension, introduced in **C-Store** and adopted in **Vertica**: different queries benefit from different sort orders, **so why not store the same data sorted in several different ways?**

```
   Data needs to be REPLICATED to multiple machines anyway, so you don't
   lose data if one machine fails.

   ➜ YOU MIGHT AS WELL STORE THAT REDUNDANT DATA SORTED DIFFERENTLY,
     so a query can use the version that best fits its access pattern.

   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │  replica 1   │  │  replica 2   │  │  replica 3   │
   │ sorted by    │  │ sorted by    │  │ sorted by    │
   │ date_key     │  │ product_sk   │  │ store_sk     │
   └──────────────┘  └──────────────┘  └──────────────┘
     redundancy you were paying for anyway, now doing double duty

   ⚑ SIMILAR TO multiple secondary indexes in a row store, BUT:
     row store  → keeps every row in ONE place; secondary indexes
                  contain POINTERS to the matching rows
     column store → normally NO POINTERS to data elsewhere,
                    only COLUMNS CONTAINING VALUES
```

---

## 17. Writing to column-oriented storage

> These optimizations make sense in data warehouses, because most of the load is **large read-only queries.** However, **they have the downside of making writes more difficult.**

```
   ❌ UPDATE-IN-PLACE (B-tree style) IS NOT POSSIBLE with compressed columns.

      Insert a row in the MIDDLE of a sorted table?
      → you would most likely have to REWRITE ALL THE COLUMN FILES.

      Because rows are identified BY THEIR POSITION within a column,
      the insertion has to update ALL COLUMNS CONSISTENTLY.

      ┌──────────────────────────────────────────────────────────┐
      │  insert here ▼                                           │
      │  col A: [a₁ a₂ ┃ a₃ a₄ a₅ …]  every subsequent position  │
      │  col B: [b₁ b₂ ┃ b₃ b₄ b₅ …]  in EVERY column shifts     │
      │  col C: [c₁ c₂ ┃ c₃ c₄ c₅ …]                             │
      └──────────────────────────────────────────────────────────┘
```

### ✅ The solution is one we already have: LSM-trees

```
   ┌───────────────────────────────────────────────────────────────────┐
   │  ALL WRITES first go to an IN-MEMORY STORE, added to a sorted     │
   │  structure, prepared for writing to disk.                         │
   │                                                                   │
   │  ⚑ It doesn't matter whether the in-memory store is row-oriented  │
   │    or column-oriented.                                            │
   │                                                                   │
   │  When enough writes accumulate, they are MERGED with the column   │
   │  files on disk and written to new files IN BULK.                  │
   │                                                                   │
   │  This is essentially what VERTICA does.                           │
   ├───────────────────────────────────────────────────────────────────┤
   │  Queries must examine BOTH the column data on disk AND the        │
   │  recent writes in memory, and combine the two.                    │
   │                                                                   │
   │  ➜ But the QUERY OPTIMIZER HIDES THIS from the user. From an      │
   │    analyst's point of view, inserts/updates/deletes are           │
   │    IMMEDIATELY REFLECTED in subsequent queries.                   │
   └───────────────────────────────────────────────────────────────────┘
```

**This is a lovely structural moment in the chapter.** The LSM-tree, introduced 20 pages earlier for OLTP key-value workloads, turns out to be exactly the right answer for writes in an analytics column store. The same idea solves both problems.

---

## 18. Aggregation: data cubes and materialized views

> Not every data warehouse is a column store — traditional row-oriented databases and other architectures are also used. **However, columnar storage can be significantly faster for ad-hoc analytical queries, so it is rapidly gaining popularity.**

### Materialized views

```
   ┌──────────────────────────┬────────────────────────────────────────┐
   │  VIRTUAL VIEW            │  MATERIALIZED VIEW                     │
   ├──────────────────────────┼────────────────────────────────────────┤
   │  Just a SHORTCUT for     │  An ACTUAL COPY of the query results,  │
   │  writing queries.        │  WRITTEN TO DISK.                      │
   │                          │                                        │
   │  When you read from it,  │  When the underlying data changes, the │
   │  the SQL engine EXPANDS  │  view MUST BE UPDATED — it's a         │
   │  it into the underlying  │  DENORMALIZED COPY.                    │
   │  query ON THE FLY.       │                                        │
   │                          │  The DB can do that automatically, but │
   │  No storage cost.        │  such updates MAKE WRITES MORE         │
   │  No write cost.          │  EXPENSIVE.                            │
   └──────────────────────────┴────────────────────────────────────────┘

   ➜ Which is why materialized views are NOT OFTEN USED IN OLTP,
     but CAN MAKE MORE SENSE in read-heavy data warehouses.
     (Whether they actually improve read performance depends on
      the individual case.)
```

### 🔷 Figure 3-12 — Two dimensions of a data cube

> A **data cube** (or **OLAP cube**) is a grid of aggregates grouped by different dimensions.

```
                                    product_sk
                    ┌────────┬────────┬────────┬────────┬─────┬──────────┐
                    │   32   │   33   │   34   │   35   │  …  │  TOTAL   │
        ┌───────────┼────────┼────────┼────────┼────────┼─────┼──────────┤
        │  140101   │ 149.60 │  31.01 │  84.58 │  28.18 │  …  │ 40710.53 │──┐
        │           │   +    │   +    │   +    │   +    │     │    +     │  │
   d    │  140102   │ 132.18 │  19.78 │  82.91 │  10.96 │  …  │ 73091.28 │  │
   a    │           │   +    │   +    │   +    │   +    │     │    +     │  │ sum
   t    │  140103   │ 196.75 │   0.00 │  12.52 │  64.67 │  …  │ 54688.10 │  │ along
   e    │           │   +    │   +    │   +    │   +    │     │    +     │  │ the
   _    │  140104   │ 178.36 │   9.98 │  88.75 │  56.16 │  …  │ 95121.09 │  │ ROW
   k    │           │        │        │        │        │     │          │  │
   e    │    …      │   …    │   …    │   …    │   …    │  …  │    …     │  │
   y    ├───────────┼────────┼────────┼────────┼────────┼─────┼──────────┤  │
        │  TOTAL    │14967.09│ 5910.43│ 7328.85│ 6885.39│  …  │  lots    │◄─┘
        └───────────┴───┬────┴────────┴────────┴────────┴─────┴──────────┘
                        │
                   sum along the COLUMN

   ┌─────────────────────────────────┐  ┌────────────────────────────────┐
   │ SELECT SUM(net_price)           │  │ SELECT SUM(net_price)          │
   │ FROM fact_sales                 │  │ FROM fact_sales                │
   │ WHERE date_key = 140101         │  │ WHERE product_sk = 32          │
   │   AND product_sk = 32           │  │  ← this is the COLUMN total    │
   │  ← this is ONE CELL             │  │    14967.09, pre-computed      │
   └─────────────────────────────────┘  └────────────────────────────────┘

   Each cell = the aggregate (e.g. SUM) of an attribute (e.g. net_price)
   of ALL FACTS with that date-product combination.

   Apply the same aggregate along each row or column → a summary
   REDUCED BY ONE DIMENSION.
```

**In reality it's more than two dimensions.** Figure 3-9 has five: date, product, store, promotion, customer. *"It's a lot harder to imagine what a five-dimensional hypercube would look like, but the principle remains the same"* — each cell holds sales for a particular date-product-store-promotion-customer combination.

### ⚖️ The trade-off

```
   ✅ ADVANTAGE — certain queries become VERY FAST, because they have
      effectively been PRE-COMPUTED. Want total sales per store
      yesterday? Just read the totals along the appropriate dimension.
      No need to scan millions of rows.

   ❌ DISADVANTAGE — it DOESN'T HAVE THE SAME FLEXIBILITY as querying
      raw data.

      Example: there is NO WAY to calculate what proportion of sales
      comes from items costing more than $100 — because PRICE ISN'T
      ONE OF THE DIMENSIONS.

   ➜ Most data warehouses therefore try to KEEP AS MUCH RAW DATA AS
     POSSIBLE, and use aggregates like data cubes ONLY AS A PERFORMANCE
     BOOST for certain queries.
```

---

## 19. Chapter Summary

```
   ╔══════════════════════════════════════════════════════════════════════╗
   ║  TWO BROAD CATEGORIES OF STORAGE ENGINE                              ║
   ╠══════════════════════════════════╦═══════════════════════════════════╣
   ║  OLTP                            ║  ANALYTICS                        ║
   ╠══════════════════════════════════╬═══════════════════════════════════╣
   ║ • USER-FACING → huge volume of   ║ • Used by BUSINESS ANALYSTS,      ║
   ║   requests                       ║   not end users                   ║
   ║ • Each query touches a SMALL     ║ • MUCH LOWER volume of queries    ║
   ║   number of records              ║ • But each query is VERY          ║
   ║ • Requests records by KEY; the   ║   DEMANDING — millions of records ║
   ║   engine uses an INDEX           ║   scanned in a short time         ║
   ║                                  ║                                   ║
   ║ ➜ DISK SEEK TIME is often        ║ ➜ DISK BANDWIDTH (not seek time)  ║
   ║   the bottleneck                 ║   is often the bottleneck         ║
   ║                                  ║ ➜ COLUMN-ORIENTED STORAGE is an   ║
   ║                                  ║   increasingly popular solution   ║
   ╚══════════════════════════════════╩═══════════════════════════════════╝
```

**On the OLTP side, two schools of thought:**

```
   ① THE LOG-STRUCTURED SCHOOL
      Only permits APPENDING to files and DELETING obsolete files,
      but NEVER UPDATES a file that has been written.
      → Bitcask, SSTables, LSM-Trees, LevelDB, Cassandra, HBase, Lucene

      "Their key idea is that they systematically turn RANDOM-ACCESS
       WRITES into SEQUENTIAL WRITES on disk, which enables higher write
       throughput due to the performance characteristics of hard drives
       and SSDs."

   ② THE UPDATE-IN-PLACE SCHOOL
      Treats the disk as FIXED-SIZE PAGES which can be OVERWRITTEN.
      → B-trees, used in ALL major relational databases and many
        non-relational ones
```

**And the key insight about why analytics is different:**

> When your queries require **sequentially scanning across a large number of rows, indexes are much less relevant.** Instead it becomes important to **encode data very compactly**, to minimize the amount of data the query needs to read from disk.

**The closing advice:**

> As an application developer, armed with this knowledge you are in a much better position to know which tool is best suited for your application. **If you need to adjust a database's tuning parameters, this understanding allows you to imagine what effect a higher or lower value may have.**

---
---

# 20. 🎁 EXTRA MATERIAL (added — not in the book)

## 20.1 The storage hierarchy, in numbers you can feel

The chapter keeps saying "disk seeks are slow" and "sequential beats random" without quantifying it. Here are the orders of magnitude, scaled so a CPU cycle is one second:

| Operation | Real latency | If 1 cycle = 1 second |
|---|---|---|
| L1 cache reference | 1 ns | **1 second** |
| L2 cache reference | 4 ns | 4 seconds |
| Main memory reference | 100 ns | 1.7 minutes |
| SSD random read | 16 µs | **4.6 hours** |
| HDD seek | 2–10 ms | **3–12 weeks** |
| Sequential read, 1 MB from SSD | 49 µs | 13 hours |
| Sequential read, 1 MB from HDD | 825 µs | 9 days |

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  A single HDD seek costs as much as ~10,000,000 CPU cycles.        │
   │                                                                    │
   │  THIS is why:                                                      │
   │    • B-trees use a branching factor of hundreds, not 2 —           │
   │      each level costs a seek, so minimize levels                   │
   │    • LSM-trees convert random writes into sequential ones          │
   │    • bloom filters are worth 10 bits/key of RAM to skip a seek     │
   │    • column stores obsess over bytes read                          │
   │                                                                    │
   │  Every design in this chapter is an argument with this table.      │
   └────────────────────────────────────────────────────────────────────┘
```

## 20.2 The RUM conjecture — why you can't have everything

The chapter says "no quick and easy rule" for choosing a storage engine. There's a formal version of that intuition, from Athanassoulis et al. (2016):

```
                          READ overhead
                               ▲
                              ╱ ╲
                             ╱   ╲
                            ╱     ╲       You may optimize for
                           ╱       ╲      any TWO of these three.
                          ╱  PICK   ╲     The third gets worse.
                         ╱    TWO    ╲
                        ╱             ╲
                       ╱_______________╲
              UPDATE overhead      MEMORY overhead

   B-TREE          : good READ,   bad UPDATE (random writes), medium MEMORY
   LSM-TREE        : good UPDATE, worse READ (check many levels), good MEMORY
   HASH INDEX      : great READ,  great UPDATE, terrible MEMORY (all keys in RAM)
   COLUMN + BITMAP : great READ (scans), terrible UPDATE, great MEMORY
```

Every engine in Chapter 3 sits somewhere on this triangle. Bloom filters are interesting precisely because they **buy back read performance by spending a little memory** — moving along one edge deliberately.

## 20.3 Compaction strategies — the knob the book doesn't turn

Kleppmann says LSM-trees "run a merging and compaction process in the background" without saying *how*. In practice this is the single most consequential tuning decision in RocksDB or Cassandra:

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │  SIZE-TIERED COMPACTION  (Cassandra default)                        │
   │                                                                     │
   │    merge SSTables when several of SIMILAR SIZE accumulate           │
   │    ▪▪▪▪ ▪▪▪▪ ▪▪▪▪ ▪▪▪▪  →  ████████████████                         │
   │                                                                     │
   │    ✅ LOW write amplification   ❌ HIGH space amplification (up to   │
   │                                    2x — needs room for the merge)   │
   │                                 ❌ a key may live in MANY SSTables   │
   │                                    → worse reads                    │
   ├─────────────────────────────────────────────────────────────────────┤
   │  LEVELED COMPACTION  (LevelDB / RocksDB default)                    │
   │                                                                     │
   │    L0 │▪▪▪▪│                    each level ~10x the previous;       │
   │    L1 │████████│                within L1+ , key ranges DON'T       │
   │    L2 │████████████████████│    overlap                             │
   │                                                                     │
   │    ✅ LOW space amplification   ❌ HIGHER write amplification        │
   │    ✅ a key is in AT MOST ONE      (~10x per level)                 │
   │       SSTable per level → better reads                              │
   └─────────────────────────────────────────────────────────────────────┘
```

**This is the concrete form of the chapter's warning** that "compaction can interfere with ongoing reads and writes." Size-tiered gives you write throughput and latency spikes; leveled gives you predictable reads and more write work. Neither is right; it depends on your workload — exactly as the book says.

## 20.4 Where each engine shows up in 2026

| Chapter 3 concept | Real systems today |
|---|---|
| Hash index / Bitcask | Riak (Bitcask), some embedded caches |
| LSM-tree | RocksDB, LevelDB, Cassandra, ScyllaDB, HBase, TiKV, CockroachDB, InfluxDB |
| B-tree | PostgreSQL, MySQL InnoDB, SQLite, SQL Server, Oracle, LMDB, MongoDB (WiredTiger) |
| SSTable + term dictionary | Lucene → Elasticsearch, OpenSearch, Solr |
| Bloom filter | RocksDB, Cassandra, HBase, Bigtable, CDN cache layers |
| Clustered index | MySQL InnoDB (always), SQL Server (optional) |
| R-tree / multi-dim | PostGIS, SQLite R*Tree, Elasticsearch geo |
| In-memory | Redis, Memcached, VoltDB, SAP HANA, Dragonfly |
| Column store | ClickHouse, DuckDB, Vertica, Redshift, BigQuery, Snowflake, Parquet, Apache Arrow |
| Star schema | dbt models, Snowflake/BigQuery warehouses, Kimball dimensional modelling |
| Data cube | Apache Kylin, Druid rollups, materialized views in Snowflake/BigQuery |

**Two shifts worth noting since the book was written (2017):** first, **DuckDB** made columnar analytics something you `pip install` rather than a warehouse you procure — the "small companies don't have data warehouses" observation is much less true now. Second, **object storage separated compute from storage** (Snowflake, BigQuery, Iceberg/Delta on S3), which changes the bandwidth math the chapter assumes.

## 20.5 A practical index-tuning checklist

The chapter says the developer must choose indexes manually but doesn't say how. A working heuristic:

```
   1. Index the columns in your WHERE, JOIN and ORDER BY clauses —
      not columns you merely SELECT.

   2. For composite indexes, order columns:  equality → range → sort.
      WHERE a = ? AND b > ? ORDER BY c   →   INDEX (a, b, c)
      (Remember the phone book: leftmost prefix is all you can use.)

   3. Low-cardinality columns alone are usually a waste in a row store
      (a boolean index matches half the table). They're EXCELLENT in a
      column store as bitmaps — the exact opposite conclusion.

   4. Every index costs you on every write. Audit for UNUSED indexes:
        Postgres:  SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;
        MySQL:     sys.schema_unused_indexes

   5. Covering indexes turn two lookups into one. Check whether your
      plan says "Index Only Scan" (Postgres) / "Using index" (MySQL).

   6. Measure, don't guess:  EXPLAIN (ANALYZE, BUFFERS) in Postgres
      shows you actual rows AND actual blocks read.
```

---

# 21. 💻 CODE APPENDIX

Everything here runs. All quoted outputs came from executing it.

## 21.1 The bash database, ported, plus the O(n) proof

```python
class BashDB:
    """Python port of the book's two bash functions."""
    def __init__(self, path):
        self.path = path; open(path, "w").close()

    def db_set(self, k, v):
        with open(self.path, "a") as f:
            f.write(f"{k},{v}\n")                  # APPEND ONLY

    def db_get(self, k):
        hit = None
        with open(self.path) as f:
            for line in f:                          # FULL SCAN — O(n)
                if line.startswith(f"{k},"):
                    hit = line[len(k)+1:].rstrip()  # keep the LAST match
        return hit
```

```
n =   1,000   lookup =   0.40 ms
n =  10,000   lookup =   2.57 ms
n = 100,000   lookup =  15.57 ms
n = 400,000   lookup =  63.47 ms      ← linear, exactly as predicted
```

## 21.2 Bitcask-style hash index with real byte offsets

```python
import struct

class HashIndexSegment:
    def __init__(self, path):
        self.path, self.index = path, {}       # key -> byte offset
        open(path, "wb").close()

    def append(self, k, v):
        kb, vb = k.encode(), str(v).encode()
        with open(self.path, "ab") as f:
            off = f.tell()                      # ← record WHERE we wrote it
            f.write(struct.pack("<II", len(kb), len(vb)) + kb + vb)
        self.index[k] = off                     # ← in-memory hash map

    def get(self, k):
        if k not in self.index: return None
        with open(self.path, "rb") as f:
            f.seek(self.index[k])               # ← exactly ONE seek
            kl, vl = struct.unpack("<II", f.read(8))
            f.read(kl)
            return f.read(vl).decode()
```

Note the binary format — length-prefixed, no escaping — exactly as the book recommends over CSV.

## 21.3 SSTable k-way merge (Figure 3-4)

```python
def sstable_merge(segments):
    """Segments listed OLDEST-first; later segments win.
       Streaming: never loads a whole segment into memory."""
    cursors = [0] * len(segments)
    out = []
    while True:
        live = [(segments[i][cursors[i]][0], i)
                for i in range(len(segments)) if cursors[i] < len(segments[i])]
        if not live: break
        lowest = min(k for k, _ in live)
        winner = None
        for k, i in live:                    # advance EVERY segment at that key
            if k == lowest:
                winner = segments[i][cursors[i]][1]   # last wins = newest
                cursors[i] += 1
        out.append((lowest, winner))
    return out
```

The memory footprint is **one key per segment**, regardless of file size. That's the property that makes merging work when segments exceed RAM.

## 21.4 A complete LSM-tree: memtable, WAL, sparse index, bloom filter, tombstones

```python
import hashlib, bisect, struct, os

class BloomFilter:
    def __init__(self, n, bits_per_key=10, k=7):
        self.m = max(8, n * bits_per_key); self.k = k
        self.bits = bytearray((self.m + 7) // 8)
    def _h(self, key):
        d = hashlib.sha256(key.encode()).digest()
        h1 = int.from_bytes(d[:8], "little")
        h2 = int.from_bytes(d[8:16], "little") | 1     # double hashing
        return [(h1 + i * h2) % self.m for i in range(self.k)]
    def add(self, key):
        for b in self._h(key): self.bits[b >> 3] |= 1 << (b & 7)
    def __contains__(self, key):
        return all(self.bits[b >> 3] >> (b & 7) & 1 for b in self._h(key))

TOMBSTONE = object()

class SSTable:
    """Sorted file + SPARSE index (one entry per block) + bloom filter."""
    BLOCK = 4
    def __init__(self, path, sorted_items):
        self.path = path; self.sparse = []
        self.bloom = BloomFilter(len(sorted_items))
        with open(path, "wb") as f:
            for i, (k, v) in enumerate(sorted_items):
                if i % self.BLOCK == 0:
                    self.sparse.append((k, f.tell()))       # ← SPARSE index
                self.bloom.add(k)
                kb = k.encode()
                vb = b"" if v is TOMBSTONE else str(v).encode()
                f.write(struct.pack("<IIb", len(kb), len(vb), v is TOMBSTONE) + kb + vb)

    def get(self, k):
        if k not in self.bloom:
            return ("MISS_BLOOM", None)                     # ZERO disk I/O
        i = bisect.bisect_right(self.sparse, (k, float('inf'))) - 1
        if i < 0: return ("MISS_RANGE", None)
        scanned = 0
        with open(self.path, "rb") as f:
            f.seek(self.sparse[i][1])                       # jump to block start
            while True:
                head = f.read(9)
                if not head: break
                kl, vl, tomb = struct.unpack("<IIb", head)
                kk = f.read(kl).decode(); vv = f.read(vl)
                scanned += 1
                if kk == k:
                    return (f"scanned {scanned} recs",
                            TOMBSTONE if tomb else vv.decode())
                if kk > k: break                            # sorted → stop early
        return (f"MISS after {scanned} recs", None)

class LSMTree:
    def __init__(self, d="/tmp/lsm", threshold=8):
        self.dir = d; os.makedirs(d, exist_ok=True)
        self.memtable = {}; self.sstables = []
        self.threshold = threshold; self.n = 0
        self.wal = open(f"{d}/wal.log", "w")

    def put(self, k, v):
        self.wal.write(f"{k}\t{v}\n"); self.wal.flush()     # ← DURABILITY FIRST
        self.memtable[k] = v
        if len(self.memtable) >= self.threshold: self.flush()

    def delete(self, k):
        self.put(k, "__TOMBSTONE__")                        # ← tombstone, not removal

    def flush(self):
        items = sorted((k, TOMBSTONE if v == "__TOMBSTONE__" else v)
                       for k, v in self.memtable.items())
        self.sstables.append(SSTable(f"{self.dir}/sst{self.n}", items)); self.n += 1
        self.memtable = {}
        self.wal.close(); self.wal = open(f"{self.dir}/wal.log", "w")  # WAL discarded

    def get(self, k):
        if k in self.memtable:
            v = self.memtable[k]
            return None if v == "__TOMBSTONE__" else v
        for s in reversed(self.sstables):                   # NEWEST FIRST
            why, v = s.get(k)
            if v is TOMBSTONE: return None
            if v is not None:  return v
        return None
```

**Note the three ordering rules that make correctness work:** the WAL is written *before* the memtable (durability), search goes *newest to oldest* (recency wins), and a tombstone *stops* the search rather than being skipped (deletion must shadow older values).

## 21.5 B-tree with page splitting

```python
import bisect

class BTree:
    def __init__(self, order=4):
        self.order = order
        self.root = {"leaf": True, "keys": [], "vals": [], "children": []}
        self.splits = 0

    def _split(self, node):
        mid = len(node["keys"]) // 2
        left  = {"leaf": node["leaf"], "keys": node["keys"][:mid],
                 "vals": node["vals"][:mid],
                 "children": node["children"][:mid+1] if not node["leaf"] else []}
        right = {"leaf": node["leaf"], "keys": node["keys"][mid:],
                 "vals": node["vals"][mid:],
                 "children": node["children"][mid+1:] if not node["leaf"] else []}
        self.splits += 1
        return node["keys"][mid], left, right      # separator pushed UP to parent

    def insert(self, k, v):
        def _ins(node):
            i = bisect.bisect_left(node["keys"], k)
            if node["leaf"]:
                node["keys"].insert(i, k); node["vals"].insert(i, v)
            else:
                res = _ins(node["children"][i])
                if res:                            # a child split — absorb it
                    sep, l, r = res
                    node["keys"].insert(i, sep); node["vals"].insert(i, None)
                    node["children"][i:i+1] = [l, r]
            if len(node["keys"]) > self.order: return self._split(node)
            return None
        res = _ins(self.root)
        if res:                                    # ROOT split → tree grows taller
            sep, l, r = res
            self.root = {"leaf": False, "keys": [sep], "vals": [None],
                         "children": [l, r]}
```

**The tree grows from the root, not the leaves** — that last block is the only place height increases, which is why B-trees stay balanced automatically.

## 21.6 Row vs column storage, measured

```python
COLS = ["date_key","product_sk","store_sk","promotion_sk",
        "customer_sk","quantity","net_price","discount_price"]

# ROW layout: one file, all columns interleaved
with open("/tmp/store/rows.dat","w") as f:
    for r in rows: f.write("|".join(str(r[c]) for c in COLS) + "\n")

# COLUMN layout: one file per column, SAME ROW ORDER in each
for c in COLS:
    with open(f"/tmp/store/col_{c}.dat","w") as f:
        f.write("\n".join(str(r[c]) for r in rows))

# The query needs only 3 of 8 columns
needed = ["date_key", "product_sk", "quantity"]

# --- row scan: must parse every field of every row ---
tot = 0
for line in open("/tmp/store/rows.dat"):
    p = line.split("|")
    if p[0] == "140105": tot += int(p[5])

# --- column scan: touch only 2 files ---
dates = open("/tmp/store/col_date_key.dat").read().split("\n")
qty   = open("/tmp/store/col_quantity.dat").read().split("\n")
tot2  = sum(int(qty[i]) for i, d in enumerate(dates) if d == "140105")
```

```
ROW store    : must read ALL   6,623,127 bytes
COLUMN store : reads only      2,399,997 bytes     → 2.8x less I/O
scan+aggregate: row 57.9 ms   column 33.1 ms       → 1.8x faster
both produced 1523  ✓
```

## 21.7 Bitmap + run-length encoding, and the sort-order discovery

```python
col = [69,69,69,69,74,31,31,31,31,29,30,30,31,31,31,68,69,69]

bitmaps = {v: [1 if x == v else 0 for x in col] for v in sorted(set(col))}

def rle(bits):
    out, cur, run = [], 0, 0
    for b in bits:
        if b == cur: run += 1
        else: out.append(run); cur = b; run = 1
    out.append(run)
    return out

# bitwise operations on real bitmaps
p31 = int("".join(map(str, bitmaps[31])), 2)
p69 = int("".join(map(str, bitmaps[69])), 2)
matched = bin(p31 | p69).count('1')      # WHERE product_sk IN (31, 69)
```

**Output — matches Figure 3-11 line for line:**

```
product_sk = 29: [9, 1, 8]           product_sk = 68: [15, 1, 2]
product_sk = 30: [10, 2, 6]          product_sk = 69: [0, 4, 12, 2]
product_sk = 31: [5, 4, 3, 3, 3]     product_sk = 74: [4, 1, 13]

WHERE product_sk IN (31,69) -> OR -> 111101111000111011 -> 13 rows
```

**And the finding worth the whole appendix**, on the 200,000-row column:

```
raw text               : 599,999 bytes
bitmap + RLE UNSORTED  : 666,814 bytes   (0.9x — LARGER than raw)
bitmap + RLE SORTED    :      34 bytes   (17,647x)
```

Compression didn't just work "less well" on unsorted data — **it actively hurt.** Sort order is the whole game.

---

# 22. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.3 — STORAGE AND RETRIEVAL                                            ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  CORE TRADE-OFF: indexes speed up READS, every index slows down WRITES.       ║
║  So DBs don't index by default — YOU choose, from query patterns.             ║
║  An index is DERIVED data: add/remove changes performance, not contents.      ║
║                                                                               ║
║  ── LOG-STRUCTURED SCHOOL (append only, never modify) ───────────────────────  ║
║  HASH INDEX (Bitcask)  in-memory map: key -> byte offset. ONE seek per read.  ║
║      ✅ fast r+w, values may exceed RAM                                        ║
║      ❌ ALL KEYS must fit in RAM;  ❌ NO efficient range queries                ║
║      compaction = keep only latest value per key; merge adjacent segments     ║
║      segments IMMUTABLE → merge in background, swap, delete old               ║
║      details: binary length-prefixed format · TOMBSTONES for delete ·         ║
║               hashmap snapshots for fast restart · checksums · 1 writer       ║
║  SSTABLE  = segment SORTED BY KEY. Three wins:                                ║
║      ① mergesort merging works even if files > memory (1 key per file in RAM) ║
║      ② index can be SPARSE (1 key per few KB) — sorting fills the gaps        ║
║      ③ blocks can be COMPRESSED; disk bandwidth is worse than CPU             ║
║  LSM-TREE = memtable (red-black tree) + WAL + cascade of SSTables             ║
║      write→memtable; full→flush as SSTable; read newest→oldest; merge in bg   ║
║      WAL exists ONLY to rebuild the memtable after a crash; discard on flush  ║
║      BLOOM FILTER kills the "key doesn't exist" worst case (no false negs)    ║
║      → LevelDB, RocksDB, Cassandra, HBase, Lucene (term→postings list)        ║
║                                                                               ║
║  ── UPDATE-IN-PLACE SCHOOL ──────────────────────────────────────────────────  ║
║  B-TREE (1970, "ubiquitous" by 1979). FIXED-SIZE PAGES (~4kB), overwritten.   ║
║      page = k keys + k+1 child refs. Branching factor in the HUNDREDS.        ║
║      height O(log n): 1 TRILLION keys ≈ 5 levels ≈ 5 page reads               ║
║      insert into a full page → SPLIT into two half-full + update parent       ║
║      ⚠️ multi-page writes can CORRUPT on crash (orphan pages)                  ║
║         → WAL / redo log: write to log BEFORE applying to tree pages          ║
║         → so every datum written AT LEAST TWICE (write amplification)         ║
║      concurrency needs LATCHES; log-structured doesn't (bg merge + swap)      ║
║      optimizations: copy-on-write (LMDB) · key abbreviation (B+ tree) ·       ║
║                     sequential leaf layout · sibling pointers · fractal trees ║
║                                                                               ║
║  B-TREE vs LSM: LSM faster WRITES (random→sequential); B-tree faster READS.   ║
║     LSM ❌ compaction hurts HIGH PERCENTILES (Ch.1 callback); key in many segs ║
║     B-tree ✅ key in exactly ONE place → range locks attach to the tree →      ║
║                attractive for transactions.  BENCHMARK YOUR OWN WORKLOAD.     ║
║                                                                               ║
║  ── OTHER INDEXES ───────────────────────────────────────────────────────────  ║
║  secondary index: keys NOT unique → value = list of row IDs, or append row ID ║
║  HEAP FILE (index→pointer, data stored once) vs CLUSTERED (row inside index)  ║
║      vs COVERING (some columns in index; query answered by index alone)       ║
║  concatenated (lastname,firstname): leftmost prefix only — useless for first  ║
║  multi-dimensional: B-tree can't do lat AND lon at once → R-trees, space-     ║
║      filling curves. Also works for (r,g,b) or (date, temperature).           ║
║  fuzzy: Lucene = FSA/trie in memory → Levenshtein automaton → edit distance   ║
║  IN-MEMORY: fast NOT because it skips disk reads (OS caches anyway) but       ║
║      because it avoids ENCODING data into a disk format. Anti-caching = evict ║
║      LRU records to disk, at record granularity (finer than OS paging).       ║
║                                                                               ║
║  ── OLTP vs OLAP ────────────────────────────────────────────────────────────  ║
║  OLTP  few records by key · random low-latency writes · latest state · GB-TB  ║
║        bottleneck = DISK SEEK TIME                                            ║
║  OLAP  aggregate over millions · bulk ETL · history of events · TB-PB         ║
║        bottleneck = DISK BANDWIDTH                                            ║
║  data warehouse = separate read-only copy so analysts can't hurt production   ║
║  STAR SCHEMA: fact table (one row = one EVENT, 100+ cols, petabytes) at the   ║
║        centre; dimension tables = who/what/where/when/how/why. Even DATE is a ║
║        dimension (so you can encode holidays). Snowflake = more normalized,   ║
║        but star preferred — simpler for analysts.                             ║
║                                                                               ║
║  ── COLUMN-ORIENTED STORAGE ─────────────────────────────────────────────────  ║
║  Store each COLUMN in its own file. Query touches 3 of 100 cols → read 3.     ║
║  INVARIANT: every column file holds rows IN THE SAME ORDER. Position = row ID.║
║  BITMAP ENCODING: one bitmap per distinct value; then RUN-LENGTH ENCODE.      ║
║      IN (a,b,c) → bitwise OR.  x AND y → bitwise AND. Valid because same order║
║  VECTORIZED PROCESSING: compressed chunk fits in L1, tight loop, SIMD.        ║
║      Compression wins TWICE: less disk I/O AND more rows per cache line.      ║
║  SORT ORDER: sort WHOLE ROWS (never columns independently!). 1st sort key     ║
║      compresses enormously; effect decays down the priority list.             ║
║      C-Store/Vertica: replicas you need anyway, each sorted DIFFERENTLY.      ║
║  WRITES: can't update-in-place → use an LSM-TREE in front. (Vertica does.)    ║
║  DATA CUBE: grid of pre-computed aggregates. Fast, but can't answer questions ║
║      about non-dimension attributes → keep the RAW DATA too.                  ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 23. ✅ Test yourself

1. **`db_set` is 1 line and performs well. `db_get` is 1 line and performs terribly. Why the asymmetry?**
   → Appending is the simplest possible write and is sequential. Reading requires locating a key among n records with no structure to help, so it's a full scan: O(n). Adding an index fixes reads and taxes writes — the chapter's central trade-off in miniature.

2. **Your keys are 200 million UUIDs. Why is Bitcask the wrong choice?**
   → The hash map must fit entirely in RAM. 200M UUID keys plus offsets is far beyond what's reasonable, and an on-disk hash map performs badly — random I/O, expensive growth, fiddly collision handling. You'd also lose range queries entirely.

3. **What exactly does sorting buy you in an SSTable? Name three things.**
   → Merging becomes a streaming mergesort that works on files larger than memory; the in-memory index can be sparse because you can scan between known offsets; and adjacent records can be grouped into compressible blocks.

4. **A bloom filter says a key is present, but it isn't in the SSTable. Is the database broken?**
   → No. Bloom filters allow false positives but never false negatives. A "yes" means *probably*, so you check the disk and find nothing. Only a "no" is authoritative — which is precisely what makes it useful for skipping reads.

5. **Why does a B-tree need a write-ahead log, when an LSM-tree's log is only for crash recovery of the memtable?**
   → Because a B-tree modifies pages in place, and some operations touch several pages at once. A crash mid-split leaves a structurally corrupt index — orphan pages. The WAL lets recovery replay the modification atomically. An LSM-tree never modifies anything, so it can't be left half-modified.

6. **Your LSM-backed service has a fine p50 but an awful p99. What's a likely cause?**
   → Compaction competing for disk. The chapter flags precisely this: the average impact is small, but a request can get stuck behind an expensive merge, and it shows up in the tail. B-trees are more predictable here.

7. **Why can't you sort each column file independently in a column store?**
   → Because row identity is encoded purely by *position* — the kth entry of every column belongs to the kth row. Sorting columns independently destroys that correspondence and you can never reassemble a row. Sorting must be done a whole row at a time.

8. **Bitmap+RLE made my column bigger, not smaller. What went wrong?**
   → The data isn't sorted. With values randomly interleaved, nearly every run has length 1, so you pay encoding overhead with no benefit. My measurement showed 0.9× on unsorted data and 17,647× on the identical data sorted.

9. **When is a data cube the wrong tool?**
   → When you need to ask questions about attributes that aren't dimensions. You can't compute "what proportion of sales came from items over $100" if price isn't a dimension. Cubes accelerate anticipated queries; keep raw data for the rest.

10. **Why are in-memory databases fast, if it isn't because they avoid disk reads?**
    → Because the OS page cache already means a disk-based engine may never touch the disk. The real saving is avoiding the cost of *encoding* in-memory structures into a disk-writable format on every operation.

---

*All quoted material, figures, examples and statistics are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 3. Diagrams have been redrawn in ASCII from the book's originals. Section 20 and the code appendix are supplementary material I added; every quoted output was produced by actually running the code shown.*
