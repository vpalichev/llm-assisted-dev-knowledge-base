
All Datomic databases are stored in the **same** PostgreSQL `datomic` database and `datomic_kvs` table.
```
┌─────────────────────────────────────────────────────────────┐
│  PostgreSQL Server (localhost:5432)                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Database: "datomic"                                  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  Table: "datomic_kvs"                           │  │  │
│  │  │  ┌─────────────────────────────────────────────┐│  │  │
│  │  │  │ id (key)              │ val (blob)          ││  │  │
│  │  │  ├───────────────────────┼─────────────────────┤│  │  │
│  │  │  │ test-db/...           │ <binary data>       ││  │  │
│  │  │  │ test-db/...           │ <binary data>       ││  │  │
│  │  │  │ datomic-testing/...   │ <binary data>       ││  │  │
│  │  │  │ datomic-testing/...   │ <binary data>       ││  │  │
│  │  │  │ prod/...              │ <binary data>       ││  │  │
│  │  │  └─────────────────────────────────────────────┘│  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```
**Key points:**

- **1 PostgreSQL database** (`datomic`) — configured in transactor properties
- **1 table** (`datomic_kvs`) — key-value store for all data
- **N Datomic databases** — distinguished by name prefix in the `id` column

Datomic uses the table as a key-value store. Each Datomic database's data has keys prefixed with its name, keeping them isolated logically but stored physically together.

------------------------
The **transactor** is Datomic's central coordinator process. It's the "single writer" in Datomic's architecture.

**What it does:**
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Peer 1    │     │   Peer 2    │     │   Peer 3    │
│  (your app) │     │  (your app) │     │  (your app) │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │   writes only     │                   │
       └───────────────────┼───────────────────┘
                           ▼
                    ┌─────────────┐
                    │ TRANSACTOR  │  ← serializes all writes
                    │  (1 only)   │  ← ACID guarantees
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ PostgreSQL  │
                    └─────────────┘
                           ▲
       ┌───────────────────┼───────────────────┐
       │     reads (direct)│                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Peer 1    │     │   Peer 2    │     │   Peer 3    │
└─────────────┘     └─────────────┘     └─────────────┘
```
**Why single transactor?**

- No write conflicts — one source of truth
- Simpler consistency model
- Reads scale horizontally (peers read directly from storage)
- Writes go through one point (but Datomic is optimized for read-heavy workloads)

**In your setup:**

- `transactor.cmd` — the Java process running the transactor
- `dev-transactor.properties` — tells it which storage to use
- Port 4334 — where peers connect for writes









------------------------



org.postgresql/postgresql {:mvn/version "42.7.1"}



Done. Here's the summary:

**Files created/modified:**

- [dev-transactor.properties](vscode-webview://16rkfkgf77vh70hu4jofnr27e51ln002l5acl0go7ha0520h7nt4/dev-transactor.properties) — transactor config for PostgreSQL
- [deps.edn](vscode-webview://16rkfkgf77vh70hu4jofnr27e51ln002l5acl0go7ha0520h7nt4/deps.edn) — added PostgreSQL JDBC driver
- [core.clj](vscode-webview://16rkfkgf77vh70hu4jofnr27e51ln002l5acl0go7ha0520h7nt4/src/connectivity_test/core.clj) — added `test-postgres` function

**Next steps to test:**

1. Create the PostgreSQL database:
    
    ```sql
    CREATE DATABASE datomic;
    CREATE USER datomic WITH PASSWORD 'datomic';
    GRANT ALL PRIVILEGES ON DATABASE datomic TO datomic;
    ```
    
2. Start the transactor (in a separate terminal):
    
    ```powershell
    D:\bin\datomic\datomic-pro-1.0.7469\bin\transactor.cmd d:\TEMP\Testing_new_projects\datomic-connectivity-testing\dev-transactor.properties
    ```
    
3. Test from REPL:
    
    ```clojure
    (require '[connectivity-test.core :as ct] :reload)
    (ct/test-postgres)
    ```









# Datomic PostgreSQL Backend Setup
This project tests Datomic connectivity with PostgreSQL as a persistent storage backend.
## Prerequisites
- PostgreSQL running locally
- Datomic Pro installed (e.g., `D:\bin\datomic\datomic-pro-1.0.7469`)
## PostgreSQL Setup
Run these SQL commands to set up the database:
```sql

-- 1. Create database and user

CREATE DATABASE datomic;

CREATE USER datomic WITH PASSWORD 'datomic';

GRANT ALL PRIVILEGES ON DATABASE datomic TO datomic;

  

-- 2. Connect to datomic database

\c datomic

  

-- 3. Create required table

CREATE TABLE datomic_kvs

(

  id text NOT NULL,

  rev integer,

  map text,

  val bytea,

  CONSTRAINT pk_id PRIMARY KEY (id)

);

GRANT ALL ON TABLE datomic_kvs TO datomic;

```
Or use the provided Datomic scripts:
```powershell

psql -U postgres -f "D:\bin\datomic\datomic-pro-1.0.7469\bin\sql\postgres-db.sql"

psql -U postgres -f "D:\bin\datomic\datomic-pro-1.0.7469\bin\sql\postgres-user.sql"

psql -U postgres -d datomic -f "D:\bin\datomic\datomic-pro-1.0.7469\bin\sql\postgres-table.sql"

```
## Start Transactor
The transactor must be run from the Datomic installation directory:
```powershell

Push-Location D:\bin\datomic\datomic-pro-1.0.7469

.\bin\transactor.cmd "d:\TEMP\Testing_new_projects\datomic-connectivity-testing\dev-transactor.properties"

```
Expected output:
```
System started datomic:sql://localhost:4334/<DB-NAME>
```
## Test Connection
From a REPL in the project directory:
```clojure

(require '[connectivity-test.core :as ct] :reload)

  

;; Test in-memory (no transactor needed)

(ct/test-in-memory)

  

;; Test PostgreSQL backend (requires running transactor)

(ct/test-postgres)

```
## Transactor Properties
The `dev-transactor.properties` file configures the transactor:

| Property | Description |

|----------|-------------|

| `protocol=sql` | Use SQL storage backend |

| `host=localhost` | Transactor host |

| `port=4334` | Transactor port |

| `sql-url` | JDBC connection URL |

| `memory-index-max` | Required for SQL protocol |

  

## Troubleshooting

  

### "relation datomic_kvs does not exist"

The `datomic_kvs` table hasn't been created. Run the table creation SQL above.

  

### "missing-transactor-property 'memory-index-max'"

Add memory settings to `dev-transactor.properties`:

```properties

memory-index-threshold=32m

memory-index-max=256m

object-cache-max=128m

```

  

### "The system cannot find the path specified"

Run the transactor from the Datomic installation directory, not from the project directory.





#### What if i wanted to create some kind of datomic-testing database what do I do

Just change the database name in the URI. Datomic stores multiple databases in the same PostgreSQL `datomic_kvs` table.

```clojure
(require '[datomic.api :as d])

;; Create a new database called "datomic-testing"
(def uri "datomic:sql://datomic-testing?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic")

(d/create-database uri)  ;=> true (created)
(def conn (d/connect uri))
(d/basis-t (d/db conn))  ;=> 66 (fresh database)
```

The `datomic-testing` part in the URI is the Datomic database name — you can have as many as you want:

|URI|Datomic DB|
|---|---|
|`datomic:sql://test-db?jdbc:...`|test-db|
|`datomic:sql://datomic-testing?jdbc:...`|datomic-testing|
|`datomic:sql://prod?jdbc:...`|prod|

All stored in the same PostgreSQL database, but isolated from each other.



