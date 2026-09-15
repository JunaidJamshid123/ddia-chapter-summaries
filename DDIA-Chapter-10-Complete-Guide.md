# DDIA — Part III: Derived Data
# Chapter 10: Batch Processing
### Complete study guide — theory, every diagram redrawn, worked examples, and what changed since 2017

> *A system cannot be successful if it is too strongly influenced by a single person. Once the initial design is complete and fairly robust, the real test begins as people with many different viewpoints undertake their own experiments.*
> — Donald Knuth

---

# PART III INTRODUCTION — DERIVED DATA

## 0. Why there's a Part III at all

> Parts I and II *"assembled from the ground up all the major considerations that go into a distributed database… **However, this discussion assumed that THERE WAS ONLY ONE DATABASE IN THE APPLICATION.**"*

```
   "In a large application you often need to be able to ACCESS AND PROCESS DATA
    IN MANY DIFFERENT WAYS, and THERE IS NO ONE DATABASE WHICH CAN SATISFY ALL
    THOSE DIFFERENT NEEDS SIMULTANEOUSLY."

   ➜ applications use a COMBINATION of datastores, indexes, caches, analytics
     systems — and MECHANISMS FOR MOVING DATA FROM ONE STORE TO ANOTHER.

   ⚠️ "This aspect of system-building is OFTEN OVERLOOKED BY VENDORS who claim
      that their product can satisfy all your needs. In reality, INTEGRATING
      DISPARATE SYSTEMS IS ONE OF THE MOST IMPORTANT THINGS that needs to be
      done in a non-trivial application."
```

## 1. The distinction that organizes the whole of Part III

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  SYSTEM OF RECORD                 ║  DERIVED DATA SYSTEM              ║
   ║  (a.k.a. SOURCE OF TRUTH)         ║                                   ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  Holds the AUTHORITATIVE version. ║  "the result of TAKING SOME       ║
   ║  New data is written HERE FIRST.  ║   EXISTING DATA from another      ║
   ║                                   ║   system and TRANSFORMING or      ║
   ║  "Each fact is represented        ║   PROCESSING it in some way."     ║
   ║   EXACTLY ONCE (the               ║                                   ║
   ║   representation is typically     ║  🔑 "IF YOU LOSE DERIVED DATA, YOU║
   ║   NORMALIZED)."                   ║    CAN RE-CREATE IT FROM THE      ║
   ║                                   ║    ORIGINAL SOURCE."              ║
   ║  "If there is any discrepancy…    ║                                   ║
   ║   the value in the system of      ║  Examples: CACHES · DENORMALIZED  ║
   ║   record is BY DEFINITION the     ║  VALUES · INDEXES · MATERIALIZED  ║
   ║   correct one."                   ║  VIEWS · recommendation models    ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝

   "Technically speaking, derived data is REDUNDANT… However, it is often
    ESSENTIAL for getting good performance on read queries."

   ⚑ THE CRUCIAL POINT: "Most databases, storage engines and query languages
     are NOT INHERENTLY a system of record or a derived system. A DATABASE IS
     JUST A TOOL: HOW YOU USE IT IS UP TO YOU."

   ➜ "By being clear about WHICH DATA IS DERIVED FROM WHICH OTHER DATA, you
     can bring clarity to an otherwise confusing system architecture. THIS
     POINT WILL BE A RUNNING THEME THROUGHOUT PART III."
```

---
---

# CHAPTER 10 — BATCH PROCESSING

## 2. Three types of system

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  ① SERVICES (ONLINE SYSTEMS)                                          ║
   ║     Waits for a request, handles it AS QUICKLY AS POSSIBLE, responds. ║
   ║     PRIMARY METRIC: RESPONSE TIME. Availability matters a lot.        ║
   ╟───────────────────────────────────────────────────────────────────────╢
   ║  ② BATCH PROCESSING SYSTEMS (OFFLINE SYSTEMS)   ← THIS CHAPTER        ║
   ║     Takes a LARGE AMOUNT OF INPUT, runs a job, produces output.       ║
   ║     "Jobs often take a while (from A FEW MINUTES TO SEVERAL DAYS), so ║
   ║      there NORMALLY ISN'T A USER WAITING for the job to finish."      ║
   ║     Usually SCHEDULED PERIODICALLY (e.g. once a day).                 ║
   ║     PRIMARY METRIC: THROUGHPUT.                                       ║
   ╟───────────────────────────────────────────────────────────────────────╢
   ║  ③ STREAM PROCESSING (NEAR-REAL-TIME / NEARLINE)     ← CHAPTER 11     ║
   ║     "somewhere BETWEEN online and offline… a job operates on EVENTS   ║
   ║      SHORTLY AFTER THEY HAPPEN, whereas a batch job operates on A     ║
   ║      FIXED SET OF INPUT DATA."                                        ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 📖 A note on MapReduce's status, written in 2017

> MapReduce *"was (perhaps OVER-ENTHUSIASTICALLY) called 'the algorithm that makes Google so massively scalable'… However, MapReduce is in some ways ALSO A STEP BACKWARDS from more sophisticated parallel processing techniques that were developed for data warehouses MANY YEARS BEFORE. **Although the importance of MapReduce is now declining, it is still worth understanding, because it provides a clear picture of WHY AND HOW batch processing is useful.**"*

```
   📜 AND THE HISTORY IS OLDER THAN YOU THINK:
   "Long before programmable digital computers were invented, PUNCH CARD
    TABULATING MACHINES — such as the HOLLERITH MACHINES used in the 1890 US
    CENSUS — implemented a semi-mechanized form of batch processing…
    And MapReduce bears AN UNCANNY RESEMBLANCE to the ELECTROMECHANICAL IBM
    CARD-SORTING MACHINES that were widely used for business data processing
    in the 1940s and 1950s. AS USUAL, HISTORY HAS A TENDENCY OF REPEATING
    ITSELF."
```

---

## 3. Batch processing with Unix tools

### The worked example: log analysis

A single line of an nginx access log:

```
216.58.210.78 - - [27/Feb/2015:17:55:11 +0000] "GET /css/typography.css HTTP/1.1"
200 3377 "http://martin.kleppmann.com/" "Mozilla/5.0 (Macintosh; Intel Mac OS X
10_9_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/40.0.2214.115 Safari/537.36"
```

Interpreted via the log format `$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"`.

### Finding the 5 most popular pages — the pipeline

```bash
cat /var/log/nginx/access.log |     # ① read the log file
  awk '{print $7}'            |     # ② extract field 7 = the requested URL
  sort                        |     # ③ sort alphabetically → duplicates adjacent
  uniq -c                     |     # ④ collapse adjacent duplicates, WITH A COUNT
  sort -r -n                  |     # ⑤ sort numerically, reversed
  head -n 5                         # ⑥ keep the top 5
```

```
   OUTPUT:
      4189 /favicon.ico
      3631 /2013/05/24/improving-security-of-ssh-private-keys.html
      2124 /2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html
      1369 /
       915 /css/typography.css

   ⚑ EASY TO MODIFY:
      omit CSS files     → awk '$7 !~ /\.css$/ {print $7}'
      top IP addresses   → awk '{print $1}'

   "It will process GIGABYTES of log files in A MATTER OF SECONDS… Surprisingly
    many data analyses can be done in A FEW MINUTES using some combination of
    awk, sed, grep, sort, uniq and xargs, and THEY PERFORM SURPRISINGLY WELL."
```

> 📖 *Footnote:* *"Some people love to point out that `cat` is unnecessary here… However, THE LINEAR PIPELINE IS MORE APPARENT when written like this."*

### The same thing as a program

```ruby
counts = Hash.new(0)                      # ① a counter per URL, default 0

File.open('/var/log/nginx/access.log') do |file|
  file.each do |line|
    url = line.split[6]                   # ② field 7 (zero-indexed)
    counts[url] += 1                      # ③ increment
  end
end

top5 = counts.map{|url, count| [count, url] }.sort.reverse[0...5]   # ④
top5.each{|count, url| puts "#{count} #{url}" }                     # ⑤
```

> *"which of the two you prefer is PARTLY A MATTER OF TASTE. However, besides the superficial syntactic differences, THERE IS A BIG DIFFERENCE IN THE EXECUTION FLOW, which becomes apparent if you run this analysis ON A LARGE FILE."*

### 🔑 Sorting vs in-memory aggregation — the real difference

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  RUBY: IN-MEMORY HASH TABLE       ║  UNIX PIPELINE: SORTING           ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║  WORKING SET depends only on the  ║  No hash table at all — relies on ║
   ║  NUMBER OF DISTINCT URLs.         ║  SORTING a list in which          ║
   ║  "if there are A MILLION LOG      ║  duplicates are simply repeated.  ║
   ║   ENTRIES FOR A SINGLE URL, the   ║                                   ║
   ║   space required is STILL JUST    ║  ✅ "can make EFFICIENT USE OF     ║
   ║   ONE URL plus the counter."      ║     DISKS" when the working set   ║
   ║                                   ║     EXCEEDS MEMORY                ║
   ║  ✅ fine if distinct URLs fit in   ║                                   ║
   ║     ~1 GB — "EVEN ON A LAPTOP"    ║  Chunks sorted in memory → spill  ║
   ║  ❌ breaks when they don't         ║  to disk as segments → MERGE.     ║
   ║                                   ║  ← EXACTLY SSTables from Ch.3     ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝

   "GNU Coreutils `sort` AUTOMATICALLY HANDLES LARGER-THAN-MEMORY DATASETS by
    SPILLING TO DISK, and AUTOMATICALLY PARALLELIZES sorting across multiple
    CPU cores… THE BOTTLENECK IS LIKELY TO BE THE RATE AT WHICH THE INPUT FILE
    CAN BE READ FROM DISK."

   ⚑ "Remember that OPTIMIZING FOR SEQUENTIAL I/O was a recurring theme in
     Chapter 3. THE SAME PATTERN REAPPEARS HERE."
```

---

## 4. The Unix philosophy

> Doug McIlroy, inventor of Unix pipes, **1964**: *"We should have some ways of connecting programs like [a] garden hose — SCREW IN ANOTHER SEGMENT when it becomes necessary to massage data in another way. This is the way of I/O also."*

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE FOUR PRINCIPLES (as described in 1978)                           ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  ① "MAKE EACH PROGRAM DO ONE THING WELL. To do a new job, BUILD       ║
   ║     AFRESH rather than complicate old programs by adding new          ║
   ║     'features'."                                                      ║
   ║                                                                       ║
   ║  ② "EXPECT THE OUTPUT OF EVERY PROGRAM TO BECOME THE INPUT TO         ║
   ║     ANOTHER, AS YET UNKNOWN, PROGRAM. Don't clutter output with       ║
   ║     extraneous information. Avoid stringently columnar or binary      ║
   ║     input formats. Don't insist on interactive input."                ║
   ║                                                                       ║
   ║  ③ "DESIGN AND BUILD SOFTWARE, EVEN OPERATING SYSTEMS, TO BE TRIED    ║
   ║     EARLY, IDEALLY WITHIN WEEKS. Don't hesitate to THROW AWAY THE     ║
   ║     CLUMSY PARTS and rebuild them."                                   ║
   ║                                                                       ║
   ║  ④ "USE TOOLS IN PREFERENCE TO UNSKILLED HELP to lighten a            ║
   ║     programming task, even if you have to DETOUR TO BUILD THE TOOLS   ║
   ║     and expect to throw some of them out afterwards."                 ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   "This approach… sounds REMARKABLY LIKE THE AGILE AND DEVOPS MOVEMENTS OF
    TODAY. SURPRISINGLY LITTLE HAS CHANGED IN FOUR DECADES."
```

### 💡 `sort` as the exemplar

> *"`sort` is a great example of a program that does one thing well. It is ARGUABLY A BETTER SORTING IMPLEMENTATION THAN MOST PROGRAMMING LANGUAGES HAVE IN THEIR STANDARD LIBRARY (which do not spill to disk and do not use multiple threads). And yet, **`sort` is barely useful in isolation.** It only becomes powerful in combination with the other Unix tools, such as `uniq`. **IT WOULD HAVE BEEN EASY FOR THE IMPLEMENTER OF `sort` TO ADD `uniq` AS A FEATURE, BUT THEY RESISTED THE TEMPTATION.**"*

---

## 5. What makes composability possible — three properties

### ① A uniform interface

```
   "If you want to be able to connect ANY program's output to ANY program's
    input, that means ALL PROGRAMS MUST USE THE SAME INPUT/OUTPUT INTERFACE.

    In Unix, that interface is A FILE (more precisely, A FILE DESCRIPTOR).
    A file is just AN ORDERED SEQUENCE OF BYTES."

   ┌─────────────────────────────────────────────────────────────────────┐
   │  THE SAME INTERFACE REPRESENTS ALL OF THESE:                        │
   │     • an actual file on the filesystem                              │
   │     • a communication channel to another process (socket, stdin,    │
   │       stdout)                                                       │
   │     • a device driver (/dev/audio, /dev/lp0)                        │
   │     • a socket representing a TCP connection                        │
   │                                                                     │
   │  "It's easy to take this for granted, but it's actually QUITE       │
   │   REMARKABLE that these VERY DIFFERENT THINGS can share a uniform   │
   │   interface, so they can easily be PLUGGED TOGETHER."               │
   └─────────────────────────────────────────────────────────────────────┘

   BY CONVENTION most tools treat the bytes as ASCII TEXT, with \n as the
   record separator.
   📖 "The choice of \n is ARBITRARY — arguably, the ASCII RECORD SEPARATOR
      0x1E would have been a better choice, since it's INTENDED FOR THIS
      PURPOSE — but in any case, the fact that all these programs have
      STANDARDIZED on the same separator ALLOWS THEM TO INTEROPERATE."
```

> 📖 **The web as the other great uniform interface:** *"A URL identifies a particular resource on a website, and you can LINK TO ANY URL FROM ANY OTHER WEBSITE… This seems obvious today, but it was A KEY INSIGHT towards making the web the success that it is."*

```
   ⚠️ THE HONEST ASSESSMENT:
   "The uniform interface of ASCII text MOSTLY WORKS, BUT IT'S NOT EXACTLY
    BEAUTIFUL: our log analysis example used {print $7} to extract the URL,
    WHICH IS NOT VERY READABLE. In an ideal world this could have perhaps been
    {print $request_url}."

   😐 AND THE CONTRAST WITH TODAY:
   "Not many pieces of software interoperate and compose as well as Unix tools
    do: YOU CAN'T EASILY PIPE THE CONTENTS OF YOUR EMAIL ACCOUNT AND YOUR
    ONLINE SHOPPING HISTORY through a custom analysis tool, into a spreadsheet,
    and post the results to a social network or a wiki. TODAY IT'S AN
    EXCEPTION, NOT THE NORM, to have programs that work together as smoothly
    as Unix tools do.
    …This leads to BALKANIZATION OF DATA."
```

### ② Separation of logic and wiring

```
                     ┌──────────────────────────────────────┐
   keyboard ────────►│ stdin                        stdout  │────────► screen
   or a file ───────►│         YOUR PROGRAM                 │────────► or a file
   or a pipe ───────►│  (doesn't know or care which)        │────────► or a pipe
                     └──────────────────────────────────────┘

   "the Unix approach works best if A PROGRAM DOESN'T WORRY ABOUT PARTICULAR
    FILE PATHS, and simply uses stdin and stdout. This allows a shell user to
    WIRE UP THE INPUT AND OUTPUT IN WHATEVER WAY THEY WANT."

   ⚑ "One could say this is a form of LOOSE COUPLING, LATE BINDING or
     INVERSION OF CONTROL."

   ➜ YOU CAN JOIN IN: write a tool that translates user-agent strings into
     browser names, or IP addresses into country codes, and SIMPLY PLUG IT
     INTO THE PIPELINE. "`sort` DOESN'T CARE whether it's communicating with
     another part of the operating system or with a program written by you."

   ⚠️ THE LIMITS: "Programs that need MULTIPLE INPUTS OR OUTPUTS are possible
      but TRICKY. You CAN'T PIPE a program's output into A NETWORK CONNECTION."
      📖 (Plan 9 and Inferno are MORE CONSISTENT: they represent a TCP
         connection as a file in /net/tcp.)
```

### ③ Transparency and experimentation

```
   ✅ "The input files are normally treated as IMMUTABLE. This means you can
      RUN THE COMMANDS AS OFTEN AS YOU WANT, trying various options, WITHOUT
      DAMAGING THE INPUT FILES."
   ✅ "You can END THE PIPELINE AT ANY POINT, pipe the output into `less`, and
      look at it. THIS IS GREAT FOR DEBUGGING."
   ✅ "You can WRITE THE OUTPUT OF ONE STAGE TO A FILE and use that as input to
      the next stage. This allows you to RESTART THE LATER STAGE WITHOUT
      RE-RUNNING THE ENTIRE PIPELINE."

   ❌ "the BIGGEST LIMITATION of Unix tools is that THEY RUN ONLY ON A SINGLE
      MACHINE — and that's where tools like Hadoop come in."
```

---

## 6. MapReduce and distributed filesystems

> **"MapReduce is A BIT LIKE UNIX TOOLS, BUT DISTRIBUTED across potentially thousands of machines. Like Unix tools, it is a FAIRLY BLUNT, BRUTE-FORCE, BUT SURPRISINGLY EFFECTIVE TOOL."**

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  UNIX                            │  MAPREDUCE                           │
   ├──────────────────────────────────┼──────────────────────────────────────┤
   │  a single process                │  a single JOB                        │
   │  stdin / stdout                  │  files on a DISTRIBUTED FILESYSTEM   │
   │  input unmodified, no side       │  input unmodified, no side effects   │
   │  effects beyond the output       │  beyond the output                   │
   │                                  │  "output files are written ONCE, in  │
   │                                  │   a SEQUENTIAL fashion"              │
   └──────────────────────────────────┴──────────────────────────────────────┘
```

### HDFS

```
   HDFS = Hadoop Distributed FileSystem, an open source re-implementation of
   Google's GFS. Based on the SHARED-NOTHING principle.

   ┌──────────────────────────────────────────────────────────────────────┐
   │              ┌──────────────┐                                        │
   │              │   NAMENODE   │  keeps track of WHICH FILE BLOCKS      │
   │              │  (central)   │  are stored on WHICH MACHINE           │
   │              └──────┬───────┘                                        │
   │        ┌────────────┼────────────┬────────────┐                      │
   │        ▼            ▼            ▼            ▼                      │
   │   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐                     │
   │   │ daemon │  │ daemon │  │ daemon │  │ daemon │  a daemon on EVERY  │
   │   │ ▓disk▓ │  │ ▓disk▓ │  │ ▓disk▓ │  │ ▓disk▓ │  general-purpose    │
   │   └────────┘  └────────┘  └────────┘  └────────┘  machine            │
   │   └──── conceptually ONE BIG FILESYSTEM over all those disks ────┘   │
   └──────────────────────────────────────────────────────────────────────┘

   FAULT TOLERANCE: blocks REPLICATED on multiple machines — either plain
   copies, or ERASURE CODING (Reed-Solomon), "which allows lost data to be
   recovered with LOWER STORAGE OVERHEAD than full replication."

   ⚑ vs NAS/SAN: those use a CENTRALIZED storage appliance with CUSTOM
     HARDWARE and special networks (Fibre Channel). HDFS uses "A CONVENTIONAL
     DATACENTER NETWORK WITHOUT SPECIAL HARDWARE."

   📊 SCALE: "the biggest HDFS deployments run on TENS OF THOUSANDS OF
      MACHINES, with combined storage capacity of HUNDREDS OF PETABYTES."
```

> 📖 **HDFS vs object storage (S3, Azure Blob, Swift):** *"One difference is that with HDFS, COMPUTING TASKS CAN BE SCHEDULED TO RUN ON THE MACHINE THAT STORES A COPY of a particular file, whereas object stores usually KEEP STORAGE AND COMPUTATION SEPARATE… Note that IF ERASURE CODING IS USED, THE LOCALITY ADVANTAGE IS LOST."*

---

## 7. MapReduce job execution

### 🔑 The four steps — and how they map onto the Unix pipeline

```
   ┌───┬─────────────────────────────────────┬────────────────────────────┐
   │ ① │ Read input files, break into RECORDS│ each line (\n separated)   │
   ├───┼─────────────────────────────────────┼────────────────────────────┤
   │ ② │ Extract a KEY and VALUE from each   │ awk '{print $7}'  ← MAPPER │
   ├───┼─────────────────────────────────────┼────────────────────────────┤
   │ ③ │ SORT all key-value pairs BY KEY     │ sort    ← IMPLICIT! You    │
   │   │                                     │         don't write this.  │
   ├───┼─────────────────────────────────────┼────────────────────────────┤
   │ ④ │ Iterate over the sorted list,       │ uniq -c           ← REDUCER│
   │   │ combining adjacent same-key values  │                            │
   └───┴─────────────────────────────────────┴────────────────────────────┘

   🔑 "Note that step 3, the sort step, IS IMPLICIT in MapReduce — you don't
     have to write it, because the output from the mapper is ALWAYS SORTED
     before it is given to the reducer. THIS SORTING STEP IS ARGUABLY THE MOST
     IMPORTANT ASPECT OF MAPREDUCE."

   ➜ AND THE ROLES, STATED CLEANLY:
     "the role of the MAPPER is to PREPARE THE DATA INTO A FORM THAT IS
      SUITABLE FOR SORTING, and the role of the REDUCER is to PROCESS THE DATA
      THAT HAS BEEN SORTED."
```

### 🔷 Figure 10-1 — A MapReduce job with three mappers and three reducers

```
   HDFS INPUT
   DIRECTORY        MAP TASKS                      REDUCE TASKS         HDFS OUTPUT
   ┌───┐      ┌──────────────────┐           ┌──────────────────┐       ┌───────┐
   │   │      │ MAP TASK 1       │  ┌─fetch─►│ REDUCE TASK 1    │       │       │
   │m1 │─────►│  ┌────────┐ m1,r1│──┤        │  m1,r1 ┐         │       │       │
   │   │      │  │ Mapper │ m1,r2│──┼──┐     │  m2,r1 ├─merge──►│──────►│  r1   │
   │   │      │  └────────┘ m1,r3│──┼──┼──┐  │  m3,r1 ┘ Reducer │       │       │
   ├───┤      ├──────────────────┤  │  │  │  ├──────────────────┤       ├───────┤
   │   │      │ MAP TASK 2       │  │  │  │  │ REDUCE TASK 2    │       │       │
   │m2 │─────►│  ┌────────┐ m2,r1│──┘  │  │  │  m1,r2 ┐         │       │       │
   │   │      │  │ Mapper │ m2,r2│─────┼──┼─►│  m2,r2 ├─merge──►│──────►│  r2   │
   │   │      │  └────────┘ m2,r3│─────┼──┼─►│  m3,r2 ┘ Reducer │       │       │
   ├───┤      ├──────────────────┤     │  │  ├──────────────────┤       ├───────┤
   │   │      │ MAP TASK 3       │     │  │  │ REDUCE TASK 3    │       │       │
   │m3 │─────►│  ┌────────┐ m3,r1│─────┘  │  │  m1,r3 ┐         │       │       │
   │   │      │  │ Mapper │ m3,r2│────────┘  │  m2,r3 ├─merge──►│──────►│  r3   │
   │   │      │  └────────┘ m3,r3│───────────►  m3,r3 ┘ Reducer │       │       │
   └───┘      └──────────────────┘           └──────────────────┘       └───────┘
                                  └──── THE SHUFFLE ────┘

   • NUMBER OF MAP TASKS  = number of input file BLOCKS (typically hundreds of
     MB each)
   • NUMBER OF REDUCE TASKS = CONFIGURED BY THE JOB AUTHOR
   • which reducer gets a key = HASH OF THE KEY (← Chapter 6 partitioning)
```

### 💡 Putting the computation near the data

> *"The MapReduce scheduler tries to run each mapper ON ONE OF THE MACHINES THAT STORES A REPLICA OF THE INPUT FILE, provided that machine has enough spare RAM and CPU… This principle is known as **PUTTING THE COMPUTATION NEAR THE DATA**: it SAVES COPYING THE INPUT FILE OVER THE NETWORK, reducing network load and increasing locality."*
>
> *"In most cases, THE APPLICATION CODE IS NOT YET PRESENT on the machine assigned the task, so the framework FIRST COPIES THE CODE (e.g. jar files) TO THE APPROPRIATE MACHINES."*

**Note the inversion.** Normally you move data to the program. Here the *program* — a few megabytes of JARs — is shipped to wherever the *data* already lives, because the data is hundreds of megabytes.

### The shuffle, explained precisely

```
   ① Each MAPPER partitions its output BY REDUCER (hash of key).
   ② Each partition is written to A SORTED FILE ON THE MAPPER'S LOCAL DISK,
      "using a technique similar to SSTables and LSM-trees" (← Chapter 3).
   ③ When a mapper finishes, the scheduler NOTIFIES THE REDUCERS that they can
      start FETCHING.
   ④ Reducers connect to each mapper and DOWNLOAD the sorted file for their
      partition.
   ⑤ The reducer MERGES the files together, PRESERVING SORT ORDER.

   ⚑ "The process of partitioning by reducer, sorting, and copying data
     partitions from mappers to reducers is known as THE SHUFFLE —
     A CONFUSING TERM: UNLIKE SHUFFLING A DECK OF CARDS, THERE IS NO
     RANDOMNESS IN MAPREDUCE."

   The reducer is then called with a key and an ITERATOR that INCREMENTALLY
   SCANS over all records with that key — "which may IN SOME CASES BE BIGGER
   THAN WHAT CAN FIT IN MEMORY."
```

### Workflows

```
   "The range of problems you can solve with A SINGLE MapReduce job are
    LIMITED." (One job gets page views per URL; the MOST POPULAR URLs needs a
    SECOND round of sorting.)

   ➜ jobs are CHAINED INTO WORKFLOWS. "The Hadoop MapReduce framework DOES NOT
     HAVE ANY PARTICULAR SUPPORT for workflows, so this chaining is done
     IMPLICITLY BY DIRECTORY NAME."

   ┌────────┐ writes  ┌──────────┐ reads  ┌────────┐ writes  ┌──────────┐
   │ JOB 1  │────────►│ /tmp/out1│───────►│ JOB 2  │────────►│ /tmp/out2│
   └────────┘         └──────────┘        └────────┘         └──────────┘
   "From the framework's point of view, THEY ARE TWO INDEPENDENT JOBS."

   ⚑ "Chained MapReduce jobs are therefore LESS LIKE PIPELINES OF UNIX
     COMMANDS (which pass output directly using only a small in-memory
     buffer), and MORE LIKE a sequence of commands where EACH COMMAND'S OUTPUT
     IS WRITTEN TO A TEMPORARY FILE."

   • A job's output is ONLY VALID WHEN THE JOB COMPLETES SUCCESSFULLY —
     "MapReduce DISCARDS THE PARTIAL OUTPUT of a failed job."
   • SCHEDULERS handle the dependencies: Oozie, Azkaban, Luigi, AIRFLOW,
     Pinball.
   📊 "Workflows consisting of 50 TO 100 MAPREDUCE JOBS are common when
      building RECOMMENDATION SYSTEMS."
```

---

## 8. Reduce-side joins and grouping

### Why MapReduce joins look nothing like database joins

```
   "In a database, if you execute a query that involves only A SMALL NUMBER OF
    RECORDS, the database would typically USE AN INDEX… However, MAPREDUCE HAS
    NO CONCEPT OF INDEXES — at least not in the usual sense.

    When a MapReduce job is given a set of files as input, IT READS THE ENTIRE
    CONTENT OF ALL OF THOSE FILES; a database would call this a FULL TABLE
    SCAN."

   ⚑ IS THAT CRAZY? No: "in ANALYTIC queries it is common to want to CALCULATE
     AGGREGATES OVER A LARGE NUMBER OF RECORDS. In this case, scanning the
     entire input might be QUITE A REASONABLE THING TO DO, especially if you
     can PARALLELIZE the processing."

   ➜ "When we talk about joins in the context of batch processing, we mean
     RESOLVING ALL OCCURRENCES OF SOME ASSOCIATION WITHIN A DATASET… a job is
     processing the data FOR ALL USERS SIMULTANEOUSLY, not merely looking up
     the data for one particular user."
```

### 🔷 Figure 10-2 — The join to be performed

```
   USER ACTIVITY EVENTS                      USERS DATABASE
   (the FACT table)                          (a DIMENSION table)
   ┌───────────────────────────────────┐     ┌────────┬──────────────────────┬──────────────┐
   │ user 105 clicked button … URL …   │     │user_id │ email                │date_of_birth │
   │ user 296 viewed profile of user…  │     ├────────┼──────────────────────┼──────────────┤
   │ user 251 logged out from session… │     │  100   │ atilla@example.com   │ 1991-02-15   │
   │ user 184 loaded page with URL …   │     │  101   │ beth@foo.com         │ 1952-06-29   │
   │ user 101 posted a response to …   │     │  102   │ chunzhi@test.net     │ 1967-09-08   │
   │ user 156 clicked the help button… │     │  103   │ devaraj@example.net  │ 1947-12-18   │
   │ user 301 clicked a link in email… │     │  104   │ evelyn@example.com   │ 1989-01-30   │
   │ user 123 searched for keyword …   │     │  105   │ flavio@foo.com       │ 1971-05-05   │
   └───────────────────────────────────┘     └────────┴──────────────────────┴──────────────┘
              ▲                                        ▲
       only the USER ID                    the AGE we want to correlate with
```

> **The goal:** *"if the profile contains the user's age, the system could determine WHICH PAGES ARE MOST POPULAR WITH WHICH AGE GROUPS. However, the activity events contain ONLY THE USER ID… Embedding that profile information in EVERY SINGLE ACTIVITY EVENT would most likely be TOO WASTEFUL."*

### ❌ The obvious approach, and why it fails

```
   "go over the activity events one by one, and QUERY THE USER DATABASE (on a
    remote server) for EVERY USER ID it encounters."

   ❌ throughput LIMITED BY THE ROUND-TRIP TIME to the database server
   ❌ local cache effectiveness "depends very much on the DISTRIBUTION OF DATA"
   ❌ "running a large number of queries in parallel could EASILY OVERWHELM THE
      DATABASE"
   ❌ 🔑 "querying a remote database would mean that THE BATCH JOB BECOMES
      NON-DETERMINISTIC, because the data in the remote database MIGHT CHANGE."

   ➜ "In order to achieve good throughput in a batch process, THE COMPUTATION
     MUST BE LOCAL TO ONE MACHINE AS MUCH AS POSSIBLE."
   ➜ THE FIX: take A COPY of the users database (via ETL from a backup) and put
     it in THE SAME DISTRIBUTED FILESYSTEM.
```

### 🔷 Figure 10-3 — The reduce-side sort-merge join

```
   USER ACTIVITY MAPPER
   ┌────────────────────────┐  ┌──────────┐                  REDUCER PARTITION 1
   │ user 104 loaded URL /x │─►│          │─► 104 → url:/x     (EVEN user IDs)
   │ user 173 loaded URL /y │  │  mapper  │─► 104 → url:/z    ┌────────────────┐
   │ user 104 loaded URL /z │─►│          │─► 173 → url:/y    │ 104 →          │
   └────────────────────────┘  └──────────┘          │        │   dob: 1989    │
                                                     ├───────►│   url: /x      │
   USER DATABASE MAPPER                              │        │   url: /z      │
   ┌────────────────────────┐  ┌──────────┐          │        │                │
   │ {user_id: 103, …,      │─►│          │─► 103 →  │        │ reducer outputs│
   │  date_of_birth: 1947…} │  │  mapper  │   dob:1947        │  {url:/x,      │
   │ {user_id: 104, …,      │─►│          │─► 104 →  │        │   dob:1989}    │
   │  date_of_birth: 1989…} │  └──────────┘   dob:1989        │  {url:/z,      │
   └────────────────────────┘                          │      │   dob:1989}    │
                                                       │      └────────────────┘
                                          REDUCER PARTITION 2 (ODD user IDs) …

   🔑 "When the framework partitions the mapper output by key, and then SORTS
     the key-value pairs, THE EFFECT IS THAT ALL THE ACTIVITY EVENTS AND THE
     USER RECORD WITH THE SAME USER ID BECOME ADJACENT TO EACH OTHER in the
     reducer input."
```

### Secondary sort — the detail that makes the reducer trivial

```
   "The job can even arrange the records to be sorted such that the REDUCER
    ALWAYS SEES THE RECORD FROM THE USER DATABASE FIRST, followed by the
    activity events IN TIMESTAMP ORDER — this is known as a SECONDARY SORT."

   ➜ So the reduce function becomes almost embarrassingly simple:
        reduce(user_id, values):
            dob = values.next()        # ← guaranteed to be the user record
            for event in values:       # ← then the activity events, in order
                emit(event.url, age_from(dob))

   "the reducer REMEMBERS THE DATE OF BIRTH IN A LOCAL VARIABLE… it only needs
    to KEEP ONE USER RECORD IN MEMORY at any one time, and IT NEVER NEEDS TO
    MAKE ANY REQUESTS OVER THE NETWORK."
```

### 🔑 The mental model: mappers send messages to reducers

> **"One way of looking at this architecture is that MAPPERS 'SEND MESSAGES' TO THE REDUCERS. When a mapper emits a key-value pair, THE KEY IS LIKE THE DESTINATION ADDRESS to which the value should be delivered. Even though the key is just an arbitrary string (not a physical network address), IT ACTS LIKE AN ADDRESS: all key-value pairs with the same key will be delivered to THE SAME DESTINATION."**

```
   ➜ THE SEPARATION THIS BUYS YOU:
   "Using the MapReduce programming model HAS SEPARATED THE PHYSICAL NETWORK
    COMMUNICATION ASPECTS (getting the data to the right machine) FROM THE
    APPLICATION LOGIC (processing the data once you have it). This is IN
    CONTRAST TO THE TYPICAL USE OF DATABASES, where a request to fetch data
    often occurs SOMEWHERE DEEPLY INSIDE a piece of application code."

   "Since MapReduce handles all network communication, IT ALSO SHIELDS THE
    APPLICATION CODE FROM HAVING TO WORRY ABOUT PARTIAL FAILURES."  ← Ch.8
```

### GROUP BY and sessionization

```
   "grouping and joining LOOK QUITE SIMILAR when implemented on top of
    MapReduce" — both are "bringing related data to the same place."

   SESSIONIZATION: "collating all the activity events for a particular user
   session, in order to find out THE SEQUENCE OF ACTIONS THE USER TOOK."
      → used for A/B TESTING: "whether users who were shown a new version of
        your website are MORE LIKELY TO MAKE A PURCHASE."
      → "If you have MULTIPLE WEB SERVERS, the activity events for a
        particular user are most likely SCATTERED ACROSS VARIOUS DIFFERENT
        SERVERS' LOG FILES." Group by session cookie / user ID.
```

### ⚠️ Handling skew — the linchpin problem

```
   "The pattern of 'bringing all records with the same key to the same place'
    BREAKS DOWN if there is A VERY LARGE AMOUNT OF DATA RELATED TO A SINGLE
    KEY." → CELEBRITIES with millions of followers.
   📖 "Such disproportionately active database records are known as LINCHPIN
      OBJECTS."   (← Chapter 6's hot keys, again)

   ⚠️ "Since a MapReduce job is ONLY COMPLETE WHEN ALL of its mappers and
      reducers have completed, ANY SUBSEQUENT JOBS MUST WAIT FOR THE SLOWEST
      REDUCER."

   ┌──────────────────────────────────────────────────────────────────────┐
   │  FIX 1 — SKEW JOIN (Pig/Hive) / SHARDED JOIN (Crunch)                │
   │    • send records for a linchpin object to A RANDOM REDUCER (instead │
   │      of a deterministic hash)                                        │
   │    • send the OTHER join input's records for that object TO ALL      │
   │      REDUCERS                                                        │
   │    ➜ "the burden is EVENLY SHARED, at the cost of having to          │
   │      REPLICATE the other join input to multiple reducers."           │
   ├──────────────────────────────────────────────────────────────────────┤
   │  FIX 2 — TWO-STAGE GROUPING (for GROUP BY on a skewed key)           │
   │    STAGE 1: send records to a RANDOM reducer → each computes a       │
   │             PARTIAL AGGREGATE for a subset                           │
   │    STAGE 2: combine the partial aggregates into ONE VALUE PER KEY    │
   └──────────────────────────────────────────────────────────────────────┘
```

---

## 9. Map-side joins — skipping the reducers entirely

> *"The reduce-side approach has the advantage that you DO NOT NEED TO MAKE ANY ASSUMPTIONS about the input data… However, the downside is that ALL THAT SORTING, COPYING TO REDUCERS, AND MERGING can be quite expensive."*
>
> A map-side join uses *"a CUT-DOWN MapReduce job in which there are NO REDUCERS AND NO SORTING. Instead, each mapper simply READS ONE INPUT FILE BLOCK from HDFS and WRITES ONE OUTPUT FILE to HDFS — that is all."*

### ① Broadcast hash join

```
   APPLIES WHEN: a LARGE dataset is joined with a SMALL one, and the small one
   FITS IN MEMORY IN EACH MAPPER.

   ┌───────────────────────────────────────────────────────────────────────┐
   │   SMALL INPUT (users db)                                              │
   │        ┌────────┐                                                     │
   │        │  users │───────┬────────────┬────────────┐                   │
   │        └────────┘       │            │            │  "BROADCAST"      │
   │                         ▼            ▼            ▼                   │
   │   LARGE INPUT     ┌──────────┐ ┌──────────┐ ┌──────────┐              │
   │   ┌───┬───┬───┐   │ mapper 1 │ │ mapper 2 │ │ mapper 3 │              │
   │   │b1 │b2 │b3 │──►│ +hashtbl │ │ +hashtbl │ │ +hashtbl │              │
   │   └───┴───┴───┘   └──────────┘ └──────────┘ └──────────┘              │
   │   each mapper scans ITS BLOCK of the large input and looks up each    │
   │   record in ITS OWN FULL COPY of the small input's hash table.        │
   └───────────────────────────────────────────────────────────────────────┘

   "BROADCAST reflects that each mapper for a partition of the large input
    reads THE ENTIRETY of the small input; HASH reflects its use of a hash
    table."

   Supported as: Pig "REPLICATED JOIN" · Hive "MAPJOIN" · Cascading · Crunch ·
   also used by Impala.

   💡 ALTERNATIVE: store the small input in A READ-ONLY INDEX ON LOCAL DISK.
      "The frequently-used parts will remain in the OS PAGE CACHE, so this can
       provide random-access lookups ALMOST AS FAST as an in-memory hash
       table, BUT WITHOUT ACTUALLY REQUIRING THE DATASET TO FIT IN MEMORY."
```

### ② Partitioned hash join

```
   APPLIES WHEN: both inputs are partitioned THE SAME WAY (same key, same hash
   function, SAME NUMBER OF PARTITIONS).

   activity events   users db
   ┌───┐             ┌───┐
   │ 0 │◄───────────►│ 0 │  ← mapper 0 handles ONLY partition 0 of BOTH
   │ 1 │◄───────────►│ 1 │
   │ 2 │◄───────────►│ 2 │     e.g. partitioned by the LAST DECIMAL DIGIT
   │ …ceil           │ … │     of the user ID → ten partitions each side
   │ 9 │◄───────────►│ 9 │
   └───┘             └───┘

   ✅ "each mapper only needs to load A SMALLER AMOUNT OF DATA into its hash
      table."
   ⚑ "If the inputs are generated by PRIOR MAPREDUCE JOBS that already perform
     this grouping, then this can be A REASONABLE ASSUMPTION."
   Known in Hive as BUCKETED MAP JOINS.
```

### ③ Map-side merge join

```
   APPLIES WHEN: the inputs are partitioned the same way AND SORTED on the
   same key.

   "it DOES NOT MATTER whether the inputs are small enough to fit in memory,
    because a mapper can perform THE SAME MERGING OPERATION THAT WOULD
    NORMALLY BE DONE BY A REDUCER."

   ⚑ "If a map-side merge join is possible, that probably means PRIOR
     MAPREDUCE JOBS BROUGHT THE DATASETS INTO THIS FORM in the first place."
```

### ⚠️ The consequence for downstream jobs

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  REDUCE-SIDE JOIN OUTPUT         │  MAP-SIDE JOIN OUTPUT                │
   │  partitioned and sorted BY THE   │  partitioned and sorted THE SAME WAY │
   │  JOIN KEY                        │  AS THE LARGE INPUT                  │
   └──────────────────────────────────┴──────────────────────────────────────┘

   ➜ "KNOWING ABOUT THE PHYSICAL DATA LAYOUT of datasets in HDFS becomes
     IMPORTANT when optimizing join strategies: it is not sufficient to just
     know the encoding format and directory name, BUT ALSO THE NUMBER OF
     PARTITIONS, AND THE KEYS BY WHICH IT IS PARTITIONED AND SORTED."

   → this metadata lives in HCATALOG and the HIVE METASTORE.
```

---

## 10. The output of batch workflows

> *"We neglected an important question: WHAT IS THE RESULT of all that processing? WHY ARE WE RUNNING ALL THESE JOBS IN THE FIRST PLACE?"*
>
> *"Where does batch processing fit in? IT IS NEITHER TRANSACTION PROCESSING, NOR IS IT ANALYTICS… The output of a batch process is often NOT A REPORT, but SOME OTHER KIND OF STRUCTURE."*

### ① Building search indexes

```
   "Google's ORIGINAL USE of MapReduce was to build indexes for their search
    engine, implemented as a workflow of FIVE TO TEN MapReduce jobs."
   (They later moved away from it — but "even today, Hadoop MapReduce remains
    a good way of building indexes for Lucene/Solr.")

   HOW: "the mappers PARTITION the set of documents as needed, each reducer
   BUILDS THE INDEX FOR ITS PARTITION, and the index files are written to
   HDFS." (← DOCUMENT-PARTITIONED indexes, Chapter 6)

   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  REBUILD WHOLESALE               │  BUILD INCREMENTALLY                 │
   │  "periodically re-run the entire │  Lucene writes NEW SEGMENT FILES and │
   │   indexing workflow… and REPLACE │  asynchronously MERGES/COMPACTS them │
   │   the previous index files"      │  in the background (← Chapter 3)     │
   │  ❌ expensive if few docs changed │                                      │
   │  ✅ "VERY EASY TO REASON ABOUT:   │                                      │
   │     DOCUMENTS IN, INDEXES OUT"   │                                      │
   └──────────────────────────────────┴──────────────────────────────────────┘
```

### ② Key-value stores as batch output

```
   Used for RECOMMENDATION SYSTEMS and CLASSIFIERS: "a database that can be
   queried by USER ID to obtain SUGGESTED FRIENDS, or by PRODUCT ID to get a
   list of RELATED PRODUCTS."

   ❌ THE OBVIOUS BAD IDEA: write from the mapper/reducer DIRECTLY to the
      production database, one record at a time.
      • "making a network request for every single record is ORDERS OF
        MAGNITUDE SLOWER than the normal throughput of a batch task"
      • "if ALL the mappers or reducers CONCURRENTLY WRITE to the same output
        database… THAT DATABASE CAN EASILY BE OVERWHELMED"
      • 🔑 "writing to an external system from inside a job produces
        EXTERNALLY VISIBLE SIDE-EFFECTS WHICH CANNOT BE HIDDEN. Thus you have
        to worry about the results from PARTIALLY COMPLETED JOBS being visible
        to other systems, and the complexities of task attempts and
        SPECULATIVE EXECUTION."

   ✅ THE GOOD IDEA: "BUILD A BRAND-NEW DATABASE INSIDE THE BATCH JOB, and
      write it as files to the job's output directory in HDFS… Those data files
      are then IMMUTABLE once written, and can be LOADED IN BULK into servers
      that handle read-only queries."
      → Voldemort, Terrapin, ElephantDB, HBase bulk loading.

   ⚑ WHY IT'S A NATURAL FIT: "using a mapper to extract a key, and then
     SORTING by that key, IS ALREADY A LOT OF THE WORK REQUIRED TO BUILD AN
     INDEX." And since the files are write-once-then-immutable, the data
     structures are SIMPLE — "they DO NOT REQUIRE A WAL."

   THE VOLDEMORT SWITCHOVER, which is elegant:
      server keeps serving OLD files → new files copied from HDFS to local
      disk → server ATOMICALLY SWITCHES OVER → "if anything goes wrong, it can
      EASILY SWITCH BACK to the old files, since they are STILL THERE AND
      IMMUTABLE."
```

### 🔑 The philosophy of batch outputs — the best section in the chapter

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "a program READS ITS INPUT and WRITES ITS OUTPUT. In the process,    ║
   ║   THE INPUT IS LEFT UNCHANGED, any previous output is COMPLETELY      ║
   ║   REPLACED, and THERE ARE NO OTHER SIDE-EFFECTS. This means that you  ║
   ║   can RE-RUN A COMMAND AS OFTEN AS YOU LIKE, tweaking or debugging    ║
   ║   it, WITHOUT MESSING UP THE STATE OF YOUR SYSTEM."                   ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ① 🔑 HUMAN FAULT TOLERANCE
      "If you introduce a bug and the output is wrong or corrupted, you can
       SIMPLY ROLL BACK to a previous version of the code, RE-RUN the job, and
       THE OUTPUT WILL BE CORRECT AGAIN. Or even simpler, you can KEEP THE OLD
       OUTPUT IN A DIFFERENT DIRECTORY and SWITCH BACK TO IT.

       ⚠️ DATABASES WITH READ/WRITE TRANSACTIONS DO NOT HAVE THIS PROPERTY: if
          you deploy buggy code that writes bad data to the database, then
          ROLLING BACK THE CODE WILL DO NOTHING TO FIX THE DATA."

   ② FASTER FEATURE DEVELOPMENT — "This principle of MINIMIZING
      IRREVERSIBILITY is beneficial for agile software development."

   ③ SAFE AUTOMATIC RETRIES — "This automatic retry is ONLY SAFE BECAUSE
      INPUTS ARE IMMUTABLE, and outputs from failed tasks are DISCARDED."

   ④ REUSABLE INPUTS — the same files feed MONITORING JOBS that "evaluate
      whether a job's output has the expected characteristics (for example, by
      COMPARING IT TO THE OUTPUT FROM THE PREVIOUS RUN)."

   ⑤ SEPARATION OF LOGIC FROM WIRING — "one team can focus on implementing a
      job that DOES ONE THING WELL, while other teams decide WHERE AND WHEN to
      run it."
```

**Point ① is the one to remember.** Immutability doesn't just buy you retries — it buys you the ability to recover from *your own mistakes*, which mutable databases fundamentally cannot offer.

```
   ⚠️ WHERE HADOOP IMPROVES ON UNIX:
   "because most Unix tools assume UNTYPED TEXT FILES, everything has to do A
    LOT OF INPUT PARSING ({print $7}). On Hadoop, some of those LOW-VALUE
    SYNTACTIC CONVERSIONS ARE ELIMINATED by using more structured file
    formats: AVRO and PARQUET… efficient schema-based encoding, and SCHEMA
    EVOLUTION over time."     ← Chapters 3 and 4, converging
```

---

## 11. MapReduce vs distributed databases

> *"When the MapReduce paper was published, it was — in some sense — NOT AT ALL NEW. All of the processing and parallel join algorithms we discussed had ALREADY BEEN IMPLEMENTED IN so-called MASSIVELY PARALLEL PROCESSING (MPP) DATABASES MORE THAN A DECADE PREVIOUSLY."* (Gamma, Teradata, Tandem NonStop SQL.)

```
   🔑 THE BIGGEST DIFFERENCE:
   "MPP databases focus on PARALLEL EXECUTION OF ANALYTIC SQL QUERIES on a
    cluster, while MapReduce + a distributed filesystem provides SOMETHING
    MUCH MORE LIKE A GENERAL-PURPOSE OPERATING SYSTEM THAT CAN RUN ARBITRARY
    PROGRAMS."
```

### ① Diversity of storage

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  MPP DATABASE                    │  HDFS                                │
   │  requires CAREFUL UP-FRONT       │  files are JUST BYTE SEQUENCES —     │
   │  MODELING before importing into  │  "database records, but equally well │
   │  a PROPRIETARY STORAGE FORMAT    │  IMAGES, VIDEOS, SENSOR READINGS,    │
   │                                  │  SPARSE MATRICES, FEATURE VECTORS,   │
   │                                  │  GENOME SEQUENCES"                   │
   └──────────────────────────────────┴──────────────────────────────────────┘

   "To put it bluntly, Hadoop opened up the possibility of INDISCRIMINATELY
    DUMPING DATA INTO HDFS, and ONLY LATER FIGURING OUT HOW TO PROCESS IT."

   🔑 "in practice, it appears that SIMPLY MAKING DATA AVAILABLE QUICKLY —
     EVEN IF IT IS IN A QUIRKY, DIFFICULT-TO-USE, RAW FORMAT — IS OFTEN MORE
     VALUABLE THAN TRYING TO DECIDE ON THE IDEAL DATA MODEL UP-FRONT."
     → the DATA LAKE / ENTERPRISE DATA HUB
     → "Indiscriminate data dumping SHIFTS THE BURDEN OF INTERPRETING THE
        DATA… the interpretation becomes THE CONSUMER'S PROBLEM
        (SCHEMA-ON-READ)."     ← Chapter 2
     → 🍣 "This approach has been dubbed THE SUSHI PRINCIPLE: 'RAW DATA IS
          BETTER'."

   ➜ Hence Hadoop as an ETL TOOL: dump raw → clean with MapReduce → import
     into an MPP warehouse. "Data modeling STILL HAPPENS, but it is in A
     SEPARATE STEP, DECOUPLED FROM THE DATA COLLECTION."
```

### ② Diversity of processing models

```
   MPP: "MONOLITHIC, TIGHTLY-INTEGRATED pieces of software that take care of
   storage layout, query planning, scheduling and execution… can achieve VERY
   GOOD PERFORMANCE on the types of queries for which it is designed. Moreover,
   SQL allows expressive queries WITHOUT HAVING TO WRITE CODE, making it
   accessible to GRAPHICAL TOOLS used by business analysts, such as Tableau."

   ❌ BUT: "NOT ALL KINDS OF PROCESSING CAN BE SENSIBLY EXPRESSED AS SQL
      QUERIES" — machine learning, recommendation systems, relevance ranking,
      image analysis. "These are often VERY SPECIFIC to a particular
      application… so they INEVITABLY REQUIRE WRITING CODE, NOT JUST QUERIES."

   ➜ "Having two processing models, SQL and MapReduce, WAS NOT ENOUGH: EVEN
     MORE DIFFERENT MODELS WERE NEEDED! And due to the OPENNESS of the Hadoop
     platform, it was feasible to implement a whole range of approaches, WHICH
     WOULD NOT HAVE BEEN POSSIBLE WITHIN THE CONFINES OF A MONOLITHIC MPP
     DATABASE."

   🔑 "Crucially, those various processing models can be run ON A SINGLE
     SHARED-USE CLUSTER, ALL ACCESSING THE SAME FILES ON HDFS… NOT HAVING TO
     MOVE DATA AROUND makes it a lot easier to derive value from the data."
     (HBase = OLTP-ish; Impala = MPP-ish. Neither uses MapReduce; both use HDFS.)
```

### ③ Designing for frequent faults — and the surprising real reason

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  MPP DATABASE                    │  MAPREDUCE                           │
   │  a node crash ABORTS THE ENTIRE  │  tolerates the failure of AN         │
   │  QUERY; resubmit or auto-retry.  │  INDIVIDUAL TASK, retrying at TASK   │
   │  Acceptable because queries run  │  GRANULARITY.                        │
   │  for SECONDS OR MINUTES.         │                                      │
   │  Prefers to KEEP DATA IN MEMORY  │  "VERY EAGER TO WRITE DATA TO DISK"  │
   └──────────────────────────────────┴──────────────────────────────────────┘
```

> **"But how realistic are these assumptions? In most clusters, machine failures DO occur, but THEY ARE NOT VERY FREQUENT — probably rare enough that most jobs would not experience a machine failure. IS IT REALLY WORTH INCURRING SIGNIFICANT OVERHEADS for the sake of fault tolerance?"**

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE ANSWER: GOOGLE'S MIXED-USE DATACENTERS AND PREEMPTION            ║
   ║                                                                       ║
   ║  Online production services and offline batch jobs RUN ON THE SAME    ║
   ║  MACHINES. Every task has a RESOURCE ALLOCATION enforced by           ║
   ║  CONTAINERS, and a PRIORITY. "if a higher-priority task needs more    ║
   ║  resources, LOWER-PRIORITY TASKS ON THE SAME MACHINE CAN BE           ║
   ║  TERMINATED (PREEMPTED)."                                             ║
   ║                                                                       ║
   ║  📊 THE NUMBERS: "a MapReduce task that runs for AN HOUR has an       ║
   ║     approximately 5% RISK OF BEING TERMINATED to make space for a     ║
   ║     higher-priority process. This rate is MORE THAN AN ORDER OF       ║
   ║     MAGNITUDE HIGHER than the rate of failures due to hardware        ║
   ║     issues, machine reboot or other reasons."                         ║
   ║                                                                       ║
   ║     "At this rate, if a job has 100 TASKS that run for 10 MINUTES,    ║
   ║      there is a RISK GREATER THAN 50% that at least one task will be  ║
   ║      TERMINATED BEFORE IT IS FINISHED."                               ║
   ║                                                                       ║
   ║  🔑 "And this is why MapReduce is designed to tolerate frequent       ║
   ║    unexpected task termination: IT'S NOT BECAUSE THE HARDWARE IS      ║
   ║    PARTICULARLY UNRELIABLE, IT'S BECAUSE THE FREEDOM TO ARBITRARILY   ║
   ║    TERMINATE PROCESSES ENABLES BETTER RESOURCE UTILIZATION."          ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ⚑ "Batch jobs effectively 'PICK UP THE SCRAPS UNDER THE TABLE', using any
     computing resources that remain after the high-priority processes have
     taken what they need."

   ⚠️ "Among OPEN SOURCE cluster schedulers, PREEMPTION IS LESS WIDELY USED…
      In an environment where tasks are NOT SO OFTEN TERMINATED, THE DESIGN
      DECISIONS OF MAPREDUCE MAKE LESS SENSE."
```

**This is the chapter's finest piece of detective work.** MapReduce's much-criticized eagerness to write to disk isn't a performance oversight — it's a rational response to an economic decision about datacenter utilization that most Hadoop users never shared.

---

## 12. Beyond MapReduce — dataflow engines

> *"implementing a complex processing job using the RAW MAPREDUCE APIs is actually QUITE HARD AND LABORIOUS — for instance, you would need to IMPLEMENT THE ABOVE JOIN ALGORITHMS FROM SCRATCH."*
>
> *"MapReduce is VERY ROBUST: you can use it to process almost arbitrarily large quantities of data on an unreliable multitenant system with frequent task terminations, and IT WILL STILL GET THE JOB DONE, ALBEIT SLOWLY. On the other hand, other tools are sometimes ORDERS OF MAGNITUDE FASTER."*

### The problem: materialization of intermediate state

```
   MATERIALIZATION = "to EAGERLY COMPUTE the result of some operation and to
   WRITE IT OUT, rather than computing it on demand when requested."

   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  UNIX PIPES                      │  MAPREDUCE                           │
   │  stream output to input          │  FULLY MATERIALIZE intermediate      │
   │  INCREMENTALLY, using only a     │  state to HDFS between every job     │
   │  SMALL IN-MEMORY BUFFER          │                                      │
   └──────────────────────────────────┴──────────────────────────────────────┘

   ❌ DOWNSIDE 1 — NO OVERLAP
      "A MapReduce job can ONLY START WHEN ALL TASKS IN THE PRECEDING JOBS
       HAVE COMPLETED, whereas processes connected by a Unix pipe are STARTED
       AT THE SAME TIME.
       Skew or varying load mean a job often has A FEW STRAGGLER TASKS that
       take MUCH LONGER… Having to wait until ALL of the preceding job's tasks
       have completed SLOWS DOWN THE EXECUTION OF THE WORKFLOW AS A WHOLE."

   ❌ DOWNSIDE 2 — REDUNDANT MAPPERS
      "Mappers are often REDUNDANT: they just READ BACK THE SAME FILE THAT WAS
       JUST WRITTEN BY A REDUCER… In many cases, the mapper code COULD BE PART
       OF THE PREVIOUS REDUCER."

   ⚑ WHEN MATERIALIZATION IS RIGHT: "if the output from the first job is a
     dataset that you want to PUBLISH WIDELY within your organization… allows
     LOOSE COUPLING, so that jobs don't need to know who is producing their
     input or consuming their output."
```

### 🔷 Dataflow engines: Spark, Tez, Flink

```
   "they handle AN ENTIRE WORKFLOW AS ONE JOB, rather than breaking it up into
    independent sub-jobs. Since they explicitly MODEL THE FLOW OF DATA through
    several processing stages, these systems are known as DATAFLOW ENGINES."

   Functions are called OPERATORS (not forced into map/reduce roles), and the
   engine offers SEVERAL WAYS to connect one operator's output to another's:

   ┌──────────────────────────────────────────────────────────────────────┐
   │ ① REPARTITION AND SORT by key   → enables sort-merge joins, grouping │
   │ ② REPARTITION, SKIP THE SORT    → "saves effort on PARTITIONED HASH  │
   │                                    JOINS, where the partitioning     │
   │                                    matters but ORDER IS IRRELEVANT,  │
   │                                    because building the hash table   │
   │                                    RANDOMIZES THE ORDER ANYWAY"      │
   │ ③ BROADCAST to all partitions   → enables broadcast hash joins       │
   └──────────────────────────────────────────────────────────────────────┘
```

### The six advantages over MapReduce

```
   ① "EXPENSIVE WORK SUCH AS SORTING need only be inserted WHERE IT IS
      ACTUALLY REQUIRED, rather than ALWAYS HAPPENING BY DEFAULT between every
      map and reduce stage."
   ② "AVOIDS UNNECESSARY MAP TASKS, since the work done by a mapper can often
      be INCORPORATED INTO THE PRECEDING REDUCE OPERATOR."
   ③ "Because ALL JOINS AND DATA DEPENDENCIES ARE EXPLICITLY DECLARED, the
      scheduler HAS AN OVERVIEW of what data is required where, so it can make
      LOCALITY OPTIMIZATIONS" — e.g. co-locate producer and consumer so data
      passes through A SHARED MEMORY BUFFER.
   ④ "It is usually sufficient for intermediate state to be KEPT IN MEMORY OR
      WRITTEN TO LOCAL DISK, which requires LESS I/O than writing to HDFS
      (where it must be REPLICATED to several machines)."
   ⑤ "Operators can START EXECUTING AS SOON AS THEIR INPUT IS READY; NO NEED
      TO WAIT for the entire preceding stage to finish."
   ⑥ "EXISTING JVM PROCESSES CAN BE REUSED to run new operators, which reduces
      STARTUP OVERHEADS."

   ⚑ "workflows implemented in Pig, Hive or Cascading can be SWITCHED FROM
     MAPREDUCE TO TEZ WITH A SIMPLE CONFIGURATION CHANGE, WITHOUT MODIFYING
     CODE."
```

### Fault tolerance without materialization

```
   MapReduce: intermediate state is ON HDFS = DURABLE → "if a task fails, it
   can just be RESTARTED on another machine, and READ THE SAME INPUT AGAIN."

   Spark/Flink/Tez: state is NOT on HDFS → "if a machine fails and the
   intermediate state on that machine is LOST, IT IS RECOMPUTED from other
   data that is still available."
      • SPARK uses the RDD ("resilient distributed dataset") abstraction for
        TRACKING THE ANCESTRY of data
      • FLINK CHECKPOINTS OPERATOR STATE

   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  ⚠️ RECOMPUTATION REQUIRES DETERMINISM                                 ║
   ║                                                                       ║
   ║  "If the operator is restarted and the RECOMPUTED DATA IS NOT THE     ║
   ║   SAME as the original lost data, it becomes VERY HARD for downstream ║
   ║   operators to RESOLVE THE CONTRADICTIONS… The solution is normally   ║
   ║   to KILL THE DOWNSTREAM OPERATORS AS WELL."                          ║
   ║                                                                       ║
   ║  "it is EASY FOR NON-DETERMINISTIC BEHAVIOR TO ACCIDENTALLY CREEP     ║
   ║   IN":                                                                ║
   ║     • hash table ITERATION ORDER is often unspecified                 ║
   ║     • probabilistic/statistical algorithms use RANDOM NUMBERS         ║
   ║     • ANY USE OF THE SYSTEM CLOCK or external data sources            ║
   ║  ➜ FIX: e.g. generate pseudo-random numbers using A FIXED SEED.       ║
   ╚═══════════════════════════════════════════════════════════════════════╝

   ⚑ "Recovering by recomputing is NOT ALWAYS THE RIGHT ANSWER: if the
     intermediate data is MUCH SMALLER than the source data, or if the
     computation is VERY CPU-INTENSIVE, it is probably CHEAPER TO MATERIALIZE."
```

### The one thing that can't be pipelined

> *"A SORTING OPERATION INEVITABLY NEEDS TO CONSUME ITS ENTIRE INPUT before it can produce any output — because it's possible that THE VERY LAST INPUT RECORD IS THE ONE WITH THE LOWEST KEY, and thus needs to be the very first output record. Any operator that requires sorting will thus need to ACCUMULATE STATE, at least temporarily."*

---

## 13. Graphs and iterative processing

> ⚠️ **The naming confusion, flagged by the book:** *"Dataflow engines typically arrange the operators in a job as a DIRECTED ACYCLIC GRAPH (DAG). **This is not the same as graph processing**: in dataflow engines, THE FLOW OF DATA is structured as a graph, while THE DATA ITSELF typically consists of relational-style tuples. In graph processing, THE DATA ITSELF HAS THE FORM OF A GRAPH. ANOTHER UNFORTUNATE NAMING CONFUSION!"*

```
   Many graph algorithms traverse ONE EDGE AT A TIME, "joining one vertex with
   an adjacent vertex in order to PROPAGATE SOME INFORMATION, and REPEATING
   UNTIL SOME CONDITION IS MET." (e.g. PageRank, or transitive closure.)

   ❌ "this idea of 'REPEATING UNTIL DONE' CANNOT BE EXPRESSED IN PLAIN
      MAPREDUCE, since it only performs A SINGLE PASS over the data."

   ➜ THE ITERATIVE WORKAROUND:
      ① external scheduler runs a batch process for ONE STEP
      ② when it completes, the scheduler CHECKS THE COMPLETION CONDITION
      ③ if not finished, GO BACK TO ①

   ⚠️ "implementing it with MapReduce is often VERY INEFFICIENT, because
      MapReduce DOES NOT ACCOUNT FOR THE ITERATIVE NATURE: it will ALWAYS READ
      THE ENTIRE INPUT DATASET AND PRODUCE A COMPLETELY NEW OUTPUT DATASET,
      EVEN IF ONLY A SMALL PART OF THE GRAPH HAS CHANGED."
```

### The Pregel / BSP model

```
   BULK SYNCHRONOUS PARALLEL (BSP). Implemented by Apache Giraph, Spark's
   GraphX, Flink's Gelly. Popularized by Google's PREGEL paper.

   ┌──────────────────────────────────────────────────────────────────────┐
   │  In MapReduce, MAPPERS "send a message" to a reduce call.            │
   │  In Pregel,    ONE VERTEX "sends a message" TO ANOTHER VERTEX,       │
   │                typically ALONG THE EDGES of the graph.               │
   │                                                                      │
   │      ITERATION n          │          ITERATION n+1                   │
   │   ┌───┐  msg   ┌───┐      │       ┌───┐        ┌───┐                 │
   │   │ A │──────► │ B │      │       │ A │        │ B │ ← processes the │
   │   └───┘        └───┘      │       └───┘        └───┘   message here  │
   │                           │                                          │
   │              ← the framework delivers ALL messages sent in the       │
   │                PREVIOUS iteration, in FIXED ROUNDS                   │
   └──────────────────────────────────────────────────────────────────────┘

   🔑 "The difference to MapReduce is that in the Pregel model, A VERTEX
     REMEMBERS ITS STATE IN MEMORY FROM ONE ITERATION TO THE NEXT, so the
     function ONLY NEEDS TO PROCESS NEW INCOMING MESSAGES. IF NO MESSAGES ARE
     BEING SENT IN SOME PART OF THE GRAPH, NO WORK NEEDS TO BE DONE."

   ⚑ "It's a bit similar to THE ACTOR MODEL, if you think of each vertex as an
     actor — EXCEPT that vertex state and messages ARE FAULT-TOLERANT AND
     DURABLE, and communication proceeds IN FIXED ROUNDS. ACTORS NORMALLY HAVE
     NO SUCH TIMING GUARANTEE."
```

### Fault tolerance and the parallelism problem

```
   FAULT TOLERANCE: "PERIODICALLY CHECKPOINTING the state of all vertices at
   the end of an iteration… If a node fails, the simplest solution is to ROLL
   BACK THE ENTIRE GRAPH COMPUTATION TO THE LAST CHECKPOINT."
   "Even though the underlying network may DROP, DUPLICATE OR ARBITRARILY
    DELAY messages, Pregel implementations guarantee messages are PROCESSED
    EXACTLY ONCE at their destination vertex in the following iteration."

   ⚠️ THE PARALLELISM PROBLEM — and it's serious:
   "Ideally the graph would be partitioned such that vertices are CO-LOCATED
    if they need to communicate a lot. HOWEVER, FINDING SUCH AN OPTIMIZED
    PARTITIONING IS HARD — in practice, the graph is OFTEN SIMPLY PARTITIONED
    BY HASH OF VERTEX ID, MAKING NO ATTEMPT TO GROUP RELATED VERTICES."

   ➜ "graph algorithms often have A LOT OF CROSS-MACHINE COMMUNICATION
     OVERHEAD, and the intermediate state (messages) IS OFTEN BIGGER THAN THE
     ORIGINAL GRAPH."

   🔑 THE BLUNT ADVICE: "if your graph CAN FIT IN MEMORY ON A SINGLE COMPUTER,
     it's QUITE LIKELY THAT A SINGLE-MACHINE (MAYBE EVEN SINGLE-THREADED)
     ALGORITHM WILL OUTPERFORM A DISTRIBUTED BATCH PROCESS. Even if the graph
     is bigger than memory but fits on the DISKS of a single computer,
     single-machine processing using GRAPHCHI is a viable option."
     ← the "Scalability! But at what COST?" argument from Part II, again
```

---

## 14. High-level APIs and the move toward declarative

> *"As the problem of PHYSICALLY OPERATING batch processes at such scale has been considered MORE OR LESS SOLVED, attention has turned to other areas: IMPROVING THE PROGRAMMING MODEL, improving efficiency, and BROADENING THE SET OF PROBLEMS these technologies can solve."*

```
   Hive, Pig, Cascading, Crunch → then Spark and Flink's own dataflow APIs,
   "often taking inspiration from FLUMEJAVA."

   These use RELATIONAL-STYLE BUILDING BLOCKS: joining on a field, grouping by
   key, filtering, aggregating. "Internally, these are implemented using the
   VARIOUS JOIN AND GROUPING ALGORITHMS THAT WE DISCUSSED."

   💡 AND THEY ENABLE INTERACTIVE USE: "you write analysis code INCREMENTALLY
      IN A SHELL, and RUN IT FREQUENTLY to observe what it is doing…
      REMINISCENT OF THE UNIX PHILOSOPHY."
```

### Why declarative joins matter

```
   "the framework can ANALYZE THE PROPERTIES OF THE JOIN INPUTS, and
    AUTOMATICALLY DECIDE WHICH OF THE AFOREMENTIONED JOIN ALGORITHMS would be
    most suitable. Hive, Spark and Flink have COST-BASED QUERY OPTIMIZERS that
    can do this, and even CHANGE THE ORDER OF JOINS so that the amount of
    intermediate state is MINIMIZED."

   ➜ "it is NICE NOT TO HAVE TO UNDERSTAND AND REMEMBER ALL THE VARIOUS JOIN
     ALGORITHMS we discussed in this chapter."    ← Chapter 2's argument, again
```

### ⚖️ But MapReduce's callback model has a real advantage

```
   ┌──────────────────────────────────┬──────────────────────────────────────┐
   │  SQL / FULLY DECLARATIVE         │  MAPREDUCE CALLBACKS                 │
   ├──────────────────────────────────┼──────────────────────────────────────┤
   │  optimizer picks the algorithm;  │  "that function is FREE TO CALL      │
   │  column pruning; VECTORIZED      │   ARBITRARY CODE… you can DRAW UPON  │
   │  EXECUTION (Ch.3)                │   A LARGE ECOSYSTEM OF EXISTING      │
   │                                  │   LIBRARIES to do things like        │
   │  Spark generates JVM BYTECODE;   │   PARSING, NATURAL LANGUAGE          │
   │  Impala uses LLVM to GENERATE    │   ANALYSIS, IMAGE ANALYSIS, and      │
   │  NATIVE CODE for inner loops     │   NUMERICAL OR STATISTICAL           │
   │                                  │   ALGORITHMS."                       │
   │                                  │                                      │
   │                                  │  ⚑ databases DO have user-defined    │
   │                                  │    functions, but they are "often    │
   │                                  │    CUMBERSOME, and NOT WELL          │
   │                                  │    INTEGRATED WITH PACKAGE MANAGERS  │
   │                                  │    (Maven, npm, Rubygems)."          │
   └──────────────────────────────────┴──────────────────────────────────────┘

   🔑 THE CONVERGENCE: "By incorporating declarative aspects in their
     high-level APIs, and having query optimizers… BATCH PROCESSING FRAMEWORKS
     BEGIN TO LOOK MORE LIKE MPP DATABASES (and can achieve comparable
     performance). At the same time, by having the extensibility of being able
     to RUN ARBITRARY CODE and READ DATA IN ARBITRARY FORMATS, THEY RETAIN
     THEIR FLEXIBILITY ADVANTAGE."

   AND FROM THE OTHER SIDE: "as MPP databases become MORE PROGRAMMABLE AND
   FLEXIBLE, THE TWO ARE BEGINNING TO LOOK MORE ALIKE: in the end, THEY ARE
   ALL JUST SYSTEMS FOR STORING AND PROCESSING DATA."
```

---

## 15. Chapter Summary

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  THE DESIGN PRINCIPLES, INHERITED FROM UNIX                           ║
   ║  • inputs are IMMUTABLE                                               ║
   ║  • outputs are intended to BECOME THE INPUT TO ANOTHER (AS YET        ║
   ║    UNKNOWN) PROGRAM                                                   ║
   ║  • complex problems are solved by COMPOSING SMALL TOOLS THAT "DO ONE  ║
   ║    THING WELL"                                                        ║
   ║                                                                       ║
   ║  THE UNIFORM INTERFACE:  Unix → FILES AND PIPES                       ║
   ║                     MapReduce → A DISTRIBUTED FILESYSTEM              ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### The two problems distributed batch frameworks must solve

```
   ① PARTITIONING
      "mappers are partitioned according to INPUT FILE BLOCKS. The output is
       RE-PARTITIONED, SORTED AND MERGED into a configurable number of reducer
       partitions. The purpose is to BRING ALL THE RELATED DATA TOGETHER IN
       THE SAME PLACE."

   ② FAULT TOLERANCE
      "MapReduce FREQUENTLY WRITES TO DISK, which makes it EASY to recover
       from an individual failed task, BUT WHICH SLOWS DOWN EXECUTION IN THE
       FAILURE-FREE CASE. Dataflow engines perform LESS MATERIALIZATION and
       keep MORE IN MEMORY, which means they need to RECOMPUTE MORE DATA if a
       node fails. DETERMINISTIC OPERATORS REDUCE THE AMOUNT that needs to be
       recomputed."
```

### The three join algorithms

| Join | Requires | How it works |
|---|---|---|
| **Sort-merge** | nothing | Mappers extract the key; partitioning, sorting and merging bring all same-key records to one reduce call |
| **Broadcast hash** | one input **small enough to fit in memory** | Load the small input into a hash table in *every* mapper; scan the large input and probe |
| **Partitioned hash** | both inputs **partitioned identically** (same key, hash function, partition count) | Apply the hash-table approach independently per partition |

### 🔑 The restricted programming model, and what it buys

> *"Distributed batch processing engines have a DELIBERATELY RESTRICTED PROGRAMMING MODEL: callback functions are assumed to be STATELESS, and to have NO EXTERNALLY VISIBLE SIDE-EFFECTS besides their designated output. This allows the framework to HIDE SOME OF THE HARD DISTRIBUTED SYSTEMS PROBLEMS: in the face of crashes and network issues, TASKS CAN BE RETRIED SAFELY, and the output from any failed tasks is DISCARDED. If several tasks for a partition succeed, ONLY ONE OF THEM ACTUALLY MAKES ITS OUTPUT VISIBLE."*
>
> *"Therefore, your code DOES NOT NEED TO WORRY ABOUT IMPLEMENTING FAULT TOLERANCE MECHANISMS: the framework can guarantee that THE FINAL OUTPUT IS THE SAME AS IF NO FAULTS HAD OCCURRED. **These reliable semantics are MUCH STRONGER than what you usually have in online services** that handle user requests."*

### And the hook into Chapter 11

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  "the input data is BOUNDED: it has A KNOWN, FIXED SIZE — for example ║
   ║   a set of log files at some point in time, or a snapshot of a        ║
   ║   database. Because it is bounded, A JOB KNOWS WHEN IT HAS FINISHED   ║
   ║   READING THE ENTIRE INPUT, and so A JOB EVENTUALLY COMPLETES."       ║
   ║                                                                       ║
   ║  "In the next chapter, the input is UNBOUNDED — you still have a job, ║
   ║   but its inputs are NEVER-ENDING STREAMS OF DATA. In this case, A    ║
   ║   JOB IS NEVER COMPLETE, because at any time THERE MAY STILL BE MORE  ║
   ║   WORK COMING IN."                                                    ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

---
---

# 16. 🎁 WHAT'S CHANGED SINCE 2017

This chapter has aged more visibly than any other in the book — not because the *principles* are wrong, but because **almost every product named in it has been displaced.** The book half-predicts this ("the importance of MapReduce is now declining"), but the extent is striking.

## 16.1 MapReduce and Hadoop are effectively dead

```
   • GOOGLE retired MapReduce internally long ago, replaced by FLUME/DATAFLOW
     (which became APACHE BEAM) and then by the systems behind BigQuery.
   • HADOOP MAPREDUCE is legacy. Nobody starts new projects on it.
   • CLOUDERA AND HORTONWORKS MERGED (2019) — the two great Hadoop
     distributors combining was the clearest market signal available. MapR was
     sold off to HPE in 2019.
   • APACHE PIG, CASCADING, CRUNCH, GIRAPH: all essentially dormant. TEZ
     survives mainly as a Hive execution backend.
   • HDFS survives in on-premise installations, but for new work it has been
     comprehensively displaced by OBJECT STORAGE (S3, GCS, Azure Blob, R2).

   🔑 THE ARCHITECTURAL REASON: Hadoop's founding bet was DATA LOCALITY —
     "putting the computation near the data," because network bandwidth was
     the bottleneck. Datacenter networking got dramatically faster (25/100
     GbE became routine), while object storage got dramatically cheaper and
     infinitely elastic. That flipped the economics:

     2017 THINKING              2026 THINKING
     ─────────────              ─────────────
     couple storage + compute   SEPARATE storage and compute
     (move code to the data)    (scale them INDEPENDENTLY; spin compute
                                 up and down; pay only for what you use)

     The book's own footnote anticipates this — it notes that with erasure
     coding "THE LOCALITY ADVANTAGE IS LOST." That turned out to be the
     future, not an edge case.
```

## 16.2 Spark won, and then got competition

```
   SPARK became the default general-purpose engine, largely because:
     • DataFrame/Dataset APIs + SPARK SQL made the declarative move the book
       describes at the end of the chapter
     • CATALYST optimizer + TUNGSTEN code generation delivered on "batch
       engines begin to look more like MPP databases"
     • ADAPTIVE QUERY EXECUTION (Spark 3.0, 2020) re-optimizes MID-QUERY using
       actual runtime statistics — including AUTOMATIC SKEW JOIN HANDLING,
       which is the manual technique from §8 promoted into the engine
     • one engine for batch, streaming, SQL and ML

   🆕 NEW COMPETITION for mid-sized work:
     • DUCKDB — an in-process analytical engine. "Scalability! But at what
       COST?" (cited twice in this chapter) became a product: for datasets up
       to hundreds of GB, a single machine with DuckDB routinely beats a
       cluster. This is the chapter's own single-machine advice, vindicated.
     • POLARS — same idea for DataFrames, in Rust.
     • CLICKHOUSE — column store fast enough to make many batch jobs
       unnecessary.
     • VELOX / DATAFUSION / ARROW — shared, vectorized execution kernels that
       several engines now build on.
```

## 16.3 The lakehouse: the book's "data lake" idea, fixed

```
   The chapter praises indiscriminate dumping (the SUSHI PRINCIPLE) but the
   industry learned the failure mode: DATA LAKES BECAME DATA SWAMPS — no
   schemas, no transactions, no way to update a row, no time travel.

   🆕 TABLE FORMATS solved it by adding a metadata layer over Parquet files:

   ┌─────────────────────────────────────────────────────────────────────┐
   │  APACHE ICEBERG  (now the de facto winner — Snowflake, AWS,          │
   │                   Databricks after the Tabular acquisition, Google)  │
   │  DELTA LAKE      (Databricks)                                        │
   │  APACHE HUDI     (Uber lineage, strong on upserts/CDC)               │
   │                                                                      │
   │  WHAT THEY ADD TO A PILE OF FILES IN OBJECT STORAGE:                │
   │    ✅ ACID TRANSACTIONS (atomic commit via a metadata pointer swap —  │
   │       structurally the SAME TRICK as Voldemort's atomic switchover   │
   │       described in §10!)                                             │
   │    ✅ TIME TRAVEL / snapshot isolation over a table                   │
   │    ✅ SCHEMA EVOLUTION using FIELD IDs (the Protobuf tag idea, Ch.4)  │
   │    ✅ row-level UPDATE/DELETE/MERGE (needed for GDPR deletion)        │
   │    ✅ hidden partitioning, compaction, statistics for pruning         │
   └─────────────────────────────────────────────────────────────────────┘

   ➜ THE LAKEHOUSE is explicitly the reconciliation the chapter's final
     sentence predicts: "in the end, THEY ARE ALL JUST SYSTEMS FOR STORING AND
     PROCESSING DATA." Warehouses gained open formats and arbitrary code;
     lakes gained transactions and SQL.
```

## 16.4 Orchestration matured beyond Oozie

```
   The book lists "Oozie, Azkaban, Luigi, Airflow, Pinball." What happened:

   • AIRFLOW won decisively and is now an Apache top-level project with a huge
     ecosystem; Airflow 2.x rewrote the scheduler and added the TaskFlow API.
   • DAGSTER and PREFECT emerged as modern alternatives with a notable
     philosophical difference: DAGSTER models DATA ASSETS (what you're
     producing) rather than TASKS (what you're running) — which is
     essentially the book's own SYSTEM-OF-RECORD-vs-DERIVED-DATA framing from
     the Part III introduction, made into a scheduling primitive.
   • DBT changed who writes batch jobs. Analysts write SELECT statements; dbt
     handles dependency ordering, materialization, testing and documentation.
     "Analytics engineering" became a job title. This is arguably the biggest
     practical change in batch processing since the book was written.
```

## 16.5 Batch and stream converged (as Chapter 11 will foreshadow)

```
   🆕 APACHE BEAM formalized "write once, run as batch or streaming" with a
      unified model (and became Google Cloud Dataflow's API).
   🆕 FLINK grew from the batch engine described here into the dominant
      STREAM processor, with batch as a special case of streaming
      (bounded streams).
   🆕 SPARK STRUCTURED STREAMING took the opposite route: streaming as
      repeated micro-batches over an unbounded table.
   🆕 INCREMENTAL VIEW MAINTENANCE (Materialize, Feldera/DBSP, RisingWave)
      attacks the chapter's own complaint that iterative MapReduce "will
      ALWAYS READ THE ENTIRE INPUT DATASET… EVEN IF ONLY A SMALL PART HAS
      CHANGED." These systems compute exactly the delta.
```

## 16.6 What has aged perfectly

```
   ✅ THE UNIX PHILOSOPHY section. Timeless, and arguably more relevant now
      that people compose pipelines from cloud services.
   ✅ HUMAN FAULT TOLERANCE — immutable inputs let you fix a bug by re-running.
      This is now the core justification for the lakehouse, for event sourcing,
      and for "reproducible pipelines" as a discipline.
   ✅ THE JOIN ALGORITHMS. Broadcast, partitioned hash and sort-merge are
      exactly what Spark, Flink, Trino, Snowflake and DuckDB implement today.
      Learning them from this chapter still pays off directly.
   ✅ THE SKEW DISCUSSION. Still the #1 practical cause of a slow Spark job.
   ✅ "if your graph fits in memory on one machine, a single-threaded algorithm
      will probably beat a distributed one." More true every year.
   ✅ SEPARATION OF LOGIC AND WIRING; the value of declarative interfaces;
      the observation that batch engines and MPP databases are converging —
      all confirmed.
```

## 16.7 A 2026 decision table

```
   ┌────────────────────────────────────┬────────────────────────────────────┐
   │  YOUR SITUATION                    │  REACH FOR                         │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  data fits on one machine (< ~1 TB)│  DUCKDB or POLARS. Seriously.      │
   │                                    │  Don't start a cluster.            │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  genuinely large-scale batch       │  SPARK on object storage, over     │
   │                                    │  ICEBERG or DELTA tables           │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  SQL transformations in a warehouse│  DBT (+ Snowflake/BigQuery/         │
   │                                    │  Databricks/DuckDB)                │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  interactive query over a lake     │  TRINO, DuckDB, or the warehouse's │
   │                                    │  own engine                        │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  scheduling and dependencies       │  AIRFLOW (ubiquitous) or DAGSTER   │
   │                                    │  (asset-oriented)                  │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  arbitrary code over big data      │  Spark, Ray (for ML), or Beam       │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  graph analytics                   │  try SINGLE-MACHINE FIRST          │
   │                                    │  (NetworkX/igraph/cuGraph)         │
   ├────────────────────────────────────┼────────────────────────────────────┤
   │  a new Hadoop MapReduce job        │  don't                             │
   └────────────────────────────────────┴────────────────────────────────────┘
```

*(As with earlier chapters: this reflects developments through my knowledge cutoff, and this area moves fast — treat specific products and claims as a starting point to verify.)*

---

# 17. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA PART III + CH.10 — BATCH PROCESSING                                     ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  PART III PREMISE: no ONE database satisfies all needs → you integrate many.  ║
║  SYSTEM OF RECORD = authoritative, each fact ONCE, normalized.                ║
║  DERIVED DATA = transformed from a source; IF LOST, RE-CREATABLE. Caches,     ║
║     indexes, materialized views, denormalized copies, ML models.              ║
║  "A database is JUST A TOOL — the distinction depends on HOW YOU USE IT."     ║
║                                                                               ║
║  THREE SYSTEM TYPES: SERVICES (metric = RESPONSE TIME) · BATCH (metric =      ║
║  THROUGHPUT, input BOUNDED, job COMPLETES) · STREAM (input UNBOUNDED, ch.11). ║
║  Batch is ancient: Hollerith machines, 1890 US Census; IBM card sorters 1940s.║
║                                                                               ║
║  ── UNIX ──────────────────────────────────────────────────────────────────── ║
║  awk|sort|uniq -c|sort -rn|head processes GIGABYTES IN SECONDS.               ║
║  RUBY HASH TABLE vs UNIX SORT: working set = DISTINCT KEYS. If it fits in RAM,║
║  hash table is fine; if not, SORTING SPILLS TO DISK and merges (= SSTables,   ║
║  ch.3). GNU sort does this AND parallelizes automatically.                    ║
║  PHILOSOPHY (1978): ① do ONE THING WELL ② expect output to become ANOTHER      ║
║  PROGRAM'S INPUT ③ build to be TRIED EARLY, throw away clumsy parts ④ build   ║
║  TOOLS. "Sounds remarkably like Agile and DevOps. Little has changed in four  ║
║  decades." `sort` COULD have absorbed `uniq` — THEY RESISTED THE TEMPTATION.  ║
║  COMPOSABILITY NEEDS: ① a UNIFORM INTERFACE (the FILE DESCRIPTOR — files,     ║
║  pipes, sockets, devices all the same) ② SEPARATION OF LOGIC AND WIRING       ║
║  (stdin/stdout = loose coupling, inversion of control) ③ TRANSPARENCY         ║
║  (immutable inputs, stop the pipe anywhere, restart from a stage).            ║
║  LIMIT: ONE MACHINE ONLY.                                                     ║
║                                                                               ║
║  ── MAPREDUCE ─────────────────────────────────────────────────────────────── ║
║  = Unix tools, distributed. Job ≈ process; HDFS ≈ stdin/stdout.               ║
║  HDFS: shared-nothing, daemon per machine, central NAMENODE, replication or   ║
║  ERASURE CODING. Tens of thousands of machines, hundreds of petabytes.        ║
║  FOUR STEPS: records → MAP extracts (k,v) → SORT (IMPLICIT — you never write  ║
║  it, and it is "ARGUABLY THE MOST IMPORTANT ASPECT") → REDUCE over same-key.  ║
║  MAPPER = prepare data for sorting. REDUCER = process sorted data.            ║
║  # map tasks = # input BLOCKS. # reduce tasks = CONFIGURED. Key→reducer by    ║
║  HASH. PUT THE COMPUTATION NEAR THE DATA (ship the JAR, not the data).        ║
║  THE SHUFFLE = partition by reducer + sort locally + reducers FETCH + merge.  ║
║  ("no randomness despite the name.")                                          ║
║  WORKFLOWS: chained BY DIRECTORY NAME; framework sees INDEPENDENT JOBS.       ║
║  Schedulers: Oozie/Azkaban/Luigi/AIRFLOW/Pinball. 50–100 jobs is normal.      ║
║                                                                               ║
║  ── JOINS ─────────────────────────────────────────────────────────────────── ║
║  No indexes → FULL TABLE SCAN, which is FINE for analytics.                   ║
║  ❌ Querying a remote DB per record: slow, overwhelms the DB, and makes the    ║
║     job NON-DETERMINISTIC. → COPY the DB into HDFS instead.                   ║
║  SORT-MERGE (reduce-side): both mappers emit the JOIN KEY; sorting makes all  ║
║     related records ADJACENT in the reducer. SECONDARY SORT guarantees the    ║
║     user record arrives FIRST → reducer holds ONE record in memory, NO        ║
║     NETWORK CALLS.                                                            ║
║  🔑 MAPPERS "SEND MESSAGES" TO REDUCERS — the key is the ADDRESS. This        ║
║     SEPARATES NETWORK COMMUNICATION FROM APPLICATION LOGIC.                   ║
║  GROUP BY works identically. SESSIONIZATION groups by session cookie.         ║
║  SKEW / LINCHPIN OBJECTS (celebrities): one slow reducer BLOCKS THE WHOLE JOB.║
║     FIX: SKEW JOIN (hot key → RANDOM reducer; replicate the other side to ALL)║
║     or TWO-STAGE GROUPING (random partial aggregates, then combine).          ║
║  MAP-SIDE JOINS (no reducers, no sorting):                                    ║
║     BROADCAST HASH — small side fits in memory, loaded into EVERY mapper      ║
║        (Pig "replicated join", Hive "MapJoin"). Or a read-only disk index     ║
║        relying on the OS PAGE CACHE.                                          ║
║     PARTITIONED HASH — both sides partitioned IDENTICALLY (Hive "bucketed").  ║
║     MAP-SIDE MERGE — both partitioned AND sorted; mapper does the merge.      ║
║  ⚠️ Output partitioning DIFFERS: reduce-side → by JOIN KEY; map-side → like    ║
║     the LARGE INPUT. Physical layout metadata lives in HCatalog/Hive metastore║
║                                                                               ║
║  ── OUTPUTS ───────────────────────────────────────────────────────────────── ║
║  SEARCH INDEXES (Google's original use, 5–10 jobs) · ML MODELS · KV STORES.   ║
║  ❌ NEVER write to the production DB from inside a job: slow, overwhelms it,   ║
║     and creates EXTERNALLY VISIBLE SIDE EFFECTS that break all-or-nothing.    ║
║  ✅ BUILD A NEW DATABASE AS FILES, then BULK LOAD (Voldemort, Terrapin,        ║
║     ElephantDB, HBase). Immutable → NO WAL NEEDED. Voldemort serves old files ║
║     during the copy, then ATOMICALLY SWITCHES — and can switch BACK.          ║
║  🔑 HUMAN FAULT TOLERANCE: buggy code? ROLL BACK AND RE-RUN. "Databases with   ║
║     read/write transactions DO NOT HAVE THIS PROPERTY." Immutability also     ║
║     makes automatic retries SAFE, and enables monitoring jobs over the same   ║
║     inputs. Avro/Parquet remove Unix's untyped-text parsing tax.              ║
║                                                                               ║
║  ── MAPREDUCE vs MPP ──────────────────────────────────────────────────────── ║
║  MPP = parallel analytic SQL. MapReduce = a general-purpose OS for arbitrary  ║
║  programs. STORAGE DIVERSITY: dump raw, interpret later (SCHEMA-ON-READ, DATA ║
║  LAKE, 🍣 THE SUSHI PRINCIPLE). PROCESSING DIVERSITY: SQL can't express ML,    ║
║  image analysis, relevance ranking → many models on ONE SHARED CLUSTER.       ║
║  FAULT DESIGN: MPP aborts the whole query and prefers memory; MapReduce       ║
║  retries PER TASK and writes to disk eagerly.                                 ║
║  🔑 THE REAL REASON: GOOGLE'S MIXED-USE DATACENTERS PREEMPT low-priority       ║
║     tasks. A 1-HOUR TASK HAS A ~5% CHANCE OF BEING KILLED — an ORDER OF       ║
║     MAGNITUDE above hardware failure. 100 tasks × 10 min → >50% chance one    ║
║     dies. "NOT BECAUSE THE HARDWARE IS UNRELIABLE, BUT BECAUSE THE FREEDOM TO ║
║     TERMINATE ENABLES BETTER UTILIZATION." Open source schedulers rarely      ║
║     preempt → MapReduce's design makes LESS SENSE there.                      ║
║                                                                               ║
║  ── DATAFLOW ENGINES (Spark, Tez, Flink) ──────────────────────────────────── ║
║  MapReduce MATERIALIZES all intermediate state to HDFS → ❌ can't start until  ║
║  ALL preceding tasks finish (STRAGGLERS block everything) and ❌ mappers often ║
║  just re-read what a reducer wrote.                                           ║
║  DATAFLOW = ONE JOB for the WHOLE WORKFLOW, arbitrary OPERATORS, connected by ║
║  repartition+sort / repartition-only / broadcast.                             ║
║  WINS: sort ONLY WHERE NEEDED · no redundant mappers · locality optimizations ║
║  from the declared DAG · state in memory or local disk · operators START      ║
║  EARLY · JVM reuse. Same code runs on either engine (config change).          ║
║  FAULT TOLERANCE by RECOMPUTATION (Spark RDD lineage, Flink checkpoints) —    ║
║  ⚠️ REQUIRES DETERMINISM. Non-determinism creeps in via hash iteration order,  ║
║  random numbers, the CLOCK, external sources. Fix with FIXED SEEDS.           ║
║  Recompute isn't always right: if intermediate data is small or the compute   ║
║  is expensive, MATERIALIZE. SORTING can never be fully pipelined.             ║
║                                                                               ║
║  ── GRAPHS ────────────────────────────────────────────────────────────────── ║
║  ⚠️ A dataflow DAG ≠ GRAPH PROCESSING. "Another unfortunate naming confusion!" ║
║  "Repeat until done" CAN'T be expressed in plain MapReduce → external         ║
║  iteration, which re-reads EVERYTHING every round.                            ║
║  PREGEL / BSP (Giraph, GraphX, Gelly): vertices SEND MESSAGES along edges;    ║
║  a vertex REMEMBERS STATE BETWEEN ITERATIONS, so idle regions do NO WORK.     ║
║  Like ACTORS but with DURABLE state and FIXED ROUNDS. Checkpoint per          ║
║  iteration. ⚠️ Partitioning is usually just HASH OF VERTEX ID → huge crosstalk;║
║  messages often EXCEED THE GRAPH SIZE. ➜ IF IT FITS ON ONE MACHINE, USE ONE.  ║
║                                                                               ║
║  ── HIGH-LEVEL APIs ───────────────────────────────────────────────────────── ║
║  Hive/Pig/Cascading/Crunch → Spark & Flink DataFrame APIs (from FlumeJava).   ║
║  DECLARATIVE JOINS let COST-BASED OPTIMIZERS pick the algorithm and reorder   ║
║  joins. Declarative filters/projections enable COLUMN PRUNING and VECTORIZED  ║
║  EXECUTION (Spark → JVM bytecode; Impala → LLVM native code).                 ║
║  BUT callbacks keep the ecosystem advantage (parsing, NLP, image, stats       ║
║  libraries + real package managers) that database UDFs lack.                  ║
║  ➜ BATCH ENGINES AND MPP DATABASES ARE CONVERGING. "In the end, they are all  ║
║    just systems for storing and processing data."                             ║
║                                                                               ║
║  SUMMARY: the two hard problems are PARTITIONING and FAULT TOLERANCE. The     ║
║  restricted model (stateless callbacks, no side effects) is what lets the     ║
║  framework retry safely and discard failed output — GUARANTEES STRONGER THAN  ║
║  ANY ONLINE SERVICE. Input is BOUNDED, so THE JOB COMPLETES. → Ch.11: unbounded║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 18. ✅ Test yourself

1. **What distinguishes a system of record from a derived data system, and why does the distinction matter?**
   → A system of record holds the authoritative copy; derived data is computed from it and can always be rebuilt if lost. The distinction isn't a property of the tool — it's a property of how you use it — and making it explicit clarifies which parts of your architecture have which inputs and outputs.

2. **The Unix pipeline has no hash table, yet it handles datasets the Ruby script can't. Why?**
   → The Ruby version needs the *distinct keys* to fit in memory. The pipeline relies on sorting, which spills to disk and merges — the same sequential-I/O trick as SSTables. GNU `sort` also parallelizes across cores automatically.

3. **Which step of MapReduce do you never write, and why is it the most important?**
   → The sort between map and reduce. It's what makes all records with the same key adjacent, which is what makes joins, grouping and aggregation possible with a reducer that holds almost no state.

4. **Why is querying a production database from inside a mapper a bad idea, beyond the obvious performance problem?**
   → It makes the job non-deterministic, because the remote data can change between runs. That breaks reproducibility and, in a dataflow engine, breaks fault recovery by recomputation.

5. **When can you use a broadcast hash join instead of a sort-merge join?**
   → When one input is small enough to fit in memory (or on local disk as a read-only index) in every mapper. You then skip the reducers, the shuffle and the sort entirely — each mapper scans its block of the large input and probes its own copy of the small one.

6. **A celebrity user makes one reducer take ten times as long as the others. What's the fix?**
   → A skew join: route records for the hot key to a *random* reducer rather than a hashed one, and replicate the matching records from the other input to all reducers. For grouping, use two stages — random partial aggregates, then combine.

7. **What is "human fault tolerance" and why can't a normal database offer it?**
   → Because batch inputs are immutable and outputs are replaced wholesale, a bug means you roll back the code, re-run, and the output is correct again. A database with read/write transactions has already destroyed the old state — rolling back the code doesn't un-write the bad data.

8. **Why does MapReduce write so eagerly to disk, when machine failures are actually rare?**
   → Because on Google's mixed-use clusters, low-priority batch tasks get *preempted* to free resources for production services. A one-hour task has roughly a 5% chance of being killed — an order of magnitude above hardware failure. The design targets deliberate termination, not unreliable hardware.

9. **Dataflow engines avoid materializing intermediate state. What do they give up, and what must you guarantee in return?**
   → They give up durable intermediate results, so lost state must be *recomputed* from lineage. That only works if operators are deterministic — which means no unseeded randomness, no clock reads, no external lookups, and no reliance on hash iteration order.

10. **Your graph fits in 200 GB and you have a 20-node cluster. What does the chapter advise?**
    → Try a single machine first. Graph partitioning is usually just a hash of vertex ID, so cross-machine message traffic often exceeds the size of the graph itself. A single-threaded algorithm, or a disk-based one like GraphChi, will frequently beat the cluster.

---

*All quoted material, figures and examples attributed to the book are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Part III introduction and Chapter 10. Diagrams have been redrawn in ASCII from the book's originals. Section 16 (post-2017 developments) is supplementary material I added; that landscape moves quickly, so treat specific products and claims there as a starting point worth verifying.*
