# DDIA — Chapter 4: Encoding and Evolution
### Complete study guide — theory, every diagram redrawn, working encoders, and what changed since 2017

> *Everything changes and nothing stands still.*
> — Heraclitus of Ephesus, as quoted by Plato in *Cratylus* (360 BC)

---

## 0. The map of this chapter

Chapter 1 introduced **evolvability**. Chapter 4 is the concrete, byte-level answer to *how you actually achieve it*.

```
┌───────────────────────────────────────────────────────────────────────────┐
│                                                                           │
│  PART A — FORMATS FOR ENCODING DATA                                       │
│  ──────────────────────────────────                                       │
│    language-specific  →  JSON/XML/CSV  →  binary JSON  →  SCHEMA-DRIVEN   │
│    (Java, pickle)        (textual)        (MessagePack)   Thrift          │
│                                                            Protobuf       │
│                                                            Avro           │
│                                                                           │
│  PART B — MODES OF DATA FLOW                                              │
│  ────────────────────────────                                             │
│    ① via DATABASES        write now, read later ("message to your future  │
│                            self")                                         │
│    ② via SERVICES         REST, SOAP, RPC                                 │
│    ③ via MESSAGE PASSING  brokers, actors                                 │
│                                                                           │
│  THE THREAD RUNNING THROUGH ALL OF IT:                                    │
│      BACKWARD COMPATIBILITY  +  FORWARD COMPATIBILITY                     │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Why compatibility matters: code changes are never instantaneous

> Applications inevitably change over time… In most cases, a change of application features **also requires a change to the data it stores.**

The two data models from Chapter 2 handle this differently:

```
   RELATIONAL                            SCHEMA-ON-READ ("schemaless")
   ──────────                            ─────────────────────────────
   Assumes ALL data conforms to          Doesn't enforce a schema, so the
   ONE schema. It can be changed         database CAN CONTAIN A MIXTURE of
   (ALTER statements), but there is      older and newer data formats,
   EXACTLY ONE schema in force at        written at different times.
   any one point in time.
```

**But here is the real problem, and it's organizational, not technical:**

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │  SERVER-SIDE — the ROLLING UPGRADE (staged rollout)                 │
   │                                                                     │
   │   deploy the new version to A FEW NODES AT A TIME, check it's       │
   │   running smoothly, gradually work through all the nodes            │
   │                                                                     │
   │   ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                         │
   │   │NEW │ │NEW │ │old │ │old │ │old │ │old │  ← mid-deploy, RIGHT NOW │
   │   └────┘ └────┘ └────┘ └────┘ └────┘ └────┘                         │
   │                                                                     │
   │   ➜ no service downtime → encourages MORE FREQUENT RELEASES         │
   │     and better evolvability                                         │
   ├─────────────────────────────────────────────────────────────────────┤
   │  CLIENT-SIDE                                                        │
   │   "you're at the mercy of the user, who may not install the update  │
   │    for some time"  (…or ever)                                       │
   └─────────────────────────────────────────────────────────────────────┘
```

### 🔑 The two directions of compatibility

```
                                TIME ────────────────────────────►

                        ┌──────────────┐              ┌──────────────┐
                        │  OLD CODE    │              │  NEW CODE    │
                        └──────┬───────┘              └──────┬───────┘
                               │                             │
             writes            │                             │  writes
             OLD DATA          ▼                             ▼  NEW DATA
                        ┌─────────────────────────────────────────────┐
                        │                  DATA                       │
                        └─────────────────────────────────────────────┘
                               │                             │
                               │                             │
   ╔═══════════════════════════▼═════════╗   ╔═══════════════▼═══════════════════╗
   ║  BACKWARD COMPATIBILITY             ║   ║  FORWARD COMPATIBILITY            ║
   ║  NEWER code can read data written   ║   ║  OLDER code can read data written ║
   ║  by OLDER code                      ║   ║  by NEWER code                    ║
   ╠═════════════════════════════════════╣   ╠═══════════════════════════════════╣
   ║  ✅ NORMALLY NOT HARD                ║   ║  ⚠️  TRICKIER                      ║
   ║                                     ║   ║                                   ║
   ║  As author of the newer code, you   ║   ║  It requires OLDER code to IGNORE ║
   ║  KNOW the format written by older   ║   ║  ADDITIONS made by a newer        ║
   ║  code, so you can explicitly handle ║   ║  version — code that was written  ║
   ║  it (if necessary by keeping the    ║   ║  before those additions existed   ║
   ║  old code around to read old data). ║   ║  and knows nothing about them.    ║
   ╚═════════════════════════════════════╝   ╚═══════════════════════════════════╝
```

**A memory aid that actually helps:** point the arrow at the *data*. Backward compatibility means reaching **back** in time to read old data. Forward compatibility means old code coping with data from the **future**.

---

# PART A — FORMATS FOR ENCODING DATA

## 2. The two representations, and the translation between them

```
   ┌──────────────────────────────┐         ┌──────────────────────────────┐
   │  ① IN MEMORY                 │         │  ② SELF-CONTAINED BYTES      │
   │                              │         │                              │
   │  objects, structs, lists,    │ ENCODE  │  a sequence of bytes         │
   │  arrays, hash tables, trees  │ ──────► │  (e.g. a JSON document)      │
   │                              │         │                              │
   │  optimized for efficient     │ ◄────── │  for writing to a file or    │
   │  access and manipulation     │ DECODE  │  sending over a network      │
   │  by the CPU — TYPICALLY      │         │                              │
   │  USING POINTERS              │         │  ⚑ A POINTER WOULDN'T MAKE   │
   │                              │         │    SENSE to another process  │
   └──────────────────────────────┘         └──────────────────────────────┘

   ENCODE  = serialization = marshalling
   DECODE  = parsing = deserialization = unmarshalling
```

> ⚠️ **Terminology clash:** *serialization* is also used in the context of **transactions** (Chapter 7) with a completely different meaning. The book sticks with **encoding**.

> ⚠️ **And a second clarification worth stating loudly:** *"encoding has nothing to do with encryption."*

---

## 3. Language-specific formats — and why not to use them

`java.io.Serializable`, Ruby's `Marshal`, Python's `pickle`, Kryo for Java.

> These are very convenient, because they allow in-memory objects to be saved and restored **with minimal additional code.** However, they also have **a number of deep problems**:

```
   ❌ ① LOCKED TO ONE LANGUAGE
      Reading the data in another language is very difficult. You commit
      yourself to your current programming language for potentially a very
      long time, and preclude integrating with other organizations' systems.

   ❌ ② SECURITY — this one is serious
      To restore data in the same object types, the decoder must be able to
      INSTANTIATE ARBITRARY CLASSES. If an attacker can get your application
      to decode an arbitrary byte sequence, they can instantiate arbitrary
      classes — which often allows REMOTE ARBITRARY CODE EXECUTION.

   ❌ ③ VERSIONING IS AN AFTERTHOUGHT
      Built for quick and easy encoding; they neglect the inconvenient
      problems of forward and backward compatibility.

   ❌ ④ EFFICIENCY IS AN AFTERTHOUGHT
      Java's built-in serialization is "notorious for its bad performance
      and bloated encoding."

   ➜ VERDICT: "generally a bad idea to use your language's built-in
     encoding for anything other than very transient purposes."
```

> **📌 2026 UPDATE — problem ② got worse before it got better.** Insecure deserialization has been a fixture of the OWASP Top 10 for a decade (now folded into **A08: Software and Data Integrity Failures**). Since the book was written: the 2017 Equifax breach, a long tail of Java gadget-chain CVEs, and Log4Shell (2021) all traced back to the same root cause — *turning untrusted bytes into live objects.*
>
> The ecosystem has responded structurally rather than with patches. Java added **serialization filters** (JEP 290, Java 9) and has an official plan to **remove `Serializable` entirely** (JEP 154 deprecated the old mechanism; Project Amber is designing a replacement). Python's `pickle` docs now open with a red security warning. The modern rule is stronger than Kleppmann's: **never deserialize untrusted input into arbitrary types, in any language, ever.** Prefer formats where decoding produces plain data, not live objects.

---

## 4. JSON, XML and CSV

> They are **widely known, widely supported, and almost as widely disliked.** XML is often criticised for being too verbose and unnecessarily complicated. JSON's popularity is mainly due to **built-in support by web browsers** (being a subset of JavaScript) and simplicity relative to XML.

### The subtle problems (beyond the syntax arguments)

```
   ⚠️ ① AMBIGUITY AROUND NUMBERS
      XML and CSV : cannot distinguish a NUMBER from a STRING of digits
                    (except via an external schema)
      JSON        : distinguishes strings and numbers, BUT does not
                    distinguish INTEGER from FLOATING-POINT, and does
                    NOT SPECIFY A PRECISION
```

**The Twitter example is the perfect illustration and worth knowing by heart:**

```
   Integers greater than 2^53 cannot be exactly represented in an IEEE 754
   double-precision float. JavaScript uses doubles for all numbers.

   Twitter uses a 64-BIT NUMBER to identify each tweet.

   ➜ THE WORKAROUND: Twitter's API returns each tweet ID TWICE —
        {
          "id":     1234567890123456789,   ← JSON number  (corrupted in JS!)
          "id_str": "1234567890123456789"  ← decimal STRING (safe)
        }

   An entire public API carries a duplicate field forever, because a
   data format didn't specify numeric precision.
```

```
   ⚠️ ② NO BINARY STRINGS
      JSON and XML have good Unicode support, but NO support for binary
      strings (byte sequences without a character encoding).
      Workaround: Base64-encode the binary as text, and use the schema to
      say "interpret this as Base64."
      "This works, but it's somewhat hacky."
      ➜ AND IT COSTS YOU 33% SIZE OVERHEAD.

   ⚠️ ③ SCHEMA SUPPORT IS OPTIONAL AND COMPLICATED
      Both XML Schema and JSON Schema exist. They are "quite powerful, and
      thus quite complicated to learn and implement." XML schemas are fairly
      widespread; MANY JSON-BASED TOOLS DON'T BOTHER.
      ⚑ Since correct interpretation of data DEPENDS on the schema,
        applications that skip schemas need extra code to encode/decode
        correctly.

   ⚠️ ④ CSV HAS NO SCHEMA AT ALL
      The application defines the meaning of each row and column. Add a
      column and you handle it manually. And CSV is VAGUE — what happens
      if a value contains a comma or a newline? Escaping rules HAVE been
      formally specified (RFC 4180), but NOT ALL PARSERS IMPLEMENT IT.
```

### 🔑 But the book defends them, and the reason is sociological

> Despite these flaws, JSON, XML and CSV are **good enough for many purposes.** It's likely they will remain popular, **especially as data interchange formats** (sending data from one organization to another). In these situations, **as long as people agree on what the format is, it often doesn't matter how pretty or efficient the format is. The difficulty of getting different organizations to agree on anything outweighs most other concerns.**

That last sentence is the most quotable line in the chapter. **Format choice is often a coordination problem, not an engineering one.**

> **📌 2026 UPDATE — JSON Schema grew up.** In 2017 JSON Schema was an expired IETF draft that "many tools don't bother using." Today **JSON Schema 2020-12** is stable and near-universal: it's the type system underneath **OpenAPI 3.1**, and it's how you constrain **LLM structured outputs** (OpenAI, Anthropic, and Google all take JSON Schema for tool/function definitions). Kleppmann's "sometimes helpful, sometimes a hindrance" verdict has tilted noticeably toward helpful.
>
> Two other developments: **JSON5** and **JSONC** added comments and trailing commas for config files (the `tsconfig.json` problem), and **`BigInt`** landed in JavaScript (ES2020) — though it still doesn't survive `JSON.parse`, so Twitter's `id_str` hack remains necessary. The 2^53 problem is 15 years old and still unsolved at the format level.

---

## 5. Binary encodings of JSON

> For data used **only internally within your organization**, there is less pressure to use a lowest-common-denominator format. For a small dataset the gains are negligible, **but once you get into the terabytes, the choice of data format can have a big impact.**

```
   A PROFUSION of binary JSON encodings:
      MessagePack, BSON, BJSON, UBJSON, BISON, Smile
   And for XML:
      WBXML, Fast Infoset

   "These formats have been adopted in various niches, but NONE of them
    are as widely adopted as the textual versions of JSON and XML."

   ⚑ THE CRITICAL LIMITATION:
     Since they DON'T PRESCRIBE A SCHEMA, they must include
     ALL OBJECT FIELD NAMES within the encoded data.
```

### Example 4-1 — the record encoded throughout the chapter

```json
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": ["daydreaming", "hacking"]
}
```

### 🔷 Figure 4-1 — MessagePack (66 bytes)

**I implemented MessagePack from scratch. The output is byte-for-byte identical to the book:**

```
83 a8 75 73 65 72 4e 61 6d 65 a6 4d 61 72 74 69 6e ae 66 61
76 6f 72 69 74 65 4e 75 6d 62 65 72 cd 05 39 a9 69 6e 74 65
72 65 73 74 73 92 ab 64 61 79 64 72 65 61 6d 69 6e 67 a7 68
61 63 6b 69 6e 67                                    → 66 bytes ✓
```

**The byte-by-byte breakdown:**

```
   ┌──────┬──────────────────┬─────────────────────────────────────────────┐
   │ byte │ meaning          │ how it decomposes                           │
   ├──────┼──────────────────┼─────────────────────────────────────────────┤
   │  83  │ object,3 entries │ top 4 bits = 0x80 (object)                  │
   │      │                  │ bottom 4 bits = 0x03 (three fields)         │
   │  a8  │ string, length 8 │ top 4 bits = 0xa0 (string)                  │
   │      │                  │ bottom 4 bits = 0x08 (8 bytes long)         │
   │ 75…65│ "userName"       │ 8 bytes ASCII. NO terminator, NO escaping   │
   │      │                  │ needed — the length was given in advance    │
   │  a6  │ string, length 6 │                                             │
   │ 4d…6e│ "Martin"         │                                             │
   │  ae  │ string,length 14 │ "favoriteNumber"                            │
   │  cd  │ uint16 follows   │ then 05 39 = 1337                           │
   │  a9  │ string, length 9 │ "interests"                                 │
   │  92  │ array, 2 entries │ 0x90 = array, 0x02 = two items              │
   │  ab  │ string,length 11 │ "daydreaming"                               │
   │  a7  │ string, length 7 │ "hacking"                                   │
   └──────┴──────────────────┴─────────────────────────────────────────────┘

   ⚑ What if an object has MORE THAN 15 FIELDS, so the count won't fit in
     4 bits? It gets a DIFFERENT TYPE INDICATOR, and the count is encoded
     in 2 or 4 bytes.
```

### 😐 The disappointing verdict

```
   textual JSON (whitespace removed) : 81 bytes
   MessagePack                       : 66 bytes
   ────────────────────────────────────────────
   saving                            : 15 bytes (18%)

   "It's not clear whether such a small space reduction (and perhaps a
    speedup in parsing) is worth the loss of human-readability."

   ➜ WHY SO LITTLE? Because "userName", "favoriteNumber" and "interests"
     — 31 bytes of field names — are STILL IN THERE.

   In the following sections we will encode the SAME record in 32 BYTES.
```

**I verified the field names really are still present in the binary:** searching the 66-byte output for the literal strings `userName`, `favoriteNumber` and `interests` finds all three. **That's the entire problem, and it's what a schema solves.**

---

## 6. Thrift and Protocol Buffers

> Apache **Thrift** (originally Facebook) and **Protocol Buffers** (originally Google), both open-sourced in 2007–08. Both **require a schema** for any data that is encoded.

### The two schema languages, side by side

**Thrift IDL:**

```thrift
struct Person {
  1: required string       userName,
  2: optional i64          favoriteNumber,
  3: optional list<string> interests
}
```

**Protocol Buffers:**

```protobuf
message Person {
    required string user_name       = 1;
    optional int64  favorite_number = 2;
    repeated string interests       = 3;
}
```

Both come with a **code generation tool** that turns the schema into classes in your language of choice.

### 🔷 Figure 4-2 — Thrift BinaryProtocol (59 bytes)

> Confusingly, Thrift has **two** binary encoding formats… *(footnote: actually three — BinaryProtocol, CompactProtocol and DenseProtocol, though DenseProtocol is C++-only so doesn't count as cross-language. Plus two JSON-based formats. "What fun!")*

**My implementation, byte-identical to the book:**

```
0b 00 01 00 00 00 06 4d 61 72 74 69 6e 0a 00 02 00 00 00 00
00 00 05 39 0f 00 03 0b 00 00 00 02 00 00 00 0b 64 61 79 64
72 65 61 6d 69 6e 67 00 00 00 07 68 61 63 6b 69 6e 67 00
                                                     → 59 bytes ✓
```

```
   ┌────────────────────────────────────────────────────────────────────┐
   │ 0b          type 11 (string)                                       │
   │ 00 01       FIELD TAG = 1        ← not the name "userName"!        │
   │ 00 00 00 06 length 6                                               │
   │ 4d 61 72 74 69 6e                "Martin"                          │
   ├────────────────────────────────────────────────────────────────────┤
   │ 0a          type 10 (i64)                                          │
   │ 00 02       FIELD TAG = 2                                          │
   │ 00 00 00 00 00 00 05 39          1337 in a FULL 8 BYTES            │
   ├────────────────────────────────────────────────────────────────────┤
   │ 0f          type 15 (list)                                         │
   │ 00 03       FIELD TAG = 3                                          │
   │ 0b          item type 11 (string)                                  │
   │ 00 00 00 02 2 list items                                           │
   │   00 00 00 0b  "daydreaming"                                       │
   │   00 00 00 07  "hacking"                                           │
   ├────────────────────────────────────────────────────────────────────┤
   │ 00          END OF STRUCT                                          │
   └────────────────────────────────────────────────────────────────────┘
```

### 🔑 The big difference from MessagePack

> **There are no field names.** Instead, the encoded data contains **field tags** — the numbers 1, 2 and 3 from the schema. **Field tags are like aliases for fields — a compact way of saying what field we're talking about, without having to spell out the field name.**

**I confirmed this programmatically:** searching the 59-byte output for `userName`, `favoriteNumber` or `interests` finds **none of them.** 31 bytes of field names, gone.

### 🔷 Figure 4-3 — Thrift CompactProtocol (34 bytes)

Semantically equivalent to BinaryProtocol, but **packs the same information into 34 bytes** via two tricks:

```
   TRICK ① PACK FIELD TYPE AND TAG DELTA INTO A SINGLE BYTE

      ┌───────────────┬───────────────┐
      │  4 bits       │  4 bits       │
      │  tag DELTA    │  field type   │      one byte instead of three
      └───────────────┴───────────────┘

      0 0 0 1 1 0 0 0  = 0x18  → tag +=1 (so tag 1), type 8 (string)
      0 0 0 1 0 1 1 0  = 0x16  → tag +=1 (so tag 2), type 6 (i64)
      0 0 0 1 1 0 0 1  = 0x19  → tag +=1 (so tag 3), type 9 (list)

   TRICK ② VARIABLE-LENGTH INTEGERS

      Rather than a full 8 bytes for 1337, use 2 bytes, with the TOP BIT
      of each byte indicating WHETHER MORE BYTES FOLLOW.

           f2       =  1 1 1 1 0 0 1 0
                       ▲ └──────────┘
                    "more"   7 data bits
           14       =  0 0 0 1 0 1 0 0
                       ▲ └──────────┘
                     "last"  7 data bits

      ➜ numbers between  -64 and 63    → 1 byte
        numbers between -8192 and 8191 → 2 bytes
        bigger numbers use more bytes
```

**My output, byte-identical:**

```
18 06 4d 61 72 74 69 6e 16 f2 14 19 28 0b 64 61 79 64 72 65
61 6d 69 6e 67 07 68 61 63 6b 69 6e 67 00        → 34 bytes ✓
```

**The varint saving, measured:**

```
   1337 as i64 in BinaryProtocol : 8 bytes → 00 00 00 00 00 00 05 39
   1337 as zigzag varint         : 2 bytes → f2 14
                                   ──────
                                   6 bytes saved on ONE field
```

### 🔷 Figure 4-4 — Protocol Buffers (33 bytes)

> Protocol Buffers does the bit packing slightly differently, but is otherwise **very similar to Thrift's CompactProtocol.**

```
0a 06 4d 61 72 74 69 6e 10 b9 0a 1a 0b 64 61 79 64 72 65 61
6d 69 6e 67 1a 07 68 61 63 6b 69 6e 67           → 33 bytes ✓
```

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  0a = 0 0 0 0 1 0 1 0   → field tag 1, wire type 2 (string)     │
   │       └────┬────┘└─┬─┘     (tag << 3) | wire_type               │
   │          tag=1   type=2                                          │
   │                                                                  │
   │  10 = 0 0 0 1 0 0 0 0   → field tag 2, wire type 0 (varint)     │
   │  b9 0a                  → 1337, varint, NO zigzag               │
   │                                                                  │
   │  1a = 0 0 0 1 1 0 1 0   → field tag 3, wire type 2              │
   │  1a                     → field tag 3 AGAIN                     │
   └─────────────────────────────────────────────────────────────────┘
```

**⚠️ Note the important detail about `required` vs `optional`:**

> In the schemas above, each field was marked either `required` or `optional`, **but this makes no difference to how the field is encoded** — nothing in the binary data indicates whether a field was required. **The difference is simply that `required` enables a runtime check that fails if the field is not set**, which can be useful for catching bugs.

> **📌 2026 UPDATE — `required` is basically dead.** proto3 (released 2016, now the default) **removed `required` entirely**, precisely because of the evolution hazard described later in this chapter — once a field is required, you can never remove it without breaking every old reader. proto3 also made all scalar fields implicitly optional with zero defaults, then reintroduced explicit `optional` in 3.15 (2021) so you can distinguish "absent" from "zero."
>
> In 2023 Google introduced **Protobuf Editions**, which replaces the proto2/proto3 split with per-feature opt-ins — an admission that the syntax-version model was too coarse. The direction of travel is unmistakable: **the industry concluded that Kleppmann's warning about `required` was right, and removed the feature.**

---

## 7. Field tags and schema evolution

> As you can see from the examples, **an encoded record is just the concatenation of its encoded fields.** Each field is identified by its **tag number** and annotated with a **datatype**. If a field value is not set, **it is simply omitted.**

```
   ╔════════════════════════════════════════════════════════════════════════╗
   ║  THE RULES, DERIVED FROM THAT ONE OBSERVATION                          ║
   ╠════════════════════════════════════════════════════════════════════════╣
   ║                                                                        ║
   ║  ✅ YOU CAN CHANGE A FIELD'S NAME                                       ║
   ║     …because the encoded data never refers to field names.             ║
   ║                                                                        ║
   ║  ❌ YOU CANNOT CHANGE A FIELD'S TAG                                     ║
   ║     …that would make ALL EXISTING ENCODED DATA INVALID.                ║
   ║                                                                        ║
   ║  ✅ YOU CAN ADD NEW FIELDS — with a NEW tag number                      ║
   ║     Old code hitting an unrecognized tag can simply IGNORE it.         ║
   ║     ⚑ The DATATYPE ANNOTATION is what lets the parser know HOW MANY    ║
   ║       BYTES TO SKIP.  ← this is the whole mechanism of forward compat  ║
   ║                                                                        ║
   ║  ⚠️  A NEW FIELD CANNOT BE `required`                                   ║
   ║     If it were, the check would FAIL when new code reads old data,     ║
   ║     because old code never wrote that field.                           ║
   ║     ➜ every field added after initial deployment must be OPTIONAL      ║
   ║       or HAVE A DEFAULT VALUE.                                         ║
   ║                                                                        ║
   ║  ⚠️  REMOVING is the mirror image                                       ║
   ║     • you can only remove a field that is OPTIONAL                     ║
   ║       (a required field can NEVER be removed)                          ║
   ║     • you can NEVER REUSE THE SAME TAG NUMBER — data written           ║
   ║       somewhere may still include the old tag                          ║
   ╚════════════════════════════════════════════════════════════════════════╝
```

### 💻 I proved forward compatibility works, with a real old parser

I wrote a v1 decoder that knows **only tags 1, 2, 3**, then fed it data written by a v2 encoder that adds **tag 4 = photoURL**:

```
NEW code wrote 52 bytes including tag 4 (photoURL)

OLD code decoded : {'userName': 'Martin', 'favoriteNumber': 1337,
                    'interests': ['daydreaming', 'hacking']}
OLD code skipped : [(4, 'http://x.co/p.jpg')]
                    └─ did not crash. Just ignored it.
```

**The mechanism, made explicit:** the key byte `0x22` decomposes to tag 4, wire type 2 (length-delimited). The parser doesn't know what field 4 *is*, but wire type 2 tells it "read a varint length, then skip that many bytes." **Forward compatibility is literally a property of the wire format's self-describing length information** — not of any cleverness in the application.

### 💀 And I demonstrated why tag reuse is catastrophic

```
v1 wrote tag 2 = favoriteNumber (int64) = 1337
v2 removed favoriteNumber, then RECYCLED tag 2 as 'accountBalance'

v2 reading OLD data: {'userName': 'Martin', 'accountBalance': 1337}

=> The old favourite number 1337 is now silently a BALANCE of 1337.
   No error. No crash. Just WRONG DATA.
```

This is why the rule is absolute. **The failure mode isn't a crash you'd notice in testing — it's silent data corruption in production.**

> **📌 2026 UPDATE — tooling now enforces what used to be discipline.** Protobuf added `reserved` to make tag retirement explicit:
> ```protobuf
> message Person {
>   reserved 2;                      // never reuse this tag
>   reserved "favorite_number";      // nor this name
>   string user_name = 1;
> }
> ```
> And **Buf** (buf.build) has largely replaced hand-rolled `protoc` pipelines. `buf breaking` diffs your schema against `main` in CI and **fails the build on incompatible changes** — tag reuse, type changes, field removals. The **Buf Schema Registry** does for Protobuf what Confluent's registry did for Avro. In 2017 "don't reuse tags" was advice you had to remember; today it's a pre-merge check.

### Data types and schema evolution

```
   ⚠️ CHANGING A FIELD'S DATATYPE — possible, but risky

      int32 ──► int64
        NEW code reading OLD data : ✅ fine, parser fills missing bits
                                       with zero
        OLD code reading NEW data : ⚠️ old code still uses a 32-bit
                                       variable. If the decoded 64-bit
                                       value WON'T FIT, IT IS TRUNCATED.
```

### 🔀 A genuinely interesting asymmetry: `repeated` vs `list`

```
   ┌───────────────────────────────────┬───────────────────────────────────┐
   │  PROTOCOL BUFFERS: `repeated`     │  THRIFT: a dedicated `list` type  │
   ├───────────────────────────────────┼───────────────────────────────────┤
   │  Has NO list/array datatype at    │  Has a real list datatype,        │
   │  all. Instead a `repeated` marker │  parameterized with the element   │
   │  — a third option alongside       │  type.                            │
   │  required and optional.           │                                   │
   │                                   │                                   │
   │  The encoding is exactly what it  │                                   │
   │  says: THE SAME FIELD TAG SIMPLY  │                                   │
   │  APPEARS MULTIPLE TIMES.          │                                   │
   │                                   │                                   │
   │  ✅ SO YOU CAN CHANGE optional →   │  ❌ CANNOT do that evolution       │
   │     repeated!                     │                                   │
   │     • new code reading old data:  │  ✅ BUT it supports NESTED LISTS   │
   │       sees a list of 0 or 1 items │     (list<list<string>>),         │
   │     • old code reading new data:  │     which protobuf's flat         │
   │       sees ONLY THE LAST ELEMENT  │     `repeated` cannot express     │
   └───────────────────────────────────┴───────────────────────────────────┘
```

**I verified the `repeated` encoding:** tag byte `0x1a` appears **twice** in the 33-byte protobuf output — once per interest. There is no list header at all.

---

## 8. Avro

> Started in **2009 as a sub-project of Hadoop**, as a result of **Thrift not being a good fit for Hadoop's use cases.** Avro is "interestingly different" from Protocol Buffers and Thrift.

**Two schema languages** — Avro IDL for humans, JSON for machines:

```
record Person {
    string               userName;
    union { null, long } favoriteNumber = null;
    array<string>        interests;
}
```

```json
{
    "type": "record",
    "name": "Person",
    "fields": [
        {"name": "userName",       "type": "string"},
        {"name": "favoriteNumber", "type": ["null", "long"], "default": null},
        {"name": "interests",      "type": {"type": "array", "items": "string"}}
    ]
}
```

### 🔷 Figure 4-5 — Avro (32 bytes — the most compact of all)

```
0c 4d 61 72 74 69 6e 02 f2 14 04 16 64 61 79 64 72 65 61 6d
69 6e 67 0e 68 61 63 6b 69 6e 67 00              → 32 bytes ✓
```

```
   ┌──────────────────┬──────────────────────────────────────────────────┐
   │ 0c               │ length 6 (zigzag: 6<<1 = 12 = 0x0c)              │
   │ 4d 61 72 74 69 6e│ "Martin"                                         │
   │ 02               │ UNION BRANCH 1 (long, not null)                  │
   │ f2 14            │ 1337                                             │
   │ 04               │ 2 array items follow                             │
   │ 16               │ length 11 → "daydreaming"                        │
   │ 0e               │ length 7  → "hacking"                            │
   │ 00               │ END OF ARRAY                                     │
   └──────────────────┴──────────────────────────────────────────────────┘
```

### 🔑 The radical part

> If you examine the byte sequence, **there is nothing to identify fields or their datatypes.** The encoding simply consists of **values concatenated together.** A string is just a length prefix followed by UTF-8 bytes, **but there's nothing in the encoded data that tells you that it is a string. It could just as well be an integer, or something else entirely.**

```
   NO field names.  NO field tags.  NO type annotations.  JUST VALUES.

   ➜ To parse, you go through the fields IN THE ORDER THEY APPEAR IN THE
     SCHEMA, using the schema to tell you the datatype of each field.

   ⚠️ THE CONSEQUENCE:
     "the binary data can ONLY be decoded correctly if the code reading
      the data is using the EXACT SAME SCHEMA as the code that wrote it.
      ANY MISMATCH would mean incorrectly decoded data."

   ➜ So how on earth does Avro support schema evolution?
```

### The writer's schema and the reader's schema

```
   ╔═══════════════════════════╗              ╔═══════════════════════════╗
   ║   WRITER'S SCHEMA         ║              ║   READER'S SCHEMA         ║
   ║                           ║              ║                           ║
   ║  whatever version of the  ║              ║  the schema the reading   ║
   ║  schema the encoding app  ║              ║  application is RELYING   ║
   ║  knows about — may be     ║              ║  ON — code may have been  ║
   ║  compiled into the app    ║              ║  generated from it at     ║
   ║                           ║              ║  build time               ║
   ╚═════════════╤═════════════╝              ╚═════════════╤═════════════╝
                 │                                          │
                 └──────────────────┬───────────────────────┘
                                    ▼
              ╔═══════════════════════════════════════════════╗
              ║  🔑 THE KEY IDEA OF AVRO                       ║
              ║                                               ║
              ║  They DON'T HAVE TO BE THE SAME —             ║
              ║  they only need to be COMPATIBLE.             ║
              ║                                               ║
              ║  When data is decoded, the Avro library       ║
              ║  looks at BOTH SCHEMAS SIDE BY SIDE and       ║
              ║  TRANSLATES from the writer's schema into     ║
              ║  the reader's schema.                         ║
              ╚═══════════════════════════════════════════════╝
```

### 🔷 Figure 4-6 — Avro schema resolution

```
   WRITER'S SCHEMA                        READER'S SCHEMA
   ┌─────────────────────┬──────────┐     ┌────────────────────┬──────────┐
   │ Datatype            │Field name│     │ Datatype           │Field name│
   ├─────────────────────┼──────────┤     ├────────────────────┼──────────┤
   │ string              │userName  │──┐  │ long               │userID    │◄─ no
   │                     │          │  │  │                    │          │   match
   │ union{null,long}    │favorite  │──┼─►│ union{null,int}    │favorite  │   → DEFAULT
   │                     │Number    │  │  │                    │Number    │
   │                     │          │  └─►│ string             │userName  │
   │ array<string>       │interests │────►│ array<string>      │interests │
   │                     │          │     │                    │          │
   │ string              │photoURL  │──✗  └────────────────────┴──────────┘
   └─────────────────────┴──────────┘
                              │
                              └─ no matching reader field → IGNORED

   ➜ FIELDS ARE MATCHED BY NAME, NOT BY POSITION.
     • different ORDER? no problem.
     • writer field with no reader match? IGNORED.
     • reader field with no writer match? FILLED IN WITH THE DEFAULT
       VALUE DECLARED IN THE READER'S SCHEMA.
```

**I implemented the resolution algorithm on exactly the schemas from Figure 4-6:**

```
writer fields: ['userName', 'favoriteNumber', 'interests', 'photoURL']
reader fields: ['userID', 'favoriteNumber', 'userName', 'interests']  ← DIFFERENT ORDER

resolved  : {'userID': 0, 'favoriteNumber': 1337,
             'userName': 'Martin', 'interests': ['daydreaming','hacking']}
ignored   : ['photoURL']    ← writer-only, silently dropped
defaulted : ['userID']      ← reader-only, filled from the reader's default
```

Note that `userName` moved from position 1 to position 3 and it made **no difference at all**. That's the property tag numbers were invented to provide — Avro gets it from names instead.

### Avro's schema evolution rules

```
   FORWARD compatibility  = NEW schema as writer, OLD schema as reader
   BACKWARD compatibility = NEW schema as reader, OLD schema as writer

   ╔═════════════════════════════════════════════════════════════════════╗
   ║  THE ONE RULE: you may only ADD or REMOVE a field                   ║
   ║                THAT HAS A DEFAULT VALUE.                            ║
   ╠═════════════════════════════════════════════════════════════════════╣
   ║  Add a field WITHOUT a default   → new readers can't read old data  ║
   ║                                    → BREAKS BACKWARD COMPATIBILITY  ║
   ║  Remove a field WITHOUT a default→ old readers can't read new data  ║
   ║                                    → BREAKS FORWARD COMPATIBILITY   ║
   ╚═════════════════════════════════════════════════════════════════════╝
```

**⚠️ And a design decision worth appreciating:**

> In some programming languages, `null` is an acceptable default for any variable, **but this is not the case in Avro**: to allow a field to be null you must use a **union type** — `union { null, long, string } field;`. You can only use `null` as a default **if it is one of the branches of the union.**
>
> "This is a little more verbose than having everything nullable by default, **but it helps prevent bugs by being explicit about what can and cannot be null.**"

*(The book cites Tony Hoare's "Null References: The Billion Dollar Mistake" here — the citation is the argument.)*

Consequently **Avro has no `optional`/`required` markers at all** — union types and default values do that job instead. *(Footnote: the default must be of the type of the **first** branch of the union — a specific limitation of Avro, not of union types generally.)*

```
   Other changes:
   • CHANGING A DATATYPE  — possible, provided Avro can convert the type
   • CHANGING A FIELD NAME — the reader's schema can contain ALIASES, so it
     matches an old writer's field name against the alias.
     ➜ BACKWARD COMPATIBLE BUT NOT FORWARD COMPATIBLE
   • ADDING A BRANCH TO A UNION — likewise backward but not forward compatible
```

### ❓ But how does the reader know the writer's schema?

> We can't just include the entire schema with every record — the schema would likely be **much bigger than the encoded data**, making all the space savings futile.

**Three answers, depending on context:**

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │ ① LARGE FILE WITH LOTS OF RECORDS  (the Hadoop case)                │
   │                                                                     │
   │    Millions of records, all with the same schema.                   │
   │    ➜ include the writer's schema ONCE AT THE BEGINNING OF THE FILE. │
   │      Avro specifies a file format for this: OBJECT CONTAINER FILES. │
   │                                                                     │
   │      ┌──────────┬─────────────────────────────────────────────┐     │
   │      │  SCHEMA  │ record record record record record record … │     │
   │      └──────────┴─────────────────────────────────────────────┘     │
   ├─────────────────────────────────────────────────────────────────────┤
   │ ② DATABASE WITH INDIVIDUALLY WRITTEN RECORDS                        │
   │                                                                     │
   │    Different records written at different times with DIFFERENT      │
   │    writer's schemas. You cannot assume they all match.              │
   │    ➜ include a VERSION NUMBER at the start of every record, and     │
   │      keep a LIST OF SCHEMA VERSIONS in the database. The reader     │
   │      extracts the version, fetches that writer's schema, decodes.   │
   │      (LinkedIn's Espresso works this way.)                          │
   │                                                                     │
   │      ┌─────┬────────────────┐        ┌──────────────────────┐       │
   │      │ v7  │ encoded record │  ───►  │ schema registry: v7  │       │
   │      └─────┴────────────────┘        └──────────────────────┘       │
   ├─────────────────────────────────────────────────────────────────────┤
   │ ③ SENDING RECORDS OVER A NETWORK CONNECTION                         │
   │                                                                     │
   │    Two processes on a bidirectional connection can NEGOTIATE THE    │
   │    SCHEMA VERSION ON CONNECTION SETUP, then use it for the lifetime │
   │    of the connection. (The Avro RPC protocol does this.)            │
   └─────────────────────────────────────────────────────────────────────┘
```

> **A database of schema versions is a useful thing to have in any case, since it acts as documentation, and gives you a chance to check schema compatibility.** As a version number you could use a simple incrementing integer, or a hash of the schema.

> **📌 2026 UPDATE — that "useful thing to have" became core infrastructure.** In 2017 the schema registry was a suggestion in a paragraph. Today it's a standard component of every streaming platform:
>
> - **Confluent Schema Registry** — the dominant implementation. Stores Avro/Protobuf/JSON Schema, assigns integer IDs, and the Kafka wire format is literally `[magic byte][4-byte schema ID][payload]` — option ② from the book, standardized.
> - It **enforces compatibility modes at registration time** (`BACKWARD`, `FORWARD`, `FULL`, `*_TRANSITIVE`). Try to register an incompatible schema and the *producer fails to start* rather than corrupting a topic.
> - **AWS Glue Schema Registry**, **Apicurio**, **Azure Schema Registry**, and the **Buf Schema Registry** are the other major players.
>
> The conceptual shift: schema compatibility moved from **a thing you reason about** to **a thing your CI and your broker refuse to let you break.**

### Dynamically generated schemas — Avro's real advantage

> One advantage of Avro's approach is that **the schema doesn't contain any tag numbers.** But why is that important?

```
   THE SCENARIO: dump a relational database to a file in a binary format.

   ┌────────────────────────────────────────────────────────────────────┐
   │  WITH AVRO                                                         │
   │    • generate an Avro schema from the relational schema            │
   │      (one record schema per table; each COLUMN becomes a FIELD;    │
   │       the COLUMN NAME maps to the FIELD NAME)                      │
   │    • dump it all to an Avro object container file                  │
   │                                                                    │
   │    Database schema changes (one column added, one removed)?        │
   │    ➜ just GENERATE A NEW AVRO SCHEMA and export.                   │
   │      "The data export process does not need to pay ANY attention   │
   │       to the schema change — it can simply do the conversion       │
   │       every time it runs."                                         │
   │    ➜ Readers see the fields changed, but SINCE FIELDS ARE          │
   │      IDENTIFIED BY NAME, the new writer's schema still matches     │
   │      up against the old reader's schema.                           │
   ├────────────────────────────────────────────────────────────────────┤
   │  WITH THRIFT / PROTOBUF                                            │
   │    • FIELD TAGS WOULD HAVE TO BE ASSIGNED BY HAND                  │
   │    • every schema change → an administrator manually updates the   │
   │      mapping from column names to field tags                       │
   │    • automating it is possible, but the generator would have to be │
   │      VERY careful never to reassign a previously used tag          │
   │                                                                    │
   │    ➜ "This kind of dynamically generated schema simply wasn't a    │
   │       design goal of Thrift or Protocol Buffers, whereas it WAS    │
   │       for Avro."                                                   │
   └────────────────────────────────────────────────────────────────────┘
```

**This is the cleanest justification for Avro's existence.** Names are generatable; tags require a human with memory of every tag ever used.

### Code generation and dynamically typed languages

```
   ┌──────────────────────────┬──────────────────────────────────────────┐
   │ STATICALLY TYPED         │ DYNAMICALLY TYPED                        │
   │ (Java, C++, C#)          │ (JavaScript, Ruby, Python)               │
   ├──────────────────────────┼──────────────────────────────────────────┤
   │ ✅ Code generation is     │ ❌ "not much point" — there is no        │
   │    USEFUL:                │    compile-time type checker to satisfy  │
   │    • efficient in-memory  │ ❌ often FROWNED UPON — these languages  │
   │      structures           │    otherwise avoid an explicit           │
   │    • type-checking        │    compilation step                      │
   │    • IDE autocompletion   │ ❌ with a DYNAMICALLY GENERATED schema,  │
   │                           │    code generation is "an unnecessary    │
   │                           │    obstacle to getting to the data"      │
   └──────────────────────────┴──────────────────────────────────────────┘

   ➜ AVRO'S ANSWER: code generation is OPTIONAL.
     With an object container file (which embeds the writer's schema),
     you can open it with the Avro library and look at the data
     JUST LIKE A JSON FILE. The file is SELF-DESCRIBING.

     Especially useful with dynamically typed data-processing languages
     like Apache Pig — "you can just open some Avro files and start
     analyzing them without even thinking about schemas."
```

---

## 9. The merits of schemas

> Their schema languages are **much simpler than XML Schema or JSON Schema**, which support much more detailed validation rules (*"the string value of this field must match this regular expression"*, *"the integer must be between 0 and 100"*). Being simpler to implement and use, they support **a fairly wide range of programming languages.**

### These ideas are old

```
   ASN.1 — a schema definition language FIRST STANDARDIZED IN 1984.
   • used to define various network protocols
   • its binary encoding DER IS STILL USED TO ENCODE SSL CERTIFICATES (X.509)
   • supports schema evolution using TAG NUMBERS, like Protobuf and Thrift
   • ⚠️ "also very complex and badly documented, so ASN.1 is probably not
        a good choice for new applications"

   Also: most relational databases have a PROPRIETARY BINARY NETWORK
   PROTOCOL, with a driver (ODBC/JDBC) that decodes responses into
   in-memory data structures.
```

### 🔑 The four properties that make schema-driven binary encodings worth it

```
   ① MUCH MORE COMPACT than "binary JSON" variants, since they can
      OMIT FIELD NAMES from the encoded data.

   ② THE SCHEMA IS A VALUABLE FORM OF DOCUMENTATION — and because the
      schema is REQUIRED FOR DECODING, YOU CAN BE SURE IT IS UP TO DATE.
      (Manually maintained documentation easily diverges from reality.)
      ← this is the underrated one

   ③ KEEPING A DATABASE OF SCHEMAS lets you CHECK FORWARD AND BACKWARD
      COMPATIBILITY OF SCHEMA CHANGES, BEFORE ANYTHING IS DEPLOYED.

   ④ For statically typed languages, CODE GENERATION enables
      TYPE-CHECKING AT COMPILE TIME.
```

> **In summary, schema evolution allows the same kind of flexibility as schemaless/schema-on-read JSON databases provide, while also providing better guarantees about your data and better tooling.**

That sentence resolves the Chapter 2 debate. You don't have to choose between flexibility and safety — **schema evolution gives you both**, if you're willing to accept a build step.

### 📊 The size comparison, measured across all five of my implementations

```
   XML (approx)              172 B  ████████████████████████████████████████
   JSON (text)                81 B  ██████████████████
   MessagePack                66 B  ███████████████
   Thrift BinaryProtocol      59 B  █████████████
   Thrift CompactProtocol     34 B  ███████
   Protocol Buffers           33 B  ███████
   Avro                       32 B  ███████

   JSON → Avro:  81 B → 32 B  =  2.5x smaller
   XML  → Avro: 172 B → 32 B  =  5.4x smaller
```

At terabyte scale, 2.5× is the difference between one machine and three.

---
---

# PART B — MODES OF DATA FLOW

> Compatibility is **a relationship between one process that encodes the data, and another process that decodes it.** That's a fairly abstract idea — there are many ways data can flow. **Who encodes the data, and who decodes it?**

```
                    ┌───────────────────────────────────┐
                    │      THREE MODES OF DATA FLOW     │
                    └───────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
   ┌──────────┐            ┌─────────────────┐          ┌───────────────┐
   │ DATABASE │            │ SERVICES        │          │ MESSAGE       │
   │          │            │ (REST / RPC)    │          │ PASSING       │
   ├──────────┤            ├─────────────────┤          ├───────────────┤
   │ write    │            │ request/response│          │ one-way, via  │
   │ now,     │            │ synchronous,    │          │ a BROKER,     │
   │ read     │            │ low latency     │          │ asynchronous  │
   │ later    │            │                 │          │               │
   │          │            │ client encodes  │          │ sender encodes│
   │ writer   │            │ req, server     │          │ recipient     │
   │ encodes, │            │ decodes+encodes │          │ decodes; they │
   │ reader   │            │ resp, client    │          │ never meet    │
   │ decodes  │            │ decodes         │          │               │
   └──────────┘            └─────────────────┘          └───────────────┘
```

---

## 10. Data flow through databases

> The process that writes encodes; the process that reads decodes. There may be a single process, in which case the reader is simply a later version of the same process — **you can think of storing something in the database as sending a message to your future self.**

```
   BACKWARD COMPATIBILITY  → clearly necessary, "otherwise your future
                             self won't be able to decode what you
                             previously wrote"

   FORWARD COMPATIBILITY   → also often required, because during a
                             rolling upgrade some instances run new code
                             and some run old code, all hitting the same DB

        ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
        │  NEW   │  │  NEW   │  │  old   │  │  old   │
        └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘
            └───────────┴─────┬─────┴───────────┘
                              ▼
                       ┌─────────────┐
                       │  DATABASE   │  ← values written by new code,
                       └─────────────┘    read by old code still running
```

### ⚠️ The additional snag — and it's a real bug people ship

### 🔷 Figure 4-7 — Data loss when old code updates a record written by new code

```
   ┌─────────────────────────────────────────────────────────────────────────┐
   │                                                                         │
   │  DB, written by NEW version of code (includes new photoURL field):     │
   │     ┌─────────────────────────────────────────┐                        │
   │     │ {                                       │                        │
   │     │   "userName":       "Martin",           │                        │
   │     │   "favoriteNumber": 1337,               │                        │
   │     │   "interests":      ["hacking"],        │                        │
   │     │   "photoURL":       "http://…"   ◄──────┼── new field            │
   │     │ }                                       │                        │
   │     └────────────────────┬────────────────────┘                        │
   │                          │  ① READ AND DECODE INTO MODEL OBJECT        │
   │                          ▼                                             │
   │     ┌───────────────────────────────────────────┐                      │
   │     │ public class Person {          OLD CODE   │                      │
   │     │   private String userName;                │                      │
   │     │   private Long favoriteNumber;            │                      │
   │     │   private List<String> interests;         │                      │
   │     │   // …no photoURL field at all!           │                      │
   │     │ }                                         │                      │
   │     └────────────────────┬──────────────────────┘                      │
   │                          │  ② UPDATE, RE-ENCODE AND WRITE BACK         │
   │                          │     person.setFavoriteNumber(42);           │
   │                          │     db.write(person.toJSON());              │
   │                          ▼                                             │
   │     ┌─────────────────────────────────────────┐                        │
   │     │ {                                       │                        │
   │     │   "userName":       "Martin",           │                        │
   │     │   "favoriteNumber": 42,                 │                        │
   │     │   "interests":      ["hacking"]         │                        │
   │     │ }                                       │                        │
   │     └─────────────────────────────────────────┘                        │
   │                    💥 photoURL IS GONE 💥                               │
   └─────────────────────────────────────────────────────────────────────────┘
```

> The desirable behaviour is usually for **the old code to keep the new field intact, even though it couldn't be interpreted.** The encoding formats discussed above support such preservation of unknown fields, **but sometimes you need to take care at an application level** — if you decode into model objects and later re-encode them, the unknown field might be lost in that translation. **"Solving this is not a hard problem, you just need to be aware of it."**

**💻 I reproduced the bug and the fix:**

```
DB, written by NEW code : {'userName':'Martin', 'favoriteNumber':1337,
                           'interests':['hacking'],
                           'photoURL':'http://example.com/martin.jpg'}
DB, after OLD code wrote: {'userName':'Martin', 'favoriteNumber':42,
                           'interests':['hacking']}
*** photoURL LOST: True ***
```

The fix is about six lines — capture what you don't recognize, and merge it back on write:

```python
class PersonV1Safe:
    FIELDS = ["userName", "favoriteNumber", "interests"]
    def __init__(self, d):
        self.d       = {k: d.get(k) for k in self.FIELDS}
        self.unknown = {k: v for k, v in d.items() if k not in self.FIELDS}  # ← keep it
    def to_json(self):
        return {**self.d, **self.unknown}                                    # ← give it back
```

```
With unknown-field preservation: {…, 'photoURL': 'http://example.com/martin.jpg'}
*** photoURL PRESERVED: True ***
```

**Note where the bug lives.** The *encoding format* preserved photoURL perfectly. The loss happened in the ORM boundary — the translation between wire format and application objects. This is exactly why the book says "take care at an application level."

### 🔑 Data outlives code

> A database generally allows any value to be updated at any time. **Within a single database you may have some values written five milliseconds ago, and some values written five years ago.**
>
> When you deploy a new version of your application you may replace the old version entirely within a few minutes. **The same is not true of database contents: the five-year-old data will still be there, in the original encoding**, unless you have explicitly rewritten it.

```
   ┌──────────────────────────────────────────────────────────────────┐
   │                    D A T A   O U T L I V E S   C O D E           │
   └──────────────────────────────────────────────────────────────────┘

   CODE:  ├─v1─┤├─v2─┤├─v3─┤├─v4─┤├─v5─┤├─v6─┤├─v7─┤  ← replaced in minutes
   DATA:  ├────────────────────────────────────────────────────────┤
          └─ records written under v1 are STILL SITTING THERE ─────┘
```

> Rewriting (migrating) data into a new schema is possible, but **expensive on a large dataset, so most databases avoid it.** Most relational databases allow simple schema changes — such as adding a column with a null default — **without rewriting existing data.** When an old row is read, the database fills in nulls for missing columns. *(Footnote: except MySQL, which often rewrites an entire table even when not strictly necessary.)*
>
> **LinkedIn's Espresso uses Avro for storage**, allowing it to use Avro's schema evolution rules. **Schema evolution thus allows the entire database to appear as if it was encoded with a single schema**, even though the underlying storage contains records from various historical eras.

### Archival storage

```
   Taking a snapshot for backup, or loading into a data warehouse?

   • The dump is typically encoded using THE LATEST SCHEMA, even if the
     source contained a mixture of schema versions. Since you're copying
     anyway, you might as well encode the copy consistently.
   • The dump is written IN ONE GO and is thereafter IMMUTABLE
     ➜ AVRO OBJECT CONTAINER FILES are a good fit
   • Also a good opportunity to encode in an ANALYTICS-FRIENDLY
     COLUMN-ORIENTED format such as PARQUET  ← callback to Chapter 3
```

> **📌 2026 UPDATE — this paragraph turned into an entire industry.** The "immutable columnar dump" idea is now the foundation of the **lakehouse**. **Apache Parquet** won decisively as the storage format, and on top of it sit three table formats that add ACID transactions, time travel, and schema evolution to files on object storage: **Apache Iceberg** (now the de facto winner, backed by Snowflake, AWS, Databricks after the Tabular acquisition), **Delta Lake**, and **Apache Hudi**.
>
> Critically, **Iceberg does schema evolution using field IDs, not column names or positions** — which is exactly the Protobuf tag idea from this chapter, applied to table columns. Rename a column in Iceberg and old data still reads correctly, because the ID is stable. Chapter 4's core lesson, rediscovered at a different layer.
>
> **Apache Arrow** also deserves mention: it's the in-memory columnar counterpart to Parquet, and it's what makes zero-copy data exchange between Python, R, Java and Rust possible. Arrow Flight even replaces the "proprietary database network protocol" the book mentions.

---

## 11. Data flow through services: REST and RPC

```
   The most common arrangement: CLIENTS and SERVERS.
   The servers expose an API over the network; clients connect and make
   requests. THE API EXPOSED BY THE SERVER IS KNOWN AS A SERVICE.

   THE WEB WORKS THIS WAY:
      browsers GET  → HTML, CSS, JavaScript, images
      browsers POST → submit data
      The API is a standardized set of protocols and data formats
      (HTTP, URLs, SSL/TLS, HTML). Because browsers, servers and website
      authors mostly agree on these, you can use any browser to access
      any website (AT LEAST IN THEORY!).
```

**But browsers aren't the only clients** — native mobile/desktop apps make requests too, and client-side JavaScript uses `XMLHttpRequest` (**Ajax**). In those cases the response is **not HTML for a human, but data for further processing** (such as JSON).

```
   ⚑ AND A SERVER CAN ITSELF BE A CLIENT
     (a typical web app server acts as client to a database)

   ➜ Used to decompose a large application into smaller services by area
     of functionality:
          SERVICE-ORIENTED ARCHITECTURE (SOA)
          — "more recently refined and rebranded as MICROSERVICES"

   🔑 THE KEY DESIGN GOAL:
     make the application easier to change and maintain by making
     services INDEPENDENTLY DEPLOYABLE AND EVOLVABLE. Each service owned
     by ONE TEAM, releasing frequently WITHOUT COORDINATING WITH OTHER
     TEAMS.

     ➜ "we should EXPECT old and new versions of servers and clients to
        be running at the same time, and so the data encoding used must
        be COMPATIBLE ACROSS VERSIONS OF THE SERVICE API"
```

### Web services — three contexts, not just "the web"

```
   ① CLIENT APP ON A USER'S DEVICE → service over HTTP
      (native mobile app, or JS web app using Ajax)
      typically over the PUBLIC INTERNET

   ② ONE SERVICE → ANOTHER SERVICE, SAME ORGANIZATION
      often within the same datacenter, as part of SOA/microservices
      (software supporting this is sometimes called MIDDLEWARE)

   ③ ONE SERVICE → A SERVICE OWNED BY A DIFFERENT ORGANIZATION
      usually via the internet; data exchange between organizations'
      backend systems. Includes public APIs: credit card processing,
      OAuth for shared access to user data.
```

### REST vs SOAP

```
   ╔══════════════════════════════════╦══════════════════════════════════╗
   ║  REST                            ║  SOAP                            ║
   ╠══════════════════════════════════╬══════════════════════════════════╣
   ║  NOT a protocol — a DESIGN       ║  An XML-BASED PROTOCOL for making║
   ║  PHILOSOPHY building on the      ║  network API requests.           ║
   ║  principles of HTTP.             ║                                  ║
   ║                                  ║  Most commonly used over HTTP,   ║
   ║  Emphasizes:                     ║  BUT AIMS TO BE INDEPENDENT of   ║
   ║   • simple data formats          ║  HTTP and AVOIDS USING MOST HTTP ║
   ║   • URLs for identifying         ║  FEATURES.                       ║
   ║     resources                    ║                                  ║
   ║   • HTTP features for cache      ║  Comes with "a sprawling and     ║
   ║     control, authentication,     ║  complex multitude of related    ║
   ║     content type negotiation     ║  standards" (WS-*).              ║
   ║                                  ║                                  ║
   ║  An API following these          ║  Described by WSDL (XML-based).  ║
   ║  principles is called RESTFUL.   ║  WSDL enables CODE GENERATION.   ║
   ║                                  ║  ⚠️ WSDL is NOT human-readable;  ║
   ║  Gaining popularity, especially  ║     SOAP messages are often too  ║
   ║  for CROSS-ORGANIZATIONAL        ║     complex to construct by hand ║
   ║  integration; associated with    ║     → heavy reliance on tooling  ║
   ║  microservices.                  ║  ⚠️ Interoperability between     ║
   ║                                  ║     vendors often causes problems║
   ║                                  ║                                  ║
   ║                                  ║  "still used in many large       ║
   ║                                  ║   enterprises, but has fallen    ║
   ║                                  ║   out of favor in most smaller   ║
   ║                                  ║   companies"                     ║
   ╚══════════════════════════════════╩══════════════════════════════════╝

   📖 The RESTful equivalent of WSDL is called SWAGGER.
   📖 Footnote: despite the acronyms, SOAP is NOT a requirement for SOA.
      SOAP is a technology; SOA is a general approach.
```

> **📌 2026 UPDATE.** SOAP is now genuinely legacy — you'll meet it in banking, insurance, healthcare (HL7), and government integrations, essentially nowhere new. **Swagger was renamed OpenAPI** in 2016 and is now the universal REST description format; **OpenAPI 3.1** aligned fully with JSON Schema 2020-12, which finally made REST schemas as rigorous as SOAP's ever were, without the WS-* baggage.
>
> Two things the book couldn't have covered:
>
> **GraphQL** (Facebook, open-sourced 2015, mainstream by ~2018) attacks a problem REST has: over-fetching and under-fetching. The client specifies exactly which fields it wants. On evolution specifically, GraphQL takes an interesting stance — **it has no API versions at all.** You add fields freely (forward compatible by construction, since clients only receive what they ask for) and mark removals with `@deprecated`. It's the "additive changes only" discipline, enforced by the query language.
>
> **tRPC / Connect / typed RPC over HTTP** — for TypeScript-monorepo teams, tRPC gives end-to-end type safety with no code generation and no schema file. This is arguably the return of "make the remote call look local," which the next section explains is a fundamentally flawed idea — but scoped narrowly enough (same repo, same language, same deploy) that most of Kleppmann's objections don't bite.

---

## 12. Remote Procedure Calls (RPC) — the fundamental flaw

```
   A LONG LINEAGE, "many of which received a lot of hype but have
   serious problems":

      Enterprise JavaBeans (EJB), Java RMI  →  limited to Java
      DCOM                                  →  limited to Microsoft
      CORBA                                 →  "excessively complex, and
                                               does not provide backward
                                               or forward compatibility"
```

> All are based on the idea of a **Remote Procedure Call**, around since the 1970s. RPC tries to make a request to a remote network service **look the same as calling a function in your programming language, within the same process** — this is called **location transparency.**
>
> **"Although this seems convenient at first, the approach is fundamentally flawed."**

### 🔑 The six ways a network request differs from a local function call

```
   ╔═══════════════════════════════════╦═══════════════════════════════════╗
   ║  LOCAL FUNCTION CALL              ║  NETWORK REQUEST                  ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║ ① PREDICTABLE. Succeeds or fails  ║ UNPREDICTABLE. Request or response║
   ║   depending only on parameters    ║ may be LOST; the remote machine   ║
   ║   UNDER YOUR CONTROL.             ║ may be SLOW or UNAVAILABLE —      ║
   ║                                   ║ ENTIRELY OUTSIDE YOUR CONTROL.    ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║ ② Returns a result, throws an     ║ HAS A THIRD OUTCOME: it may       ║
   ║   exception, or never returns     ║ RETURN WITHOUT A RESULT, due to a ║
   ║   (infinite loop / crash).        ║ TIMEOUT. Then YOU SIMPLY DON'T    ║
   ║                                   ║ KNOW WHAT HAPPENED — you have no  ║
   ║                                   ║ way of knowing whether the        ║
   ║                                   ║ request got through or not.       ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║ ③ Retrying isn't a concept.       ║ If you RETRY, the requests may be ║
   ║                                   ║ getting through and only the      ║
   ║                                   ║ RESPONSES getting lost → THE      ║
   ║                                   ║ ACTION HAPPENS MULTIPLE TIMES,    ║
   ║                                   ║ unless you build IDEMPOTENCE /    ║
   ║                                   ║ deduplication into the protocol.  ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║ ④ Takes about the SAME TIME every ║ MUCH SLOWER, and LATENCY IS       ║
   ║   time.                           ║ WILDLY VARIABLE — under a         ║
   ║                                   ║ millisecond at good times, MANY   ║
   ║                                   ║ SECONDS when congested, for       ║
   ║                                   ║ EXACTLY THE SAME WORK.            ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║ ⑤ You can efficiently pass        ║ ALL PARAMETERS MUST BE ENCODED    ║
   ║   REFERENCES (POINTERS) to        ║ into a byte sequence. Fine for    ║
   ║   objects in local memory.        ║ primitives; "quickly becomes      ║
   ║                                   ║ problematic with larger objects." ║
   ╠═══════════════════════════════════╬═══════════════════════════════════╣
   ║ ⑥ One process, one language.      ║ Client and service may be in      ║
   ║                                   ║ DIFFERENT LANGUAGES → the RPC     ║
   ║                                   ║ framework must TRANSLATE          ║
   ║                                   ║ DATATYPES. "This can end up ugly, ║
   ║                                   ║ since not all languages have the  ║
   ║                                   ║ same types" — recall JavaScript's ║
   ║                                   ║ 2^53 problem.                     ║
   ╚═══════════════════════════════════╩═══════════════════════════════════╝
```

> **"There's no point trying to make a remote service look too much like a local object in your programming language, because it's a fundamentally different thing. Part of the appeal of REST is that it doesn't try to hide the fact that it's a network protocol"** (although that doesn't stop people building RPC libraries on top of REST).

### Current directions for RPC

> Despite all these problems, **RPC isn't going away.**

| Framework | Encoding |
|---|---|
| **Thrift**, **Avro** | RPC support included |
| **gRPC** | RPC implementation using **Protocol Buffers** |
| **Finagle** | uses Thrift |
| **Rest.li** | JSON over HTTP |

```
   THE NEW GENERATION IS MORE EXPLICIT that a remote request differs from
   a local call:

   • FUTURES (PROMISES) — Finagle and Rest.li use them to encapsulate
     asynchronous actions THAT MAY FAIL. Futures also simplify making
     requests to MULTIPLE SERVICES IN PARALLEL and combining results.
   • STREAMS — gRPC supports calls consisting of not just one request and
     one response, but A SERIES of requests and responses over time.
   • SERVICE DISCOVERY — letting a client find out which IP and port a
     particular service is on.
```

### ⚖️ The honest comparison — when to use which

```
   ┌─────────────────────────────────┬─────────────────────────────────────┐
   │  CUSTOM RPC + BINARY ENCODING   │  RESTful API                        │
   ├─────────────────────────────────┼─────────────────────────────────────┤
   │  ✅ BETTER PERFORMANCE than      │  ✅ Good for EXPERIMENTATION AND     │
   │     something generic like       │     DEBUGGING — make requests from  │
   │     JSON over REST               │     a browser or curl, no code      │
   │                                  │     generation, no software install │
   │                                  │  ✅ Supported by ALL mainstream      │
   │                                  │     languages and platforms         │
   │                                  │  ✅ A VAST ECOSYSTEM: servers,       │
   │                                  │     caches, load balancers, proxies,│
   │                                  │     firewalls, monitoring,          │
   │                                  │     debugging, testing tools        │
   └─────────────────────────────────┴─────────────────────────────────────┘

   ➜ "REST seems to be the PREDOMINANT STYLE FOR PUBLIC APIs. The main
     focus of RPC frameworks is on requests between services OWNED BY THE
     SAME ORGANIZATION, typically within the same datacenter."
```

**That boundary — public API vs internal service — is the durable lesson**, and it has held up completely.

> **📌 2026 UPDATE — gRPC won the internal-service slot, exactly as predicted.** It's the default for service-to-service traffic in Kubernetes environments, and it's what service meshes (Istio, Linkerd) are built to route. Things that changed:
>
> - **HTTP/2** (which gRPC requires) is universal; **HTTP/3 / QUIC** is now widely deployed and helps most with the tail-latency problem in point ④.
> - **gRPC-Web** and **Connect** (buf.build) solved gRPC's browser problem — browsers can't speak raw gRPC, so these bridge it.
> - **Service meshes** externalized retries, timeouts, circuit breaking and load balancing out of the RPC library and into a sidecar. Notably, this *doesn't* fix Kleppmann's point ③ — automatic mesh retries make the idempotence problem **worse**, because now infrastructure you didn't write is duplicating your requests. Idempotency keys are more necessary in 2026 than in 2017, not less.
> - **The 2^53 problem persisted into gRPC:** protobuf's canonical JSON mapping encodes `int64` as a **string**, for exactly the reason Twitter did. Same bug, same fix, ten years later.

### Data encoding and evolution for RPC

```
   🔑 A SIMPLIFYING ASSUMPTION you can make for services (but NOT for
      databases):

      "It is reasonable to assume that ALL THE SERVERS WILL BE UPDATED
       FIRST, AND ALL THE CLIENTS SECOND."

      ➜ Therefore you only need:
           BACKWARD COMPATIBILITY ON REQUESTS
           FORWARD  COMPATIBILITY ON RESPONSES

        ┌──────────┐   request  (old client → new server)  ┌──────────┐
        │  CLIENT  │ ───────────────────────────────────►  │  SERVER  │
        │  (old)   │            needs BACKWARD compat      │  (new)   │
        │          │ ◄───────────────────────────────────  │          │
        └──────────┘   response (new server → old client)  └──────────┘
                                needs FORWARD compat
```

**Compatibility properties are inherited from the encoding:**

| Scheme | Evolution story |
|---|---|
| **Thrift, gRPC (Protobuf), Avro RPC** | Evolve according to the compatibility rules of the encoding format |
| **SOAP** | Requests/responses specified with XML Schemas. Can be evolved, but **there are subtle pitfalls** |
| **RESTful APIs** | Most commonly JSON (no formal schema) for responses; JSON or URI/form-encoded parameters for requests. **Adding optional request parameters and adding new response fields are usually considered compatible** |

### ⚠️ The organizational problem that has no technical solution

> Service compatibility is made harder by the fact that **RPC is often used across organizational boundaries**, so the provider of a service **often has no control over its clients and cannot force them to upgrade.** Thus compatibility needs to be maintained **for a long time, perhaps indefinitely.** If a compatibility-breaking change is required, the provider often ends up **maintaining multiple versions of the API side by side.**

```
   AND THERE IS NO AGREEMENT ON HOW API VERSIONING SHOULD WORK:

   • version number in the URL      /v2/users
   • version in the HTTP Accept header
   • for API-key clients: store the client's requested version ON THE
     SERVER, updated through a separate administrative interface
     (this is what Stripe does)
```

> **📌 2026 UPDATE — Stripe's approach won the argument.** The pattern the book mentions in passing has become the reference design for long-lived public APIs. Stripe pins each account to the API version current when it signed up, then maintains a **chain of request/response transformation functions** — a new request is upgraded through each version to the current internal representation, and the response is downgraded back. The internal codebase only ever knows one version.
>
> Stripe has since published on this, and the pattern has been reimplemented widely (e.g. `cadwyn` for FastAPI). The insight: **don't maintain N copies of your API; maintain one API plus N-1 small, composable migrations.** It's the same idea as database schema migrations, applied to API shapes.

---

## 13. Message-passing data flow

> Asynchronous message-passing systems are **somewhere between RPC and databases.**

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  Similar to RPC      : a client's request (a MESSAGE) is delivered   │
   │                        to another process with LOW LATENCY           │
   │  Similar to databases: the message is NOT sent via a direct network  │
   │                        connection, but goes via an intermediary — a  │
   │                        MESSAGE BROKER (message queue, message-       │
   │                        oriented middleware) — which STORES IT        │
   │                        TEMPORARILY                                   │
   └──────────────────────────────────────────────────────────────────────┘

     ┌──────────┐        ┌────────────────────┐        ┌──────────┐
     │ PRODUCER │ ─────► │   MESSAGE BROKER   │ ─────► │ CONSUMER │
     │          │        │   (named queue     │        │          │
     │ publishes│        │    or topic)       │  ─────►│ CONSUMER │
     │ and      │        │                    │        │          │
     │ FORGETS  │        │  stores the message│  ─────►│ CONSUMER │
     └──────────┘        └────────────────────┘        └──────────┘
```

### The five advantages over direct RPC

```
   ① BUFFERS if the recipient is unavailable or overloaded
      → improves system reliability
   ② AUTOMATICALLY REDELIVERS to a process that crashed
      → prevents messages from being lost
   ③ THE SENDER NEED NOT KNOW the recipient's IP address and port
      → particularly useful in cloud deployments where VMs come and go
   ④ ONE MESSAGE CAN GO TO SEVERAL RECIPIENTS
   ⑤ LOGICALLY DECOUPLES sender from recipient
      → the sender just publishes and DOESN'T CARE WHO CONSUMES
```

> **The difference compared to RPC:** message-passing is usually **one-way** — a sender normally doesn't expect a reply. A process *can* send a response, but usually on a **separate channel**. **This is what makes it asynchronous: the sender doesn't wait for delivery, it simply sends and then forgets about it.**

### Message brokers

```
   PAST     : commercial enterprise software — TIBCO, IBM WebSphere,
              webMethods
   RECENTLY : open source — RabbitMQ, ActiveMQ, HornetQ, NATS,
              APACHE KAFKA

   HOW THEY'RE USED (semantics vary by implementation):
      one process sends a message to a NAMED QUEUE OR TOPIC, and the
      broker ensures delivery to one or more CONSUMERS or SUBSCRIBERS.
      There can be MANY PRODUCERS and MANY CONSUMERS on the same topic.

   A topic provides ONE-WAY data flow. But a consumer may itself publish
   to ANOTHER TOPIC (chaining them together), or to a REPLY QUEUE
   consumed by the original sender — giving a request-response dataflow
   similar to RPC.
```

```
   🔑 THE ENCODING POINT:
     "Message brokers typically DON'T ENFORCE ANY PARTICULAR DATA MODEL
      — a message is just a sequence of bytes with some metadata, SO YOU
      CAN USE ANY ENCODING FORMAT."

     If the encoding is backward and forward compatible, you have THE
     GREATEST FLEXIBILITY to change publishers and consumers
     INDEPENDENTLY, AND DEPLOY THEM IN ANY ORDER.

   ⚠️ And the Figure 4-7 warning applies here too: if a consumer
     RE-PUBLISHES messages to another topic, BE CAREFUL TO PRESERVE
     UNKNOWN FIELDS.
```

> **📌 2026 UPDATE — Kafka became the centre of gravity, and "any encoding format" became "Avro or Protobuf, via a registry."** Because a Kafka topic is durable and long-lived, its schema is effectively a **long-term API contract** between teams — messages written today may be replayed years from now, so *data outlives code* applies with full force. Confluent's registry with `FULL_TRANSITIVE` compatibility is the standard answer.
>
> Also notable: **Kafka removed ZooKeeper** (KRaft mode, production-ready 2022), **Apache Pulsar** offers built-in schema enforcement at the broker, **NATS JetStream** grew durability, and **cloud-native queues** (SQS, Pub/Sub, EventBridge, Kinesis) took a large share of the simpler use cases. The **CloudEvents** CNCF spec standardized event *metadata* (id, source, type, time) across all of them — a layer the book predates.

### Distributed actor frameworks

```
   THE ACTOR MODEL — a programming model for concurrency in a SINGLE
   PROCESS. Rather than dealing with threads directly (and race
   conditions, locking, deadlock), LOGIC IS ENCAPSULATED IN ACTORS.

   • Each actor communicates by sending/receiving ASYNCHRONOUS MESSAGES
   • MESSAGE DELIVERY IS NOT GUARANTEED — in certain error scenarios,
     messages will be lost
   • Each actor processes ONLY ONE MESSAGE AT A TIME → no thread worries
   • Each actor can be scheduled independently by the framework

   DISTRIBUTED ACTOR FRAMEWORKS scale this across multiple nodes. The SAME
   message-passing mechanism is used whether sender and recipient are on
   the same node or not. If different nodes, the message is TRANSPARENTLY
   ENCODED, sent over the network, and decoded on the other side.
```

### 🔑 Why location transparency works here but not in RPC

> **Location transparency works better in the actor model than in RPC, because the actor model ALREADY ASSUMES THAT MESSAGES MAY BE LOST, even within a single process.** Although latency over the network is likely higher, **there is less of a fundamental mismatch between local and remote communication.**

```
   RPC's mistake:  pretend the network is a function call
                   → local semantics are STRONGER than remote reality
                   → the abstraction LEAKS

   Actor's trick:  make LOCAL calls as weak as REMOTE ones
                   → local semantics ALREADY MATCH remote reality
                   → nothing to leak

   ⚑ You don't fix a leaky abstraction by strengthening the weak side.
     You fix it by WEAKENING THE STRONG SIDE.
```

**But rolling upgrades still bite:**

| Framework | Handling |
|---|---|
| **Akka** | Uses Java's built-in serialization by default — **no forward or backward compatibility.** You can replace it with **Protocol Buffers** and gain rolling upgrades |
| **Orleans** | Default custom encoding **doesn't support rolling upgrades.** To deploy a new version you must **set up a new cluster, move traffic over, and shut down the old one.** Custom serialization plugins can be used |
| **Erlang OTP** | *"Surprisingly hard to make changes to record schemas (despite the system having many features designed for high availability)."* Rolling upgrades are possible but **need careful planning.** The experimental `maps` datatype may help |

The Erlang note is a nice irony worth pausing on: **the platform most famous for nine-nines uptime has one of the hardest schema-evolution stories.** High availability at the process level doesn't automatically give you evolvability at the data level.

> **📌 2026 UPDATE.** **Akka changed its licence** in 2022 (BSL, not open source), which fractured the ecosystem and spawned the **Apache Pekko** fork. Akka's default serializer is now Jackson rather than Java serialization — a direct fix for the problem the book flags. **Orleans** is now a first-class part of .NET and got a new version-tolerant serializer in Orleans 7 that supports rolling upgrades properly. **Dapr** added actors as a portable, sidecar-based building block. And **Erlang/Elixir** got `maps` fully standardized — the fix the book hoped for arrived.

---

## 14. Chapter Summary

> We saw how the details of these encodings affects **not only their efficiency, but more importantly also the architecture of applications and your options for deploying them.**

```
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║  WHY ROLLING UPGRADES MATTER                                          ║
   ╠═══════════════════════════════════════════════════════════════════════╣
   ║  • release new versions WITHOUT DOWNTIME                              ║
   ║    → encourages FREQUENT SMALL RELEASES over RARE BIG ONES            ║
   ║  • makes deployments LESS RISKY                                       ║
   ║    → faulty releases detected and ROLLED BACK before affecting        ║
   ║      a large number of users                                          ║
   ║  ➜ "hugely beneficial for EVOLVABILITY"                               ║
   ║                                                                       ║
   ║  BUT: we must assume DIFFERENT NODES RUN DIFFERENT VERSIONS.          ║
   ║  So all data flowing around the system must have                      ║
   ║        BACKWARD COMPATIBILITY (new code reads old data)               ║
   ║      + FORWARD  COMPATIBILITY (old code reads new data)               ║
   ╚═══════════════════════════════════════════════════════════════════════╝
```

**The three families of encoding:**

| Family | Verdict |
|---|---|
| **Language-specific** | Restricted to one language, **often fail to provide forward and backward compatibility** |
| **Textual (JSON, XML, CSV)** | Widespread. **Compatibility depends on how you use them.** Optional schema languages, "sometimes helpful, sometimes a hindrance." **Vague about datatypes** — be careful with numbers and binary strings |
| **Binary schema-driven (Thrift, Protobuf, Avro)** | **Compact, efficient, clearly defined compatibility semantics.** Schema useful for documentation and code generation. **Downside: data must be decoded before it's human-readable** |

**The three modes of data flow:**

```
   ① DATABASES        writer encodes, reader decodes
   ② RPC and REST     client encodes request → server decodes request,
                      encodes response → client decodes response
   ③ ASYNC MESSAGING  sender encodes, recipient decodes, via broker or actor
```

> **"We can conclude that with a bit of care, backward/forward compatibility and rolling upgrades are quite achievable. May your application's evolution be rapid and your deployments be frequent."**

---
---

# 15. 🎁 WHAT'S CHANGED SINCE 2017 — a consolidated view

You asked specifically about the book's age. Here's the honest scorecard.

## 15.1 What has aged perfectly (most of it)

```
   ✅ The backward/forward compatibility framing — still the correct
      mental model, unchanged.
   ✅ "Data outlives code" — more true than ever with data lakes and
      multi-year Kafka retention.
   ✅ The six reasons RPC ≠ local function call — every single one still
      bites, and service meshes made #3 (retry/idempotence) worse.
   ✅ Never reuse a field tag — now enforced by tooling rather than
      remembered by humans.
   ✅ Figure 4-7's unknown-field-loss bug — still ships regularly, now
      usually via ORM/DTO layers and API gateways.
   ✅ "The difficulty of getting different organizations to agree on
      anything outweighs most other concerns" — sociology doesn't
      get deprecated.
   ✅ The 2^53 JSON number problem — completely unfixed. protobuf's JSON
      mapping still stringifies int64 for the same reason.
```

## 15.2 What has genuinely moved on

| Topic in the book | 2026 status |
|---|---|
| `required` in Protobuf | **Removed in proto3.** Protobuf Editions (2023) replaced the syntax-version model |
| "many JSON tools don't bother with schemas" | **JSON Schema 2020-12** is stable, underpins OpenAPI 3.1 and LLM structured output |
| Swagger | **Renamed OpenAPI**; 3.1 aligns with JSON Schema |
| "a database of schema versions is a useful thing to have" | **Schema registries are standard infrastructure** (Confluent, Buf, Glue, Apicurio) with enforced compatibility modes |
| SOAP "still used in many large enterprises" | Legacy only — banking, insurance, HL7, government |
| Thrift | **Largely displaced by gRPC.** Still at Facebook/Meta internally |
| Avro | Still dominant in Kafka + Hadoop-lineage systems; Protobuf gaining |
| Kafka | Became the default event backbone; **ZooKeeper removed (KRaft)** |
| Akka | **Licence changed to BSL (2022)** → Apache Pekko fork |
| Orleans | Version-tolerant serializer added in v7 |
| Erlang `maps` "experimental" | Fully standard |
| Parquet mentioned in passing | Foundation of the **lakehouse**: Iceberg, Delta, Hudi |
| API versioning "no agreement" | **Stripe's version-pinning + transformation chain** became the reference pattern |

## 15.3 Genuinely new things the book couldn't cover

```
   🆕 GRAPHQL — no API versions at all; additive-only evolution enforced
      by the query language. Clients receive only what they request, so
      adding fields is forward compatible by construction.

   🆕 ZERO-COPY FORMATS — Cap'n Proto, FlatBuffers, Apache Arrow.
      The insight: DON'T PARSE AT ALL. The on-disk/on-wire layout IS the
      in-memory layout, so "decoding" is a pointer cast.
      ┌───────────────────────────────────────────────────────────┐
      │ Protobuf/Avro : bytes ──[parse]──► objects ──► use        │
      │ Arrow/FlatBuf : bytes ──────────────────────► use         │
      └───────────────────────────────────────────────────────────┘
      Trade-off: larger encoded size, less flexible evolution, but
      microseconds instead of milliseconds. Arrow in particular made
      cross-language dataframe interop (pandas↔Polars↔DuckDB↔Spark)
      practical.

   🆕 CBOR (RFC 8949) — a standardized binary JSON that actually got
      adopted, unlike the "profusion" the book lists. It's the encoding
      under COSE, WebAuthn/passkeys, and much of IoT.

   🆕 CLOUDEVENTS — CNCF spec standardizing event METADATA (id, source,
      type, subject, time) independently of the payload encoding.

   🆕 ICEBERG'S FIELD IDs — schema evolution for table columns using
      stable numeric IDs. Literally the Protobuf tag idea, one layer up.

   🆕 RUST + serde — a serialization framework where the format is a
      generic parameter, so one derive macro gives you JSON, CBOR,
      MessagePack, Bincode and more. A genuinely different design point
      from code generation.

   🆕 LLM TOOL SCHEMAS — JSON Schema is now the interface between
      applications and language models. An unexpected second life for a
      spec the book called "sometimes a hindrance."
```

## 15.4 A modern decision table

```
   ┌────────────────────────────────┬──────────────────────────────────────┐
   │  SITUATION                     │  REACH FOR                           │
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  Public API, external clients  │  REST + JSON + OpenAPI 3.1           │
   │                                │  (debuggability and ecosystem win)   │
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  Internal service-to-service   │  gRPC + Protobuf + Buf CI checks     │
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  Kafka / event streaming       │  Avro or Protobuf + a schema         │
   │                                │  registry, FULL_TRANSITIVE mode      │
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  Analytics / archival storage  │  Parquet, in an Iceberg table        │
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  Client-driven field selection │  GraphQL                             │
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  TypeScript monorepo           │  tRPC (no codegen, end-to-end types) │
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  Latency-critical, same-DC     │  FlatBuffers / Cap'n Proto (zero-copy)│
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  Config files                  │  YAML/TOML/JSON5 — human-first       │
   ├────────────────────────────────┼──────────────────────────────────────┤
   │  Anything, ever                │  NOT your language's built-in        │
   │                                │  serializer                          │
   └────────────────────────────────┴──────────────────────────────────────┘
```

---

# 16. 💻 CODE APPENDIX

All five encoders below are complete implementations. **Every one produces output byte-identical to the corresponding figure in the book** — I checked the hex, not just the length.

## 16.1 Shared primitives

```python
def varint(n):
    """LEB128 unsigned varint: 7 data bits per byte, top bit = 'more to come'."""
    out = bytearray()
    while True:
        b = n & 0x7f
        n >>= 7
        if n: out.append(b | 0x80)      # set continuation bit
        else: out.append(b); return bytes(out)

def zigzag(n):
    """Map signed -> unsigned so small negatives stay small.
       0 -> 0,  -1 -> 1,  1 -> 2,  -2 -> 3, ..."""
    return (n << 1) ^ (n >> 63) if n < 0 else n << 1
```

Note **who uses zigzag and who doesn't**: Thrift CompactProtocol and Avro do; Protobuf's plain `int64` does *not* (only `sint32`/`sint64` do). That's why 1337 encodes as `f2 14` in Thrift/Avro but `b9 0a` in Protobuf.

## 16.2 MessagePack → Figure 4-1 (66 bytes ✓)

```python
def msgpack(obj):
    if isinstance(obj, dict):
        out = bytearray([0x80 | len(obj)])            # fixmap: 0x80 | count
        for k, v in obj.items(): out += msgpack(k) + msgpack(v)
        return bytes(out)
    if isinstance(obj, list):
        out = bytearray([0x90 | len(obj)])            # fixarray: 0x90 | count
        for v in obj: out += msgpack(v)
        return bytes(out)
    if isinstance(obj, str):
        b = obj.encode()
        return bytes([0xa0 | len(b)]) + b             # fixstr: 0xa0 | length
    if isinstance(obj, int):
        return b"\xcd" + struct.pack(">H", obj)       # uint16
```

## 16.3 Thrift BinaryProtocol → Figure 4-2 (59 bytes ✓)

```python
T_STRING, T_I64, T_LIST, T_STOP = 11, 10, 15, 0

def thrift_binary(rec):
    out = bytearray()
    out.append(T_STRING); out += struct.pack(">h", 1)      # type, then FIELD TAG
    b = rec["userName"].encode(); out += struct.pack(">i", len(b)) + b
    out.append(T_I64);    out += struct.pack(">h", 2)
    out += struct.pack(">q", rec["favoriteNumber"])        # a FULL 8 bytes for 1337
    out.append(T_LIST);   out += struct.pack(">h", 3)
    out.append(T_STRING); out += struct.pack(">i", len(rec["interests"]))
    for s in rec["interests"]:
        b = s.encode(); out += struct.pack(">i", len(b)) + b
    out.append(T_STOP)                                     # 0x00 end of struct
    return bytes(out)
```

## 16.4 Thrift CompactProtocol → Figure 4-3 (34 bytes ✓)

```python
def thrift_compact(rec):
    out = bytearray(); prev = 0
    def field_header(tag, typ):
        nonlocal prev
        delta = tag - prev; prev = tag
        return bytes([(delta << 4) | typ])        # TAG DELTA + TYPE in ONE byte

    out += field_header(1, 8)                     # 8 = binary/string
    b = rec["userName"].encode(); out += varint(len(b)) + b
    out += field_header(2, 6)                     # 6 = i64
    out += varint(zigzag(rec["favoriteNumber"]))  # 2 bytes, not 8
    out += field_header(3, 9)                     # 9 = list
    out.append((len(rec["interests"]) << 4) | 8)  # count + element type
    for s in rec["interests"]:
        b = s.encode(); out += varint(len(b)) + b
    out.append(0x00)
    return bytes(out)
```

## 16.5 Protocol Buffers → Figure 4-4 (33 bytes ✓)

```python
def protobuf(rec):
    out = bytearray()
    def key(tag, wire): return varint((tag << 3) | wire)   # THE key byte format

    b = rec["userName"].encode()
    out += key(1, 2) + varint(len(b)) + b                  # wire 2 = length-delimited
    out += key(2, 0) + varint(rec["favoriteNumber"])       # wire 0 = varint
    for s in rec["interests"]:                             # 'repeated': TAG REPEATS
        b = s.encode(); out += key(3, 2) + varint(len(b)) + b
    return bytes(out)
```

## 16.6 Avro → Figure 4-5 (32 bytes ✓)

```python
def avro(rec):
    out = bytearray()
    b = rec["userName"].encode()
    out += varint(zigzag(len(b))) + b            # NO tag. NO type. Just a length.
    out += varint(zigzag(1))                     # union branch 1 (long, not null)
    out += varint(zigzag(rec["favoriteNumber"]))
    out += varint(zigzag(len(rec["interests"]))) # array block count
    for s in rec["interests"]:
        b = s.encode(); out += varint(zigzag(len(b))) + b
    out.append(0x00)                             # end of array
    return bytes(out)
```

Compare 16.6 with 16.5 line by line. **Avro's encoder has no concept of a field identifier anywhere.** That's the whole design, and it's why the reader must have a compatible schema.

## 16.7 An old parser that survives an unknown field

```python
def protobuf_decode_v1(data, known={1:"userName", 2:"favoriteNumber", 3:"interests"}):
    """OLD parser. Knows tags 1-3 only. Must not choke on tag 4."""
    i, out, skipped = 0, {}, []
    while i < len(data):
        k = data[i]; i += 1
        tag, wire = k >> 3, k & 7
        if wire == 2:                                  # length-delimited
            ln = data[i]; i += 1
            val = data[i:i+ln].decode(); i += ln
        elif wire == 0:                                # varint
            val, shift = 0, 0
            while True:
                b = data[i]; i += 1
                val |= (b & 0x7f) << shift; shift += 7
                if not b & 0x80: break
        if tag in known:
            name = known[tag]
            if name == "interests": out.setdefault(name, []).append(val)
            else: out[name] = val
        else:
            skipped.append((tag, val))    # ← the WIRE TYPE told us how to skip
    return out, skipped
```

**The whole of forward compatibility is in the `else` branch.** The parser doesn't need to know what tag 4 means — it only needs to know how long it is, and the wire type tells it that.

## 16.8 Avro schema resolution (Figure 4-6)

```python
def avro_resolve(written, writer_schema, reader_schema):
    w = {f[0]: written[f[0]] for f in writer_schema}
    result, ignored, defaulted = {}, [], []
    for name, typ, default in reader_schema:
        if name in w:
            result[name] = w[name]        # MATCHED BY NAME — position irrelevant
        else:
            result[name] = default        # reader-only field → reader's default
            defaulted.append(name)
    for name, _ in writer_schema:
        if name not in [f[0] for f in reader_schema]:
            ignored.append(name)          # writer-only field → dropped
    return result, ignored, defaulted
```

## 16.9 The Figure 4-7 fix

```python
class PersonV1Safe:
    """Preserve fields you don't understand, and write them back."""
    FIELDS = ["userName", "favoriteNumber", "interests"]
    def __init__(self, d):
        self.d       = {k: d.get(k) for k in self.FIELDS}
        self.unknown = {k: v for k, v in d.items() if k not in self.FIELDS}
    def to_json(self):
        return {**self.d, **self.unknown}
```

---

# 17. 📌 ONE-PAGE CHEAT SHEET

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  DDIA CH.4 — ENCODING AND EVOLUTION                                           ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  Code changes are NOT instantaneous: rolling upgrades server-side, users who  ║
║  never update client-side. So old+new code and old+new data COEXIST.          ║
║     BACKWARD compat = NEW code reads OLD data  (easy: you know the old format)║
║     FORWARD  compat = OLD code reads NEW data  (hard: must ignore the unknown)║
║                                                                               ║
║  ── ENCODING = in-memory objects → self-contained bytes ────────────────────  ║
║  (= serialization = marshalling. NOT encryption. NOT txn serializability.)    ║
║                                                                               ║
║  LANGUAGE-SPECIFIC (pickle, java.io.Serializable): ❌ one language, ❌ RCE     ║
║     security holes, ❌ versioning afterthought, ❌ slow+bloated. Avoid.        ║
║  JSON/XML/CSV: ⚠️ number ambiguity (2^53! Twitter ships id AND id_str),        ║
║     ⚠️ no binary strings (Base64, +33%), ⚠️ optional complex schemas,          ║
║     ⚠️ CSV has no schema and vague escaping.                                  ║
║     ✅ BUT "the difficulty of getting organizations to agree outweighs most    ║
║        other concerns" — good enough as an INTERCHANGE format.                ║
║  BINARY JSON (MessagePack, BSON…): only 81→66 bytes, because FIELD NAMES ARE  ║
║     STILL IN THERE. No schema = must self-describe.                           ║
║                                                                               ║
║  ── SCHEMA-DRIVEN BINARY: sizes measured on the book's example record ──────  ║
║  Thrift BinaryProtocol 59 B  tag+type+full-width ints                         ║
║  Thrift CompactProtocol 34 B  tag DELTA + type packed in 1 byte; zigzag varint║
║  Protocol Buffers      33 B  key = (tag<<3)|wiretype; 'repeated' = tag repeats║
║  Avro                  32 B  NO tags, NO types, NO names — PURE VALUES        ║
║                                                                               ║
║  ── THRIFT/PROTOBUF EVOLUTION: everything hinges on FIELD TAGS ─────────────  ║
║  ✅ rename a field freely (names aren't in the bytes)                          ║
║  ❌ NEVER change a tag   ❌ NEVER reuse a tag (silent data corruption!)         ║
║  ✅ add fields with NEW tags — old code skips them using the WIRE TYPE         ║
║  ⚠️ new fields must be optional/defaulted; only optional fields can be removed ║
║  int32→int64 ok forward; old code reading it back may TRUNCATE                ║
║  protobuf `repeated` allows optional→repeated evolution; Thrift `list` allows ║
║     nested lists. Pick your trade.                                            ║
║                                                                               ║
║  ── AVRO: writer's schema vs reader's schema ───────────────────────────────  ║
║  They need only be COMPATIBLE, not identical. Resolution MATCHES BY NAME:     ║
║     writer-only field → IGNORED   |   reader-only field → READER'S DEFAULT    ║
║  RULE: only add/remove fields THAT HAVE A DEFAULT.                            ║
║  No optional/required — union types + defaults instead. null must be a union  ║
║     branch (deliberate: Hoare's billion-dollar mistake).                      ║
║  Renaming = reader ALIASES = backward but NOT forward compatible.             ║
║  HOW DOES THE READER GET THE WRITER'S SCHEMA?                                 ║
║     big file → object container file (schema once at the top)                 ║
║     database → version number per record + a schema registry                  ║
║     network  → negotiate once at connection setup                             ║
║  ✨ AVRO'S EDGE: no tags → DYNAMICALLY GENERATED SCHEMAS (dump a DB table      ║
║     straight to Avro; columns→fields by name; no human assigns tags).         ║
║  Codegen: useful in static languages, an obstacle in dynamic ones. Avro makes ║
║     it OPTIONAL — container files are self-describing.                        ║
║                                                                               ║
║  WHY SCHEMAS: compact · self-updating documentation · compatibility CHECKABLE ║
║     BEFORE deploy · compile-time type checking. = schemaless flexibility WITH ║
║     better guarantees and tooling.                                            ║
║                                                                               ║
║  ── MODE 1: DATABASES ──────────────────────────────────────────────────────  ║
║  "storing something = sending a message to your future self"                  ║
║  Needs BOTH directions (rolling upgrades → old code reads new rows).          ║
║  ⚠️ FIG 4-7 BUG: old code decodes → updates → re-encodes → UNKNOWN FIELD LOST. ║
║     The FORMAT preserved it; your ORM/DTO layer threw it away. Keep+merge it. ║
║  📌 DATA OUTLIVES CODE. Five-year-old rows sit in their original encoding.     ║
║     Archival dumps → rewrite in ONE schema → Avro container / Parquet.        ║
║                                                                               ║
║  ── MODE 2: SERVICES ───────────────────────────────────────────────────────  ║
║  REST = philosophy built ON HTTP. SOAP = XML protocol that AVOIDS HTTP, WS-*, ║
║     WSDL, tool-dependent, poor interop, now legacy.                           ║
║  RPC ≠ local call, SIX ways: unpredictable · TIMEOUT = you don't know if it   ║
║     happened · retries need IDEMPOTENCE · wildly variable latency · no        ║
║     pointers · cross-language type mismatch. Location transparency is a LIE.  ║
║  Modern RPC is honest about it: futures, streams, service discovery.          ║
║  ➜ REST for PUBLIC APIs (debuggable, universal, huge ecosystem).              ║
║    RPC for INTERNAL same-org, same-datacenter (faster).                       ║
║  Services simplify to: servers upgrade FIRST, clients SECOND →                ║
║     BACKWARD compat on REQUESTS, FORWARD compat on RESPONSES.                 ║
║  ⚠️ Cross-org = you can't force clients to upgrade = compatibility FOREVER.    ║
║                                                                               ║
║  ── MODE 3: MESSAGE PASSING ────────────────────────────────────────────────  ║
║  Broker = buffer + redelivery + no IP needed + multicast + decoupling.        ║
║  ONE-WAY and asynchronous: send and forget. Brokers enforce NO data model →   ║
║     any encoding → so compatibility gives you INDEPENDENT DEPLOY IN ANY ORDER.║
║  ACTORS: location transparency WORKS here because the model ALREADY assumes   ║
║     messages can be lost locally. Don't strengthen the weak side — WEAKEN THE ║
║     STRONG SIDE. Still need compat for rolling upgrades (Akka→Protobuf).      ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

---

# 18. ✅ Test yourself

1. **Which direction of compatibility is harder, and why?**
   → Forward. Backward compatibility is easy because the new author knows the old format and can handle it explicitly. Forward compatibility requires code written *before* a change to cope gracefully with that change — it must ignore what it cannot understand, which only works if the format carries enough self-description to skip unknown data.

2. **MessagePack saved only 15 bytes over JSON. Avro saved 49. Where did the difference come from?**
   → Field names. MessagePack has no schema, so `userName`, `favoriteNumber` and `interests` — 31 bytes — must travel with the data. Avro has a schema on both ends, so it ships nothing but values.

3. **You rename a Protobuf field from `user_name` to `username`. What breaks?**
   → Nothing on the wire. Field names never appear in the encoded bytes; only tags do. You may break generated-code call sites in your own source, but no stored or in-flight data is affected.

4. **You delete field 7 and later add a new field, reusing tag 7. What happens?**
   → Silent data corruption. Old records still carry a tag-7 value of the old type and meaning, and the new code will happily interpret it as the new field. No exception is raised. My demo showed a favourite number of 1337 becoming an account balance of 1337.

5. **Avro has no field tags. So how can a reader handle a writer that reordered its fields?**
   → Schema resolution matches fields **by name**, comparing writer's and reader's schemas side by side. Order is irrelevant. Writer-only fields are ignored; reader-only fields get the reader's declared default.

6. **Why can't you add an Avro field without a default value?**
   → A reader using the new schema will encounter old records that lack the field, and it has nothing to put there. That breaks backward compatibility. Symmetrically, removing a field without a default breaks forward compatibility.

7. **The encoding format preserved `photoURL` perfectly, yet the field was lost. Where did it go?**
   → In the application's own translation layer. The old code decoded into a model class that had no `photoURL` member, then re-encoded from that class. The bytes were fine; the object model dropped it. Fix by capturing unrecognized keys and merging them back on write.

8. **Why is location transparency defensible in the actor model but not in RPC?**
   → Because actors already assume message delivery can fail, even locally. RPC's local semantics are *stronger* than network reality, so the abstraction leaks. Actors weakened the local side to match the remote side, so there's no gap to leak through.

9. **For services you only need backward compatibility on requests and forward on responses. Why doesn't that simplification apply to databases?**
   → Because with services you can assume servers upgrade before clients, which fixes the direction of every mismatch. A database has no such ordering: a row can be written by any version and read by any version, in any combination, for years.

10. **Given the book is from 2017, what's the single biggest change to how teams handle schema evolution today?**
    → Enforcement moved from human discipline to tooling. Schema registries (Confluent, Buf) and CI checks (`buf breaking`) now reject incompatible changes before they're deployed. The rules Kleppmann describes are unchanged — they're just no longer optional.

---

*All quoted material, figures, examples and byte sequences attributed to the book are from Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly, 2017), Chapter 4. Diagrams have been redrawn in ASCII from the book's originals. Section 15 (2026 updates), the code appendix, and all marked 📌 UPDATE notes are supplementary material I added — treat the post-2017 details as a starting point worth verifying, since that landscape keeps moving. Every byte sequence shown was produced by running the code in the appendix and verified against the book's figures.*
