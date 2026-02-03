# Datomic PostgreSQL Connection Guide

## Architecture (The Onion)
```
┌─────────────────────────────────────────────────────┐
│  PostgreSQL Database: "datomic"                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  Table: "datomic_kvs"                         │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  Datomic DB: "my-app"      (basis-t: N) │  │  │
│  │  ├─────────────────────────────────────────┤  │  │
│  │  │  Datomic DB: "test-db"    (basis-t: M)  │  │  │
│  │  ├─────────────────────────────────────────┤  │  │
│  │  │  Datomic DB: "analytics"  (basis-t: K)  │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```
- **1 PostgreSQL database** → configured in transactor properties
- **1 table** (`datomic_kvs`) → key-value store for all Datomic data
- **N Datomic databases** → isolated by name in URI, stored in same table

## Data Flow
```
WRITES: Peer ──▶ Transactor (port 4334) ──▶ PostgreSQL
READS:  Peer ─────────────────────────────▶ PostgreSQL (direct)
```

## Connection URI

### From config map (recommended)
```clojure
(def db-config
  {:datomic-db   "my-app"           ; Datomic database name
   :pg-host      "localhost"
   :pg-port      5432
   :pg-database  "datomic"          ; PostgreSQL database
   :pg-user      "datomic"
   :pg-password  "datomic"})

(defn make-uri [{:keys [datomic-db pg-host pg-port pg-database pg-user pg-password]}]
  ;; => "datomic:sql://my-app?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic"
  (str "datomic:sql://" datomic-db
       "?jdbc:postgresql://" pg-host ":" pg-port "/" pg-database
       "?user=" pg-user "&password=" pg-password))

(def uri (make-uri db-config))
;=> "datomic:sql://my-app?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic"
```

### Raw format
```
datomic:sql://<datomic-db>?jdbc:postgresql://<pg-host>:<pg-port>/<pg-database>?user=<pg-user>&password=<pg-password>
```

## deps.edn
```clojure
{:deps {com.datomic/peer {:mvn/version "1.0.7075"}
        org.postgresql/postgresql {:mvn/version "42.7.1"}}}
```

## Connect
```clojure
(require '[datomic.api :as d])
(def uri "datomic:sql://my-db?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic")
(d/create-database uri)  ; only needed once per db-name
(def conn (d/connect uri))
```

## Start Transactor First
```powershell
cd D:\bin\datomic\datomic-pro-1.0.7469
.\bin\transactor.cmd <path-to>\dev-transactor.properties
```

## Transactor Properties (dev-transactor.properties)
```properties
protocol=sql
host=localhost
port=4334
sql-url=jdbc:postgresql://localhost:5432/datomic
sql-user=datomic
sql-password=datomic
sql-driver-class=org.postgresql.Driver
memory-index-threshold=32m
memory-index-max=256m
object-cache-max=128m
```

## Create a New Datomic Database
```clojure
;; Just change :datomic-db - no PostgreSQL setup needed
(def testing-uri (make-uri (assoc db-config :datomic-db "testing")))
(d/create-database testing-uri)  ;=> true
(def conn (d/connect testing-uri))
```
All Datomic databases share the same PostgreSQL `datomic_kvs` table.

## PostgreSQL Prerequisites
Database `datomic` with table `datomic_kvs` must exist. See `D:\bin\datomic\datomic-pro-1.0.7469\bin\sql\postgres-table.sql`.

---

## Practical Guide: Schema → Data → Query

### Step 1: Define Schema
Schema defines attributes. Each attribute needs `:db/ident`, `:db/valueType`, and `:db/cardinality`.
```clojure
(def schema
  [;; Person
   {:db/ident       :person/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one}
   {:db/ident       :person/email
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity}

   ;; File
   {:db/ident       :file/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one}
   {:db/ident       :file/path
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one}

   ;; Meeting (with refs to person and file)
   {:db/ident       :meeting/title
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one}
   {:db/ident       :meeting/date
    :db/valueType   :db.type/instant
    :db/cardinality :db.cardinality/one}
   {:db/ident       :meeting/participants
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/many}
   {:db/ident       :meeting/files
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/many}])

;; Transact schema (must be done before data!)
@(d/transact conn schema)
```

**Common value types:** `:db.type/string`, `:db.type/long`, `:db.type/instant`, `:db.type/boolean`, `:db.type/ref`

**Cardinalities:** `:db.cardinality/one` (single value), `:db.cardinality/many` (set of values)

### Step 2: Insert Data
Use temporary string IDs to reference entities within the same transaction.
```clojure
@(d/transact conn
  [;; Entities with temp IDs for cross-referencing
   {:db/id "alice" :person/name "Alice" :person/email "alice@example.com"}
   {:db/id "bob"   :person/name "Bob"   :person/email "bob@example.com"}
   {:db/id "spec"  :file/name "spec.pdf" :file/path "/docs/spec.pdf"}
   {:db/id "deck"  :file/name "slides.pptx" :file/path "/docs/slides.pptx"}

   ;; Meeting referencing other entities by temp ID
   {:meeting/title "Project Kickoff"
    :meeting/date  #inst "2025-02-01T10:00:00"
    :meeting/participants ["alice" "bob"]
    :meeting/files ["spec" "deck"]}])
```

**Note:** `@` dereferences the future - blocks until transaction completes.

### Step 3: Query Data
Always query against `(d/db conn)` - a snapshot of the database.

**Simple query:**
```clojure
(d/q '[:find ?name ?email
       :where
       [?e :person/name ?name]
       [?e :person/email ?email]]
     (d/db conn))
;=> #{["Alice" "alice@example.com"] ["Bob" "bob@example.com"]}
```

**Join across entities:**
```clojure
(d/q '[:find ?title ?person-name
       :where
       [?m :meeting/title ?title]
       [?m :meeting/participants ?p]
       [?p :person/name ?person-name]]
     (d/db conn))
;=> #{["Project Kickoff" "Alice"] ["Project Kickoff" "Bob"]}
```

**Pull nested data (powerful!):**
```clojure
(d/q '[:find (pull ?m [* {:meeting/participants [:person/name :person/email]}
                        {:meeting/files [:file/name :file/path]}])
       :where [?m :meeting/title _]]
     (d/db conn))
;=> [[{:meeting/title "Project Kickoff"
;      :meeting/date #inst "2025-02-01T10:00:00"
;      :meeting/participants [{:person/name "Alice" :person/email "alice@example.com"}
;                             {:person/name "Bob" :person/email "bob@example.com"}]
;      :meeting/files [{:file/name "spec.pdf" :file/path "/docs/spec.pdf"}
;                      {:file/name "slides.pptx" :file/path "/docs/slides.pptx"}]}]]
```

### Quick Reference
| Operation | Code |
|-----------|------|
| Transact schema | `@(d/transact conn schema)` |
| Insert data | `@(d/transact conn [{...} {...}])` |
| Get db snapshot | `(d/db conn)` |
| Simple query | `(d/q '[:find ... :where ...] (d/db conn))` |
| Pull nested | `(d/q '[:find (pull ?e [...]) :where ...] (d/db conn))` |
| List all DBs | `(d/get-database-names "datomic:sql://*?jdbc:...")` |


