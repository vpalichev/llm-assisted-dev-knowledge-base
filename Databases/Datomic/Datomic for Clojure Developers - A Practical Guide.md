# Table of Contents

- [[#1. Introduction to Datomic]]
- [[#2. Setting Up Datomic]]
- [[#3. Connection Management]]
- [[#4. Defining Schema]]
- [[#5. CRUD Operations]]
- [[#6. Datomic Queries]]
- [[#7. Migration from EDN]]
- [[#8. Time-Travel Queries]]
- [[#9. Production Considerations]]
- [[#10. Troubleshooting]]
- [[#Next Steps]]

---

This guide teaches Datomic fundamentals through a real-world reporting application. You'll learn how to model data, define schemas, execute queries, and migrate existing data—all using concrete examples from a production codebase.

## 1. Introduction to Datomic

[[#Table of Contents|Back to TOC]]

### What Makes Datomic Different

Datomic is an immutable database. Every fact you store includes a timestamp, and nothing is ever deleted—only superseded. This gives you:

- **Time-travel queries**: Query the database as it existed at any point in history
- **Audit trails for free**: Every change is preserved with its transaction time
- **Simplified concurrency**: Reads never block writes; you always query a consistent snapshot
- **Local query execution**: The Peer library caches data in your process; queries run without network round-trips

### Core Data Model

Datomic stores **datoms**—atomic facts with five components:

```
[entity-id  attribute  value  transaction-id  added?]
[42         :report/code  "REP-001"  1000  true]
```

This says: "Entity 42 has attribute `:report/code` with value `"REP-001"`, asserted in transaction 1000."

When you "update" a value, Datomic adds a new datom with `added? = true` and retracts the old one with `added? = false`. Both remain in history.

### Datomic Editions

| Edition | Use Case | Storage |
| --- | --- | --- |
| **Datomic Pro** | Production, commercial use | DynamoDB, PostgreSQL, Cassandra, etc. |
| **Datomic Dev-Local** | Development, testing | Local filesystem |
| **Datomic Free** | Discontinued | Was: local H2/filesystem |

This guide uses Datomic Pro with the `dev` protocol for local development. The code patterns apply to all editions.

### Why Migrate from EDN Files?

Consider this EDN-based storage:

```clojure
;; data/reports.edn
{:reports
 [{:code "REP-001"
   :form-type "Форма 1 - Объем инвестиций"
   :period-type "month"
   :period-value "2025-01"
   :submitted-by "ivanov"
   :status "На рассмотрении"
   :data {"domestic-investment" "1500000"
          "foreign-investment" "500000"}
   :created-at 1736956800000}
  ;; hundreds more...
  ]}
```

Problems with EDN files:

- **No concurrent access**: Multiple processes can corrupt data
- **No queries**: Finding reports by status requires loading everything
- **No transactions**: Partial writes on crash leave inconsistent state
- **No history**: Updates destroy previous values

Datomic solves all of these while keeping Clojure's data-oriented philosophy.

---

## 2. Setting Up Datomic

[[#Table of Contents|Back to TOC]]

### Prerequisites

- Java 11+ (verify: `java -version`)
- Clojure CLI tools
- Datomic Pro distribution (download from [my.datomic.com](https://my.datomic.com/))

### Directory Structure

```
C:\datomic-pro\           # Datomic installation (no spaces in path)
C:\projects\reporting-planner\
  ├── deps.edn
  ├── src\clj\reporting_planner\
  │   ├── db.clj          # Connection management, schema
  │   ├── reports_datomic.clj  # Report CRUD operations
  │   └── migrate.clj     # EDN-to-Datomic migration
  └── data\
      └── reports.edn     # Legacy data
```

### Transactor Configuration

Create `C:\datomic-pro\config\dev-transactor.properties`:

```properties
license-key=YOUR_LICENSE_KEY
protocol=dev
host=localhost
port=4334
storage-data-dir=data
log-dir=log
memory-index-threshold=32m
memory-index-max=128m
object-cache-max=256m
```

Start the Transactor:

```powershell
cd C:\datomic-pro
bin\transactor.bat config\dev-transactor.properties
```

Expected output:

```
System started datomic:dev://localhost:4334/<DB-NAME>
```

### Project Configuration

**deps.edn:**

```clojure
{:paths ["src/clj" "resources"]

 :deps {org.clojure/clojure {:mvn/version "1.11.1"}
        com.datomic/datomic-pro {:mvn/version "1.0.7187"}}

 :mvn/repos {"my.datomic.com" {:url "https://my.datomic.com/repo"}}}
```

**~/.m2/settings.xml** (Maven credentials):

```xml
<settings>
  <servers>
    <server>
      <id>my.datomic.com</id>
      <username>YOUR_EMAIL</username>
      <password>YOUR_DOWNLOAD_KEY</password>
    </server>
  </servers>
</settings>
```

**config.edn:**

```clojure
{:datomic {:uri "datomic:dev://localhost:4334/reporting-planner"}}
```

---

## 3. Connection Management

[[#Table of Contents|Back to TOC]]

### The Connection Pattern

Datomic connections are thread-safe and long-lived. Create once, reuse everywhere:

```clojure
(ns reporting-planner.db
  (:require [datomic.api :as d]
            [reporting-planner.config :as config]))

(defonce ^:private conn-atom (atom nil))

(defn get-db-uri []
  (or (get-in config/config [:datomic :uri])
      "datomic:dev://localhost:4334/reporting-planner"))

(defn create-database! []
  (let [uri (get-db-uri)]
    (d/create-database uri)))

(defn get-connection []
  (if-let [conn @conn-atom]
    conn
    (let [uri (get-db-uri)
          _   (create-database!)
          conn (d/connect uri)]
      (install-schema! conn)  ; defined below
      (reset! conn-atom conn)
      conn)))

(defn get-db []
  (d/db (get-connection)))
```

**Key points:**

- `d/create-database` is idempotent—safe to call repeatedly
- `d/connect` returns a connection to an existing database
- `d/db` returns an immutable database value (a snapshot)
- The atom caches the connection; subsequent calls return the cached instance

### Connection vs Database Value

```clojure
;; Connection: mutable reference to the database
(def conn (d/connect uri))

;; Database value: immutable snapshot at a point in time
(def db-now (d/db conn))

;; Queries use database values, not connections
(d/q '[:find ?e :where [?e :report/code]] db-now)

;; Transactions use connections
@(d/transact conn [{:report/code "REP-001"}])
```

---

## 4. Defining Schema

[[#Table of Contents|Back to TOC]]

### Mapping EDN to Datomic Attributes

Your EDN report:

```clojure
{:code "REP-001"
 :form-type "Форма 1 - Объем инвестиций"
 :period-type "month"
 :period-value "2025-01"
 :reporting-period "month:2025-01"
 :current-date "2025-01-15"
 :submission-date "2025-01-15"
 :submitted-by "ivanov"
 :status "На рассмотрении"
 :data {"domestic-investment" "1500000"
        "foreign-investment" "500000"}
 :created-at 1736956800000}
```

Becomes this schema:

```clojure
(def schema
  [;; Identity attribute - enforces uniqueness
   {:db/ident       :report/code
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity
    :db/doc         "Unique sequential report code (REP-001, REP-002, ...)"}

   {:db/ident       :report/form-type
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "Form type name"}

   {:db/ident       :report/period-type
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "Period granularity: month, quarter, year"}

   {:db/ident       :report/period-value
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "Period identifier: 2025-01, Q1-2025, 2025"}

   {:db/ident       :report/reporting-period
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "Combined period: month:2025-01"}

   {:db/ident       :report/current-date
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one}

   {:db/ident       :report/submission-date
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one}

   {:db/ident       :report/submitted-by
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "Username of submitter"}

   {:db/ident       :report/status
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "Workflow status"}

   ;; Complex nested data stored as serialized EDN
   {:db/ident       :report/data
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "Form field values as serialized EDN"}

   {:db/ident       :report/created-at
    :db/valueType   :db.type/long
    :db/cardinality :db.cardinality/one
    :db/doc         "Unix timestamp milliseconds"}])
```

### Schema Attribute Properties

| Property | Values | Purpose |
| --- | --- | --- |
| `:db/valueType` | `:db.type/string`, `:db.type/long`, `:db.type/instant`, `:db.type/ref`, etc. | Data type |
| `:db/cardinality` | `:db.cardinality/one`, `:db.cardinality/many` | Single value vs set |
| `:db/unique` | `:db.unique/identity`, `:db.unique/value` | Uniqueness constraint |
| `:db/index` | `true` | Enable fast lookup (automatic for `:db/unique`) |

### Identity vs Value Uniqueness

- **`:db.unique/identity`**: Upsert behavior. Transacting `{:report/code "REP-001" :report/status "Approved"}` updates existing entity with that code.
- **`:db.unique/value`**: Uniqueness only. Attempting to add a duplicate throws an exception.

### Installing Schema

Schema installation is idempotent—transacting existing attributes is a no-op:

```clojure
(defn install-schema! [conn]
  @(d/transact conn schema)
  (println "Schema installed"))
```

Call this at connection time; it's safe to run on every startup.

---

## 5. CRUD Operations

[[#Table of Contents|Back to TOC]]

### Create: Transacting New Reports

```clojure
(ns reporting-planner.reports-datomic
  (:require [datomic.api :as d]
            [reporting-planner.db :as db]))

(defn serialize-data [m]
  (pr-str m))

(defn deserialize-data [s]
  (when s (read-string s)))

(defn get-next-report-code []
  (let [db-val (db/get-db)
        max-num (or (d/q '[:find (max ?num) .
                           :where
                           [?e :report/code ?code]
                           [(re-find #"REP-(\d+)" ?code) [_ ?num-str]]
                           [(Integer/parseInt ?num-str) ?num]]
                         db-val)
                    0)]
    (format "REP-%03d" (inc max-num))))

(defn create-report! [username report-data]
  (let [code (get-next-report-code)
        form-data (get report-data "data" {})
        tx-data [{:report/code code
                  :report/form-type (get report-data "form-type")
                  :report/period-type (get report-data "period-type")
                  :report/period-value (get report-data "period-value")
                  :report/reporting-period (str (get report-data "period-type")
                                                ":"
                                                (get report-data "period-value"))
                  :report/submission-date (get report-data "current-date")
                  :report/current-date (get report-data "current-date")
                  :report/submitted-by username
                  :report/status "На рассмотрении"
                  :report/data (serialize-data form-data)
                  :report/created-at (System/currentTimeMillis)}]
        result @(d/transact (db/get-connection) tx-data)]
    {:success true
     :code code
     :tempids (:tempids result)}))
```

**Transaction return value:**

```clojure
{:db-before #datomic.db.Db{...}
 :db-after  #datomic.db.Db{...}
 :tx-data   [#datom[...] ...]
 :tempids   {-9223301668109598134 17592186045418}}
```

### Read: Querying Reports

**Get all reports:**

```clojure
(defn get-all-reports []
  (let [db-val (db/get-db)
        entities (d/q '[:find [(pull ?e [*]) ...]
                        :where [?e :report/code]]
                      db-val)]
    (mapv (fn [e]
            {:code (:report/code e)
             :form-type (:report/form-type e)
             :period-type (:report/period-type e)
             :period-value (:report/period-value e)
             :reporting-period (:report/reporting-period e)
             :submission-date (:report/submission-date e)
             :current-date (:report/current-date e)
             :submitted-by (:report/submitted-by e)
             :status (:report/status e)
             :data (deserialize-data (:report/data e))
             :created-at (:report/created-at e)})
          entities)))
```

**Get single report by code (using identity lookup):**

```clojure
(defn get-report-by-code [code]
  (let [db-val (db/get-db)
        ;; Lookup ref: [unique-attr value] resolves to entity id
        entity (d/entity db-val [:report/code code])]
    (when (:report/code entity)  ; entity exists
      {:code (:report/code entity)
       :form-type (:report/form-type entity)
       :status (:report/status entity)
       :data (deserialize-data (:report/data entity))
       ;; ... other fields
       })))
```

**Filter by status:**

```clojure
(defn get-reports-by-status [status]
  (let [db-val (db/get-db)]
    (d/q '[:find [(pull ?e [:report/code :report/form-type :report/status]) ...]
           :in $ ?status
           :where
           [?e :report/status ?status]]
         db-val
         status)))
```

### Update: Modifying Reports

Datomic updates work through upsert on identity attributes:

```clojure
(defn update-report-status! [code new-status]
  (let [tx-data [{:report/code code  ; identity attr finds existing entity
                  :report/status new-status}]]
    @(d/transact (db/get-connection) tx-data)))

(defn update-report-data! [code new-data]
  @(d/transact (db/get-connection)
               [{:report/code code
                 :report/data (serialize-data new-data)}]))
```

### Delete: Retracting Entities

```clojure
(defn delete-report! [code]
  (let [db-val (db/get-db)
        eid (:db/id (d/entity db-val [:report/code code]))]
    (when eid
      @(d/transact (db/get-connection)
                   [[:db/retractEntity eid]]))))
```

Note: Retraction removes the entity from current view but preserves it in history.

---

## 6. Datomic Queries

[[#Table of Contents|Back to TOC]]

### Datalog Basics

Query structure:

```clojure
(d/q '[:find ?code ?status      ; what to return
       :in $ ?submitter         ; inputs: $ = database, others are params
       :where                   ; pattern matching clauses
       [?e :report/submitted-by ?submitter]
       [?e :report/code ?code]
       [?e :report/status ?status]]
     db-val
     "ivanov")
```

### Pull Expressions

Pull retrieves multiple attributes in one operation:

```clojure
;; Pull specific attributes
(d/q '[:find [(pull ?e [:report/code :report/status]) ...]
       :where [?e :report/code]]
     db-val)
;; => [{:report/code "REP-001" :report/status "На рассмотрении"} ...]

;; Pull all attributes
(d/q '[:find [(pull ?e [*]) ...]
       :where [?e :report/code]]
     db-val)
```

### Aggregates

```clojure
;; Count reports by status
(d/q '[:find ?status (count ?e)
       :where
       [?e :report/status ?status]]
     db-val)
;; => [["На рассмотрении" 5] ["Утвержден" 3]]

;; Get maximum report number
(d/q '[:find (max ?num) .  ; . returns scalar instead of set
       :where
       [?e :report/code ?code]
       [(re-find #"REP-(\d+)" ?code) [_ ?num-str]]
       [(Integer/parseInt ?num-str) ?num]]
     db-val)
```

### Return Formats

```clojure
;; Set of tuples (default)
[:find ?a ?b :where ...]  ; => #{[v1 v2] [v3 v4]}

;; Collection (single column)
[:find [?a ...] :where ...]  ; => [v1 v2 v3]

;; Single tuple
[:find [?a ?b] :where ...]  ; => [v1 v2]

;; Scalar
[:find ?a . :where ...]  ; => v1
```

---

## 7. Migration from EDN

[[#Table of Contents|Back to TOC]]

### Reading Legacy Data

```clojure
(ns reporting-planner.migrate
  (:require [clojure.edn :as edn]
            [clojure.java.io :as io]
            [reporting-planner.reports-datomic :as reports-db]))

(defn read-edn-file [path]
  (with-open [r (io/reader path)]
    (edn/read (java.io.PushbackReader. r))))
```

### Import Single Report

```clojure
(defn import-report-from-edn! [report-map]
  (try
    (let [tx-data [{:report/code (:code report-map)
                    :report/form-type (:form-type report-map)
                    :report/period-type (:period-type report-map)
                    :report/period-value (:period-value report-map)
                    :report/reporting-period (:reporting-period report-map)
                    :report/current-date (:current-date report-map)
                    :report/submission-date (:submission-date report-map)
                    :report/submitted-by (:submitted-by report-map)
                    :report/status (:status report-map)
                    :report/data (pr-str (:data report-map))
                    :report/created-at (:created-at report-map)}]]
      @(d/transact (db/get-connection) tx-data)
      (:code report-map))
    (catch Exception e
      (println "Failed to import" (:code report-map) (.getMessage e))
      nil)))
```

### Batch Migration

```clojure
(defn migrate-reports-to-datomic! []
  (let [edn-data (read-edn-file "data/reports.edn")
        reports (:reports edn-data [])
        results (doall (map import-report-from-edn! reports))
        successes (filter some? results)]
    {:total (count reports)
     :success (count successes)
     :failed (- (count reports) (count successes))}))
```

**REPL session:**

```clojure
user=> (require '[reporting-planner.migrate :as m])
user=> (m/migrate-reports-to-datomic!)
{:total 15 :success 15 :failed 0}
```

### Handling Complex Nested Data

The `:data` field contains arbitrary form values:

```clojure
{"domestic-investment" "1500000"
 "foreign-investment" "500000"
 "production-volume" "2500000"}
```

Options for storing this:

1. **Serialize as EDN string** (chosen approach): Simple, preserves arbitrary structure
2. **Component entities**: More queryable, but requires schema for every field
3. **JSON type** (Datomic 1.0.6610+): Native JSON storage

The serialization approach:

```clojure
(defn serialize-data [m]
  (when m (pr-str m)))

(defn deserialize-data [s]
  (when s (read-string s)))

;; Storing
{:report/data (serialize-data {"field1" "value1"})}

;; Retrieving
(deserialize-data (:report/data entity))
```

---

## 8. Time-Travel Queries

[[#Table of Contents|Back to TOC]]

### Querying Historical States

```clojure
;; Get database as of a specific time
(def db-yesterday (d/as-of (db/get-db) #inst "2025-01-14"))

;; Query past state
(d/q '[:find ?status .
       :where [?e :report/code "REP-001"]
              [?e :report/status ?status]]
     db-yesterday)

;; Get history database (sees all datoms, including retracted)
(def db-history (d/history (db/get-db)))

;; Find all status changes for a report
(d/q '[:find ?status ?tx ?added
       :where
       [?e :report/code "REP-001"]
       [?e :report/status ?status ?tx ?added]]
     db-history)
;; => #{["На рассмотрении" 13194139533318 true]
;;      ["На рассмотрении" 13194139533325 false]
;;      ["Утвержден" 13194139533325 true]}
```

### Transaction Timestamps

```clojure
;; Get transaction time
(defn get-tx-instant [db tx-id]
  (:db/txInstant (d/entity db tx-id)))

;; Find when report was created
(d/q '[:find ?inst .
       :where
       [?e :report/code "REP-001" ?tx]
       [?tx :db/txInstant ?inst]]
     db-val)
```

---

## 9. Production Considerations

[[#Table of Contents|Back to TOC]]

### Connection Lifecycle

```clojure
(defn shutdown! []
  (when-let [conn @conn-atom]
    (d/release conn)
    (reset! conn-atom nil)))

;; Call on application shutdown
(.addShutdownHook (Runtime/getRuntime)
                  (Thread. shutdown!))
```

### Schema Evolution

Adding new attributes is safe—just transact them:

```clojure
@(d/transact conn
  [{:db/ident :report/reviewed-by
    :db/valueType :db.type/string
    :db/cardinality :db.cardinality/one}])
```

Renaming/removing attributes requires migration. Use `:db/ident` rename:

```clojure
@(d/transact conn
  [{:db/id :report/old-name
    :db/ident :report/new-name}])
```

### Error Handling

```clojure
(defn safe-transact [conn tx-data]
  (try
    {:success true
     :result @(d/transact conn tx-data)}
    (catch java.util.concurrent.ExecutionException e
      (let [cause (.getCause e)]
        (cond
          (instance? IllegalArgumentException cause)
          {:success false :error :validation :message (.getMessage cause)}

          (instance? java.lang.IllegalStateException cause)
          {:success false :error :conflict :message (.getMessage cause)}

          :else
          {:success false :error :unknown :message (.getMessage e)})))))
```

---

## 10. Troubleshooting

[[#Table of Contents|Back to TOC]]

### Connection Failures

```
CompilerException java.lang.RuntimeException: Could not find datomic.api
```

→ Missing dependency. Verify `deps.edn` includes `com.datomic/datomic-pro` and Maven credentials are configured.

```
ExceptionInfo Unable to connect to localhost:4334
```

→ Transactor not running. Start it with `bin\transactor.bat config\dev-transactor.properties`.

### Schema Errors

```
:db.error/unique-conflict
```

→ Attempting to assert duplicate value for `:db.unique/identity` or `:db.unique/value` attribute.

```
:db.error/invalid-entity-id
```

→ Referencing an entity ID that doesn't exist. Check lookup refs are correct.

### Windows-Specific Issues

| Symptom | Fix |
| --- | --- |
| Transactor silently fails | Check properties file encoding (UTF-8, no BOM) |
| "Address in use" | Another process on port 4334; change port or kill process |
| Path errors | Use forward slashes in properties: `storage-data-dir=C:/datomic/data` |

### REPL Diagnostics

```clojure
;; Check connection
(d/db (db/get-connection))

;; List all attributes in schema
(d/q '[:find ?ident
       :where [?e :db/ident ?ident]
              [?e :db/valueType]]
     (db/get-db))

;; Count entities with an attribute
(d/q '[:find (count ?e) .
       :where [?e :report/code]]
     (db/get-db))
```

---

## Next Steps

[[#Table of Contents|Back to TOC]]

Once you have basic CRUD working:

1. **Learn advanced Datalog**: Rules, recursive queries, query functions
2. **Explore entity refs**: Model relationships between entities
3. **Transaction functions**: Custom atomic operations in the transactor
4. **Filtered databases**: Security via database-level filtering
5. **Peer Server**: Deploy Client API for lighter-weight clients

Resources:

- [Datomic Documentation](https://docs.datomic.com/)
- [Learn Datalog Today](http://www.learndatalogtoday.org/)
- [Day of Datomic tutorials](https://github.com/Datomic/day-of-datomic)