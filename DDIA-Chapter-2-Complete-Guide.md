# DDIA — Chapter 2: Data Models and Query Languages
### Complete study guide — theory, every diagram redrawn, and runnable code

> *"The limits of my language mean the limits of my world."*
> — Ludwig Wittgenstein, *Tractatus Logico-Philosophicus* (1922)

That epigraph is the thesis of the chapter. A data model doesn't just decide how you *store* things — it decides which questions are easy to ask, which are awkward, and which never occur to you at all.

---

## 0. The map of this chapter

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   PART A — THREE DATA MODELS                                             │
│   ─────────────────────────────                                          │
│                                                                          │
│      RELATIONAL  ◄──────►  DOCUMENT  ◄──────►  GRAPH                     │
│      (tables)              (trees)             (anything → anything)     │
│                                                                          │
│      good at              good at              good at                   │
│      many-to-many         one-to-many          many-to-many              │
│      + joins              + locality           + arbitrary depth         │
│                                                                          │
│   PART B — THEIR QUERY LANGUAGES                                         │
│   ──────────────────────────────                                         │
│                                                                          │
│      SQL          MapReduce      Cypher                                  │
│      relational   aggregation    SPARQL                                  │
│      algebra      pipeline       Datalog                                 │
│                                                                          │
│      running theme:  DECLARATIVE  >  IMPERATIVE                          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**The single most useful takeaway:** the right model is decided by *the shape of your relationships*, not by hype.

```
   one-to-many only    ─────►   DOCUMENT
   some many-to-many   ─────►   RELATIONAL
   many-to-many        ─────►   GRAPH
   everywhere
```

---

## 1. Data models are layers of abstraction

> **Data models are perhaps the most important part of developing software, because they have such a profound effect: not only on how the software is written, but also how we think about the problem we are solving.**

Most applications layer one data model on another. For each layer the key question is: **how is it represented in terms of the next-lower layer?**

```
   ┌───────────────────────────────────────────────────────────────────┐
   │  LAYER 1 — THE REAL WORLD                                         │
   │  people · organizations · goods · actions · money flows · sensors │
   └───────────────────────────────┬───────────────────────────────────┘
                                   │  application developer models it as…
                                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │  LAYER 2 — OBJECTS / DATA STRUCTURES + APIs                       │
   │  usually SPECIFIC to your application                             │
   └───────────────────────────────┬───────────────────────────────────┘
                                   │  to store them, express in a…
                                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │  LAYER 3 — GENERAL-PURPOSE DATA MODEL       ◄── THIS CHAPTER      │
   │  JSON / XML documents · relational tables · graph model           │
   └───────────────────────────────┬───────────────────────────────────┘
                                   │  database engineers represent it as…
                                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │  LAYER 4 — BYTES in memory, on disk, on a network  ◄── CHAPTER 3  │
   │  in a form that allows querying, searching, manipulating          │
   └───────────────────────────────┬───────────────────────────────────┘
                                   │  hardware engineers represent it as…
                                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │  LAYER 5 — electrical currents · pulses of light · magnetic fields│
   └───────────────────────────────────────────────────────────────────┘
```

**Why layering works:** each layer **hides the complexity of the layers below by providing a clean data model.** That's what lets database vendors and application developers work together effectively without either needing to understand the other's job.

**The catch:**

> **Every data model embodies assumptions about how it is going to be used.** Some kinds of usage are easy and some are not supported; some operations are fast and some perform badly; some data transformations feel natural and some are awkward.

This chapter lives at **Layer 3**. Chapter 3 handles Layer 4.

---

# PART A — RELATIONAL MODEL vs DOCUMENT MODEL

## 2. Where the relational model came from

**Edgar Codd, 1970:** data is organized into **relations** (SQL: *tables*), where each relation is an **unordered collection of tuples** (rows).

It was a *theoretical proposal*, and many people at the time doubted it could be implemented efficiently. By the mid-1980s, RDBMSs and SQL had become the default tool for anyone storing data with regular structure — and **that dominance lasted 25–30 years, an eternity in computing.**

**Its roots:** business data processing on 1960s–70s mainframes. Mundane by today's standards:
- **Transaction processing** — sales, bank transactions, airline reservations, warehouse stock-keeping
- **Batch processing** — customer invoicing, payroll, reporting

**The goal that mattered:** other databases of the time **forced developers to think about the internal representation of the data.** The relational model's purpose was to **hide that implementation detail behind a cleaner interface.**

### The graveyard of challengers

```
   1970s ─────────────────────────────────────────────────────────► today

   ┌──────────────┐
   │ HIERARCHICAL │──┐
   │  (IMS, 1968) │  │
   └──────────────┘  │
                     ├──►  ┌────────────┐
   ┌──────────────┐  │     │ RELATIONAL │ ══════════════════════════►  still here
   │   NETWORK    │──┘     │ (SQL)      │
   │  (CODASYL)   │        └────────────┘
   └──────────────┘              ▲
                                 │
   ┌──────────────┐              │
   │ OBJECT DBs   │──────────────┤   late 1980s–early 90s: came and went
   └──────────────┘              │
   ┌──────────────┐              │
   │  XML DBs     │──────────────┤   early 2000s: niche adoption only
   └──────────────┘              │
   ┌──────────────┐              │
   │    NoSQL     │──────────────┘   2010s: the current challenger
   └──────────────┘

   "Each competitor generated a lot of hype in its time, but it never lasted."
```

And remarkably, relational databases **generalized very well** beyond business data processing — online publishing, discussion, social networking, e-commerce, games, SaaS productivity apps. Much of the web still runs on them.

---

## 3. The birth of NoSQL

> The term **NoSQL is unfortunate**, since it doesn't refer to any particular technology — it was intended simply as a **catchy Twitter hashtag** for a 2009 meetup on open source, distributed, non-relational databases.

It struck a nerve, spread, and was **retroactively re-interpreted as "Not Only SQL."**

**Four driving forces:**

```
   1. SCALABILITY        need greater scale than relational easily achieves —
                         very large datasets or very high write throughput

   2. OPEN SOURCE        widespread preference for free/OSS over commercial
                         database products

   3. SPECIALIZED        query operations not well supported by the
      QUERIES            relational model

   4. SCHEMA             frustration with restrictiveness of relational
      FRUSTRATION        schemas; desire for something more dynamic
                         and expressive
```

**Kleppmann's verdict — polyglot persistence:**

> Different applications have different requirements, and the best choice for one use case may differ from another. It therefore seems likely that in the foreseeable future, **relational databases will continue to be used alongside a broad variety of non-relational data stores.**

Note the tone: no winner declared. That restraint is deliberate and it holds up well a decade later.

---

## 4. The object-relational mismatch

Most application development is object-oriented. If data lives in relational tables, **an awkward translation layer** is needed between application objects and tables/rows/columns. This disconnect is called an **impedance mismatch**.

> 📖 **Where the term comes from:** electronics. Every circuit has an impedance (resistance to alternating current) on its inputs and outputs. When you connect one circuit's output to another's input, **power transfer is maximized if the impedances match.** A mismatch causes signal reflections and other troubles.

**ORMs** (ActiveRecord, Hibernate) reduce the boilerplate, **but they can't completely hide the differences between the two models.**

### 🔷 Figure 2-1 — A LinkedIn profile in a relational schema

The profile has a unique `user_id`. Fields like `first_name` appear **exactly once per user** → columns on `users`. But most people have **several** jobs, **varying** periods of education, and **any number** of contact details → **one-to-many** relationships.

```
                       users table
   ┌─────────┬────────────┬───────────┬──────────────────────────┐
   │ user_id │ first_name │ last_name │ summary                  │
   ├─────────┼────────────┼───────────┼──────────────────────────┤
   │   251   │   Bill     │   Gates   │ Co-chair of … blogger.   │
   └────┬────┴────────────┴───────────┴──────────────────────────┘
        │    ┌───────────┬─────────────┬──────────┐
        │    │ region_id │ industry_id │ photo_id │
        │    ├───────────┼─────────────┼──────────┤
        │    │   us:91   │     131     │ 57817532 │
        │    └─────┬─────┴──────┬──────┘
        │          │            │
        │          │ MANY-TO-ONE│  (many users live in one region)
        │          ▼            ▼
        │  ┌──────────────────────┐   ┌──────────────────────────┐
        │  │    regions table     │   │     industries table     │
        │  ├───────┬──────────────┤   ├──────┬───────────────────┤
        │  │ us:7  │ Greater      │   │  43  │ Financial Services│
        │  │       │ Boston Area  │   │  48  │ Construction      │
        │  │ us:91 │ Greater      │   │ 131  │ Philanthropy      │
        │  │       │ Seattle Area │   └──────┴───────────────────┘
        │  └───────┴──────────────┘
        │
        │  ONE-TO-MANY  (one user has many of each of these)
        │
        ├──────────────────────────────────────────────────────┐
        │                    positions table                   │
        │  ┌─────┬─────────┬──────────────────┬──────────────┐ │
        │  │ id  │ user_id │    job_title     │ organization │ │
        │  ├─────┼─────────┼──────────────────┼──────────────┤ │
        │  │ 458 │   251   │ Co-chair         │ Bill&Melinda │ │
        │  │ 457 │   251   │ Co-founder,Chair │ Microsoft    │ │
        │  └─────┴─────────┴──────────────────┴──────────────┘ │
        ├──────────────────────────────────────────────────────┤
        │                    education table                   │
        │  ┌─────┬─────────┬────────────────────┬──────┬─────┐ │
        │  │ id  │ user_id │    school_name     │start │ end │ │
        │  ├─────┼─────────┼────────────────────┼──────┼─────┤ │
        │  │ 807 │   251   │ Harvard University │ 1973 │1975 │ │
        │  │ 806 │   251   │ Lakeside School    │ NULL │NULL │ │
        │  └─────┴─────────┴────────────────────┴──────┴─────┘ │
        ├──────────────────────────────────────────────────────┤
        │                  contact_info table                  │
        │  ┌─────┬─────────┬─────────┬───────────────────────┐ │
        │  │ id  │ user_id │  type   │         url           │ │
        │  ├─────┼─────────┼─────────┼───────────────────────┤ │
        │  │ 155 │   251   │ blog    │ thegatesnotes.com     │ │
        │  │ 156 │   251   │ twitter │ twitter.com/BillGates │ │
        │  └─────┴─────────┴─────────┴───────────────────────┘ │
        └──────────────────────────────────────────────────────┘
```

### Three ways to represent one-to-many in SQL

| # | Approach | Notes |
|---|---|---|
| **1** | **Normalized separate tables** with a foreign key to `users` (Figure 2-1) | Traditional SQL model, pre-SQL:1999 |
| **2** | **Structured datatypes / XML columns** — multi-valued data inside a single row, with querying and indexing *inside* the document | Added in later SQL standards. Supported to varying degrees by Oracle, DB2, SQL Server, PostgreSQL. Postgres also has vendor extensions for JSON and arrays |
| **3** | **Encode as a JSON/XML blob in a text column**, let the application interpret it | ⚠️ You typically **cannot query values inside** that encoded column |

### Example 2-1 — The same profile as one JSON document

```json
{
  "user_id":     251,
  "first_name":  "Bill",
  "last_name":   "Gates",
  "summary":     "Co-chair of the Bill & Melinda Gates... Active blogger.",
  "region_id":   "us:91",
  "industry_id": 131,
  "photo_url":   "/p/7/000/253/05b/308dd6e.jpg",
  "positions": [
    {"job_title": "Co-chair",             "organization": "Bill & Melinda Gates Foundation"},
    {"job_title": "Co-founder, Chairman", "organization": "Microsoft"}
  ],
  "education": [
    {"school_name": "Harvard University",       "start": 1973, "end": 1975},
    {"school_name": "Lakeside School, Seattle", "start": null, "end": null}
  ],
  "contact_info": {
    "blog":    "http://thegatesnotes.com",
    "twitter": "http://twitter.com/BillGates"
  }
}
```

**Why this fits:** a résumé is **mostly a self-contained document.** JSON is much simpler than XML. Document databases — MongoDB, RethinkDB, CouchDB, Espresso — support this model directly.

**The locality win:**

```
   RELATIONAL (Fig 2-1)                      DOCUMENT (Example 2-1)
   ─────────────────────                     ──────────────────────
   To fetch one profile you must either:     One query. Everything is
                                             in one place.
     • run MULTIPLE queries
       (one per table, by user_id)                    ┌──────────┐
   or                                                 │ ████████ │
     • do a MESSY MULTI-WAY JOIN                      │ ████████ │  ← one
                                                      │ ████████ │    contiguous
   ┌────┐  ┌────┐  ┌────┐  ┌────┐                     │ ████████ │    read
   │users│ │pos.│  │edu.│  │cont│                     └──────────┘
   └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘
     └───────┴───────┴───────┘
        4 lookups / 1 big join
```

⚠️ **Kleppmann's own caution:** "Some developers feel that the JSON model reduces the impedance mismatch. **However, as we shall see in Chapter 4, there are also problems with JSON as a data encoding format.** The lack of a schema is often cited as an advantage" — and that claim gets examined critically in §7.

### 🔷 Figure 2-2 — One-to-many relationships form a TREE

The one-to-many relationships imply a tree structure, and **JSON makes that tree explicit**:

```
                              ┌───────────────┐
                              │    user 251   │
                              └───────┬───────┘
              ┌───────────────┬───────┴───────┬──────────────────┐
              │               │               │                  │
         first_name       last_name      ┌─────────┐        ┌──────────┐
         last_name         summary       │positions│        │education │
         (scalars)                       └────┬────┘        └─────┬────┘
                                    ┌─────────┼─────────┐    ┌────┴────┐
                                    │         │         │    │         │
                                 ┌─────┐  ┌─────┐  ┌─────┐ ┌─────┐ ┌─────┐
                                 │job 1│  │job 2│  │job 3│ │edu 1│ │edu 2│
                                 └──┬──┘  └──┬──┘  └──┬──┘ └──┬──┘ └──┬──┘
                                    │        │        │       │       │
                              job_title  job_title job_title  │       │
                            organization organization ...  school_name│
                                                              start   │
                                                              end     ...

   Every node has EXACTLY ONE parent.  ← this is the defining property
   That's what makes it a TREE, and it's what documents are good at.
```

---

## 5. Many-to-one and many-to-many relationships

### Why `region_id: "us:91"` instead of `"Greater Seattle Area"`?

If the UI is a free-text field, store the string. But there are real advantages to **standardized lists** with a dropdown/autocompleter:

```
   ✔ Consistent style and spelling across all profiles
   ✔ Avoids ambiguity (several cities share a name)
   ✔ Name stored in ONE place → easy to update everywhere
     (e.g. a city renamed due to political events)
   ✔ Localization — translate the standardized list, and the region
     displays in the viewer's own language
   ✔ Better SEARCH — "philanthropists in Washington state" can match,
     because the region list encodes that Seattle is in Washington.
     The raw string "Greater Seattle Area" does NOT tell you that.
```

### 🔑 The core principle: it's a question of duplication

```
   ┌──────────────────────────────────┬──────────────────────────────────┐
   │  STORE AN ID                     │  STORE THE TEXT                  │
   ├──────────────────────────────────┼──────────────────────────────────┤
   │  Human-meaningful info           │  Human-meaningful info is        │
   │  ("Philanthropy") lives in       │  DUPLICATED into every record    │
   │  exactly ONE place.              │  that uses it.                   │
   │                                  │                                  │
   │  Everything else refers to it    │  Update = touch every copy       │
   │  by an ID that has no meaning    │    → overhead on writes          │
   │  to humans.                      │    → RISK OF INCONSISTENCY       │
   │                                  │      (some copies updated,       │
   │  ➜ The ID NEVER NEEDS TO CHANGE, │       others not)                │
   │    even if the thing it          │                                  │
   │    identifies changes.           │                                  │
   └──────────────────────────────────┴──────────────────────────────────┘

              Removing such duplication is the key idea behind
                         N O R M A L I Z A T I O N
```

> 💡 **Kleppmann's rule of thumb on normal forms:** "Literature on the relational model distinguishes several different normal forms, but **the distinctions are of little practical interest. As a rule of thumb, if you're duplicating values that could be stored in just one place, the schema is not normalized.**"

That one sentence replaces a semester of 1NF/2NF/3NF/BCNF memorization.

### ⚠️ But normalizing needs many-to-one — which documents handle badly

```
   MANY people live in ONE region.        ◄── many-to-one
   MANY people work in ONE industry.      ◄── many-to-one

   ┌─────────────────────────┬─────────────────────────────────────────┐
   │ RELATIONAL DB           │ DOCUMENT DB                             │
   ├─────────────────────────┼─────────────────────────────────────────┤
   │ Normal to refer to rows │ Joins not needed for one-to-many trees, │
   │ in other tables by ID,  │ so SUPPORT FOR JOINS IS OFTEN WEAK.     │
   │ because JOINS ARE EASY. │                                         │
   │                         │ (At time of writing: supported in       │
   │                         │  RethinkDB, NOT in MongoDB, and only    │
   │                         │  in pre-declared views in CouchDB.)     │
   └─────────────────────────┴─────────────────────────────────────────┘

   No DB-side join?  →  EMULATE IT IN APPLICATION CODE with multiple queries.
                        The work is SHIFTED from the database to your code.
```

### 📈 Data gets more interconnected over time

> Even if the **initial version** of an application fits well in a join-free document model, **data has a tendency of becoming more interconnected as features are added.**

Two example features that break the tree:

| Feature | Why it needs many-to-many |
|---|---|
| **Organizations and schools as entities** | `organization` and `school_name` are currently just strings. Make them **references** — then each company/school gets its own page with logo and news feed, and every résumé mentioning it can link to it and show the logo (Figure 2-3) |
| **Recommendations** | One user writes a recommendation for another, shown with the **author's name and photo**. If the recommender updates their photo, **every recommendation they wrote must reflect it** — so the recommendation must **reference the author's profile**, not copy it |

### 🔷 Figure 2-4 — Extending résumés with many-to-many relationships

```
   ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐              ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
     DOCUMENT: user 251                       DOCUMENT: user 467
   │                          │              │                          │
     ┌──────────┐   ┌───────┐                  ┌───────┐   ┌──────────┐
   │ │positions │──►│ job 1 │ │   ┌───────┐  │ │ job 1 │◄──│positions │ │
     │          │   │job_ti…│─────► org 1 ◄─────│job_ti…│   │          │
   │ │          │   └───────┘ │   └───────┘  │ └───────┘   │          │ │
     │          │   ┌───────┐     ┌───────┐    ┌───────┐   │          │
   │ │          │──►│ job 2 │─────► org 2 │  │ │ job 2 │◄──│          │ │
     └──────────┘   │job_ti…│ │   └───────┘    │job_ti…│   └──────────┘
   │                └───────┘     ┌───────┐  │ └───────┘                │
                                │ │ org 3 │
   │ ┌──────────┐   ┌───────┐     └───────┘  │ ┌───────┐   ┌──────────┐ │
     │education │──►│ edu 1 │ │   ┌───────┐    │ edu 1 │◄──│education │
   │ │          │   │ start │─────►school1◄──────start │   │          │ │
     │          │   │  end  │ │   └───────┘  │ │  end  │   │          │
   │ │          │   └───────┘     ┌───────┐    └───────┘   │          │ │
     │          │──►┌───────┐ │   │school2│  │ ┌───────┐◄──│          │
   │ └──────────┘   │ edu 2 │     └───────┘    │ edu 2 │   └──────────┘ │
                    └───────┘ │   ┌───────┐  │ └───────┘
   │ ┌──────────┐   ┌───────┐     │school3│                             │
     │recommend-│──►│ rec 1 │ │   └───────┘  │
   │ │  ations  │   └───┬───┘                                           │
     └──────────┘       └──────┼──────────────► (references user 467)
   └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘              └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘

   ┌ ─ ─ ┐  Data inside a dotted rectangle CAN be grouped into one document.
            But references to orgs, schools and other users must be
            REFERENCES — and require JOINS when queried.
```

**That's the whole argument in one picture.** The dotted boxes are what documents are good at. Everything crossing a dotted line is what they're bad at.

---

## 6. Are document databases repeating history?

This debate is **much older than NoSQL.** It goes back to the earliest computerized database systems.

### IMS and the hierarchical model

**IBM's Information Management System (IMS)** — the most popular database for business data processing in the 1970s. Originally developed for **stock-keeping in the Apollo space program**, first commercially released **1968**, and *still in use and maintained today* on IBM mainframes.

> Its **hierarchical model** has **remarkable similarities to the JSON model** used by document databases. It represented all data as a **tree of records nested within records** — much like Figure 2-2.

```
   IMS (1968)                          DOCUMENT DBs (2010s)
   ──────────                          ────────────────────
   ✅ one-to-many: works well          ✅ one-to-many: works well
   ❌ many-to-many: difficult          ❌ many-to-many: difficult
   ❌ no joins                         ❌ joins weak or absent

   Developers had to choose:           Developers have to choose:
     • DUPLICATE (denormalize), or       • DUPLICATE (denormalize), or
     • manually resolve references       • emulate joins in app code

   "These problems of the 1960s were very much like the problems that
    developers are running into with document databases today."
```

Two solutions were proposed to fix the hierarchical model, and they fought the **"great debate" through most of the 1970s**:

### 6.1 The network model (CODASYL)

Standardized by the **Conference on Data Systems Languages (CODASYL)**.

> The CODASYL model is a **generalization of the hierarchical model.** In a tree, every record has **exactly one parent**; in the network model, a record can have **multiple parents.** This allows many-to-one and many-to-many relationships to be modeled.

```
   HIERARCHICAL (tree)                NETWORK (CODASYL)
   ───────────────────                ─────────────────

          ┌───┐                            ┌───┐   ┌───┐
          │ A │                            │ A │   │ B │
          └─┬─┘                            └─┬─┘   └─┬─┘
        ┌───┴───┐                            └───┬───┘
      ┌─▼─┐   ┌─▼─┐                            ┌─▼─┐
      │ B │   │ C │                            │ C │  ← TWO parents. Legal here.
      └───┘   └───┘                            └───┘

   exactly ONE parent per record      MULTIPLE parents allowed
```

**How links worked:**

> The links between records **are not foreign keys, but more like pointers in a programming language** (while still being stored on disk). **The only way of accessing a record was to follow a path from a root record along these chains of links.** This was called an **access path**.

```
   ACCESS PATH — the developer's burden

   root ──►[rec]──►[rec]──►[rec]──►[rec]──► ??? the one you want

   Simplest case: like traversing a linked list.
   With many-to-many: SEVERAL DIFFERENT PATHS lead to the same record,
   and the programmer had to KEEP TRACK OF THEM ALL IN THEIR HEAD.

   A query = moving a CURSOR through the database, iterating over lists
   of records and following access paths. If a record had multiple parents,
   the application code had to track all the relationships.

   Even CODASYL committee members admitted this was like
   "navigating around an n-dimensional data space."
```

**The trade-off, fairly stated:** manual access-path selection made **the most efficient use of very limited 1970s hardware** (tape drives, whose seeks are extremely slow). The problem was that it made query and update code **complicated and inflexible.**

> With both the hierarchical and network model, **if you didn't have a path to the data you wanted, you were in a difficult situation.** You could change the access paths — but then you had to go through a lot of hand-written query code and **rewrite it.** It was difficult to change an application's data model.

### 6.2 The relational model's answer

> What the relational model did, by contrast, was to **lay out all the data in the open**: a relation is simply a collection of tuples, and that's it. **No labyrinthine nested structures, no complicated access paths to follow.**

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  You can read ANY or ALL rows in a table, selecting those that     │
   │  match an ARBITRARY condition.                                     │
   │                                                                    │
   │  You can read a particular row by designating some columns as a    │
   │  KEY and matching on those.                                        │
   │                                                                    │
   │  You can INSERT a new row into any table, without worrying about   │
   │  foreign key relationships to and from other tables.               │
   └────────────────────────────────────────────────────────────────────┘
```

### 🔑 The killer insight: the query optimizer

```
   CODASYL                              RELATIONAL
   ───────                              ──────────
   Access paths chosen                  Access paths chosen
   BY THE DEVELOPER,                    AUTOMATICALLY by the
   by hand, per query                   QUERY OPTIMIZER

        │                                     │
        ▼                                     ▼
   Want to query a new way?             Want to query a new way?
   → rewrite the query code             → just DECLARE A NEW INDEX.
                                          Queries automatically use
                                          whichever indexes are most
                                          appropriate. You don't need
                                          to change your queries at all.
```

> Query optimizers are complicated beasts, consuming many years of research. **But a key insight of the relational model was this: you only need to build a query optimizer ONCE, and then all applications that use the database can benefit from it.**
>
> If you don't have an optimizer, it's easier to hand-code the access paths for a particular query than to write a general-purpose optimizer — **but the general-purpose solution wins in the long run.**

This is one of the most quotable passages in the book. It's an argument about **where to put engineering effort**, not about syntax.

### 6.3 So — are document DBs the new CODASYL? No.

```
   Document databases reverted to the hierarchical model in ONE aspect:
   storing NESTED RECORDS within their parent record rather than in a
   separate table.

   BUT for many-to-one and many-to-many, relational and document
   databases are NOT FUNDAMENTALLY DIFFERENT:

   ┌────────────────────┬────────────────────────┐
   │ RELATIONAL         │ DOCUMENT               │
   ├────────────────────┼────────────────────────┤
   │ FOREIGN KEY        │ DOCUMENT REFERENCE     │
   └─────────┬──────────┴───────────┬────────────┘
             └───────────┬──────────┘
                         ▼
        Both are a unique identifier, RESOLVED AT READ TIME
        by a join or follow-up queries.

        ➜ "To date, document databases have not followed
           the path of CODASYL."
```

The difference that matters: CODASYL resolved the join **at insert time** via pointers. Relational and document both resolve **at query time**. (Footnote in the book: foreign key *constraints* let you restrict modifications, but constraints aren't required by the relational model.)

---

## 7. Relational vs document today — the four axes

Restricting to data-model differences (fault tolerance is Ch. 5, concurrency is Ch. 7):

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║              ARGUMENTS FOR THE DOCUMENT MODEL                         ║
   ║   ① closer to app data structures  ② schema flexibility               ║
   ║   ③ better performance due to LOCALITY                                ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║              THE RELATIONAL MODEL COUNTERS WITH                       ║
   ║   ④ better support for JOINS, many-to-one and many-to-many            ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

### 7.1 Which leads to simpler application code?

**Use a document model if** your data has a document-like structure — *a tree of one-to-many relationships, where typically the entire tree is loaded at once.*

> The relational technique of **shredding** — splitting a document-like structure into multiple tables — can lead to **cumbersome schemas and unnecessarily complicated application code.**

**The document model's limitation:**

> You **cannot refer directly to a nested item** within a document. Instead you must say something like *"the second item in the list of positions for user 251"* — **much like an access path in the hierarchical model.**

(As long as documents aren't too deeply nested, this usually isn't a problem. But note the echo of CODASYL.)

**When poor join support hurts — and when it doesn't:**

```
   ✅ FINE:   an analytics application using a document DB to record
              which events occurred at which time.
              Many-to-many may NEVER be needed.

   ❌ HURTS:  if your application DOES use many-to-many.
              Options, both bad:
                • DENORMALIZE → app code must keep the denormalized
                  data consistent
                • EMULATE JOINS in app code with multiple requests
                  → moves complexity into the application,
                  → usually SLOWER than a join performed by
                    specialized code inside the database
              "Can lead to significantly more complex application
               code and worse performance."
```

### 🔑 The decision table (this is the sentence to memorize)

> It's **not possible to say in general** which data model leads to simpler application code; **it depends on the kinds of relationships that exist between data items.**

```
   ┌────────────────────────┬────────────┬─────────────┬───────────┐
   │  Your data is…         │  DOCUMENT  │ RELATIONAL  │   GRAPH   │
   ├────────────────────────┼────────────┼─────────────┼───────────┤
   │  tree-shaped           │    BEST    │     ok      │  overkill │
   │  (one-to-many only)    │            │             │           │
   ├────────────────────────┼────────────┼─────────────┼───────────┤
   │  some many-to-many     │  awkward   │    BEST     │    good   │
   ├────────────────────────┼────────────┼─────────────┼───────────┤
   │  HIGHLY INTERCONNECTED │ VERY       │ acceptable  │ MOST      │
   │                        │ AWKWARD    │             │ NATURAL   │
   └────────────────────────┴────────────┴─────────────┴───────────┘
```

### 7.2 Schema flexibility — "schemaless" is the wrong word

Most document databases (and JSON support in relational DBs) **do not enforce any schema.** No schema means arbitrary keys and values can be added, and readers have no guarantees about which fields exist.

> Document databases are sometimes called **schemaless**, but **that's misleading**, as the code that reads the data usually assumes some kind of structure — **there is an implicit schema, but it is not enforced by the database.**

```
   ╔════════════════════════════╦════════════════════════════════════════╗
   ║      SCHEMA-ON-READ        ║           SCHEMA-ON-WRITE              ║
   ║      (document DBs)        ║       (traditional relational)         ║
   ╠════════════════════════════╬════════════════════════════════════════╣
   ║ Structure is IMPLICIT,     ║ Schema is EXPLICIT, and the database   ║
   ║ interpreted only when the  ║ ENSURES all written data conforms.     ║
   ║ data is READ.              ║                                        ║
   ╠════════════════════════════╬════════════════════════════════════════╣
   ║          ANALOGY IN PROGRAMMING LANGUAGES                           ║
   ╠════════════════════════════╬════════════════════════════════════════╣
   ║ DYNAMIC (run-time)         ║ STATIC (compile-time)                  ║
   ║ type checking              ║ type checking                          ║
   ╚════════════════════════════╩════════════════════════════════════════╝

   Just as static vs dynamic typing advocates have big debates,
   schema enforcement is CONTENTIOUS — "in general there's no
   right or wrong answer."
```

### The migration example — where the difference bites

Say you store each user's full name in one field, and want to split it into first and last name.

**Document database — handle it at read time:**

```javascript
if (user && user.name && !user.first_name) {
    // Documents written before Dec 8, 2013 don't have first_name
    user.first_name = user.name.split(" ")[0];
}
```

**Relational database — migrate:**

```sql
ALTER TABLE users ADD COLUMN first_name text;

UPDATE users SET first_name = split_part(name, ' ', 1);      -- PostgreSQL
UPDATE users SET first_name = substring_index(name, ' ', 1); -- MySQL
```

**⚠️ Kleppmann pushes back on the folklore here — this is a genuinely useful correction:**

> Schema changes have a bad reputation of being slow and requiring downtime. **This reputation is not entirely deserved: most relational database systems execute the `ALTER TABLE` statement in a few milliseconds** — with the exception of **MySQL, which copies the entire table**, which can mean minutes or hours of downtime on a large table. (Tools exist to work around this: `pt-online-schema-change`, Large Hadron Migrator.)

And the honest symmetry:

> Running the `UPDATE` on a large table is likely to be slow **on any database**, since every row must be re-written. If that's not acceptable, the application can **leave `first_name` as NULL and fill it in at read time** — *exactly like it would with a document database.*

So the two approaches converge under pressure. The difference is smaller than the marketing suggests.

### When is schema-on-read genuinely better?

```
   ✅ USE SCHEMA-ON-READ when the data is HETEROGENEOUS:

      • Many different types of object, and it's not practical to put
        each type in its own table
      • The structure is determined by EXTERNAL SYSTEMS you don't
        control, and which may change at any time

      → "a schema may hurt more than it helps"

   ✅ USE SCHEMA-ON-WRITE when all records are expected to have the
      SAME structure:

      → "schemas are a useful mechanism for documenting and
         enforcing that structure"
```

### 7.3 Data locality for queries

A document is usually stored as **a single continuous string** — JSON, XML, or a binary variant like MongoDB's **BSON**.

```
   ✅ THE ADVANTAGE
   If your application often needs the ENTIRE document (e.g. to render
   a web page), storage locality is a real performance win.
   Split across tables → multiple index lookups → more disk seeks.

   ⚠️ THE THREE CAVEATS
   ① Applies ONLY if you need large parts of the document at once.
   ② The DB typically loads the ENTIRE document even if you access a
      small portion — wasteful on large documents.
   ③ On UPDATE, the entire document usually must be RE-WRITTEN. Only
      modifications that don't change the encoded size can easily be
      done in place.

   ➜ RECOMMENDATION: keep documents fairly small, and avoid writes
     that increase document size.

   ➜ "These performance limitations significantly reduce the set of
      situations in which document databases are useful."
```

**Locality isn't exclusive to documents.** Relational and column systems do it too:

| System | Locality mechanism |
|---|---|
| **Google Spanner** | Schema can declare a table's rows be **interleaved (nested)** within a parent table |
| **Oracle** | **Multi-table index cluster tables** |
| **Bigtable** (Cassandra, HBase) | The **column-family** concept |

### 7.4 Convergence — the models are growing together

```
                RELATIONAL                          DOCUMENT
                    │                                   │
     XML support since mid-2000s              RethinkDB supports
     (all but MySQL): local modifications,    relational-like JOINS
     indexing and querying INSIDE XML                    │
                    │                          Some MongoDB drivers
     PostgreSQL 9.3+ and DB2 10.5+:            auto-resolve database
     similar support for JSON                  references (client-side
                    │                          join — slower: extra
     "Given the popularity of JSON for         network round-trips,
      web APIs, it is likely that other        less optimized)
      relational DBs will follow."                       │
                    │                                    │
                    └──────────────►  ◄──────────────────┘
                              CONVERGENCE

   "It seems that relational and document databases are becoming more
    similar over time, and THAT IS A GOOD THING: the data models
    COMPLEMENT each other. A hybrid of the relational and document
    models is a good route for databases to take in future."
```

> 🎓 **The footnote that ties history in a bow:** Codd's **original 1970 description** of the relational model allowed something quite similar to JSON documents — he called them **nonsimple domains**. The idea: a value in a row doesn't have to be a primitive; **it could be a nested relation**, giving arbitrarily nested trees as values — *"much like the JSON or XML support that was added to SQL over 30 years later."*

The industry spent three decades rediscovering a footnote from the original paper.

---

# PART B — QUERY LANGUAGES FOR DATA

## 8. Declarative vs imperative

When the relational model arrived it brought a new way of querying: **SQL is declarative**, whereas IMS and CODASYL used **imperative code**.

### The same question, three ways

**Imperative** — "get me the sharks, and here's exactly how":

```javascript
function getSharks() {
    var sharks = [];
    for (var i = 0; i < animals.length; i++) {
        if (animals[i].family === "Sharks") {
            sharks.push(animals[i]);
        }
    }
    return sharks;
}
```

**Relational algebra** — σ (sigma) is the *selection* operator:

```
    sharks = σ  family = "Sharks"  (animals)
```

**SQL** — which "followed the structure of the relational algebra fairly closely":

```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

### The distinction, precisely

```
   ┌──────────────────────────────────┬──────────────────────────────────┐
   │  IMPERATIVE                      │  DECLARATIVE                     │
   ├──────────────────────────────────┼──────────────────────────────────┤
   │  Tells the computer to perform   │  You specify the PATTERN of the  │
   │  CERTAIN OPERATIONS in a         │  data you want:                  │
   │  CERTAIN ORDER.                  │    • what conditions results     │
   │                                  │      must meet                   │
   │  You can step through it line    │    • how to transform it         │
   │  by line: evaluate conditions,   │      (sort, group, aggregate)    │
   │  update variables, decide        │                                  │
   │  whether to loop again.          │  …but NOT HOW to achieve it.     │
   │                                  │                                  │
   │                                  │  The QUERY OPTIMIZER decides     │
   │                                  │  indexes, join methods, order.   │
   └──────────────────────────────────┴──────────────────────────────────┘
```

### 🔑 Three reasons declarative wins

```
   ① CONCISENESS
      Typically more concise and easier to work with than an imperative API.

   ② IMPLEMENTATION HIDING  ◄── "more importantly"
      Hides engine details, so the DB can introduce PERFORMANCE
      IMPROVEMENTS WITHOUT ANY CHANGES TO YOUR QUERIES.

      Worked example from the book:
      ┌──────────────────────────────────────────────────────────────┐
      │ In the imperative code, `animals` appears in a PARTICULAR    │
      │ ORDER. Suppose the DB wants to reclaim disk space by moving  │
      │ records around, changing that order. Can it do so safely?    │
      │                                                              │
      │  SQL version:        doesn't guarantee ordering → SAFE ✅     │
      │  Imperative version: the DB can NEVER BE SURE whether the    │
      │                      code relies on ordering → UNSAFE ❌      │
      │                                                              │
      │ "The fact that SQL is MORE LIMITED in functionality gives    │
      │  the database MUCH MORE ROOM for automatic optimizations."   │
      └──────────────────────────────────────────────────────────────┘

   ③ PARALLELISM
      CPUs get faster by adding CORES, not clock speed.
      Imperative code is very hard to parallelize — it specifies
      instructions that MUST run in a particular order.
      Declarative languages specify only the PATTERN of results, not
      the algorithm → the database is free to use a parallel
      implementation.
```

**Point ② is the deep one.** Limiting what a language can express is what makes optimization possible. Expressiveness and optimizability trade off against each other.

---

## 9. Declarative queries on the web (the CSS analogy)

A beautiful detour: the same argument, in a completely different environment.

**The markup** — a nav where the current page is marked `class="selected"`:

```html
<ul>
    <li class="selected">
        <p>Sharks</p>
        <ul>
            <li>Great White Shark</li>
            <li>Tiger Shark</li>
            <li>Hammerhead Shark</li>
        </ul>
    </li>
    <li>
        <p>Whales</p>
        <ul>
            <li>Blue Whale</li>
            <li>Humpback Whale</li>
            <li>Fin Whale</li>
        </ul>
    </li>
</ul>
```

**Goal:** give the currently selected page title a blue background.

**Declarative — CSS:**

```css
li.selected > p {
    background-color: blue;
}
```

The selector **declares the pattern**: all `<p>` whose *direct parent* is an `<li>` with class `selected`. `<p>Sharks</p>` matches; `<p>Whales</p>` doesn't.

**Declarative — XSL** (the XPath `li[@class='selected']/p` is equivalent):

```xml
<xsl:template match="li[@class='selected']/p">
    <fo:block background-color="blue">
        <xsl:apply-templates/>
    </fo:block>
</xsl:template>
```

**Imperative — the DOM API:**

```javascript
var liElements = document.getElementsByTagName("li");
for (var i = 0; i < liElements.length; i++) {
    if (liElements[i].className === "selected") {
        var children = liElements[i].childNodes;
        for (var j = 0; j < children.length; j++) {
            var child = children[j];
            if (child.nodeType === Node.ELEMENT_NODE && child.tagName === "P") {
                child.setAttribute("style", "background-color: blue");
            }
        }
    }
}
```

> "This code imperatively sets the element to have a blue background, **but the code is awful.**"

### The two serious problems (not just verbosity)

```
   ❌ PROBLEM 1 — STATE DOESN'T UNDO ITSELF

      If the `selected` class is REMOVED (user clicks another page),
      the blue colour WON'T BE REMOVED — even if you re-run the code.
      The item stays highlighted until page reload.

      With CSS: the browser automatically detects that
      `li.selected > p` no longer applies and removes the blue
      background AS SOON AS the class is removed.

   ❌ PROBLEM 2 — NO FREE PERFORMANCE

      Want to use a faster new API like getElementsByClassName() or
      document.evaluate()?  YOU have to rewrite the code.

      Browser vendors can improve CSS and XPath performance WITHOUT
      BREAKING COMPATIBILITY — you get the speedup for free.
```

> **In a web browser, declarative CSS styling is much better than manipulating styles imperatively in JavaScript. Similarly, in databases, declarative query languages like SQL turned out to be much better than imperative query APIs.**

*(Footnote: IMS and CODASYL both used imperative query APIs — applications typically iterated over records one at a time in COBOL.)*

---

## 10. MapReduce querying

> MapReduce is **neither a declarative query language, nor a fully imperative query API, but somewhere in between**: the logic is expressed with **snippets of code, which are called repeatedly by the processing framework.**

Based on `map` (a.k.a. *collect*) and `reduce` (a.k.a. *fold* / *inject*) from functional programming. A limited form is supported by MongoDB and CouchDB for read-only queries across many documents.

**The scenario:** you're a marine biologist logging ocean sightings. You want a report of **how many sharks were sighted per month.**

### Version 1 — SQL (PostgreSQL)

```sql
SELECT date_trunc('month', observation_timestamp) AS observation_month,
       sum(num_animals)                           AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```

`date_trunc('month', timestamp)` rounds a timestamp down to the start of the calendar month. The query **filters** → **groups by month** → **sums**.

### Version 2 — MongoDB MapReduce

```javascript
db.observations.mapReduce(
    function map() {                                       // ②③
        var year  = this.observationTimestamp.getFullYear();
        var month = this.observationTimestamp.getMonth() + 1;
        emit(year + "-" + month, this.numAnimals);
    },
    function reduce(key, values) {                         // ④⑤
        return Array.sum(values);
    },
    {
        query: { family: "Sharks" },                       // ①
        out:   "monthlySharkReport"                        // ⑥
    }
);
```

```
   ① The shark filter is specified DECLARATIVELY
      (a MongoDB-specific extension to MapReduce)
   ② `map` is called ONCE FOR EVERY DOCUMENT matching `query`,
      with `this` set to the document
   ③ `map` emits a KEY (e.g. "2013-12") and a VALUE (animal count)
   ④ Emitted pairs are GROUPED BY KEY; `reduce` is called once
      per distinct key
   ⑤ `reduce` adds up the animals for that month
   ⑥ Output written to the collection `monthlySharkReport`
```

### 🔷 The dataflow, traced through the book's own example

Two documents in the collection:

```javascript
{ observationTimestamp: Date.parse("Mon, 25 Dec 1995 12:34:56 GMT"),
  family: "Sharks", species: "Carcharodon carcharias", numAnimals: 3 }

{ observationTimestamp: Date.parse("Tue, 12 Dec 1995 16:17:18 GMT"),
  family: "Sharks", species: "Carcharias taurus",      numAnimals: 4 }
```

```
   INPUT DOCUMENTS            MAP                 GROUP BY KEY         REDUCE
   ───────────────            ───                 ────────────         ──────

   ┌─────────────────┐
   │ 25 Dec 1995     │   emit("1995-12", 3) ─┐
   │ Sharks          │──►                    │
   │ numAnimals: 3   │                       │   ┌──────────────┐
   └─────────────────┘                       ├──►│ "1995-12"    │──► reduce(
                                             │   │   → [3, 4]   │      "1995-12",
   ┌─────────────────┐                       │   └──────────────┘       [3,4])
   │ 12 Dec 1995     │   emit("1995-12", 4) ─┘                        = 7
   │ Sharks          │──►
   │ numAnimals: 4   │
   └─────────────────┘

   ┌─────────────────┐
   │ 20 Dec 1995     │   ✗ filtered out by  query:{family:"Sharks"}
   │ Whales          │
   └─────────────────┘
```

### ⚠️ The restrictions on map and reduce — and why they exist

> They must be **pure functions**: they only use the data passed to them as input, **cannot perform additional database queries**, and **must not have side effects.**

```
   WHY?  These restrictions allow the database to:
         • run the functions ANYWHERE
         • in ANY ORDER
         • and RE-RUN them on failure

   (Nevertheless powerful: they can parse strings, call library
    functions, perform calculations, and more.)
```

That trade — give up side effects, gain relocatable and retryable execution — is the same bargain that makes distributed computing work at all. It reappears in Chapter 10.

### Version 3 — the aggregation pipeline (MongoDB 2.2+)

**The usability problem with MapReduce:**

> You have to write **two carefully coordinated JavaScript functions**, which is often harder than writing a single query. Moreover, **a declarative query language offers more opportunities for a query optimizer** to improve performance.

So MongoDB added a declarative language:

```javascript
db.observations.aggregate([
    { $match: { family: "Sharks" } },
    { $group: {
        _id: {
            year:  { $year:  "$observationTimestamp" },
            month: { $month: "$observationTimestamp" }
        },
        totalAnimals: { $sum: "$numAnimals" }
    } }
]);
```

### 😄 The punchline

> The aggregation pipeline's expressiveness is **similar to a subset of SQL**, but with JSON-based syntax rather than SQL's English-sentence style; the difference is perhaps a matter of taste.
>
> **The moral of the story is that a NoSQL system may find itself accidentally reinventing SQL, albeit in disguise.**

Map the three side by side and the convergence is obvious:

```
   SQL                        AGGREGATION PIPELINE
   ───                        ────────────────────
   WHERE family='Sharks'  ≡   { $match: { family: "Sharks" } }
   GROUP BY month         ≡   { $group: { _id: {…month…}         } }
   sum(num_animals)       ≡              totalAnimals: {$sum: …}
```

**Also worth noting:** there's nothing in SQL that constrains it to a single machine, and **MapReduce doesn't have a monopoly on distributed query execution.** Higher-level languages like SQL *can* be implemented as a pipeline of MapReduce operations, but many distributed SQL implementations don't use MapReduce at all. And using JavaScript mid-query isn't MapReduce-specific — some SQL databases support JS functions too.

---

# PART C — GRAPH-LIKE DATA MODELS

## 11. When everything relates to everything

```
   mostly one-to-many, or no relationships    ──►  DOCUMENT model
   simple many-to-many                        ──►  RELATIONAL handles it
   connections become COMPLEX                 ──►  GRAPH becomes natural
```

**A graph has two kinds of object:**

```
   ┌──────────────────────────────────────────────────────────────────┐
   │   VERTICES                          EDGES                        │
   │   (nodes, entities)                 (relationships, arcs)        │
   │                                                                  │
   │        ( A )  ─────────────────────────────────►  ( B )          │
   │        tail                    label                head         │
   └──────────────────────────────────────────────────────────────────┘
```

**Typical examples:**

| Graph | Vertices | Edges | Classic algorithm |
|---|---|---|---|
| **Social graph** | People | Who knows whom | — |
| **Web graph** | Web pages | HTML links | **PageRank** → search ranking |
| **Road/rail network** | Junctions | Roads/railway lines | **Shortest path** → routing |

### 🔑 The underrated point: graphs handle HETEROGENEOUS data

> Graphs are **not limited to homogeneous data**. An equally powerful use is to **provide a consistent way of storing completely different types of object in a single data store.**

> **Facebook maintains a single graph with many different types of vertex and edge:** vertices are people, locations, events, checkins and comments; edges indicate who is friends with whom, which checkin happened in which location, who commented on which post, who attended which event.

### 🔷 Figure 2-5 — The running example

Two people: **Lucy** from Idaho, and **Alain** from Beaune, France. They are **married** and **living in London**.

```
   ┌────────────────────┐                              ┌────────────────────┐
   │ type: continent    │                              │ type: continent    │
   │ name: North America│                              │ name: Europe       │
   └─────────▲──────────┘                              └────▲──────────▲────┘
             │ within                            within │          │ within
             │                                          │          │
   ┌─────────┴──────────┐          ┌────────────────────┴──┐  ┌────┴───────────┐
   │ type: country      │          │ type: country         │  │ type: country  │
   │ name: United States│          │ name: United Kingdom  │  │ name: France   │
   └─────────▲──────────┘          └───────────▲───────────┘  └────▲───────────┘
             │ within                          │ within            │ within
             │                                 │                   │
   ┌─────────┴──────────┐          ┌───────────┴───────────┐  ┌────┴─────────────┐
   │ type: state        │          │ type: country         │  │ type: région     │
   │ name: Idaho        │          │ name: England         │  │ name_fr:Bourgogne│
   │ abbreviation: ID   │          └───────────▲───────────┘  │ name_en:Burgundy │
   └─────────▲──────────┘                      │ within       └────▲─────────────┘
             │                     ┌───────────┴───────────┐       │ within
             │                     │ type: city            │  ┌────┴─────────────┐
             │                     │ name: London          │  │type: département │
             │                     └──▲─────────────▲──────┘  │name: Côte-d'Or   │
             │            lives_in    │             │         └────▲─────────────┘
             │      ┌─────────────────┘             │              │ within
             │      │                      lives_in │         ┌────┴─────────────┐
             │      │                               │         │ type: city       │
             │ born_in                              │         │ name: Beaune     │
             │      │                               │         └────▲─────────────┘
   ┌─────────┴──────┴───┐   married   ┌─────────────┴──────┐ born_in │
   │ type: person       │◄───────────►│ type: person       │─────────┘
   │ name: Lucy         │             │ name: Alain        │
   └────────────────────┘             └────────────────────┘

   BOXES = vertices        ARROWS = edges (labels shown on the arrows)
```

**What this diagram proves — four things hard to express in a traditional relational schema:**

```
   ① DIFFERENT REGIONAL STRUCTURES PER COUNTRY
      France has départements and régions;
      the US has counties and states.
      One rigid table can't hold both cleanly.

   ② QUIRKS OF HISTORY
      A country within a country — England within the United Kingdom.

   ③ VARYING GRANULARITY
      Lucy's residence is a CITY (London);
      her birthplace is only a STATE (Idaho).

   ④ MULTILINGUAL PROPERTIES
      Bourgogne has name_fr and name_en, not a single `name`.
      Other vertices have plain `name`. No uniform schema required.
```

> **Graphs are good for evolvability:** as you add features, a graph can easily be extended to accommodate changes in your application's data structures.

*Example given: add food allergies — a vertex per allergen, an edge from person to allergen, and link allergens to foods containing them. Now you can query what's safe for each person to eat. No schema migration.*

---

## 12. Property graphs

```
   EVERY VERTEX consists of:              EVERY EDGE consists of:
   ┌────────────────────────────┐         ┌──────────────────────────────────┐
   │ • a unique identifier      │         │ • a unique identifier            │
   │ • a set of OUTGOING edges  │         │ • the TAIL vertex (starts at)    │
   │ • a set of INCOMING edges  │         │ • the HEAD vertex (ends at)      │
   │ • properties (key-value)   │         │ • a LABEL describing the kind    │
   └────────────────────────────┘         │   of relationship                │
                                          │ • properties (key-value)         │
                                          └──────────────────────────────────┘
```

### Example 2-2 — A property graph as two relational tables

```sql
CREATE TABLE vertices (
    vertex_id   integer PRIMARY KEY,
    properties  json
);

CREATE TABLE edges (
    edge_id     integer PRIMARY KEY,
    tail_vertex integer REFERENCES vertices (vertex_id),
    head_vertex integer REFERENCES vertices (vertex_id),
    label       text,
    properties  json
);

CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```

Just **two tables** for arbitrarily complex graphs. That's remarkable, and it's the reason graph modelling feels so flexible.

### Three important consequences

```
   ① NO SCHEMA RESTRICTS CONNECTIONS
      Any vertex can have an edge to ANY other vertex.
      Nothing restricts which kinds of thing can be associated.

   ② TRAVERSAL IN BOTH DIRECTIONS
      Given any vertex you can efficiently find BOTH its incoming
      and outgoing edges → follow a path forwards AND backwards.
      ➜ THIS IS WHY there are indexes on BOTH tail_vertex
        AND head_vertex.

   ③ ONE GRAPH, MANY KINDS OF INFORMATION
      Different LABELS for different kinds of relationship let you
      store several different kinds of information in a single graph
      while keeping a clean data model.
```

---

## 13. Cypher — the property graph query language

Created for **Neo4j**. Named after a character in *The Matrix* — **not** related to ciphers in cryptography.

### Example 2-3 — Inserting the left-hand portion of Figure 2-5

```cypher
CREATE
  (NAmerica:Location {name:'North America', type:'continent'}),
  (USA:Location      {name:'United States', type:'country'  }),
  (Idaho:Location    {name:'Idaho',         type:'state'    }),
  (Lucy:Person       {name:'Lucy' }),
  (Idaho) -[:WITHIN]-> (USA) -[:WITHIN]-> (NAmerica),
  (Lucy)  -[:BORN_IN]-> (Idaho)
```

**Reading the arrow notation:**

```
   (Idaho) -[:WITHIN]-> (USA)
      │        │           │
      │        │           └── HEAD vertex
      │        └────────────── edge LABEL
      └─────────────────────── TAIL vertex

   Vertices get symbolic names (USA, Idaho) so later parts of the
   query can refer to them when creating edges.
```

### Example 2-4 — Find people who emigrated from the US to Europe

```cypher
MATCH
  (person) -[:BORN_IN]->  () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
  (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name
```

**In English:** find any vertex (call it `person`) meeting **both** conditions:

```
   ① person has an outgoing BORN_IN edge to some vertex. From there,
      follow a CHAIN of outgoing WITHIN edges until you reach a
      Location whose name is "United States".

   ② That SAME person also has an outgoing LIVES_IN edge. Follow it,
      then a chain of outgoing WITHIN edges, until you reach a
      Location whose name is "Europe".

   For each such person, return the name property.
```

**The critical piece — `*0..`:**

```
   -[:WITHIN*0..]->     means "follow a WITHIN edge, ZERO OR MORE times"
                        ➜ like the * operator in a regular expression

   Why you need it:
      Lucy -[:LIVES_IN]-> London -[:WITHIN]-> England
            -[:WITHIN]-> UK -[:WITHIN]-> Europe        (3 hops)

      Alain -[:BORN_IN]-> Beaune -[:WITHIN]-> Côte-d'Or
            -[:WITHIN]-> Bourgogne -[:WITHIN]-> France
            -[:WITHIN]-> Europe                        (4 hops)

   The NUMBER OF HOPS IS NOT KNOWN IN ADVANCE. That's the defining
   difficulty of graph queries.
```

### The declarative payoff, again

**Two totally different execution strategies for the same query:**

```
   STRATEGY A — forwards                  STRATEGY B — backwards
   ────────────────────                   ──────────────────────
   Scan ALL people in the DB.             Use an index on `name` to find
   For each, examine birthplace           the US and Europe vertices.
   and residence.                         Follow all INCOMING within edges
   Return those matching.                 to collect every location inside
                                          each. Then find people via
                                          incoming BORN_IN / LIVES_IN edges.
```

> As typical for a declarative language, **you don't need to specify such execution details.** The query optimizer automatically chooses the strategy predicted to be most efficient, and you get on with writing the rest of your application.

---

## 14. Graph queries in SQL — yes, but painfully

> **In a relational database, you usually know in advance which joins you need. In a graph query, you may need to traverse a variable number of edges before you find the vertex you're looking for — i.e. THE NUMBER OF JOINS IS NOT FIXED IN ADVANCE.**

A `LIVES_IN` edge may point at a street, city, district, region, or state. It may point **directly** at your target, or be **several levels removed**.

SQL's answer since **SQL:1999**: **recursive common table expressions** (`WITH RECURSIVE`), supported in PostgreSQL, DB2, Oracle, SQL Server — and SQLite.

### Example 2-5 — The same query in SQL

```sql
WITH RECURSIVE
  -- in_usa is the set of vertex IDs of all locations within the United States
  in_usa(vertex_id) AS (
       SELECT vertex_id FROM vertices WHERE properties->>'name' = 'United States'  -- ①
    UNION
       SELECT edges.tail_vertex FROM edges                                          -- ②
         JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
         WHERE edges.label = 'within'
  ),
  -- in_europe is the set of vertex IDs of all locations within Europe
  in_europe(vertex_id) AS (                                                         -- ③
       SELECT vertex_id FROM vertices WHERE properties->>'name' = 'Europe'
     UNION
       SELECT edges.tail_vertex FROM edges
         JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
         WHERE edges.label = 'within'
  ),
  -- born_in_usa is the set of vertex IDs of all people born in the US
  born_in_usa(vertex_id) AS (                                                       -- ④
     SELECT edges.tail_vertex FROM edges
       JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
       WHERE edges.label = 'born_in'
  ),
  -- lives_in_europe is the set of vertex IDs of all people living in Europe
  lives_in_europe(vertex_id) AS (                                                   -- ⑤
     SELECT edges.tail_vertex FROM edges
       JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
       WHERE label = 'lives_in'
  )
SELECT vertices.properties->>'name'
FROM vertices
-- join to find those people who were both born in the US *and* live in Europe
JOIN born_in_usa     ON vertices.vertex_id = born_in_usa.vertex_id                  -- ⑥
JOIN lives_in_europe ON vertices.vertex_id = lives_in_europe.vertex_id;
```

```
   ① Find the vertex named "United States", add it to the set in_usa
   ② Follow all INCOMING within edges from vertices in in_usa, add them
      to the SAME set — repeat until no incoming within edges remain
   ③ Same, starting from "Europe", building in_europe
   ④ For each vertex in in_usa, follow incoming born_in edges
   ⑤ For each vertex in in_europe, follow incoming lives_in edges
   ⑥ INTERSECT the two sets of people by joining them
```

### 😐 The verdict

```
   ┌──────────────────────────────────────────────────────────────┐
   │        CYPHER: 4 lines        vs        SQL: 29 lines        │
   │                                                              │
   │  "If the same query can be written in four lines in one      │
   │   query language, but requires 29 lines in another, that     │
   │   just shows that DIFFERENT DATA MODELS ARE DESIGNED TO      │
   │   SATISFY DIFFERENT USE CASES. It's important to pick a      │
   │   data model that is suitable for your application."         │
   └──────────────────────────────────────────────────────────────┘
```

Note the framing — not "SQL is bad," but "the fit is wrong."

### 📦 Sidebar: Graph databases vs the network model — are graphs just CODASYL again?

**No.** Four important differences:

```
   ┌─────┬──────────────────────────────┬───────────────────────────────┐
   │     │  CODASYL (network model)     │  GRAPH DATABASE               │
   ├─────┼──────────────────────────────┼───────────────────────────────┤
   │  1  │ Schema specified which       │ NO SUCH RESTRICTION — any     │
   │     │ record type could be nested  │ vertex can have an edge to    │
   │     │ within which other type      │ any other vertex. Much        │
   │     │                              │ greater flexibility to adapt  │
   │     │                              │ to changing requirements.     │
   ├─────┼──────────────────────────────┼───────────────────────────────┤
   │  2  │ ONLY way to reach a record   │ Refer DIRECTLY to any vertex  │
   │     │ was to traverse an access    │ by its unique ID, or use a    │
   │     │ path to it                   │ PROPERTY INDEX to find        │
   │     │                              │ vertices by value.            │
   ├─────┼──────────────────────────────┼───────────────────────────────┤
   │  3  │ Children were an ORDERED     │ NO defined ordering of        │
   │     │ SET — the DB had to maintain │ vertices or edges. You sort   │
   │     │ ordering (affecting storage  │ results at query time only.   │
   │     │ layout), and inserts had to  │                               │
   │     │ worry about position         │                               │
   ├─────┼──────────────────────────────┼───────────────────────────────┤
   │  4  │ All queries IMPERATIVE —     │ You CAN write imperative      │
   │     │ difficult to write, easily   │ traversals, but most graph    │
   │     │ broken by schema changes     │ DBs also support high-level   │
   │     │                              │ DECLARATIVE languages like    │
   │     │                              │ Cypher or SPARQL.             │
   └─────┴──────────────────────────────┴───────────────────────────────┘
```

---

## 15. Triple-stores and SPARQL

> The triple-store model is **mostly equivalent to the property graph model, using different words to describe the same ideas.**

Everything is a three-part statement:

```
   ( SUBJECT , PREDICATE , OBJECT )

   e.g.  ( Jim , likes , bananas )
           │       │        │
           │       │        └── object
           │       └─────────── predicate (verb)
           └─────────────────── subject
```

### The two cases for the object — this is the key to the mapping

```
   CASE 1 — object is a PRIMITIVE VALUE (string, number)
   ─────────────────────────────────────────────────────
      ( lucy , age , 33 )

      ➜ predicate + object = KEY + VALUE of a PROPERTY on the
        subject vertex

      Equivalent property graph:   vertex lucy { "age": 33 }


   CASE 2 — object is ANOTHER VERTEX
   ─────────────────────────────────
      ( lucy , marriedTo , alain )

      ➜ predicate = the EDGE LABEL
        subject   = TAIL vertex
        object    = HEAD vertex

      Equivalent property graph:   (lucy) -[:marriedTo]-> (alain)
```

### Example 2-6 — Figure 2-5 data as Turtle triples

```turtle
@prefix : <urn:example:>.
_:lucy     a       :Person.
_:lucy     :name   "Lucy".
_:lucy     :bornIn _:idaho.
_:idaho    a       :Location.
_:idaho    :name   "Idaho".
_:idaho    :type   "state".
_:idaho    :within _:usa.
_:usa      a       :Location.
_:usa      :name   "United States".
_:usa      :type   "country".
_:usa      :within _:namerica.
_:namerica a       :Location.
_:namerica :name   "North America".
_:namerica :type   "continent".
```

Vertices are written `_:someName` — the name means nothing outside this file; it exists only so we know which triples refer to the same vertex. When the predicate is an **edge**, the object is a vertex (`_:idaho :within _:usa`). When the predicate is a **property**, the object is a string literal (`_:usa :name "United States"`).

### Example 2-7 — The concise form (semicolons group by subject)

```turtle
@prefix : <urn:example:>.
_:lucy     a :Person;   :name "Lucy";          :bornIn _:idaho.
_:idaho    a :Location; :name "Idaho";         :type "state";   :within _:usa.
_:usa      a :Location; :name "United States"; :type "country"; :within _:namerica.
_:namerica a :Location; :name "North America"; :type "continent".
```

### 🕸️ The semantic web — a balanced take

> The triple-store data model is **completely independent** of the semantic web — Datomic, for instance, is a triple-store that claims no connection to it. But since the two are linked in many people's minds, they're worth discussing.

**The idea, which is fundamentally simple and reasonable:**

```
   Websites already publish information as TEXT AND PICTURES for
   HUMANS to read.

   Why don't they ALSO publish it as MACHINE-READABLE DATA?

   RDF (Resource Description Framework) was intended as a mechanism
   for different websites to publish data in a consistent format,
   so data from different sites could be automatically combined into
   a "WEB OF DATA" — an internet-wide 'database of everything'.
```

**The honest assessment:**

> Unfortunately, the semantic web was **over-hyped in the early 2000s**, but so far hasn't shown any sign of being realized in practice, which has made many people cynical about it. It has also suffered from **a dizzying plethora of acronyms, overly complex standards proposals, and hubris.**
>
> **However, if you look past those failings, there is also a lot of good work that has come out of the semantic web project.** Triples can be a good internal data model for applications, even if you have no interest in publishing RDF data on the semantic web.

### Example 2-8 — RDF/XML (the verbose alternative to Turtle)

```xml
<rdf:RDF xmlns="urn:example:"
    xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">

  <Location rdf:nodeID="idaho">
    <name>Idaho</name>
    <type>state</type>
    <within>
      <Location rdf:nodeID="usa">
        <name>United States</name>
        <type>country</type>
        <within>
          <Location rdf:nodeID="namerica">
            <name>North America</name>
            <type>continent</type>
          </Location>
        </within>
      </Location>
    </within>
  </Location>

  <Person rdf:nodeID="lucy">
    <name>Lucy</name>
    <bornIn rdf:nodeID="idaho"/>
  </Person>
</rdf:RDF>
```

Same data, "much more verbosely." Turtle/N3 is preferable — "much easier on the eyes" — and tools like Apache Jena convert between formats.

### Why RDF predicates are URIs

```
   Not just:  WITHIN  or  LIVES_IN

   But:       <http://my-company.com/namespace#within>
              <http://my-company.com/namespace#lives_in>

   WHY?  So you can combine your data with someone else's. If they
         attach a DIFFERENT MEANING to "within", you won't get a
         conflict — their predicates are
              <http://other.org/foo#within>

   The URL doesn't need to resolve to anything. From RDF's point of
   view it is simply a NAMESPACE. You declare the prefix once at the
   top of the file and then forget about it.
```

### Example 2-9 — The same query in SPARQL

**SPARQL** = *SPARQL Protocol and RDF Query Language*, pronounced "sparkle." It **predates Cypher** — and **Cypher's pattern-matching is borrowed from SPARQL**, which is why they look alike.

```sparql
PREFIX : <urn:example:>

SELECT ?personName WHERE {
  ?person :name ?personName.
  ?person :bornIn  / :within* / :name "United States".
  ?person :livesIn / :within* / :name "Europe".
}
```

**Even more concise than Cypher.** The equivalences:

```
   ┌────────────────────────────────────────────────────────────────┐
   │  VARIABLE-LENGTH PATH                                          │
   │                                                                │
   │  Cypher:  (person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (location)│
   │  SPARQL:  ?person :bornIn / :within* ?location.                │
   └────────────────────────────────────────────────────────────────┘
   ┌────────────────────────────────────────────────────────────────┐
   │  MATCHING A PROPERTY                                           │
   │                                                                │
   │  Cypher:  (usa {name:'United States'})                         │
   │  SPARQL:  ?usa :name "United States".                          │
   │                                                                │
   │  ⚑ RDF doesn't distinguish properties from edges — it uses     │
   │    predicates for BOTH — so the SAME SYNTAX works for both.    │
   └────────────────────────────────────────────────────────────────┘

   (Variables start with ? in SPARQL.)
```

> **SPARQL is a nice query language — even if the semantic web never happens, it can be a powerful tool for applications to use internally.**

---

## 16. The foundation: Datalog

Much older than SPARQL or Cypher — **studied extensively by academics in the 1980s.** Less well known among engineers, but important, because **it provides the foundation that later query languages build upon.**

**Why the renewed interest:**

> Every year we have more cores per CPU, more CPUs per machine, more machines per cluster. **As programmers we still haven't figured out a good answer to how best to work with all that parallelism. But it has been suggested that declarative languages based on Datalog may be the future for parallel programming** — which has rekindled interest in Datalog recently.

**In practice:** the query language of **Datomic**, and **Cascalog** is a Datalog implementation for querying large datasets in Hadoop.

**The notation flip:**

```
   Triple-store:   ( subject , predicate , object )
   Datalog:          predicate ( subject , object )
```

### Example 2-10 — The data as Datalog facts

```prolog
name(namerica, 'North America').
type(namerica, continent).

name(usa, 'United States').
type(usa, country).
within(usa, namerica).

name(idaho, 'Idaho').
type(idaho, state).
within(idaho, usa).

name(lucy, 'Lucy').
born_in(lucy, idaho).
```

### Example 2-11 — The same query in Datalog

```prolog
within_recursive(Location, Name) :- name(Location, Name).        /* Rule 1 */

within_recursive(Location, Name) :- within(Location, Via),       /* Rule 2 */
                                    within_recursive(Via, Name).

migrated(Name, BornIn, LivingIn) :- name(Person, Name),          /* Rule 3 */
                                    born_in(Person, BornLoc),
                                    within_recursive(BornLoc, BornIn),
                                    lives_in(Person, LivingLoc),
                                    within_recursive(LivingLoc, LivingIn).

?- migrated(Who, 'United States', 'Europe').
/* Who = 'Lucy'. */
```

### How to read Datalog

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  Cypher and SPARQL jump in right away with SELECT.               │
   │  DATALOG TAKES A SMALL STEP AT A TIME.                           │
   │                                                                  │
   │  We define RULES that tell the database about NEW PREDICATES.    │
   │  Here: `within_recursive` and `migrated`.                        │
   │                                                                  │
   │  These predicates AREN'T TRIPLES STORED IN THE DATABASE —        │
   │  they are DERIVED from data or from other rules.                 │
   │                                                                  │
   │  Rules can refer to other rules, just like functions can call    │
   │  other functions or recurse. Complex queries get BUILT UP        │
   │  A SMALL PIECE AT A TIME.                                        │
   └──────────────────────────────────────────────────────────────────┘

   SYNTAX:
     • Words starting with an UPPERCASE letter are VARIABLES
     • Predicates are matched like in Cypher and SPARQL
     • name(Location, Name) matches the fact
       name(namerica, 'North America') with bindings
       Location = namerica, Name = 'North America'

   THE  :-  OPERATOR  ("if"):
     A rule APPLIES if the system can find a match for ALL predicates
     on the RIGHT-HAND side. When it applies, it's AS THOUGH the
     LEFT-HAND side was ADDED TO THE DATABASE (with variables
     replaced by their matched values).
```

### 🔷 Figure 2-6 — Determining that Idaho is in North America

This is the **fixpoint iteration**: apply the rules repeatedly until nothing new is produced.

```
  ┌─ AFTER RULE 1 ────────────┐   ┌─ AFTER RULE 2 ONCE ───────┐   ┌─ AFTER RULE 2 TWICE ──────┐
  │   within_recursive        │   │   within_recursive        │   │   within_recursive        │
  ├──────────┬────────────────┤   ├──────────┬────────────────┤   ├──────────┬────────────────┤
  │ Location │ Name           │   │ Location │ Name           │   │ Location │ Name           │
  ├──────────┼────────────────┤   ├──────────┼────────────────┤   ├──────────┼────────────────┤
  │ namerica │ North America  │   │ namerica │ North America  │   │ namerica │ North America  │
  │ usa      │ United States  │   │ usa      │ North America ★│   │ usa      │ North America  │
  │ idaho    │ Idaho          │   │ usa      │ United States  │   │ idaho    │ North America ★│
  └──────────┴────────────────┘   │ idaho    │ United States ★│   │ usa      │ United States  │
              │                   │ idaho    │ Idaho          │   │ idaho    │ United States  │
              │                   └──────────┴────────────────┘   │ idaho    │ Idaho          │
              │                               │                   └──────────┴────────────────┘
       uses within(usa,namerica)       uses within(idaho,usa)                  │
       and within(idaho,usa)                                        ✅ ANSWER FOUND:
                                                                   idaho IS IN North America

   ★ = newly derived in this round
```

**The step-by-step reasoning the book gives:**

```
   1. name(namerica, 'North America') exists in the DB
      → Rule 1 applies
      → generates  within_recursive(namerica, 'North America')

   2. within(usa, namerica) exists in the DB, AND
      within_recursive(namerica, 'North America') was generated in step 1
      → Rule 2 applies
      → generates  within_recursive(usa, 'North America')

   3. within(idaho, usa) exists in the DB, AND
      within_recursive(usa, 'North America') was generated in step 2
      → Rule 2 applies
      → generates  within_recursive(idaho, 'North America')
```

**Then Rule 3** finds people born in some `BornIn` and living in some `LivingIn`. Query with `BornIn = 'United States'`, `LivingIn = 'Europe'`, leaving the person as variable `Who` → **`Who = 'Lucy'`**, the same answer as Cypher and SPARQL.

### The trade-off

> The Datalog approach **requires a different kind of thinking** to the other query languages, but it's **very powerful, because rules can be combined and reused in different queries.** It's **less convenient for simple one-off queries, but it can cope better if your data is complex.**

```
   Cypher / SPARQL          Datalog
   ───────────────          ───────
   one query, one answer    reusable RULES that compose
   great for one-offs       great for complex, layered logic
                            (think: views that can recurse
                             and call each other)
```

---

## 17. Chapter Summary

> Data models are a huge subject, and in this chapter we have taken a quick look at a broad variety of different models.

### The historical arc

```
   ┌────────────────────┐
   │  ONE BIG TREE      │   the HIERARCHICAL model
   │  (hierarchical)    │
   └─────────┬──────────┘
             │  ❌ bad at many-to-many
             ▼
   ┌────────────────────┐
   │  RELATIONAL        │   invented to solve exactly that problem
   └─────────┬──────────┘
             │  ❌ some applications don't fit well either
             ▼
   ┌─────────────────────────────────────────────────────┐
   │  NoSQL diverged in TWO OPPOSITE DIRECTIONS:         │
   │                                                     │
   │  ①  DOCUMENT DATABASES                              │
   │     data comes in SELF-CONTAINED DOCUMENTS, and     │
   │     relationships between documents are RARE        │
   │                                                     │
   │  ②  GRAPH DATABASES  ← the opposite direction       │
   │     ANYTHING IS POTENTIALLY RELATED TO EVERYTHING   │
   └─────────────────────────────────────────────────────┘
```

> **All three models are widely used today, and each is good in its respective domain. One model can be emulated in terms of another, but the result is often awkward. That's why we have different systems for different purposes, not a single one-size-fits-all solution.**

**What document and graph databases share:** they typically **don't enforce a schema**, which can make it easier to adapt applications to changing requirements.

### The query languages covered

| Language | Model | Style |
|---|---|---|
| **SQL** | Relational | Declarative |
| **MapReduce** | Document / distributed | In between |
| **Aggregation pipeline** | Document (MongoDB) | Declarative |
| **Cypher** | Property graph | Declarative |
| **SPARQL** | Triple-store / RDF | Declarative |
| **Datalog** | Triple-store (generalized) | Declarative, rule-based |
| *CSS, XSL/XPath* | *not databases* | *Declarative — interesting parallels* |

### Models the chapter deliberately leaves out

```
   🧬 GENOME DATA — researchers need SEQUENCE-SIMILARITY SEARCHES:
      take one very long string (a DNA molecule) and match it against
      a large database of strings that are SIMILAR BUT NOT IDENTICAL.
      No database described here handles this → specialized software
      like GenBank.

   ⚛️ PARTICLE PHYSICS — the LHC works with HUNDREDS OF PETABYTES.
      At that scale, custom solutions are required to stop hardware
      cost spiralling out of control.

   🔎 FULL-TEXT SEARCH — arguably a data model in its own right,
      frequently used alongside databases. Information Retrieval is a
      large specialist subject; the book touches search indexes in
      Chapter 3 and Part III.
```

---
---

# 18. 💻 CODE APPENDIX — everything runnable

All output below came from actually executing this code.

## 18.1 The property graph in SQLite + the recursive CTE, for real

SQLite supports `WITH RECURSIVE` and `json_extract`, so Example 2-2 and Example 2-5 run unmodified in spirit:

```python
import sqlite3, json

db = sqlite3.connect(":memory:")
db.executescript("""
CREATE TABLE vertices (vertex_id INTEGER PRIMARY KEY, properties TEXT);
CREATE TABLE edges (
    edge_id     INTEGER PRIMARY KEY,
    tail_vertex INTEGER REFERENCES vertices(vertex_id),
    head_vertex INTEGER REFERENCES vertices(vertex_id),
    label       TEXT,
    properties  TEXT
);
CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
""")

V = {}
def vertex(key, **props):
    vid = len(V) + 1
    V[key] = vid
    db.execute("INSERT INTO vertices VALUES (?,?)", (vid, json.dumps(props)))

def edge(tail, label, head):
    db.execute("INSERT INTO edges (tail_vertex, head_vertex, label, properties)"
               " VALUES (?,?,?,?)", (V[tail], V[head], label, "{}"))

# All of Figure 2-5
vertex('namerica',  type='continent',   name='North America')
vertex('europe',    type='continent',   name='Europe')
vertex('usa',       type='country',     name='United States')
vertex('uk',        type='country',     name='United Kingdom')
vertex('france',    type='country',     name='France')
vertex('idaho',     type='state',       name='Idaho', abbreviation='ID')
vertex('england',   type='country',     name='England')
vertex('bourgogne', type='region',      name_fr='Bourgogne', name_en='Burgundy')
vertex('london',    type='city',        name='London')
vertex('cotedor',   type='departement', name="Côte-d'Or")
vertex('beaune',    type='city',        name='Beaune')
vertex('lucy',      type='person',      name='Lucy')
vertex('alain',     type='person',      name='Alain')

for a, b in [('usa','namerica'), ('uk','europe'), ('france','europe'),
             ('idaho','usa'), ('england','uk'), ('bourgogne','france'),
             ('london','england'), ('cotedor','bourgogne'), ('beaune','cotedor')]:
    edge(a, 'within', b)

edge('lucy','born_in','idaho');  edge('lucy','lives_in','london')
edge('alain','born_in','beaune'); edge('alain','lives_in','london')
edge('lucy','married','alain')
```

The query (Example 2-5, adapted to SQLite's `json_extract`):

```sql
WITH RECURSIVE
  in_usa(vertex_id) AS (
      SELECT vertex_id FROM vertices
       WHERE json_extract(properties,'$.name') = 'United States'
    UNION
      SELECT e.tail_vertex FROM edges e
        JOIN in_usa ON e.head_vertex = in_usa.vertex_id
       WHERE e.label = 'within'
  ),
  in_europe(vertex_id) AS (
      SELECT vertex_id FROM vertices
       WHERE json_extract(properties,'$.name') = 'Europe'
    UNION
      SELECT e.tail_vertex FROM edges e
        JOIN in_europe ON e.head_vertex = in_europe.vertex_id
       WHERE e.label = 'within'
  ),
  born_in_usa(vertex_id) AS (
      SELECT e.tail_vertex FROM edges e
        JOIN in_usa ON e.head_vertex = in_usa.vertex_id
       WHERE e.label = 'born_in'
  ),
  lives_in_europe(vertex_id) AS (
      SELECT e.tail_vertex FROM edges e
        JOIN in_europe ON e.head_vertex = in_europe.vertex_id
       WHERE e.label = 'lives_in'
  )
SELECT json_extract(v.properties,'$.name')
FROM vertices v
JOIN born_in_usa     b ON v.vertex_id = b.vertex_id
JOIN lives_in_europe l ON v.vertex_id = l.vertex_id;
```

**Actual output:**

```
Emigrated US -> Europe: ['Lucy']

Transitive closure of 'within Europe':
  ['?', 'Beaune', "Côte-d'Or", 'England', 'Europe', 'France', 'London', 'United Kingdom']
```

✅ **Lucy, and only Lucy.** Alain lives in Europe but was born in Beaune, so he's correctly excluded.

😄 **And notice the `'?'` in the second result.** That's **Bourgogne** — it has `name_fr` and `name_en` but no `name` property, so `json_extract(...,'$.name')` returns NULL. This is **schema-on-read biting you in real time**, exactly as §7.2 warns: the database happily stored a vertex with a different shape, and only the *reading* code discovered the mismatch. In a schema-on-write system, that column would have been declared and the problem caught at insert.

## 18.2 Datalog fixpoint evaluation — reproducing Figure 2-6

```python
facts_name   = {'namerica': 'North America', 'usa': 'United States', 'idaho': 'Idaho'}
facts_within = [('usa', 'namerica'), ('idaho', 'usa')]

wr = set()                     # the derived predicate within_recursive(Location, Name)

# Rule 1:  within_recursive(Loc, Name) :- name(Loc, Name).
for loc, nm in facts_name.items():
    wr.add((loc, nm))

# Rule 2:  within_recursive(Loc, Name) :- within(Loc, Via), within_recursive(Via, Name).
for iteration in (1, 2, 3):
    new = {(loc, nm) for (loc, via) in facts_within
                     for (v, nm)    in wr
                     if v == via} - wr
    if not new:
        print(f"iteration {iteration}: FIXPOINT reached")
        break
    wr |= new
    print(f"iteration {iteration}: derived {sorted(new)}")
```

**Actual output — compare it to Figure 2-6 above:**

```
After Rule 1    : [('idaho','Idaho'), ('namerica','North America'), ('usa','United States')]
After Rule 2 x1 : + ('usa','North America'), ('idaho','United States')
After Rule 2 x2 : + ('idaho','North America')            ← the answer appears
After Rule 2 x3 : fixpoint reached, nothing new

=> Is Idaho in North America?  True
```

**Three rounds, then it stops.** That "stops" is the whole trick of recursive query evaluation: you iterate until the derived set stops growing (the *fixpoint*). It's exactly what `WITH RECURSIVE`'s `UNION` does under the hood — `UNION` deduplicates, so when a round adds nothing new, the recursion terminates.

## 18.3 MapReduce vs aggregation pipeline, side by side

```python
from collections import defaultdict

observations = [
    {"ts":"1995-12-25","family":"Sharks","species":"Carcharodon carcharias","numAnimals":3},
    {"ts":"1995-12-12","family":"Sharks","species":"Carcharias taurus",      "numAnimals":4},
    {"ts":"1996-01-03","family":"Sharks","species":"Carcharias taurus",      "numAnimals":2},
    {"ts":"1995-12-20","family":"Whales","species":"Balaenoptera musculus",  "numAnimals":1},
]

# ── MapReduce: TWO carefully coordinated functions ──────────────────
def mapper(doc):
    y, m, _ = doc["ts"].split("-")
    yield f"{y}-{int(m)}", doc["numAnimals"]      # emit(key, value)

def reducer(key, values):
    return sum(values)

buckets = defaultdict(list)
for doc in observations:
    if doc["family"] == "Sharks":                 # the declarative `query:` filter
        for k, v in mapper(doc):
            buckets[k].append(v)                  # framework groups by key
result = {k: reducer(k, vs) for k, vs in buckets.items()}

# ── Aggregation pipeline: declarative STAGES ────────────────────────
def pipeline(docs, stages):
    for stage in stages:
        docs = stage(docs)
    return docs

match_sharks = lambda ds: [d for d in ds if d["family"] == "Sharks"]

def group_by_month(ds):
    acc = defaultdict(int)
    for d in ds:
        y, m, _ = d["ts"].split("-")
        acc[(int(y), int(m))] += d["numAnimals"]
    return [{"_id": {"year": y, "month": m}, "totalAnimals": t}
            for (y, m), t in sorted(acc.items())]

pipeline(observations, [match_sharks, group_by_month])
```

**Actual output:**

```
MapReduce           : {'1995-12': 7, '1996-1': 2}
   reduce('1995-12', [3, 4]) -> 7        ← exactly the book's worked example
   reduce('1996-1',  [2])    -> 2

Aggregation pipeline: [{'_id': {'year':1995,'month':12}, 'totalAnimals': 7},
                       {'_id': {'year':1996,'month':1},  'totalAnimals': 2}]
```

`reduce("1995-12", [3, 4]) → 7` is precisely the trace given in the book. Note how the pipeline version is just **a list of stages you compose** — no coordination between two functions required. That's the usability argument for declarative, made concrete.

## 18.4 The same data in all three models — a direct comparison

```python
# ─── DOCUMENT MODEL ──────────────────────────────────────────────────
# One self-contained tree. One read gets everything. No joins possible.
profile = {
    "user_id": 251, "first_name": "Bill", "last_name": "Gates",
    "region_id": "us:91",             # ← a REFERENCE; resolving it needs a join
    "positions": [                     # ← nested one-to-many: FREE
        {"job_title": "Co-chair",             "organization": "Gates Foundation"},
        {"job_title": "Co-founder, Chairman", "organization": "Microsoft"},
    ],
}

# ─── RELATIONAL MODEL ────────────────────────────────────────────────
users     = [(251, "Bill", "Gates", "us:91")]
positions = [(458, 251, "Co-chair", "Gates Foundation"),
             (457, 251, "Co-founder, Chairman", "Microsoft")]
regions   = [("us:91", "Greater Seattle Area")]
# one-to-many AND many-to-one both handled by the same mechanism: a JOIN

# ─── GRAPH MODEL ─────────────────────────────────────────────────────
vertices = {1: {"type":"person",  "name":"Bill Gates"},
            2: {"type":"company", "name":"Microsoft"},
            3: {"type":"region",  "name":"Greater Seattle Area"}}
edges    = [(1, "WORKED_AT", 2, {"title":"Co-founder, Chairman"}),
            (1, "LIVES_IN",  3, {})]
# note: the edge itself carries PROPERTIES — neither of the other
# models can attach data to the relationship without an extra table
```

**The thing to notice in that last comment.** Property graphs let you put attributes **on the edge**. In relational you need a join table with extra columns; in document you have nowhere natural to put it at all. That's a genuine expressiveness difference, not just syntax.

---

# 19. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.2 — DATA MODELS AND QUERY LANGUAGES                                  ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║                                                                               ║
║  LAYERS: real world → objects/APIs → GENERAL DATA MODEL → bytes → electrons   ║
║  Each layer hides the one below. This chapter = layer 3. Ch.3 = layer 4.      ║
║                                                                               ║
║  ── THE THREE MODELS ────────────────────────────────────────────────────────  ║
║  DOCUMENT    tree of one-to-many. Locality win. Weak joins. Shredding avoided.║
║  RELATIONAL  Codd 1970. Unordered tuples. Joins easy. Query optimizer.        ║
║  GRAPH       anything → anything. Vertices + edges. Heterogeneous. Evolvable. ║
║                                                                               ║
║  DECIDE BY THE SHAPE OF YOUR RELATIONSHIPS:                                   ║
║     one-to-many only → document | some m-to-m → relational | dense → graph    ║
║  "For highly interconnected data: document AWKWARD, relational ACCEPTABLE,    ║
║   graph MOST NATURAL."                                                        ║
║                                                                               ║
║  ── KEY CONCEPTS ────────────────────────────────────────────────────────────  ║
║  IMPEDANCE MISMATCH  OO objects vs tables. ORMs reduce but don't remove it.   ║
║  NORMALIZATION       remove duplication. Rule of thumb: if a value could live ║
║                      in one place and doesn't, it's not normalized.           ║
║                      IDs never change; human-meaningful text does.            ║
║  SHREDDING           splitting a document across tables → cumbersome schemas  ║
║  LOCALITY            one contiguous read, BUT: loads whole doc, rewrites whole║
║                      doc on update → KEEP DOCUMENTS SMALL                     ║
║  SCHEMA-ON-READ      implicit, checked when read  ≈ dynamic typing            ║
║  SCHEMA-ON-WRITE     explicit, enforced on write  ≈ static typing             ║
║                      "schemaless" is misleading — the schema is IMPLICIT      ║
║                      ALTER TABLE is ms on most DBs (MySQL copies the table)   ║
║                                                                               ║
║  ── HISTORY ─────────────────────────────────────────────────────────────────  ║
║  IMS (1968, Apollo) hierarchical = tree = today's document DBs, same problems ║
║  CODASYL network    multiple parents; ACCESS PATHS chosen BY HAND; imperative ║
║  RELATIONAL won     data "in the open"; the QUERY OPTIMIZER is built ONCE and ║
║                     every application benefits. New query? Just add an index. ║
║  Graph DBs are NOT CODASYL: no schema restriction, direct vertex access by ID,║
║                     no forced ordering, declarative languages available.      ║
║  Codd's 1970 paper already had "nonsimple domains" ≈ nested JSON. 30 yrs early║
║                                                                               ║
║  ── DECLARATIVE > IMPERATIVE ────────────────────────────────────────────────  ║
║  ① concise  ② hides implementation → DB optimizes without changing queries    ║
║  ③ parallelizable (imperative fixes an order; declarative fixes only a result)║
║  "Being MORE LIMITED gives the database MORE ROOM to optimize."               ║
║  Web analogy: CSS/XPath (declarative) beats DOM manipulation (imperative) —   ║
║  imperative styles don't un-apply, and you don't get free vendor speedups.    ║
║                                                                               ║
║  ── QUERY LANGUAGES ─────────────────────────────────────────────────────────  ║
║  SQL          declarative, relational algebra lineage                         ║
║  MapReduce    between the two: code snippets called by a framework.           ║
║               map+reduce must be PURE → runnable anywhere, any order, retryable║
║  Agg pipeline MongoDB's declarative answer → "accidentally reinventing SQL"   ║
║  CYPHER       Neo4j. (a)-[:LABEL]->(b). *0.. = zero or more hops. 4 lines.    ║
║  SQL RECURSIVE  WITH RECURSIVE does the same job in 29 lines.                 ║
║  SPARQL       RDF triples. Predates Cypher (Cypher borrowed its matching).    ║
║               No property/edge distinction → same syntax for both.            ║
║  DATALOG      predicate(subject, object). RULES with :- that COMPOSE and      ║
║               RECURSE. Fixpoint iteration. Worse for one-offs, better for     ║
║               complex layered logic. Foundation the others build on.          ║
║                                                                               ║
║  TRIPLE-STORE ≡ property graph, different words.                              ║
║     (lucy, age, 33)          → a PROPERTY on vertex lucy                      ║
║     (lucy, marriedTo, alain) → an EDGE labelled marriedTo                     ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 20. ✅ Test yourself

1. **Why is `region_id: "us:91"` better than `region: "Greater Seattle Area"`?**
   → It's a normalization question. The ID has no meaning to humans so it never needs to change; the human-readable name lives in exactly one place, making renames, localization, and hierarchy-aware search possible. Storing the string duplicates it into every record.

2. **What exactly does a document database do well, and what breaks it?**
   → Well: trees of one-to-many relationships loaded as a whole (locality, no shredding). Breaks: many-to-many, which forces either denormalization (app must maintain consistency) or emulated joins (slower, complexity moves into your code).

3. **"Document databases are schemaless." What's wrong with that sentence?**
   → The reading code always assumes *some* structure, so there is an implicit schema — it just isn't enforced by the database. The accurate term is **schema-on-read**.

4. **In what sense are document databases repeating IMS's mistakes, and in what sense aren't they?**
   → They are, in reverting to nested records with weak join support — the same 1960s dilemma of duplicate-vs-manually-resolve. They aren't CODASYL, though: they resolve references at read time by unique identifier, exactly like foreign keys, rather than via hand-maintained access paths.

5. **Why does limiting SQL's expressiveness make it faster?**
   → Because the database can't tell whether imperative code depends on row order, timing, or side effects, so it can't safely reorganize anything. A declarative query only states the desired result, leaving the engine free to reorder, re-index, and parallelize.

6. **Why must MapReduce's `map` and `reduce` be pure functions?**
   → So the framework can run them anywhere, in any order, and re-run them after a failure. Purity is what buys relocatable, retryable distributed execution.

7. **What does `-[:WITHIN*0..]->` do, and why can't a normal SQL join do it?**
   → It follows a `WITHIN` edge zero or more times. A normal join has a fixed arity decided when you write the query; graph traversals need a number of hops that isn't known in advance. SQL needs `WITH RECURSIVE` for this.

8. **Give the property-graph equivalent of `(lucy, age, 33)` and `(lucy, marriedTo, alain)`.**
   → The first is a property on the `lucy` vertex: `{"age": 33}`. The second is an edge labelled `marriedTo` with `lucy` as tail and `alain` as head. The difference is whether the object is a primitive or another vertex.

9. **Why is Datalog worse for a one-off query and better for complex data?**
   → It makes you define derived predicates before you can ask anything, which is overhead for a single question. But those rules compose, recurse, and get reused across queries — so complex logic gets built up in reusable pieces instead of one enormous query.

---

*All quoted material, figures, examples and code snippets are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 2. Diagrams have been redrawn in ASCII from the book's originals. The code appendix and its outputs are supplementary — the results shown come from actually running the code.*
