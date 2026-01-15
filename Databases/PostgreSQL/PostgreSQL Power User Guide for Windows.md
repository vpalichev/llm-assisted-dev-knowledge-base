# Table of Contents

- [[#Table of Contents]]
  - [[#Core Architecture]]
  - [[#Installation Paths (Windows)]]
  - [[#Command-Line Interface]]
  - [[#SQL Execution]]
  - [[#Indexing]]
  - [[#Advanced Data Types]]
  - [[#Window Functions]]
  - [[#Performance Analysis]]
  - [[#Connection Pooling (Windows)]]
  - [[#Backup and Recovery]]
  - [[#Replication]]
  - [[#Extensions]]
  - [[#Configuration Management]]
  - [[#Command-Line Utilities (Windows-Specific Notes)]]
  - [[#Monitoring Queries]]

---
## Core Architecture

[[#Table of Contents|Back to TOC]]

**PostgreSQL** is an object-relational database management system (ORDBMS) implementing ACID-compliant transactions and multi-version concurrency control (MVCC).

**postmaster**: The primary daemon process that manages database connections and spawns backend processes. On Windows, runs as a service (`postgresql-x64-[version]`).

**backend process**: Individual server process handling a single client connection. Windows implementation uses threads within the postmaster service rather than forking child processes.

**shared buffers**: RAM allocated for caching table and index data blocks. Configuration parameter: `shared_buffers` in `postgresql.conf`.

**WAL (Write-Ahead Log)**: Transaction log ensuring data integrity. Files stored in `data\pg_wal\` directory. Parameter `wal_level` controls verbosity.

## Installation Paths (Windows)

[[#Table of Contents|Back to TOC]]

```
C:\Program Files\PostgreSQL\[version]\
├── bin\          # Executables (psql.exe, pg_dump.exe)
├── data\         # Database cluster (PGDATA)
├── lib\          # Shared libraries
└── share\        # Documentation, extensions
```

**PGDATA**: Environment variable or path specifying the database cluster location. Default: `C:\Program Files\PostgreSQL\[version]\data\`

## Command-Line Interface

[[#Table of Contents|Back to TOC]]

**psql**: Interactive terminal client for PostgreSQL.

```cmd
psql -U username -d database_name -h hostname -p 5432
```

Windows-specific considerations:

- Use double quotes for strings containing spaces: `psql -d "My Database"`
- Environment variables: Use `SET PGPASSWORD=secret` (session-scoped) or `%APPDATA%\postgresql\pgpass.conf` for persistent credentials
- Path quoting: Windows backslashes require escaping in certain contexts: `\copy table FROM 'C:\\data\\file.csv'`

**Meta-commands** (psql-specific, prefixed with backslash):

- `\l`: List databases
- `\c database_name`: Connect to database
- `\dt`: List tables in current schema
- `\d table_name`: Describe table structure
- `\du`: List roles
- `\dn`: List schemas
- `\df`: List functions
- `\x`: Toggle expanded display (vertical output)
- `\timing`: Enable query execution timing
- `\e`: Open query in external editor (uses `%EDITOR%` environment variable)
- `\i filepath`: Execute SQL from file

**Note**: Windows command prompt uses `^` for line continuation, not backslash. PowerShell uses backtick `` ` ``.

## SQL Execution

[[#Table of Contents|Back to TOC]]

**Transaction isolation levels**:

- `READ UNCOMMITTED` (treated as READ COMMITTED in PostgreSQL)
- `READ COMMITTED` (default)
- `REPEATABLE READ`
- `SERIALIZABLE`

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- queries
COMMIT;
```

**EXPLAIN**: Query execution plan analyzer.

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT * FROM table WHERE condition;
```

Parameters:

- `ANALYZE`: Execute query and show actual timings
- `BUFFERS`: Display buffer usage statistics
- `VERBOSE`: Show complete output including schema-qualified names
- `FORMAT JSON`: Output in JSON format for parsing

**CTE (Common Table Expression)**: Named temporary result set within query scope.

```sql
WITH regional_sales AS (
    SELECT region, SUM(amount) AS total_sales
    FROM orders
    GROUP BY region
)
SELECT * FROM regional_sales WHERE total_sales > 1000;
```

**LATERAL join**: Allows subquery to reference columns from preceding FROM items.

```sql
SELECT * FROM employees e
CROSS JOIN LATERAL (
    SELECT * FROM projects p 
    WHERE p.employee_id = e.id 
    LIMIT 3
) recent_projects;
```

## Indexing

[[#Table of Contents|Back to TOC]]

**Index types**:

1. **B-tree**: Default. Handles equality and range queries. Syntax: `CREATE INDEX idx_name ON table(column);`
    
2. **Hash**: Equality comparisons only. `CREATE INDEX idx_name ON table USING hash(column);`
    
3. **GiST (Generalized Search Tree)**: Full-text search, geometric data. `CREATE INDEX idx_name ON table USING gist(column);`
    
4. **GIN (Generalized Inverted Index)**: Array, JSONB, full-text. `CREATE INDEX idx_name ON table USING gin(column);`
    
5. **BRIN (Block Range Index)**: Large tables with natural ordering. `CREATE INDEX idx_name ON table USING brin(column);`
    

**Partial index**: Index subset of rows matching WHERE clause.

```sql
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

**Expression index**: Index on computed expression rather than column.

```sql
CREATE INDEX idx_lower_email ON users(lower(email));
```

**Covering index (INCLUDE clause)**: Non-key columns stored in index for index-only scans.

```sql
CREATE INDEX idx_user_lookup ON users(email) INCLUDE (name, created_at);
```

## Advanced Data Types

[[#Table of Contents|Back to TOC]]

**JSONB**: Binary JSON storage with indexing support. Distinct from **JSON** type (text storage).

```sql
CREATE TABLE events (
    id serial PRIMARY KEY,
    data jsonb
);

-- Operators
-- -> : Get JSON object field (returns JSON)
-- ->> : Get JSON object field as text
-- #> : Get JSON object at path (returns JSON)
-- #>> : Get JSON object at path as text
-- @> : Contains (top-level)
-- ? : Key exists

SELECT data->'user'->>'name' FROM events;
SELECT * FROM events WHERE data @> '{"type": "login"}';

-- GIN index for containment queries
CREATE INDEX idx_events_data ON events USING gin(data);
```

**Arrays**: Multi-dimensional arrays of any data type.

```sql
CREATE TABLE products (
    tags text[]
);

INSERT INTO products VALUES (ARRAY['electronics', 'computer']);
SELECT * FROM products WHERE 'computer' = ANY(tags);
```

**Range types**: Represent ranges of values (timestamps, integers, numerics).

```sql
CREATE TABLE reservations (
    room_id int,
    during tsrange
);

-- Exclude overlapping reservations
ALTER TABLE reservations 
ADD CONSTRAINT no_overlap 
EXCLUDE USING gist (room_id WITH =, during WITH &&);
```

**Enum types**: User-defined enumerated types with ordered values.

```sql
CREATE TYPE status AS ENUM ('pending', 'active', 'completed');
CREATE TABLE tasks (status status);
```

## Window Functions

[[#Table of Contents|Back to TOC]]

**Window function**: Performs calculation across row set related to current row without grouping.

```sql
SELECT 
    employee_id,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank,
    LAG(salary, 1) OVER (ORDER BY hire_date) AS prev_salary
FROM employees;
```

**Frame clause**: Defines window frame within partition.

```sql
SELECT 
    date,
    amount,
    SUM(amount) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_sum
FROM transactions;
```

Frame types:

- `ROWS`: Physical row count
- `RANGE`: Logical value range
- `GROUPS`: Peer groups (rows with same ORDER BY values)

## Performance Analysis

[[#Table of Contents|Back to TOC]]

**pg_stat_statements**: Extension tracking execution statistics for SQL statements.

```sql
CREATE EXTENSION pg_stat_statements;

-- View most time-consuming queries
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**VACUUM**: Reclaims storage from dead tuples created by MVCC.

- `VACUUM`: Standard cleanup
- `VACUUM FULL`: Rewrites entire table (locks table, reclaims more space)
- `VACUUM ANALYZE`: Combines vacuum with statistics update

**autovacuum**: Background process automatically performing vacuum. Configuration parameters:

- `autovacuum_vacuum_threshold`: Minimum updates before vacuum
- `autovacuum_vacuum_scale_factor`: Fraction of table size added to threshold

**ANALYZE**: Updates planner statistics for query optimization.

```sql
ANALYZE table_name;
```

## Connection Pooling (Windows)

[[#Table of Contents|Back to TOC]]

**pgBouncer**: Lightweight connection pooler. Windows binary available separately.

**Pool modes**:

- **Session pooling**: Connection returned to pool after client disconnect
- **Transaction pooling**: Connection returned after transaction completion
- **Statement pooling**: Connection returned after statement execution

Configuration file (`pgbouncer.ini`):

```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = md5
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

Windows service installation:

```cmd
pgbouncer.exe -regservice config_file_path
```

## Backup and Recovery

[[#Table of Contents|Back to TOC]]

**pg_dump**: Logical backup utility creating SQL script or archive.

```cmd
pg_dump -U username -d database -F c -f "C:\backup\db.dump"
```

Formats (`-F` flag):

- `p`: Plain SQL text
- `c`: Custom archive (compressed, supports parallel restore)
- `d`: Directory (parallel dump)
- `t`: Tar archive

**pg_restore**: Restore from custom/directory/tar format.

```cmd
pg_restore -U username -d database -j 4 "C:\backup\db.dump"
```

`-j`: Number of parallel jobs (Windows: limited by CPU cores)

**pg_basebackup**: Physical backup of entire cluster.

```cmd
pg_basebackup -D "C:\backup\base" -F tar -z -P
```

**Point-in-Time Recovery (PITR)**: Restore to specific transaction timestamp.

Requirements:

1. Base backup from `pg_basebackup`
2. Archived WAL files (`archive_mode = on`)
3. `recovery.conf` (PostgreSQL <12) or `recovery.signal` file (PostgreSQL ≥12)

## Replication

[[#Table of Contents|Back to TOC]]

**Streaming replication**: Physical replication sending WAL records to standby servers.

**Primary server configuration** (`postgresql.conf`):

```ini
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
```

**Standby server**: Clone of primary using `pg_basebackup`, contains `standby.signal` file.

**Logical replication**: Table-level replication using publish/subscribe model.

Publisher:

```sql
CREATE PUBLICATION my_publication FOR TABLE users, orders;
```

Subscriber:

```sql
CREATE SUBSCRIPTION my_subscription 
CONNECTION 'host=publisher_host dbname=mydb' 
PUBLICATION my_publication;
```

## Extensions

[[#Table of Contents|Back to TOC]]

**Installation**: Extensions must be compiled for Windows or available as binary packages.

```sql
CREATE EXTENSION extension_name;
```

Essential extensions:

- **pg_trgm**: Trigram matching for similarity searches
- **pgcrypto**: Cryptographic functions
- **uuid-ossp**: UUID generation
- **hstore**: Key-value pairs within single column
- **PostGIS**: Spatial database functionality (requires separate installer on Windows)

## Configuration Management

[[#Table of Contents|Back to TOC]]

**postgresql.conf**: Primary configuration file in PGDATA directory.

Critical parameters:

- `max_connections`: Maximum concurrent connections (default: 100)
- `shared_buffers`: RAM for data caching (recommend: 25% of system RAM)
- `effective_cache_size`: Planner estimate of OS disk cache (recommend: 50-75% of RAM)
- `work_mem`: Memory per query operation (sort, hash)
- `maintenance_work_mem`: Memory for maintenance operations (VACUUM, CREATE INDEX)
- `checkpoint_timeout`: Time between automatic checkpoints
- `max_wal_size`: Maximum WAL size before checkpoint

**pg_hba.conf**: Client authentication configuration.

Format: `TYPE DATABASE USER ADDRESS METHOD`

```
# TYPE  DATABASE  USER      ADDRESS        METHOD
host    all       all       127.0.0.1/32   md5
host    all       postgres  0.0.0.0/0      reject
```

Methods:

- `trust`: Allow without password
- `reject`: Deny connection
- `md5`: MD5-encrypted password
- `scram-sha-256`: Modern encrypted password (preferred)
- `peer`: OS user authentication (limited support on Windows)
- `sspi`: Windows integrated authentication (Kerberos/NTLM)

**Reload configuration**: Apply changes without restart.

```sql
SELECT pg_reload_conf();
```

## Command-Line Utilities (Windows-Specific Notes)

[[#Table of Contents|Back to TOC]]

**pg_ctl**: Control PostgreSQL server.

```cmd
pg_ctl -D "C:\Program Files\PostgreSQL\15\data" start
pg_ctl -D "C:\Program Files\PostgreSQL\15\data" stop -m fast
pg_ctl -D "C:\Program Files\PostgreSQL\15\data" reload
```

Stop modes:

- `smart`: Wait for clients to disconnect
- `fast`: Terminate connections, rollback transactions (default)
- `immediate`: Abort processes (requires recovery on restart)

**Windows service management** (alternative to pg_ctl):

```cmd
net start postgresql-x64-15
net stop postgresql-x64-15
sc query postgresql-x64-15
```

**createdb/dropdb**: Database creation/deletion utilities.

```cmd
createdb -U postgres -O owner_user database_name
dropdb -U postgres database_name
```

**psql batch execution** (Windows cmd.exe):

```cmd
psql -U username -d database -f "C:\scripts\script.sql" -o "C:\output\results.txt"
```

PowerShell escaping for special characters:

```powershell
psql -U username -d database -c "SELECT * FROM table WHERE name='O''Brien'"
```

## Monitoring Queries

[[#Table of Contents|Back to TOC]]

**Active connections**:

```sql
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE state = 'active';
```

**Blocking queries**:

```sql
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query,
       blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_locks.pid = blocked_activity.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_locks.pid = blocking_activity.pid
WHERE NOT blocked_locks.granted;
```

**Table sizes**:

```sql
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

**Index usage statistics**:

```sql
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

This guide covers essential PostgreSQL concepts and operations for power users on Windows systems. Command-line examples use Windows path conventions and note potential quoting differences with Unix-like systems.
