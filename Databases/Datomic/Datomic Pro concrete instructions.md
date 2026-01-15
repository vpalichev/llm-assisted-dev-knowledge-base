# Table of Contents

- [[#Overview]]
- [[#Prerequisites]]
- [[#Installation]]
- [[#Database Schema]]
- [[#API Endpoints]]
- [[#Troubleshooting]]
- [[#Development Workflow]]
- [[#REPL Examples]]
- [[#Reset Database]]
- [[#File Locations]]
- [[#Resources]]

---

# Overview

[[#Table of Contents|Back to TOC]]

The reporting planner uses **Datomic Pro** for report storage. User and session data use EDN files.

- **Reports**: Datomic database
- **Users**: `data/users.edn`
- **Sessions**: `data/sessions.edn`

## Prerequisites

[[#Table of Contents|Back to TOC]]

1. **Java 11+**
2. **Datomic Pro 1.0.7469**

## Installation

[[#Table of Contents|Back to TOC]]

### Step 1: Download Datomic Pro

1. Visit [https://my.datomic.com/downloads/pro](https://my.datomic.com/downloads/pro)
2. Register/login to get access
3. Download **Datomic Pro 1.0.7469**
4. Extract to `D:\bin\datomic\datomic-pro-1.0.7469`

### Step 2: Create Transactor Config

Create `D:\bin\datomic\datomic-pro-1.0.7469\config\development-transactor.properties`:

```properties
protocol=dev
host=localhost
port=4334
data-dir=C:/datomic-data
memory-index-threshold=32m
memory-index-max=64m
object-cache-max=512m
```

**Memory rule**: `object-cache-max + memory-index-max` must be ≤ 75% of JVM heap.

With default 1024m heap: `512m + 64m = 576m` (< 768m ✓)

### Step 3: Start the Transactor

```powershell
cd D:\bin\datomic\datomic-pro-1.0.7469
bin\transactor.cmd config\development-transactor.properties
```

**Expected output:**

```
System started
```

Dev protocol does not display port in output. This is normal.

**Keep this terminal open.**

### Step 4: Configure deps.edn

Add Datomic's Maven repository and dependency:

```clojure
{:mvn/repos {"datomic-releases" {:url "https://datomic-releases.s3.amazonaws.com/maven/releases"}} incorrect

 :deps {org.clojure/clojure {:mvn/version "1.12.0"}
        com.datomic/datomic-pro {:mvn/version "1.0.7469"}}}
```

Verify with:

```powershell
clojure -Stree
```

If dependency fails, force refresh:

```powershell
clojure -Sforce -Stree
```

### Step 5: Configure Application

In your `config.edn`:

```clojure
{:datomic {:uri "datomic:dev://localhost:4334/reporting-planner"}}
```

URI format: `datomic:dev://HOST:PORT/DATABASE-NAME`

- `dev` - protocol (matches transactor config)
- `localhost:4334` - host and port
- `reporting-planner` - database name (you choose this)

### Step 6: Create Database

In REPL:

```clojure
(require '[datomic.api :as d])
(def uri "datomic:dev://localhost:4334/reporting-planner")
(d/create-database uri)
(def conn (d/connect uri))
```

### Step 7: Start the Application

```powershell
clj -M:run
```

## Database Schema

[[#Table of Contents|Back to TOC]]

| Attribute                 | Type           | Description                  |
| ------------------------- | -------------- | ---------------------------- |
| `:report/code`            | string, unique | Report code (REP-001)        |
| `:report/form-type`       | string         | Form type                    |
| `:report/period-type`     | string         | day, week, month, quarter, year |
| `:report/period-value`    | string         | Period value                 |
| `:report/reporting-period`| string         | Combined period string       |
| `:report/current-date`    | string         | YYYY-MM-DD                   |
| `:report/submission-date` | string         | YYYY-MM-DD                   |
| `:report/submitted-by`    | string         | Username                     |
| `:report/status`          | string         | Default: "На рассмотрении"   |
| `:report/data`            | string         | Serialized EDN map           |
| `:report/created-at`      | long           | Timestamp (ms)               |

## API Endpoints

[[#Table of Contents|Back to TOC]]

- `GET /api/reports` - List all reports
- `POST /api/reports` - Create report
- `GET /api/reports/export/:code` - Export to Excel

## Troubleshooting

[[#Table of Contents|Back to TOC]]

### ":db.error/not-enough-memory"

```
(datomic.objectCacheMax + datomic.memoryIndexMax) exceeds 75% of JVM RAM
```

**Fix**: Reduce `object-cache-max` or increase JVM heap in `bin\transactor.cmd`:

```batch
set XMS=-Xms2048m
set XMX=-Xmx2048m
```

### "Could not find artifact com.datomic:datomic-pro"

**Fix**: Add Maven repo to `deps.edn`:

```clojure
:mvn/repos {"datomic-releases" {:url "https://datomic-releases.s3.amazonaws.com/maven/releases"}}
```

### "Failed to read artifact descriptor"

**Fix**: Version mismatch. Use exact version `1.0.7469`:

```clojure
com.datomic/datomic-pro {:mvn/version "1.0.7469"}
```

### "Unable to connect to Datomic"

1. Verify transactor is running
2. Check port matches (default: 4334)
3. Check protocol matches (`dev` in both places)

## Development Workflow

[[#Table of Contents|Back to TOC]]

**Terminal 1 - Transactor:**

```powershell
cd D:\bin\datomic\datomic-pro-1.0.7469
bin\transactor.cmd config\development-transactor.properties
```

**Terminal 2 - Backend:**

```powershell
clj -M:run
```

**Terminal 3 - Frontend:**

```powershell
npx shadow-cljs watch app
```

## REPL Examples

[[#Table of Contents|Back to TOC]]

```clojure
(require '[datomic.api :as d])

(def uri "datomic:dev://localhost:4334/reporting-planner")
(def conn (d/connect uri))
(def db (d/db conn))

;; List all report codes
(d/q '[:find [?code ...]
       :where [?e :report/code ?code]]
     db)

;; Get specific report
(d/pull db '[*] [:report/code "REP-001"])

;; Count reports
(d/q '[:find (count ?e) .
       :where [?e :report/code]]
     db)
```

## Reset Database

[[#Table of Contents|Back to TOC]]

1. Stop application
2. Stop transactor (Ctrl+C)
3. Delete data directory:

    ```powershell
    rm -r C:\datomic-data
    ```

4. Restart transactor
5. Restart application (recreates database)

## File Locations

[[#Table of Contents|Back to TOC]]

| File                | Path                                                                       |
| ------------------- | -------------------------------------------------------------------------- |
| Datomic installation| `D:\bin\datomic\datomic-pro-1.0.7469`                                      |
| Transactor config   | `D:\bin\datomic\datomic-pro-1.0.7469\config\development-transactor.properties` |
| Datomic data        | `C:\datomic-data`                                                          |
| Application config  | `config.edn`                                                               |
| Dependencies        | `deps.edn`                                                                 |

## Resources

[[#Table of Contents|Back to TOC]]

- [Datomic Documentation](https://docs.datomic.com/)
- [Datomic Pro Downloads](https://my.datomic.com/downloads/pro)
- [Datomic Query Reference](https://docs.datomic.com/query/query-data-reference.html)