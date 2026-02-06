# EAVTA: What Each Component Accepts

This is a precise reference for what you can feed into each position of a Datomic datom. Every datom is `[E A V T added]`, and each position has strict rules about what it will accept.

---

## E — Entity

The entity position identifies _which thing_ a fact is about. You never provide a raw numeric entity ID yourself — Datomic assigns those internally. But you do provide _references_ that Datomic resolves to entity IDs.

### What E accepts in transaction data

**Nothing (omitted)** — When you write a map without `:db/id`, Datomic creates a brand new entity with an auto-assigned ID. This is the most common case.

```clojure
;; No :db/id — Datomic creates a new entity automatically
{:person/name "Alice"
 :person/age 30}
```

**String tempid** — A temporary placeholder that exists only for the duration of one transaction. Any string works. Its sole purpose is to let you reference the same not-yet-created entity from multiple places within the same transaction.

```clojure
;; "alice" and "bob" are tempids — they let these two entities reference each other
[{:db/id "alice" :person/name "Alice" :person/friend "bob"}
 {:db/id "bob"   :person/name "Bob"   :person/friend "alice"}]
```

After the transaction completes, the tempid strings are gone forever. The transaction result contains a `:tempids` map showing what real numeric IDs they resolved to.

**Lookup ref** — A two-element vector of `[unique-attribute value]` that resolves to an existing entity. The attribute must have `:db/unique` set to either `:db.unique/identity` or `:db.unique/value`.

```clojure
;; Find the entity whose :person/email is "alice@example.com" and update its age
{:db/id [:person/email "alice@example.com"]
 :person/age 31}
```

If no entity with that unique value exists, the behavior depends on the uniqueness type. With `:db.unique/identity`, Datomic creates a new entity (this is called "upsert"). With `:db.unique/value`, the transaction fails.

**Existing numeric entity ID** — A long integer you obtained from a previous query or transaction result. You would never make this number up — it always comes from Datomic.

```clojure
;; 17592186045421 came from a query result
{:db/id 17592186045421
 :person/age 31}
```

**The reserved tempid `"datomic.tx"`** — Refers to the transaction entity itself. Used to attach metadata to the current transaction.

```clojure
{:db/id "datomic.tx"
 :audit/user "admin@example.com"}
```

---

## A — Attribute

The attribute position identifies _which property_ is being described. Every attribute must be defined in the schema before use. The definition itself is a transaction containing a map with specific system-defined keys.

### Schema definition keys (used when creating an attribute)

An attribute is defined by transacting an entity with the following keys.

**`:db/ident`** (required) — A namespaced keyword that becomes the attribute's name. This is what you use in all code to refer to the attribute. The namespace is a convention for grouping (like `:person/name`, `:book/title`) but has no enforcement — it is just a keyword.

Accepts: any Clojure keyword, typically namespaced. Examples: `:person/name`, `:order/total`, `:db/doc`.

**`:db/valueType`** (required, immutable after creation) — Declares what kind of values this attribute can hold. Once set, it can never be changed.

Accepts exactly one of these keywords:

```
:db.type/string      — Java String, UTF-8 text
:db.type/long        — 64-bit signed integer (Java long)
:db.type/double      — 64-bit IEEE 754 floating point (preferred over float)
:db.type/float       — 32-bit IEEE 754 floating point (rarely used; prefer double)
:db.type/boolean     — true or false
:db.type/instant     — A point in time (millisecond precision, stored as Java Date)
:db.type/uuid        — 128-bit UUID
:db.type/keyword     — A Clojure keyword like :active or :status/pending
:db.type/bigint      — Arbitrary precision integer (Java BigInteger)
:db.type/bigdec      — Arbitrary precision decimal (Java BigDecimal)
:db.type/ref         — A reference to another entity (stored as entity ID)
:db.type/bytes       — Raw byte array ⚠️ DEPRECATED (no value semantics; byte arrays
                       don't compare or hash as equal even when contents match)
:db.type/uri         — A java.net.URI
:db.type/symbol      — A Clojure symbol
:db.type/tuple       — A fixed-length vector of typed values (composite)
```

**`:db/cardinality`** (required) — Whether the attribute holds one value or a set of values per entity.

Accepts exactly one of:

```
:db.cardinality/one   — The entity has at most one value for this attribute.
                        Writing a new value automatically retracts the old one.

:db.cardinality/many  — The entity has a set of values for this attribute.
                        Writing a new value adds to the set. No automatic retraction.
                        Values are unordered and unique (it is a set, not a list).
```

**`:db/doc`** (optional, mutable) — A documentation string. Any string. Can be changed after creation.

**`:db/unique`** (optional, mutable with caveats) — Enforces uniqueness of values for this attribute across all entities.

Accepts exactly one of:

```
:db.unique/value     — No two entities can have the same value for this attribute.
                       Attempting to transact a duplicate value causes the transaction to fail.

:db.unique/identity  — Same uniqueness guarantee, but additionally enables "upsert":
                       if you transact a value that already exists, Datomic merges the
                       new attributes into the existing entity instead of failing.
                       Also enables lookup refs like [:person/email "alice@example.com"].
```

Can only be set on `:db.cardinality/one` attributes. The attribute's existing data must have no duplicates before you can add uniqueness.

**`:db/isComponent`** (optional, mutable) — Declares that entities referenced by this attribute are "owned" by the parent entity.

Accepts: `true` or `false` (default `false`).

Only meaningful on `:db.type/ref` attributes. When true, retracting the parent entity automatically retracts all component child entities. Pull also automatically traverses component refs.

**`:db/noHistory`** (optional, mutable) — Controls whether Datomic retains past values of this attribute in the history database.

Accepts: `true` or `false` (default `false`).

When true, old values are not kept in the history index after indexing. Useful for high-churn attributes where past values have no business meaning (e.g., a counter that updates every second). Current value is always available; only history is discarded.

**`:db/index`** (optional, mutable) — Controls whether the attribute is included in the AVET index, which enables efficient lookup by value.

Accepts: `true` or `false` (default `false`).

Attributes with `:db/unique` are automatically indexed and do not need this flag. Use `:db/index` for non-unique attributes where you frequently query by value.

**`:db/fulltext`** (optional, immutable after creation) — Enables a fulltext search index for the attribute. Not available in Datomic Cloud.

Accepts: `true` or `false` (default `false`).

Once set, it cannot be altered (unlike most other optional schema keys). Only applicable to `:db.type/string` attributes.

**`:db.attr/preds`** (optional, mutable) — A list of predicate functions that validate values before they are accepted.

Accepts: a vector of fully qualified symbols naming functions. Each function takes one argument (the proposed value) and returns truthy or falsy. If any predicate returns falsy, the transaction fails.

```clojure
{:db/ident       :person/age
 :db/valueType   :db.type/long
 :db/cardinality :db.cardinality/one
 :db.attr/preds  ['my.app.validators/positive-integer?]}
```

**`:db.entity/preds`** (optional, mutable) — A list of predicate functions that validate the _entity as a whole_ (not just a single attribute). Each function takes two arguments: the database value after the transaction (`db-after`) and the entity ID. The transaction fails if any predicate returns falsy. Must be explicitly requested via `:db/ensure` in the transaction.

Accepts: a vector of fully qualified symbols naming functions.

**`:db.entity/attrs`** (optional, mutable) — A list of required attributes for an entity. Used with `:db/ensure` to enforce that specific attributes must be present.

Accepts: a vector of attribute keywords.

**`:db/tupleAttrs`** (for composite tuples) — Declares that this attribute is a composite tuple whose value is automatically derived from other attributes on the same entity.

Accepts: a vector of 2–8 attribute keywords. The tuple value is automatically maintained by Datomic whenever any of the source attributes change.

```clojure
{:db/ident       :student/semester-course
 :db/valueType   :db.type/tuple
 :db/cardinality :db.cardinality/one
 :db/unique      :db.unique/identity
 :db/tupleAttrs  [:student/semester :student/course]}
```

**`:db/tupleTypes`** (for heterogeneous tuples) — Declares the types of each element in a fixed-length tuple.

Accepts: a vector of 2–8 value type keywords (like `:db.type/string`, `:db.type/long`, etc.).

```clojure
{:db/ident       :coordinates/point
 :db/valueType   :db.type/tuple
 :db/cardinality :db.cardinality/one
 :db/tupleTypes  [:db.type/double :db.type/double]}
```

**`:db/tupleType`** (for homogeneous tuples) — Declares that all elements in the tuple share the same type.

Accepts: a single value type keyword. The tuple can be 2–8 elements, all of the specified type.

```clojure
{:db/ident       :color/rgb
 :db/valueType   :db.type/tuple
 :db/cardinality :db.cardinality/one
 :db/tupleType   :db.type/long}
;; Accepts values like [255 128 0]
```

### Complete attribute definition example

```clojure
{:db/ident        :person/email            ;; required — the name
 :db/valueType    :db.type/string          ;; required — the type (immutable)
 :db/cardinality  :db.cardinality/one      ;; required — one or many
 :db/unique       :db.unique/identity      ;; optional — uniqueness + upsert
 :db/doc          "Primary email address"   ;; optional — documentation
 :db.attr/preds   ['my.app/valid-email?]   ;; optional — validation
 :db/noHistory    false                    ;; optional — keep history (default)
 :db/isComponent  false                    ;; optional — not a component (default)
 :db/index        false                    ;; optional — not in AVET index (default)
 :db/fulltext     false}                   ;; optional — no fulltext search (default, immutable)
```

### What A accepts in transaction data (after schema is defined)

When writing datoms (not defining schema), the attribute position accepts a **namespaced keyword** that matches a previously transacted `:db/ident`. That is all. You write `:person/name`, and Datomic resolves it to the attribute's internal entity ID.

---

## V — Value

The value position holds the actual data. What it accepts depends entirely on the `:db/valueType` of the attribute being used.

### Accepted values by type

**`:db.type/string`** — Any Java string. No length limit enforced by Datomic, but extremely large strings (megabytes) should be stored externally (e.g., S3) with a reference in Datomic.

```clojure
:person/name "Alice Chen"
```

**`:db.type/long`** — A 64-bit signed integer. Range: -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807.

```clojure
:person/age 30
:order/quantity 5
```

**`:db.type/double`** — A 64-bit IEEE 754 floating-point number. Subject to normal floating-point precision limitations. Preferred over `:db.type/float`.

```clojure
:measurement/weight 72.5
:geo/latitude 51.5074
```

**`:db.type/float`** — A 32-bit IEEE 754 floating-point number. Rarely used in practice; `:db.type/double` is preferred for its greater precision.

```clojure
:sensor/reading (float 3.14)
```

**`:db.type/boolean`** — Exactly `true` or `false`.

```clojure
:person/active true
:order/cancelled false
```

**`:db.type/instant`** — A point in time with millisecond precision. Stored as a `java.util.Date`. In Clojure, written with the `#inst` reader literal.

```clojure
:event/date #inst "2025-01-15T10:30:00.000Z"
```

Note that instants have no timezone — they represent an absolute point in time (internally stored as milliseconds since Unix epoch).

**`:db.type/uuid`** — A 128-bit universally unique identifier. Written with the `#uuid` reader literal.

```clojure
:person/external-id #uuid "f47ac10b-58cc-4372-a567-0e02b2c3d479"
```

**`:db.type/keyword`** — A Clojure keyword. Commonly used for enumerations and status values.

```clojure
:order/status :status/pending
:person/role :role/admin
```

**`:db.type/bigint`** — An arbitrary-precision integer (Java `BigInteger`). Written with the `N` suffix in Clojure.

```clojure
:astronomy/star-count 99999999999999999999999999999N
```

**`:db.type/bigdec`** — An arbitrary-precision decimal (Java `BigDecimal`). Written with the `M` suffix in Clojure.

```clojure
:finance/precise-amount 1234.56789012345M
```

**`:db.type/ref`** — A reference to another entity. The value is an entity ID (or a tempid, or a lookup ref, or a nested map that implies a new entity). This is how relationships between entities are expressed.

```clojure
;; By entity ID (from a previous query)
:book/author 17592186045421

;; By lookup ref
:book/author [:person/email "herbert@example.com"]

;; By tempid (within the same transaction)
:book/author "temp-author"

;; By nested map (creates a new entity inline)
:book/author {:person/name "Frank Herbert"}
```

**`:db.type/bytes`** — ⚠️ **Deprecated.** A raw byte array. Deprecated because Java byte arrays lack value semantics (two byte arrays with identical contents do not compare or hash as equal). Cannot be used as a unique attribute or lookup ref. Not indexable in Datalog queries. If you need to store binary data, consider using an external store (e.g., S3) and storing a URI or string reference in Datomic.

**`:db.type/uri`** — A `java.net.URI` instance.

```clojure
:resource/url (java.net.URI. "https://example.com/doc")
```

**`:db.type/symbol`** — A Clojure symbol. Rarely used in practice.

```clojure
:rule/function 'my.app/calculate
```

**`:db.type/tuple`** — A fixed-length vector of values. The element types depend on the tuple configuration (`:db/tupleTypes`, `:db/tupleType`, or `:db/tupleAttrs`). String values within a tuple are limited to 256 characters. `nil` is a legal value for any slot in a tuple (useful for range searches, where `nil` sorts lowest).

```clojure
;; Heterogeneous tuple [:db.type/string :db.type/long]
:student/semester-course ["Fall 2025" 101]

;; Homogeneous tuple of :db.type/long
:color/rgb [255 128 0]
```

### Cardinality and values

For `:db.cardinality/one` attributes, you provide a single value. Writing a new value automatically retracts the old one.

For `:db.cardinality/many` attributes, you can provide a single value (which is added to the set) or a vector of values in map form (each is added individually). Each value in the set is stored as a separate datom.

```clojure
;; Cardinality one — replaces old value
{:db/id alice :person/age 31}   ;; If she was 30, that's automatically retracted

;; Cardinality many — adds to the set
{:db/id alice :person/tags ["clever" "kind"]}   ;; Each tag becomes its own datom
;; Later:
{:db/id alice :person/tags ["funny"]}   ;; Adds "funny" to the set; "clever" and "kind" remain
```

---

## T — Transaction

The transaction position is almost entirely system-controlled. You do not specify which transaction a datom belongs to — all datoms in a single `d/transact` call share the same transaction entity automatically.

### What the system provides automatically

**Transaction entity ID** — A numeric ID in the `:db.part/tx` partition. You never set this.

**`:db/txInstant`** — A `:db.type/instant` value recording the wall-clock time of the transaction. Datomic sets this automatically. You _can_ override it by explicitly asserting a `:db/txInstant` value on `"datomic.tx"`, but the value must be at or after the most recent transaction's instant and at or before the current wall-clock time. This is occasionally used when importing historical data.

```clojure
;; Overriding txInstant for historical data import
[{:person/name "Alice"}
 {:db/id "datomic.tx"
  :db/txInstant #inst "2020-01-01T00:00:00.000Z"}]
```

### What you can attach to the transaction entity

You can transact any attributes you have defined onto the transaction entity using the reserved tempid `"datomic.tx"`. Common use cases are audit trails:

```clojure
{:db/id "datomic.tx"
 :audit/user "admin@example.com"      ;; who made this change
 :audit/reason "GDPR data correction"  ;; why
 :audit/source :source/admin-panel}    ;; where it came from
```

These metadata attributes are defined in schema just like any other attributes. There is nothing special about them — they simply happen to be asserted on a transaction entity rather than a domain entity.

### How you interact with T in queries

You can bind the transaction component in Datalog queries using the fourth position in a clause:

```clojure
;; Find when Alice's age was last changed
(d/q '[:find ?tx ?inst
       :where
       [?alice :person/name "Alice"]
       [?alice :person/age _ ?tx]
       [?tx :db/txInstant ?inst]]
     (d/db conn))
```

For time-travel, you can use a transaction ID or a `t` value with `d/as-of`, `d/since`, and `d/history`.

---

## added — The Boolean Flag

The `added` component is the simplest in terms of what it accepts: exactly `true` or `false`. But the way you express intent versus what Datomic actually writes is worth understanding clearly.

### How you express assert/retract

In list form (low-level), you use `:db/add` and `:db/retract`:

```clojure
;; Assert: "Alice's age is 31" → added=true
[:db/add alice-id :person/age 31]

;; Retract: "Alice's age is no longer 30" → added=false
[:db/retract alice-id :person/age 30]
```

In map form (high-level), everything is an assertion by default:

```clojure
;; This is equivalent to [:db/add temp-id :person/name "Alice"]
;;                    and [:db/add temp-id :person/age 30]
{:person/name "Alice"
 :person/age 30}
```

There is no map form for retraction. To retract, you must use the list form `[:db/retract ...]`.

### What Datomic generates implicitly

When you assert a new value on a `:db.cardinality/one` attribute that already has a value, Datomic automatically generates a retraction datom for the old value. You never write this retraction — it is inferred.

When you retract an entity (using `:db/retractEntity`), Datomic generates retraction datoms for every attribute-value pair on that entity, plus cascading retractions for any component children.

```clojure
;; You write:
[:db/retractEntity alice-id]

;; Datomic generates (for every fact about Alice):
;; [alice-id :person/name  "Alice"  tx  false]
;; [alice-id :person/age   31       tx  false]
;; [alice-id :person/email "a@b.c"  tx  false]
;; ... plus retractions for all component children
```

### Where you see `added` explicitly

In normal database queries, you never see `added` — Datomic filters to show only currently-true facts. You encounter `added` in two contexts.

The **history database** returns all datoms ever written, including retractions. You can bind the fifth position in a query clause:

```clojure
(d/q '[:find ?age ?tx ?added
       :where [?e :person/age ?age ?tx ?added]]
     (d/history (d/db conn)))
```

The **transaction log** (`d/tx-range`) returns raw datoms with their `added` flag visible, which is useful for change-data-capture and event-sourcing patterns.

---

## Complete Keyword Reference

### System-defined attribute keys (for schema definitions)

```
:db/ident              keyword      required     The attribute's name
:db/valueType          keyword      required     The data type (immutable after creation)
:db/cardinality        keyword      required     One or many
:db/doc                string       optional     Documentation
:db/unique             keyword      optional     Uniqueness constraint
:db/isComponent        boolean      optional     Component ownership (ref attrs only)
:db/noHistory          boolean      optional     Discard historical values
:db/index              boolean      optional     Include in AVET index
:db/fulltext           boolean      optional     Fulltext search index (immutable; not in Cloud)
:db.attr/preds         [symbol]     optional     Attribute-level validation predicates
:db.entity/preds       [symbol]     optional     Entity-level validation predicates
:db.entity/attrs       [keyword]    optional     Required attributes for entity specs
:db/tupleAttrs         [keyword]    optional     Composite tuple source attributes
:db/tupleTypes         [keyword]    optional     Heterogeneous tuple element types
:db/tupleType          keyword      optional     Homogeneous tuple element type
```

### System-defined value type keywords

```
:db.type/string    :db.type/long      :db.type/double    :db.type/float
:db.type/boolean   :db.type/instant   :db.type/uuid
:db.type/keyword   :db.type/bigint    :db.type/bigdec
:db.type/ref       :db.type/uri       :db.type/symbol
:db.type/tuple

⚠️ Deprecated: :db.type/bytes (no value semantics)
⚠️ Rarely used: :db.type/float (prefer :db.type/double)
```

### System-defined cardinality keywords

```
:db.cardinality/one
:db.cardinality/many
```

### System-defined uniqueness keywords

```
:db.unique/value
:db.unique/identity
```

### System-defined transaction operations

```
:db/add               Assert a datom (added=true)
:db/retract           Retract a datom (added=false)
:db/retractEntity     Retract all datoms for an entity
:db/cas               Compare-and-swap (atomic conditional update)
:db/ensure            Validate entity against a spec (entity preds/attrs)
:db/force-partition   Assign tempids to specific partitions
:db/match-partition   Match tempids to partitions of existing entities
```

### Reserved tempid

```
"datomic.tx"          Refers to the current transaction entity
```

### System-provided transaction attributes

```
:db/txInstant         instant      Auto-set to wall-clock time of the transaction
```