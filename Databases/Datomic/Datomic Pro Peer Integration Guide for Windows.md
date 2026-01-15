# Table of Contents

- [[#Overview]]
- [[#Prerequisites]]
- [[#Storage Backend Options]]
- [[#Installation]]
- [[#Option 1: Dev Storage (Development)]]
- [[#Option 2: Memory Storage (Testing & CI/CD)]]
- [[#Option 3: PostgreSQL Storage (Production)]]
- [[#Starting the Application]]
- [[#Migration from Datomic Free or EDN]]
- [[#Database Schema]]
- [[#Querying the Database]]
- [[#Backup and Restore]]
- [[#Performance Tuning]]
- [[#Troubleshooting]]
- [[#Production Deployment]]
- [[#Storage Comparison]]
- [[#Quick Start Cheat Sheet]]
- [[#License]]

---

This guide explains how to set up and use **Datomic Pro** (Peer library) for the reporting planner application on **Windows**.

## Overview

[[#Table of Contents|Back to TOC]]

The reporting planner uses **Datomic Pro** for report storage. Datomic Pro offers multiple storage backends, production-ready features, and enterprise capabilities beyond Datomic Free.

**What Changed from Free:**

- Different connection URI format based on storage backend
- Multiple storage options: Dev, Memory, and PostgreSQL
- Enhanced features: transaction functions, excision, analytics support
- Better performance: improved caching, query optimization, and scalability

**Data Storage Strategy:**

- **Reports**: Stored in Datomic database
- **Users**: Still in `data/users.edn` (unchanged)
- **Sessions**: Still in `data/sessions.edn` (unchanged)

## Prerequisites

[[#Table of Contents|Back to TOC]]

1. **Java 8 or higher** - Required by Datomic
2. **Datomic Pro license** - Free Apache 2.0 license (no cost)
3. **Storage backend** - Choose based on your requirements

## Storage Backend Options

[[#Table of Contents|Back to TOC]]

| Backend | URI Format | Use Case | Persistence | Transactor Required |
| --- | --- | --- | --- | --- |
| **Dev** | `datomic:dev://` | Development, testing | File-based, local | Yes |
| **Memory** | `datomic:mem://` | Testing, CI/CD | In-memory only | No |
| **PostgreSQL** | `datomic:sql://` | Production | PostgreSQL database | Yes |

**Recommended Choices:**

- **Development**: Dev storage - simple, persistent, file-based
- **Testing/CI**: Memory storage - fast, isolated, disposable, zero configuration
- **Production**: PostgreSQL - reliable, ACID-compliant persistence

## Installation

[[#Table of Contents|Back to TOC]]

### Step 1: Install Datomic Pro

**Option A: Download from my.datomic.com**

1. Log in to [my.datomic.com](https://my.datomic.com/)
2. Download **Datomic Pro** version **1.0.7187** (latest stable as of August 2024)
3. Extract to `C:\datomic-pro-1.0.7187`

**Option B: Use Maven coordinates (recommended)**

Add to `deps.edn`:

```clojure
{:deps {com.datomic/datomic-pro {:mvn/version "1.0.7187"}}}
```

Configure Maven credentials in `C:\Users\YourUsername\.m2\settings.xml`:

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

Get your download key from [my.datomic.com](https://my.datomic.com/) → Account Settings.

---

## Option 1: Dev Storage (Development)

[[#Table of Contents|Back to TOC]]

### Overview

- File-based storage with data persistence
- No external database required
- Simple setup, local only
- Perfect for development work

### Configuration

**config.edn:**

```clojure
{:datomic {:uri "datomic:dev://localhost:4334/reporting-planner"}}
```

### Start Transactor

**PowerShell:**

```powershell
cd C:\datomic-pro-1.0.7187
.\bin\transactor config\samples\dev-transactor-template.properties
```

**Command Prompt:**

```cmd
cd C:\datomic-pro-1.0.7187
bin\transactor.bat config\samples\dev-transactor-template.properties
```

**Expected output:**

```
System started datomic:dev://localhost:4334/<DB-NAME>, storing data in: data ...
```

**Keep this window open!** The transactor must run continuously.

### Customizing Dev Storage

Edit `config\samples\dev-transactor-template.properties`:

```properties
# Change data directory
data-dir=C:/my-datomic-data

# Change port (optional)
port=4334

# Memory settings (adjust based on your system)
memory-index-threshold=32m
memory-index-max=512m
object-cache-max=1g
```

**Note**: Use forward slashes (`/`) or escaped backslashes (`\\`) in properties files.

---

## Option 2: Memory Storage (Testing & CI/CD)

[[#Table of Contents|Back to TOC]]

### Overview

- **In-memory only** - all data is LOST when the application stops
- **No transactor needed** - embedded in your application process
- **Zero configuration** - no setup, no files, no cleanup
- **Blazingly fast** - ideal for tests
- **Isolated** - each process gets its own fresh database
- **Perfect for CI/CD** - no state between test runs

### When to Use

- Unit tests and integration tests
- CI/CD pipelines (GitHub Actions, Jenkins, etc.)
- Quick prototyping without data cleanup
- Demos that always start with clean data

### When NOT to Use

- Production deployments
- Development work where you need to preserve data
- Any situation requiring data persistence

### Configuration

**config.edn:**

```clojure
{:datomic {:uri "datomic:mem://reporting-planner"}}
```

### No Transactor Required

Start your application directly:

```powershell
clj -M:run
```

**Expected output:**

```
Creating Datomic database at: datomic:mem://reporting-planner
Installing Datomic schema...
Datomic connection established
Using Datomic for report storage
Server started on http://0.0.0.0:3000
```

The database exists only while your application runs. Stop the application and all data is gone.

### Multiple Databases

Create independent in-memory databases with different names:

```clojure
;; Testing database
{:uri "datomic:mem://test-db"}

;; Development database
{:uri "datomic:mem://dev-db"}

;; Feature branch database
{:uri "datomic:mem://feature-xyz"}
```

### Use Cases

**Unit Tests:**

```clojure
(deftest test-create-report
  ;; Each test gets a fresh database
  (let [conn (d/connect "datomic:mem://test-db-1")]
    ;; ... test code ...
    ))

(deftest test-update-report
  ;; Completely independent from previous test
  (let [conn (d/connect "datomic:mem://test-db-2")]
    ;; ... test code ...
    ))
```

**CI/CD Pipeline:**

```yaml
# .github/workflows/test.yml
- name: Run Tests
  run: |
    # No setup needed!
    clj -M:test
    # No cleanup needed!
```

---

## Option 3: PostgreSQL Storage (Production)

[[#Table of Contents|Back to TOC]]

### Overview

- PostgreSQL backend - battle-tested, ACID-compliant
- Production-ready with backups, replication, monitoring
- Scalable for large datasets
- Familiar to database administrators

### Install PostgreSQL

Download and install **PostgreSQL 17.7** from [EnterpriseDB](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads):

1. Download Windows installer (64-bit)
2. Run installer
3. Set installation directory (default: `C:\Program Files\PostgreSQL\17`)
4. Set postgres user password (remember this!)
5. Use default port: **5432**
6. Complete installation

**Verify:**

```powershell
psql --version
# Should show: psql (PostgreSQL) 17.7
```

### Create Datomic Database

**Using pgAdmin 4:**

1. Open pgAdmin 4
2. Connect to PostgreSQL (localhost)
3. Right-click "Databases" → Create → Database
4. Name: `datomic`, Owner: `postgres`
5. Click "Save"

**Using Command Line:**

```powershell
# Connect to PostgreSQL
psql -U postgres

# In psql prompt:
CREATE DATABASE datomic;
CREATE USER datomic WITH PASSWORD 'datomic';
GRANT ALL PRIVILEGES ON DATABASE datomic TO datomic;
GRANT ALL ON SCHEMA public TO datomic;
\q
```

### Configure Transactor

Copy and edit the SQL transactor template:

```powershell
cd C:\datomic-pro-1.0.7187
copy config\samples\sql-transactor-template.properties config\my-sql-transactor.properties
```

**Edit `config\my-sql-transactor.properties`:**

```properties
protocol=sql
host=localhost
port=4334

# PostgreSQL JDBC connection
sql-url=jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic
sql-driver-class=org.postgresql.Driver

# Memory settings
memory-index-threshold=32m
memory-index-max=512m
object-cache-max=1g
```

### Add PostgreSQL JDBC Driver

**deps.edn:**

```clojure
{:deps {com.datomic/datomic-pro {:mvn/version "1.0.7187"}
        org.postgresql/postgresql {:mvn/version "42.7.5"}}}
```

Version **42.7.5** is the latest stable PostgreSQL JDBC driver (January 2025).

### Update Application Config

**config.edn:**

```clojure
{:datomic {
   :uri "datomic:sql://reporting-planner?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic"
 }}
```

### Start Transactor

```powershell
cd C:\datomic-pro-1.0.7187
.\bin\transactor config\my-sql-transactor.properties
```

**Expected output:**

```
System started datomic:sql://reporting-planner, storing data in: PostgreSQL ...
```

### PostgreSQL URI Format

```
datomic:sql://[db-name]?[jdbc-url]
```

**Basic:**

```clojure
"datomic:sql://reporting-planner?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic"
```

**With SSL (production):**

```clojure
"datomic:sql://reporting-planner?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic&ssl=true&sslmode=require"
```

### PostgreSQL Performance Tuning

Edit `C:\Program Files\PostgreSQL\17\data\postgresql.conf`:

```properties
# Increase shared memory
shared_buffers = 256MB

# Increase checkpoint segments
max_wal_size = 1GB

# Enable query logging (development only)
log_statement = 'all'
log_duration = on
```

Restart PostgreSQL:

```powershell
Stop-Service postgresql-x64-17
Start-Service postgresql-x64-17
```

---

## Starting the Application

[[#Table of Contents|Back to TOC]]

### Step 1: Start Transactor (if needed)

**Memory storage**: Skip this step

**Dev/PostgreSQL storage**:

```powershell
cd C:\datomic-pro-1.0.7187
.\bin\transactor config\my-transactor.properties
```

Keep this window open!

### Step 2: Start Backend

Open a new PowerShell window:

```powershell
cd C:\path\to\your\project
clj -M:run
```

**Expected output:**

```
Creating Datomic database at: datomic:dev://localhost:4334/reporting-planner
Installing Datomic schema...
Datomic connection established
Using Datomic for report storage
Server started on http://0.0.0.0:3000
```

The application automatically:

1. Creates the database if it doesn't exist
2. Installs the schema
3. Connects to Datomic

---

## Migration from Datomic Free or EDN

[[#Table of Contents|Back to TOC]]

### Migrate from `data/reports.edn`

If you have existing reports in `data/reports.edn`:

```powershell
clj -M -m reporting-planner.migrate
```

**Output:**

```
===========================================
Starting migration: EDN -> Datomic
===========================================

Reading reports from: data/reports.edn
Found 6 reports to migrate

Imported report: REP-001
Imported report: REP-002
...

Migration complete:
  Successfully imported: 6 reports
  Failed: 0 reports
```

### Migrate from Datomic Free

**1. Export from Free** (while Free transactor is running):

```clojure
(require '[datomic.api :as d])
(def conn (d/connect "datomic:free://localhost:4334/reporting-planner"))
(def reports (d/q '[:find [(pull ?e [*]) ...]
                    :where [?e :report/code]]
                  (d/db conn)))
(spit "reports-export.edn" (pr-str reports))
```

**2. Import to Pro** (after starting Pro transactor):

```clojure
(require '[datomic.api :as d])
(require '[reporting-planner.db :as db])

(def reports (clojure.edn/read-string (slurp "reports-export.edn")))

(doseq [report reports]
  @(d/transact (db/get-conn)
               [(dissoc report :db/id)]))

(println "Imported" (count reports) "reports")
```

---

## Database Schema

[[#Table of Contents|Back to TOC]]

| Attribute | Type | Unique? | Description |
| --- | --- | --- | --- |
| `:report/code` | String | Yes | Unique report code (REP-001, REP-002, ...) |
| `:report/form-type` | String | No | Type of form |
| `:report/period-type` | String | No | Period type (day, week, month, quarter, year) |
| `:report/period-value` | String | No | Period value (format depends on period-type) |
| `:report/reporting-period` | String | No | Combined period string (backward compatibility) |
| `:report/current-date` | String | No | Current date (YYYY-MM-DD) |
| `:report/submission-date` | String | No | Submission date (YYYY-MM-DD) |
| `:report/submitted-by` | String | No | Username of submitter |
| `:report/status` | String | No | Report status (default: "На рассмотрении") |
| `:report/data` | String | No | Form data (serialized EDN map) |
| `:report/created-at` | Long | No | Creation timestamp (milliseconds) |

---

## Querying the Database

[[#Table of Contents|Back to TOC]]

### Basic Queries

```clojure
(require '[reporting-planner.db :as db])
(require '[datomic.api :as d])

;; Get database value
(def db-val (db/get-db))

;; Query all report codes
(d/q '[:find [?code ...]
       :where [?e :report/code ?code]]
     db-val)
;; => ["REP-001" "REP-002" "REP-003" ...]

;; Get a specific report
(d/entity db-val [:report/code "REP-001"])
;; => {:report/code "REP-001", :report/form-type "...", ...}

;; Find reports by status
(d/q '[:find ?code ?status
       :where
       [?e :report/code ?code]
       [?e :report/status ?status]
       [?e :report/status "Одобрено"]]
     db-val)

;; Get reports from a specific user
(d/q '[:find [(pull ?e [*]) ...]
       :in $ ?user
       :where [?e :report/submitted-by ?user]]
     db-val
     "admin")
```

### Advanced Queries

**Reports submitted in the last 7 days:**

```clojure
(def seven-days-ago (- (System/currentTimeMillis) (* 7 24 60 60 1000)))

(d/q '[:find ?code ?date
       :in $ ?since
       :where
       [?e :report/code ?code]
       [?e :report/submission-date ?date]
       [?e :report/created-at ?timestamp]
       [(>= ?timestamp ?since)]]
     db-val
     seven-days-ago)
```

**Group reports by status:**

```clojure
(d/q '[:find ?status (count ?e)
       :where [?e :report/status ?status]]
     db-val)
;; => [["На рассмотрении" 5] ["Одобрено" 3] ["Отклонено" 1]]
```

---

## Backup and Restore

[[#Table of Contents|Back to TOC]]

### PostgreSQL Backups

**Automated Backup Script (`backup-postgres.bat`):**

```batch
@echo off
set PGPASSWORD=your_password
set BACKUP_DIR=C:\datomic-backups
set DATE=%date:~-4,4%%date:~-10,2%%date:~-7,2%

"C:\Program Files\PostgreSQL\17\bin\pg_dump" -U datomic datomic > %BACKUP_DIR%\backup-%DATE%.sql

echo Backup completed: %BACKUP_DIR%\backup-%DATE%.sql
```

**Schedule with Task Scheduler:**

1. Open Task Scheduler
2. Create Basic Task → "Datomic Backup"
3. Trigger: Daily at 2:00 AM
4. Action: Start program → `C:\path\to\backup-postgres.bat`

**Restore:**

```powershell
$env:PGPASSWORD="your_password"
psql -U datomic -d datomic -f C:\datomic-backups\backup-20241231.sql
```

### Dev Storage Backups

Copy the data directory:

```powershell
# Backup
Copy-Item -Path "C:\datomic-pro-1.0.7187\data" -Destination "C:\datomic-backups\data-backup-$(Get-Date -Format 'yyyyMMdd')" -Recurse

# Restore
Copy-Item -Path "C:\datomic-backups\data-backup-20241231" -Destination "C:\datomic-pro-1.0.7187\data" -Recurse -Force
```

### EDN Export/Import

**Export:**

```clojure
(require '[reporting-planner.reports-datomic :as reports-db])
(require '[clojure.pprint :refer [pprint]])

(spit "backup-reports.edn"
      (with-out-str (pprint {:reports (reports-db/get-all-reports)})))
```

**Import:**

```clojure
(require '[reporting-planner.migrate :as migrate])

(migrate/import-all-reports-from-edn!
  (clojure.edn/read-string (slurp "backup-reports.edn")))
```

---

## Performance Tuning

[[#Table of Contents|Back to TOC]]

### Memory Settings

Edit transactor properties:

```properties
# Increase memory for large datasets
memory-index-threshold=32m
memory-index-max=512m
object-cache-max=2g
```

### Query Optimization

**Use indexes effectively:**

```clojure
;; Fast: uses unique index
(d/entity db [:report/code "REP-001"])

;; Slower: requires scan
(d/q '[:find ?e .
       :where [?e :report/status "Одобрено"]]
     db)
```

**Add indexes to frequently queried attributes:**

```clojure
;; In schema
{:db/ident :report/status
 :db/valueType :db.type/string
 :db/cardinality :db.cardinality/one
 :db/index true  ; Enable index for faster queries
 :db/doc "Report status"}
```

---

## Troubleshooting

[[#Table of Contents|Back to TOC]]

### "Unable to connect to Datomic"

**Causes & Solutions:**

- Transactor not running → Check PowerShell window
- Wrong URI in `config.edn` → Verify it matches transactor
- PostgreSQL not accessible → Check service status
- Firewall blocking port 4334 → Add exception

### "Unable to find credentials"

**Causes & Solutions:**

- Maven can't download Datomic → Verify `C:\Users\YourUsername\.m2\settings.xml`
- Incorrect credentials → Check [my.datomic.com](https://my.datomic.com/)
- Download manually if needed

### "Could not find JDBC driver"

Add to `deps.edn`:

```clojure
{:deps {org.postgresql/postgresql {:mvn/version "42.7.5"}}}
```

### PostgreSQL Connection Timeouts

```powershell
# Check service
Get-Service postgresql-x64-17

# Start if stopped
Start-Service postgresql-x64-17
```

Increase timeout in transactor properties:

```properties
sql-connection-timeout=30000  ; 30 seconds
```

### Transactor Won't Start

**Check logs:**

```powershell
type C:\datomic-pro-1.0.7187\log\transactor.log
```

**Common issues:**

- Port 4334 in use → Change port in properties
- Insufficient memory → Increase JVM heap
- PostgreSQL unreachable → Verify connection

---

## Production Deployment

[[#Table of Contents|Back to TOC]]

### Architecture

```
┌─────────────┐
│   IIS or    │
│   Nginx     │  (Reverse proxy)
└──────┬──────┘
       │
   ┌───┴────┐
   │        │
┌──▼──┐  ┌──▼──┐
│ App │  │ App │  (Multiple app servers)
│ Peer│  │ Peer│
└──┬──┘  └──┬──┘
   │        │
   └───┬────┘
       │
  ┌────▼─────┐
  │Datomic   │
  │Transactor│  (Single transactor)
  └────┬─────┘
       │
  ┌────▼─────┐
  │PostgreSQL│  (Database server)
  └──────────┘
```

### Security Checklist

- [ ] Use SSL/TLS for PostgreSQL connections
- [ ] Restrict transactor port (4334) to application servers
- [ ] Use strong passwords for PostgreSQL
- [ ] Enable Windows Firewall rules
- [ ] Encrypt data at rest (PostgreSQL TDE)
- [ ] Automate regular backups
- [ ] Monitor logs for suspicious activity
- [ ] Rotate credentials regularly

### Windows Service Setup

Run transactor as a Windows service using [NSSM](https://nssm.cc/):

```powershell
# Install as service
nssm install DatomicTransactor "C:\datomic-pro-1.0.7187\bin\transactor.bat" "config\my-sql-transactor.properties"

# Set working directory
nssm set DatomicTransactor AppDirectory "C:\datomic-pro-1.0.7187"

# Start service
Start-Service DatomicTransactor
```

---

## Storage Comparison

[[#Table of Contents|Back to TOC]]

| Feature | Dev Storage | Memory Storage | PostgreSQL |
| --- | --- | --- | --- |
| **Setup** | Simple | None | Medium |
| **Dependencies** | None | None | PostgreSQL |
| **Persistence** | Yes | No | Yes |
| **Transactor** | Yes | No | Yes |
| **Performance** | Fast | Fastest | Fast |
| **Backup** | File copy | N/A | pg_dump |
| **Production** | No | No | Yes |
| **Best For** | Development | Testing/CI | Production |

---

## Quick Start Cheat Sheet

[[#Table of Contents|Back to TOC]]

### Development (Dev Storage)

```powershell
# 1. Start transactor
cd C:\datomic-pro-1.0.7187
.\bin\transactor config\samples\dev-transactor-template.properties

# 2. Config: {:datomic {:uri "datomic:dev://localhost:4334/reporting-planner"}}

# 3. Start app (new window)
clj -M:run
```

### Testing (Memory Storage)

```powershell
# 1. Config: {:datomic {:uri "datomic:mem://reporting-planner"}}

# 2. Start app (no transactor!)
clj -M:run
```

### Production (PostgreSQL)

```powershell
# 1. Ensure PostgreSQL running
Get-Service postgresql-x64-17

# 2. Start transactor
cd C:\datomic-pro-1.0.7187
.\bin\transactor config\my-sql-transactor.properties

# 3. Config: {:datomic {:uri "datomic:sql://reporting-planner?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic"}}

# 4. Start app (new window)
clj -M:run
```

---

## License

[[#Table of Contents|Back to TOC]]

Datomic Pro binaries are available under the Apache 2.0 license at no cost. Visit [my.datomic.com](https://my.datomic.com/) for more information.