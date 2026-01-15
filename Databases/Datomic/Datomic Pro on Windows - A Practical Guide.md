
This guide is for developers comfortable with JVM tooling who are new to Datomic. We'll start by clarifying Datomic's architecture, then walk through getting a working setup on Windows.

## Understanding Datomic's Architecture

Before touching configuration files, let's get the mental model straight. Datomic's terminology trips up nearly everyone at first.

### The Three Core Pieces

Datomic separates concerns that traditional databases combine:

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR APPLICATION                         │
│                    (includes Peer Library)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Peer Library: queries data locally, submits transactions│   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────────────┘
                      │ submits transactions
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                         TRANSACTOR                              │
│         Single process that serializes all writes               │
│         Writes to storage, notifies peers of changes            │
└─────────────────────┬───────────────────────────────────────────┘
                      │ reads/writes
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                          STORAGE                                │
│   (DynamoDB, PostgreSQL, Cassandra, or local dev storage)       │
└─────────────────────────────────────────────────────────────────┘
```

### The Transactor

The Transactor is a **standalone process** you run separately from your application. It has one job: serialize all writes to the database. Think of it as the single point of truth for "what happened and in what order."

Key points:

- Only **one** Transactor can run per database at a time
- It does NOT serve reads—your application reads data directly from the storage layer via the Peer library's local cache
- It broadcasts changes to connected Peers so they can update their caches
- You start it with a batch script (`bin\transactor.bat`) and a properties file

### The Peer Library

The Peer Library is a **JAR file** your application includes as a dependency. When people say "Peer," they mean your application process running with this library embedded.

What it does:

- Maintains a **local cache** of database segments (this is why Datomic reads are fast)
- Executes queries **locally** against that cache—no round-trips to a server
- Submits transactions to the Transactor (the only thing that requires network)
- Receives notifications from the Transactor when data changes

### "Peer Library" vs "Pro Library"—Are They Different?

**No.** This is just confusing terminology. When people say:

- **Peer Library** → The library your application uses to interact with Datomic
- **Pro Library** → The Peer Library for Datomic Pro (the commercial version)
- **datomic-pro** → The Maven artifact name: `com.datomic/datomic-pro`

They all refer to the same thing. The "Pro" just distinguishes it from the (now-discontinued) Datomic Free edition.

```clojure
;; This IS the "Pro Library" / "Peer Library"
;; deps.edn
{:deps {com.datomic/datomic-pro {:mvn/version "1.0.7187"}}}
```

### Peer vs. Client API (Don't Confuse These)

Datomic has two different APIs:

|Peer API|Client API|
|---|---|
|Library runs in your process|Connects to a Peer Server|
|Has local data cache|No local cache|
|Queries run in-process|Queries run on Peer Server|
|Lower latency for reads|Simpler deployment|
|`datomic.api` namespace|`datomic.client.api` namespace|

This guide covers the **Peer API**, which is what most on-premise Datomic Pro setups use.

### How Data Flows

1. **Reads**: Application → Peer Library queries local cache → Returns results
2. **Writes**: Application → Peer Library → Transactor → Storage → Transactor notifies Peers

The magic is that reads never hit the Transactor. Your application has the data locally.

---

## Setting Up Datomic Pro on Windows

Now that the concepts are clear, let's get it running.

### Prerequisites

- **Java 11+** (Java 17 or 21 recommended)
- **Datomic Pro license key** (or evaluation key from [my.datomic.com](https://my.datomic.com/))
- **Maven repository credentials** (provided with your license)

Verify Java:

```powershell
java -version
```

### Step 1: Download and Extract

1. Download Datomic Pro from [my.datomic.com](https://my.datomic.com/)
2. Extract to a path **without spaces** (important on Windows):

```powershell
# Good
C:\datomic-pro-1.0.7187

# Bad - will cause problems
C:\Program Files\Datomic Pro
```

### Step 2: Configure Your License Key

Create or edit `config\dev-transactor.properties`:

```properties
# License key (required)
license-key=YOUR_LICENSE_KEY_HERE

# Protocol - use "dev" for local development
protocol=dev

# Storage location for dev protocol
storage-data-dir=data

# Host and port the transactor listens on
host=localhost
port=4334

# Memory settings (adjust based on your system)
memory-index-threshold=32m
memory-index-max=128m
object-cache-max=256m

# Write-ahead log
data-dir=data
log-dir=log
```

**Windows Gotcha #1**: Use forward slashes in paths, or escape backslashes:

```properties
# Both work
storage-data-dir=C:/datomic/data
storage-data-dir=C:\\datomic\\data

# This breaks
storage-data-dir=C:\datomic\data
```

### Step 3: Start the Transactor

Open a terminal in your Datomic directory:

```powershell
bin\transactor.bat config\dev-transactor.properties
```

You should see:

```
Starting datomic:dev://localhost:4334/<DB-NAME>...
System started datomic:dev://localhost:4334/<DB-NAME>
```

**Windows Gotcha #2**: If you see encoding errors or the script fails silently, check that the `.properties` file uses UTF-8 encoding without BOM. Notepad can add BOM; use VS Code or Notepad++ instead.

**Windows Gotcha #3**: Firewall prompts. Windows Defender will ask to allow Java network access. Allow it for private networks at minimum.

### Step 4: Set Up Your Application

Add Datomic Pro to your project. You'll need to configure Maven to access Datomic's private repository.

#### Configure Maven Credentials

Create or edit `~/.m2/settings.xml` (typically `C:\Users\YourName\.m2\settings.xml`):

```xml
<settings>
  <servers>
    <server>
      <id>my.datomic.com</id>
      <username>YOUR_DATOMIC_USERNAME</username>
      <password>YOUR_DATOMIC_DOWNLOAD_KEY</password>
    </server>
  </servers>
</settings>
```

#### deps.edn Configuration

```clojure
{:deps {com.datomic/datomic-pro {:mvn/version "1.0.7187"}}
 
 :mvn/repos {"my.datomic.com" {:url "https://my.datomic.com/repo"}}}
```

#### Leiningen project.clj

```clojure
(defproject my-app "0.1.0"
  :dependencies [[org.clojure/clojure "1.11.1"]
                 [com.datomic/datomic-pro "1.0.7187"]]
  :repositories [["my.datomic.com" {:url "https://my.datomic.com/repo"
                                    :username :env/DATOMIC_USERNAME
                                    :password :env/DATOMIC_PASSWORD}]])
```

### Step 5: Connect and Create a Database

```clojure
(require '[datomic.api :as d])

;; Connection URI format: datomic:<protocol>://<host>:<port>/<db-name>
(def uri "datomic:dev://localhost:4334/my-first-db")

;; Create the database (returns true if created, false if exists)
(d/create-database uri)

;; Get a connection
(def conn (d/connect uri))

;; Verify it works
(d/db conn)
;; => #datomic.db.Db{...}
```

### Step 6: Define a Schema and Add Data

```clojure
;; Define schema for a simple "person" entity
(def schema
  [{:db/ident       :person/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "A person's name"}
   
   {:db/ident       :person/email
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity
    :db/doc         "A person's email (unique)"}])

;; Transact the schema
@(d/transact conn schema)

;; Add some data
@(d/transact conn
   [{:person/name  "Alice"
     :person/email "alice@example.com"}
    {:person/name  "Bob"
     :person/email "bob@example.com"}])

;; Query it
(d/q '[:find ?name ?email
       :where
       [?e :person/name ?name]
       [?e :person/email ?email]]
     (d/db conn))
;; => #{["Alice" "alice@example.com"] ["Bob" "bob@example.com"]}
```

---

## Windows-Specific Pitfalls and Solutions

### Memory Configuration

The default memory settings in `transactor.bat` may not suit your system. Edit the batch file or set environment variables:

```powershell
# Before running transactor
$env:XMX = "2g"
$env:XMS = "1g"
```

Or edit `bin\transactor.bat` directly. Find and modify the Java options:

```batch
set JAVA_OPTS=-Xms1g -Xmx2g -XX:+UseG1GC
```

### Path Length Issues

Windows has a 260-character path limit by default. Datomic's storage can create deep directory structures. Solutions:

1. **Enable long paths** (Windows 10 1607+):
    
    ```powershell
    # Run as Administrator
    New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
        -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
    ```
    
2. **Use a short base path** like `C:\d\` instead of `C:\Users\YourName\Documents\Projects\Datomic\`
    

### Line Ending Problems

If you edit config files on Windows and they stop working:

```powershell
# Check for CRLF issues
(Get-Content config\dev-transactor.properties -Raw) -match "`r`n"

# Convert to Unix line endings if needed
(Get-Content config\dev-transactor.properties -Raw) -replace "`r`n", "`n" | 
    Set-Content config\dev-transactor.properties -NoNewline
```

### Console Encoding

If you see garbled output:

```powershell
# Set UTF-8 encoding in PowerShell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
chcp 65001
```

### Transactor Won't Start: Common Causes

|Symptom|Likely Cause|Fix|
|---|---|---|
|"Address already in use"|Port 4334 occupied|Change port in properties or kill other process|
|Hangs with no output|Wrong Java version|Ensure JAVA_HOME points to JDK 11+|
|"License key invalid"|Copy/paste error|Check for invisible characters in license key|
|"Could not find artifact"|Maven credentials wrong|Verify `settings.xml` credentials|
|Silent failure|BOM in properties file|Re-save as UTF-8 without BOM|

### Running as a Windows Service

For production-like environments, wrap the Transactor in a Windows Service using [NSSM](https://nssm.cc/):

```powershell
# Install NSSM, then:
nssm install DatomicTransactor "C:\datomic-pro\bin\transactor.bat" "C:\datomic-pro\config\dev-transactor.properties"
nssm set DatomicTransactor AppDirectory "C:\datomic-pro"
nssm start DatomicTransactor
```

---

## Quick Reference

### Connection URI Formats

```
# Dev protocol (local development)
datomic:dev://localhost:4334/my-db

# SQL storage (PostgreSQL example)
datomic:sql://my-db?jdbc:postgresql://localhost:5432/datomic

# DynamoDB
datomic:ddb://us-east-1/my-table/my-db
```

### Essential REPL Commands

```clojure
(require '[datomic.api :as d])

;; Database lifecycle
(d/create-database uri)    ; Create new database
(d/delete-database uri)    ; Delete database (irreversible!)
(d/connect uri)            ; Get connection

;; Working with data
(d/db conn)                ; Get current database value
(d/transact conn tx-data)  ; Submit transaction (returns future)
(d/q query db & args)      ; Run query

;; History and time-travel
(d/history (d/db conn))    ; Database with full history
(d/as-of (d/db conn) t)    ; Database as of time t
(d/since (d/db conn) t)    ; Only changes since time t
```

### Stopping Cleanly

```powershell
# Graceful shutdown - press Ctrl+C in transactor terminal
# Or if running as service:
nssm stop DatomicTransactor
```

---

## Next Steps

Once you have a working setup:

1. **Learn Datalog**: The query language is Datomic's superpower. Start with [Learn Datalog Today](http://www.learndatalogtoday.org/).
    
2. **Understand the schema**: Datomic's schema is itself data. Explore `:db/ident`, `:db/valueType`, `:db/cardinality`.
    
3. **Explore time-travel**: `d/history`, `d/as-of`, and `d/since` let you query past states. This is built-in—no extra setup.
    
4. **Consider storage backends**: Dev protocol is fine for development. For production on Windows, PostgreSQL is a solid choice that avoids cloud dependencies.
    

---

## Troubleshooting Checklist

When something isn't working:

- [ ] Is the Transactor running? Check for the "System started" message.
- [ ] Can you connect? Try `(d/connect uri)` and check for exceptions.
- [ ] Is the URI correct? Protocol, host, port, and database name must all match.
- [ ] Are credentials configured? Check `~/.m2/settings.xml` for Maven, properties file for license.
- [ ] Any firewall issues? Windows Defender or corporate firewalls can block connections.
- [ ] Path problems? Spaces in paths and deep nesting cause issues on Windows.
- [ ] File encoding? Properties files should be UTF-8 without BOM.

When in doubt, check the Transactor's console output—it usually tells you exactly what's wrong.