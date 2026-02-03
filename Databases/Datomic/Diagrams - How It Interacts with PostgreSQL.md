
## The Core Insight: Separation of Reads and Writes

Datomic's architecture is fundamentally different from traditional databases. Instead of having a single database server handle everything, Datomic separates concerns into distinct components. The key insight is that **writes need coordination** (to maintain ACID properties), but **reads don't** (they can happen in parallel without conflict).

---

## Component Overview

Let me first show you what each piece does:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           DATOMIC SYSTEM                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐              │
│  │    PEER 1    │     │    PEER 2    │     │    PEER 3    │              │
│  │  (your app)  │     │  (your app)  │     │  (your app)  │              │
│  │              │     │              │     │              │              │
│  │ ┌──────────┐ │     │ ┌──────────┐ │     │ ┌──────────┐ │              │
│  │ │  Local   │ │     │ │  Local   │ │     │ │  Local   │ │              │
│  │ │  Cache   │ │     │ │  Cache   │ │     │ │  Cache   │ │              │
│  │ └──────────┘ │     │ └──────────┘ │     │ └──────────┘ │              │
│  └──────────────┘     └──────────────┘     └──────────────┘              │
│         │                    │                    │                      │
│         │ writes             │ writes             │ writes               │
│         ▼                    ▼                    ▼                      │
│       ┌───────────────────────────────────────────────┐                  │
│       │                 TRANSACTOR                    │                  │
│       │    (single writer, serializes all writes)     │                  │
│       └───────────────────────────────────────────────┘                  │
│                              │                                           │
│                              │ writes (datoms)                           │
│                              ▼                                           │
└──────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                 ┌───────────────────────────┐
                 │        POSTGRESQL         │
                 │     (Storage Service)     │
                 │                           │
                 │    Stores segments of     │
                 │     immutable datoms      │
                 └───────────────────────────┘
                               ▲
                               │
        ┌──────────────────────┼──────────────────────┐
        │ direct reads         │ direct reads         │ direct reads
        │                      │                      │
   from Peer 1            from Peer 2            from Peer 3
```

---

## What PostgreSQL Actually Stores

Here's a crucial point: **PostgreSQL is not being used as a relational database in the traditional sense**. Datomic uses it as a key-value store for immutable segments of data.

```
┌─────────────────────────────────────────────────────────────────┐
│                      POSTGRESQL TABLES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  DATOMIC_KVS  (key-value storage)                         │  │
│  ├─────────────────────┬─────────────────────────────────────┤  │
│  │  id (key)           │  val (binary blob)                  │  │
│  ├─────────────────────┼─────────────────────────────────────┤  │
│  │  "segment-001"      │  [compressed datoms...]             │  │
│  │  "segment-002"      │  [compressed datoms...]             │  │
│  │  "index-root"       │  [index tree structure...]          │  │
│  │  "log-tail"         │  [recent transactions...]           │  │
│  └─────────────────────┴─────────────────────────────────────┘  │
│                                                                 │
│  Datomic doesn't use PostgreSQL's:                              │
│    ✗ Foreign keys                                               │
│    ✗ Complex queries                                            │
│    ✗ Joins                                                      │
│    ✗ Transactions (beyond simple puts)                          │
│                                                                 │
│  Datomic DOES use PostgreSQL for:                               │
│    ✓ Durable storage                                            │
│    ✓ Simple key-value operations                                │
│    ✓ Replication (if configured)                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

The datoms themselves look like this inside those segments:

```
┌─────────────────────────────────────────────────────────────────┐
│                       DATOM STRUCTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Entity   Attribute    Value        Transaction   Operation]   │
│  [  E         A           V              Tx          Op     ]   │
│                                                                 │
│  Examples:                                                      │
│  [42      :person/name  "Alice"        1000         true  ]     │
│  [42      :person/age   30             1000         true  ]     │
│  [42      :person/age   30             1005         false ]     │ ← retraction
│  [42      :person/age   31             1005         true  ]     │ ← new value
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Write Path in Detail

When your application issues a transaction, here's exactly what happens:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                            WRITE PATH                                     │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  STEP 1: Your Code                                                        │
│  ─────────────────                                                        │
│  @(d/transact conn [{:person/name "Alice" :person/age 30}])               │
│         │                                                                 │
│         ▼                                                                 │
│  STEP 2: Peer Packages Transaction                                        │
│  ─────────────────────────────────                                        │
│  ┌─────────────────────────────┐                                          │
│  │ Transaction Request:        │                                          │
│  │  - tx-data (what to write)  │                                          │
│  │  - tx-id (for ordering)     │                                          │
│  └─────────────────────────────┘                                          │
│         │                                                                 │
│         │  sent over network                                              │
│         ▼                                                                 │
│  STEP 3: Transactor Receives                                              │
│  ────────────────────────────                                             │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │                       TRANSACTOR                            │          │
│  │                                                             │          │
│  │   1. Acquire write lock (only one tx at a time)             │          │
│  │   2. Expand transaction (resolve tempids, defaults)         │          │
│  │   3. Validate (schema constraints, cardinality)             │          │
│  │   4. Generate new datoms with tx-id                         │          │
│  │   5. Index the new datoms                                   │          │
│  │   6. Write to PostgreSQL                                    │          │
│  │   7. Notify all peers of new data                           │          │
│  │                                                             │          │
│  └─────────────────────────────────────────────────────────────┘          │
│         │                                                                 │
│         ▼                                                                 │
│  STEP 4: PostgreSQL Storage                                               │
│  ──────────────────────────                                               │
│  INSERT INTO datomic_kvs (id, val) VALUES ('segment-xyz', <blob>);        │
│         │                                                                 │
│         ▼                                                                 │
│  STEP 5: Broadcast to Peers                                               │
│  ──────────────────────────                                               │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                                │
│  │ Peer 1  │    │ Peer 2  │    │ Peer 3  │                                │
│  │ update! │    │ update! │    │ update! │                                │
│  └─────────┘    └─────────┘    └─────────┘                                │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## The Read Path in Detail

Reads are beautifully simple because they bypass the transactor entirely:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                             READ PATH                                     │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  STEP 1: Your Code                                                        │
│  ─────────────────                                                        │
│  (d/q '[:find ?name :where [?e :person/name ?name]]                       │
│       (d/db conn))                                                        │
│         │                                                                 │
│         ▼                                                                 │
│  STEP 2: Get Database Value                                               │
│  ──────────────────────────                                               │
│  (d/db conn) returns an immutable snapshot at point-in-time T             │
│         │                                                                 │
│         ▼                                                                 │
│  STEP 3: Check Local Cache                                                │
│  ─────────────────────────                                                │
│  ┌─────────────────────────────────────────┐                              │
│  │           PEER LOCAL CACHE              │                              │
│  │                                         │                              │
│  │   Is the data I need already cached?    │                              │
│  │                                         │                              │
│  │   YES ─────────────────────────────────────▶ Return immediately        │
│  │    │                                    │                              │
│  │   NO                                    │                              │
│  │    │                                    │                              │
│  └────┼────────────────────────────────────┘                              │
│       │                                                                   │
│       ▼                                                                   │
│  STEP 4: Fetch from PostgreSQL (if cache miss)                            │
│  ─────────────────────────────────────────────                            │
│  ┌─────────────────────────────────────────┐                              │
│  │             POSTGRESQL                  │                              │
│  │                                         │                              │
│  │   SELECT val FROM datomic_kvs           │                              │
│  │   WHERE id = 'segment-needed';          │                              │
│  │                                         │                              │
│  └─────────────────────────────────────────┘                              │
│       │                                                                   │
│       │  segment data                                                     │
│       ▼                                                                   │
│  STEP 5: Cache and Process                                                │
│  ─────────────────────────                                                │
│  ┌─────────────────────────────────────────┐                              │
│  │   • Store in local cache                │                              │
│  │   • Run Datalog query locally           │                              │
│  │   • Return results to your code         │                              │
│  └─────────────────────────────────────────┘                              │
│                                                                           │
│  NOTE: The transactor is NEVER involved in reads!                         │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Why This Architecture Scales

```
┌───────────────────────────────────────────────────────────────────────────┐
│                       SCALABILITY COMPARISON                              │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  TRADITIONAL DATABASE                      DATOMIC                        │
│  ────────────────────                      ───────                        │
│                                                                           │
│  ┌─────────┐                               ┌─────────┐                    │
│  │  App 1  │───┐                           │ Peer 1  │──────┐             │
│  └─────────┘   │                           └─────────┘      │             │
│                │    ┌────────────┐                          │  reads      │
│  ┌─────────┐   ├───▶│  DATABASE  │         ┌─────────┐      ├─────────┐   │
│  │  App 2  │───┤    │   SERVER   │         │ Peer 2  │──────┤         │   │
│  └─────────┘   │    │            │         └─────────┘      │         ▼   │
│                │    │ ALL reads  │                          │   ┌────────┐│
│  ┌─────────┐   │    │ ALL writes │         ┌─────────┐      │   │Storage ││
│  │  App 3  │───┘    │ bottleneck │         │ Peer 3  │──────┘   │  (PG)  ││
│  └─────────┘        └────────────┘         └─────────┘          └────────┘│
│                                                   │                  ▲    │
│  Problem: Single point                            │ writes           │    │
│  handles everything                               ▼                  │    │
│                                            ┌────────────┐            │    │
│                                            │ Transactor │────────────┘    │
│                                            │  (writes   │                 │
│                                            │   only)    │                 │
│                                            └────────────┘                 │
│                                                                           │
│  Result:                                  Result:                         │
│  - Reads compete with writes              - Reads scale horizontally      │
│  - Scaling = bigger server                - Add peers = more read capacity│
│  - Read replicas add complexity           - Query runs in your JVM        │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## The Immutability Advantage

A key concept that makes this all work is that **Datomic data is immutable**. When you "update" a value, you're actually adding a new fact with a newer timestamp:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                       IMMUTABILITY IN ACTION                              │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Time T=1000: Alice is 30                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │  [42, :person/age, 30, tx-1000, ADDED]                           │     │
│  └──────────────────────────────────────────────────────────────────┘     │
│                                                                           │
│  Time T=1005: Alice turns 31 (birthday!)                                  │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │  [42, :person/age, 30, tx-1005, RETRACTED]  ← old fact removed   │     │
│  │  [42, :person/age, 31, tx-1005, ADDED]      ← new fact added     │     │
│  └──────────────────────────────────────────────────────────────────┘     │
│                                                                           │
│  The old datom [42, :person/age, 30, tx-1000, ADDED] still exists!        │
│  Nothing is ever deleted. This means:                                     │
│                                                                           │
│  • (d/as-of db tx-1000) → sees Alice as 30                                │
│  • (d/as-of db tx-1005) → sees Alice as 31                                │
│  • (d/history db)       → sees both facts                                 │
│                                                                           │
│  WHY THIS MATTERS FOR CACHING:                                            │
│  ─────────────────────────────                                            │
│  Since data never changes, once a peer caches a segment,                  │
│  that cached data is ALWAYS valid. No cache invalidation needed!          │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Summary Table

| Aspect          | PostgreSQL's Role        | Datomic's Role                                       |
| --------------- | ------------------------ | ---------------------------------------------------- |
| Data model      | Simple key-value store   | Rich entity-attribute-value with time                |
| Query execution | None (just fetch blobs)  | Full Datalog engine runs in Peer                     |
| Indexing        | B-tree for keys only     | EAVT, AEVT, AVET, VAET indexes built by Datomic      |
| Transactions    | Simple single-row writes | Full ACID with serializable isolation via Transactor |
| Schema          | Minimal (just kvs table) | Rich schema with types, cardinality, uniqueness      |

The PostgreSQL database essentially becomes a very reliable, durable "disk" that Datomic writes its own data structures to. All the database intelligence—querying, indexing, caching, time-travel—lives in the Datomic layer above it.


