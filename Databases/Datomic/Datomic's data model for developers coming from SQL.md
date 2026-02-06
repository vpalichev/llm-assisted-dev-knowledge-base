If you've spent years thinking in tables, rows, and columns, Datomic will feel like learning a new language. The good news: it's a simpler language. Instead of modeling the world as rectangular tables where each row is a complete record, Datomic stores atomic facts called **datoms**. Every piece of information becomes a discrete statement about what's true, when it became true, and which entity it describes.

This shift unlocks powerful capabilities—time travel queries, automatic bidirectional relationships, and a database that never forgets—but first you need to rewire how you think about data.

## Part 1: Core concepts

### The datom is Datomic's fundamental unit

A datom is an atomic fact with four components that matter for everyday work: **Entity**, **Attribute**, **Value**, and **Transaction**. Think of it as a sentence: "Entity 42 has attribute `:person/name` with value `"Jane Doe"` as of transaction 1000."

```clojure
[entity   attribute      value         transaction]
[42       :person/name   "Jane Doe"    1000       ]
```

Internally, Datomic stores a fifth component—a boolean indicating whether this is an assertion (true) or retraction (false)—but you rarely interact with it directly.

### How a SQL row becomes multiple datoms

In SQL, you'd store a person as a single row with multiple columns:

```sql
INSERT INTO persons (id, name, email, age) 
VALUES (42, 'Jane Doe', 'jane@example.com', 30);
```

In Datomic, that same person becomes three separate datoms—one for each fact. Here's the simplest form, omitting `:db/id` entirely so Datomic assigns an entity ID automatically:

```clojure
[{:person/name "Jane Doe"
  :person/email "jane@example.com"
  :person/age 30}]
```

This map syntax is convenient shorthand. Datomic expands it into individual assertions:

```clojure
[[:db/add 42 :person/name "Jane Doe"]
 [:db/add 42 :person/email "jane@example.com"]
 [:db/add 42 :person/age 30]]
```

After the transaction, your database contains these datoms:

|E|A|V|Tx|
|---|---|---|---|
|42|:person/name|"Jane Doe"|13194|
|42|:person/email|"jane@example.com"|13194|
|42|:person/age|30|13194|

Notice that entity **42** appears in three datoms. An "entity" in Datomic isn't stored anywhere—it's a conceptual view derived from all datoms sharing the same entity ID.

### Schema defines attributes, not tables or entity types

Here's where Datomic diverges most sharply from SQL thinking. You don't define a `persons` table with fixed columns. Instead, you define **attributes** that any entity can possess.

The namespace in `:person/name` is pure convention. Datomic doesn't enforce that entity 42 must have all `:person/*` attributes, or that it can't also have `:employee/salary`. The namespace helps humans organize attributes logically, but the database treats all attributes uniformly. Any entity can have any attribute—there's no rigid structure to violate.

## Part 2: Identity and :db/id

Every entity in Datomic has an entity ID—a long integer that uniquely identifies it within the database. Understanding how to work with `:db/id` is essential for writing transactions.

### When you don't need :db/id

For simple entity creation where nothing else in the transaction references the new entity, you can omit `:db/id` entirely:

```clojure
(d/transact conn 
  {:tx-data 
   [{:person/name "Jane Doe"
     :person/email "jane@example.com"}]})
```

Datomic assigns an entity ID automatically. You'll get the assigned ID back in the transaction result if you need it.

### Temporary IDs (tempids) for cross-references within a transaction

When you need to reference an entity elsewhere in the same transaction—before it has a real ID—use a **temporary ID (tempid)**. A tempid is simply a string you assign to `:db/id`:

```clojure
(d/transact conn 
  {:tx-data 
   [{:db/id "temp-jane"
     :person/name "Jane Doe"}
    {:db/id "temp-company"
     :company/name "Acme Corp"
     :company/ceo "temp-jane"}]})  ;; references Jane by her tempid
```

The string `"temp-jane"` has no meaning to Datomic—it's just a placeholder that gets resolved to a real entity ID when the transaction commits. All references to `"temp-jane"` within the transaction resolve to the same entity.

Tempid strings must be unique within a transaction but have no persistence. You could use `"jane"`, `"person-1"`, or any string you like.

### Lookup refs for existing entities

When referencing an entity that already exists in the database, use a **lookup ref**—a two-element vector containing a unique attribute and its value:

```clojure
[:person/email "jane@example.com"]
```

This only works if `:person/email` has been declared with `:db/unique` in the schema. Lookup refs are how you say "find the entity where this unique attribute has this value" without knowing the entity ID.

```clojure
(d/transact conn 
  {:tx-data 
   [{:db/id [:person/email "jane@example.com"]  ;; lookup ref
     :person/age 31}]})  ;; update Jane's age
```

### Existing entity IDs

If you already have an entity's numeric ID (from a previous query or transaction result), you can use it directly:

```clojure
(d/transact conn 
  {:tx-data 
   [{:db/id 17592186045418
     :person/age 31}]})
```

### Summary of :db/id usage

|Situation|What to use|
|---|---|
|New entity, no references needed|Omit `:db/id`|
|New entity, referenced elsewhere in same transaction|Tempid string: `"my-tempid"`|
|Existing entity with unique attribute|Lookup ref: `[:person/email "jane@example.com"]`|
|Existing entity with known ID|Numeric ID: `17592186045418`|

## Part 3: Schema basics

Every attribute requires exactly three properties. No defaults exist; you must specify all three.

### :db/ident — the attribute's programmatic name

This is a namespaced keyword that identifies the attribute. By convention, use a namespace that groups related attributes together.

```clojure
:db/ident :file/name
:db/ident :person/email
```

The `:db` namespace and all `:db.*` namespaces are reserved for Datomic's internal use.

### :db/valueType — what kind of data it holds

This specifies the primitive type for values. Common types include `:db.type/string`, `:db.type/long`, `:db.type/instant`, and crucially for relationships, `:db.type/ref`. Once set, the value type cannot be changed.

```clojure
:db/valueType :db.type/string    ;; text
:db/valueType :db.type/long      ;; integers
:db/valueType :db.type/ref       ;; reference to another entity
```

### :db/cardinality — one value or many values

An attribute is either single-valued or multi-valued:

```clojure
:db/cardinality :db.cardinality/one   ;; at most one value per entity
:db/cardinality :db.cardinality/many  ;; a set of values per entity
```

A cardinality-one attribute like `:person/email` holds exactly one value. Asserting a new value automatically retracts the old one. A cardinality-many attribute like `:file/responsible-persons` holds a **set** of values—assertions accumulate rather than replace.

Cardinality-many still enforces the value type. If you declare `:db/valueType :db.type/ref`, every value in that set must be an entity reference. It's a homogeneous set, not a mixed bag.

### Complete schema example

Here's the schema we'll use throughout this guide—a `File` entity that references multiple `Person` entities:

```clojure
(def schema
  [;; Person attributes
   {:db/ident       :person/id
    :db/valueType   :db.type/uuid
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity
    :db/doc         "Unique identifier for a person"}
   
   {:db/ident       :person/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "A person's full name"}
   
   ;; File attributes
   {:db/ident       :file/id
    :db/valueType   :db.type/uuid
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity
    :db/doc         "Unique identifier for a file"}
   
   {:db/ident       :file/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "The file's display name"}
   
   {:db/ident       :file/responsible-persons
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/many
    :db/doc         "People responsible for this file"}])

;; Transact the schema
(d/transact conn {:tx-data schema})
```

The `:file/responsible-persons` attribute is cardinality-many with type ref—it can hold references to multiple person entities.

## Part 4: Adding multiple values to a cardinality-many attribute

This section covers every way to associate multiple values with a cardinality-many ref attribute. Understanding these patterns is essential because cardinality-many behaves differently than you might expect from SQL.

The key insight: Datomic is **accumulate-only**. The database never updates in place. When you assert a value on a cardinality-many attribute, you're adding a new fact to the set, not replacing the existing set. This is why multiple assertions accumulate rather than overwrite.

### Scenario 1: Creating a new entity with multiple refs inline

When referenced entities don't exist yet, you can create everything in a single transaction using nested maps. The outer entity references a vector of maps, and Datomic creates all entities atomically.

```clojure
(d/transact conn 
  {:tx-data 
   [{:file/id   #uuid "f47ac10b-58cc-4372-a567-0e02b2c3d479"
     :file/name "Q4-Report.pdf"
     :file/responsible-persons 
     [{:person/id   #uuid "550e8400-e29b-41d4-a716-446655440001"
       :person/name "Alice Chen"}
      {:person/id   #uuid "550e8400-e29b-41d4-a716-446655440002"
       :person/name "Bob Martinez"}]}]})
```

This single transaction creates three entities: one file and two persons. Datomic assigns entity IDs and wires up the references automatically. Behind the scenes, each nested map gets an implicit tempid, and the `:file/responsible-persons` attribute references those tempids.

**Resulting datoms:**

|E|A|V|Tx|
|---|---|---|---|
|17592|:file/id|#uuid "f47ac10b-..."|13194|
|17592|:file/name|"Q4-Report.pdf"|13194|
|17592|:file/responsible-persons|17593|13194|
|17592|:file/responsible-persons|17594|13194|
|17593|:person/id|#uuid "550e8400-...440001"|13194|
|17593|:person/name|"Alice Chen"|13194|
|17594|:person/id|#uuid "550e8400-...440002"|13194|
|17594|:person/name|"Bob Martinez"|13194|

Notice that `:file/responsible-persons` appears twice for entity 17592—once for each person reference. This is how cardinality-many works: separate datoms for each value in the set.

### Scenario 2: Creating a new entity with lookup refs to existing persons

When the referenced entities already exist, use **lookup refs** to reference them by their unique identity attribute. A lookup ref is a two-element vector: `[unique-attribute value]`.

First, ensure the persons exist:

```clojure
(d/transact conn 
  {:tx-data 
   [{:person/id #uuid "550e8400-e29b-41d4-a716-446655440003"
     :person/name "Carol Wong"}
    {:person/id #uuid "550e8400-e29b-41d4-a716-446655440004"
     :person/name "David Kim"}]})
```

Now create a file that references them:

```clojure
(d/transact conn 
  {:tx-data 
   [{:file/id   #uuid "a1b2c3d4-e5f6-4789-abcd-ef0123456789"
     :file/name "Budget-2026.xlsx"
     :file/responsible-persons 
     [[:person/id #uuid "550e8400-e29b-41d4-a716-446655440003"]
      [:person/id #uuid "550e8400-e29b-41d4-a716-446655440004"]]}]})
```

Datomic resolves the lookup refs at transaction time, finding the entity IDs for Carol and David.

**Resulting datoms for the file:**

|E|A|V|Tx|
|---|---|---|---|
|17597|:file/id|#uuid "a1b2c3d4-..."|13196|
|17597|:file/name|"Budget-2026.xlsx"|13196|
|17597|:file/responsible-persons|17595|13196|
|17597|:file/responsible-persons|17596|13196|

The lookup refs resolved to entity IDs 17595 (Carol) and 17596 (David).

### Scenario 3: Adding one more value using :db/add

To add a single new responsible person to an existing file, use the `:db/add` operation. This does not replace existing values—it adds to the set.

```clojure
(d/transact conn 
  {:tx-data 
   [[:db/add 
     [:file/id #uuid "a1b2c3d4-e5f6-4789-abcd-ef0123456789"]
     :file/responsible-persons 
     [:person/id #uuid "550e8400-e29b-41d4-a716-446655440001"]]]})
```

This adds Alice (entity from Scenario 1) to the Budget file. Carol and David remain—we're accumulating, not replacing.

**Resulting datoms (file now has three responsible persons):**

|E|A|V|Tx|
|---|---|---|---|
|17597|:file/responsible-persons|17595|13196|
|17597|:file/responsible-persons|17596|13196|
|17597|:file/responsible-persons|17593|13198|

The third datom was added in the new transaction (13198).

### Scenario 4: Adding several values in a single transaction

To add multiple responsible persons at once, include multiple `:db/add` operations in the same transaction:

```clojure
(d/transact conn 
  {:tx-data 
   [[:db/add 
     [:file/id #uuid "f47ac10b-58cc-4372-a567-0e02b2c3d479"]
     :file/responsible-persons 
     [:person/id #uuid "550e8400-e29b-41d4-a716-446655440003"]]
    [:db/add 
     [:file/id #uuid "f47ac10b-58cc-4372-a567-0e02b2c3d479"]
     :file/responsible-persons 
     [:person/id #uuid "550e8400-e29b-41d4-a716-446655440004"]]]})
```

This adds both Carol and David to the Q4-Report file in one atomic transaction.

**Resulting datoms (Q4-Report now has four responsible persons):**

|E|A|V|Tx|
|---|---|---|---|
|17592|:file/responsible-persons|17593|13194|
|17592|:file/responsible-persons|17594|13194|
|17592|:file/responsible-persons|17595|13199|
|17592|:file/responsible-persons|17596|13199|

### Scenario 5: Using map syntax to add values to an existing entity

Map syntax is more concise than explicit `:db/add` operations. For cardinality-many attributes, the map syntax **also accumulates**—it never replaces.

```clojure
(d/transact conn 
  {:tx-data 
   [{:db/id [:file/id #uuid "a1b2c3d4-e5f6-4789-abcd-ef0123456789"]
     :file/responsible-persons 
     [[:person/id #uuid "550e8400-e29b-41d4-a716-446655440002"]]}]})
```

This adds Bob to the Budget file. Despite appearing to "set" the attribute, the behavior is identical to `:db/add` for cardinality-many.

**Resulting datoms (Budget file now has four responsible persons):**

|E|A|V|Tx|
|---|---|---|---|
|17597|:file/responsible-persons|17595|13196|
|17597|:file/responsible-persons|17596|13196|
|17597|:file/responsible-persons|17593|13198|
|17597|:file/responsible-persons|17594|13200|

### Scenario 6: Removing a value using :db/retract

To remove a specific person from a file's responsible persons, use `:db/retract` with the exact value to remove:

```clojure
(d/transact conn 
  {:tx-data 
   [[:db/retract 
     [:file/id #uuid "a1b2c3d4-e5f6-4789-abcd-ef0123456789"]
     :file/responsible-persons 
     [:person/id #uuid "550e8400-e29b-41d4-a716-446655440002"]]]})
```

This removes Bob from the Budget file. The other responsible persons remain.

**Resulting datoms after retraction:**

Datomic doesn't delete datoms—it adds retraction facts. The "current" database view excludes retracted values, but the history remains queryable.

|E|A|V|Tx|Added|
|---|---|---|---|---|
|17597|:file/responsible-persons|17594|13201|false|

The `false` in the Added column marks this as a retraction.

## Part 5: Querying the results

Datalog queries in Datomic use pattern matching against datoms. Variables (prefixed with `?`) bind to values and create implicit joins when the same variable appears in multiple patterns.

### Finding all files with their responsible persons

The simplest query returns file and person entity IDs:

```clojure
(d/q '[:find ?file ?person
       :where [?file :file/name ?name]
              [?file :file/responsible-persons ?person]]
     db)
```

This returns one tuple per file-person pair. Because `:file/responsible-persons` is cardinality-many, a file with three responsible persons produces three result tuples.

**Result:**

```clojure
#{[17592 17593] [17592 17594] [17592 17595] [17592 17596]
  [17597 17595] [17597 17596] [17597 17593]}
```

### Understanding the join pattern

The query works through variable unification. The variable `?file` appears in both `:where` clauses:

```clojure
[?file :file/name ?name]              ;; ?file binds to entities with :file/name
[?file :file/responsible-persons ?person]  ;; same ?file, now getting its persons
```

Datomic finds all datoms matching the first pattern, then for each bound `?file`, finds all matching datoms for the second pattern. This is an implicit join—no explicit JOIN syntax needed.

### Using pull to get nested entity data

Pull expressions fetch attribute values directly in the query results, including nested entities:

```clojure
(d/q '[:find (pull ?file [:file/name 
                          {:file/responsible-persons [:person/name]}])
       :where [?file :file/name]]
     db)
```

**Result:**

```clojure
[[{:file/name "Q4-Report.pdf"
   :file/responsible-persons [{:person/name "Alice Chen"}
                              {:person/name "Bob Martinez"}
                              {:person/name "Carol Wong"}
                              {:person/name "David Kim"}]}]
 [{:file/name "Budget-2026.xlsx"
   :file/responsible-persons [{:person/name "Carol Wong"}
                              {:person/name "David Kim"}
                              {:person/name "Alice Chen"}]}]]
```

The map spec `{:file/responsible-persons [:person/name]}` tells Datomic to follow the reference and pull the `:person/name` attribute from each referenced entity.

### Finding files for a specific person

To query in the opposite direction—finding all files where a specific person is responsible:

```clojure
(d/q '[:find ?file-name
       :in $ ?person-name
       :where [?person :person/name ?person-name]
              [?file :file/responsible-persons ?person]
              [?file :file/name ?file-name]]
     db "Alice Chen")
```

**Result:**

```clojure
[["Q4-Report.pdf"] ["Budget-2026.xlsx"]]
```

The `:in` clause declares that the query accepts a database (`$`) and a person name parameter. The join flows through `?person`—first binding it to the entity with name "Alice Chen", then finding all files that reference that entity.

### Reverse navigation with pull

Datomic automatically indexes ref attributes in both directions. You can navigate backwards using the `_` prefix:

```clojure
(d/pull db '[:person/name {:file/_responsible-persons [:file/name]}] 
        [:person/id #uuid "550e8400-e29b-41d4-a716-446655440001"])
```

**Result:**

```clojure
{:person/name "Alice Chen"
 :file/_responsible-persons [{:file/name "Q4-Report.pdf"}
                             {:file/name "Budget-2026.xlsx"}]}
```

The `:file/_responsible-persons` pattern means "find all entities that have this person as a value for their `:file/responsible-persons` attribute." No separate index or inverse relationship needed—Datomic handles this automatically.

## Part 6: Common pitfalls for SQL developers

### Pitfall 1: Expecting cardinality-many to replace values

This is the most common mistake. In SQL, `UPDATE files SET responsible_persons = 'Alice' WHERE id = 1` replaces the value. In Datomic, asserting a value on a cardinality-many attribute **adds** to the existing set:

```clojure
;; This ADDS Carol, it doesn't replace existing persons
(d/transact conn 
  {:tx-data 
   [{:db/id [:file/id #uuid "a1b2c3d4-..."]
     :file/responsible-persons [[:person/id #uuid "...carol..."]]}]})
```

To replace all values, you must explicitly retract the old ones first:

```clojure
;; First retract all existing values, then add new ones
(d/transact conn 
  {:tx-data 
   [[:db/retract [:file/id #uuid "a1b2c3d4-..."] :file/responsible-persons]
    {:db/id [:file/id #uuid "a1b2c3d4-..."]
     :file/responsible-persons [[:person/id #uuid "...carol..."]]}]})
```

### Pitfall 2: Thinking namespaces enforce structure

Just because you have `:person/name` and `:person/email` doesn't mean an entity with `:person/name` must have `:person/email`, or that it can't also have `:vehicle/license-plate`. Namespaces are organizational conventions, not constraints.

```clojure
;; This is perfectly valid - one entity with attributes from different "types"
{:person/name "Jane Doe"
 :employee/department "Engineering"
 :vehicle/license-plate "ABC123"}
```

If you need to enforce that certain attributes always appear together, you'll need application-level validation or entity specs.

### Pitfall 3: Looking for NULL values

Datomic has no NULL. An entity either has a datom for an attribute or it doesn't. You can't query for "persons where email is null"—instead, query for persons that lack the email attribute:

```clojure
;; Find persons without an email
(d/q '[:find ?person
       :where [?person :person/name]
              (not [?person :person/email])]
     db)
```

### Pitfall 4: Expecting auto-increment IDs

Datomic entity IDs are assigned by the system and are not sequential integers starting from 1. They're large numbers that encode partition information. Don't rely on their numeric values or ordering.

If you need human-readable sequential IDs, create your own attribute (like `:invoice/number`) and manage the sequence yourself or use a UUID.

### Pitfall 5: Forgetting that history is preserved

When you retract a value, it's not deleted—it's marked as retracted at a point in time. The datom still exists in the historical database. This is a feature (audit trails, time-travel queries), but it means:

- Storage grows over time even if you "delete" data
- Sensitive data that's been "deleted" is still queryable in history
- You can't truly delete data without using excision (a separate, deliberate process)

### Pitfall 6: Using :db/id when you don't need it

If you're creating a standalone entity with no cross-references in the transaction, omit `:db/id`:

```clojure
;; Good - simple and clean
[{:person/name "Jane Doe"}]

;; Unnecessary - adds noise without benefit  
[{:db/id "temp-1" :person/name "Jane Doe"}]
```

Only use tempids when you need to reference the entity elsewhere in the same transaction.

### Pitfall 7: Confusing lookup refs with tempids

Lookup refs reference **existing** entities: `[:person/email "jane@example.com"]`

Tempids create **new** entities within a transaction: `"my-temp-id"`

Using a lookup ref for an entity that doesn't exist will cause the transaction to fail. Using a tempid when you meant to reference an existing entity will create a duplicate.

## Conclusion

Datomic's data model requires unlearning some SQL habits but rewards you with a more flexible, queryable, and historically-aware database. The key mental shifts are viewing data as accumulated facts rather than mutable rows, defining attributes rather than tables, and understanding that cardinality-many attributes form sets that grow through assertion and shrink through retraction.

For cardinality-many ref attributes specifically, remember that both map syntax and `:db/add` accumulate values—neither replaces the existing set. Use lookup refs when referencing existing entities, nested maps when creating related entities together, and `:db/retract` when removing specific values from the set. The pull syntax makes querying relationships straightforward, and Datomic's automatic bidirectional indexing means you never need to maintain inverse relationships manually.