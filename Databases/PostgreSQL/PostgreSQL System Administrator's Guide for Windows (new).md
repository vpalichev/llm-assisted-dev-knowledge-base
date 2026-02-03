## 1. Installation

### Official Installer Setup

Download from: https://www.enterprisedb.com/downloads/postgres-postgresql-downloads

**Installation defaults:**

- Install directory: `C:\Program Files\PostgreSQL\17\`
- Data directory: `C:\Program Files\PostgreSQL\17\data\`
- Default port: `5432`
- Default superuser: `postgres`

**Post-install checklist:**

1. Note the superuser password you set during install
2. Verify service is running: `services.msc` → look for "postgresql-x64-17"
3. Add bin directory to PATH (if installer didn't)

```powershell
# Add PostgreSQL bin to PATH (run as Administrator)
$pgPath = "C:\Program Files\PostgreSQL\17\bin"
[Environment]::SetEnvironmentVariable(
    "Path",
    [Environment]::GetEnvironmentVariable("Path", "Machine") + ";$pgPath",
    "Machine"
)
# Restart terminal after this
```

### Key Installation Directories

|Purpose|Path|
|---|---|
|Executables|`C:\Program Files\PostgreSQL\17\bin\`|
|Data (cluster)|`C:\Program Files\PostgreSQL\17\data\`|
|Config files|`C:\Program Files\PostgreSQL\17\data\`|
|Logs|`C:\Program Files\PostgreSQL\17\data\log\`|

---

## 2. Essential Executables

All executables are in `C:\Program Files\PostgreSQL\17\bin\`. Either add to PATH or use full paths.

### postgres.exe

The database server process itself. Rarely run directly—use `pg_ctl` or Windows services instead.

```powershell
# Direct start (foreground, for debugging only)
postgres.exe -D "C:\Program Files\PostgreSQL\17\data"
```

### psql.exe

Interactive SQL terminal. Primary tool for database interaction.

```powershell
# Connect to local default database as postgres user
psql -U postgres

# Connect to specific database
psql -U postgres -d mydb

# Connect to remote server
psql -h 192.168.1.100 -p 5432 -U postgres -d mydb

# Execute single command
psql -U postgres -c "SELECT version();"

# Execute SQL file
psql -U postgres -d mydb -f "C:\scripts\setup.sql"
```

**Windows quoting pitfall:** In PowerShell, use double quotes for SQL with single quotes inside:

```powershell
# CORRECT in PowerShell
psql -U postgres -c "SELECT * FROM users WHERE name = 'John';"

# WRONG - will fail
psql -U postgres -c 'SELECT * FROM users WHERE name = "John";'
```

### pg_dump.exe

Backup a single database.

```powershell
# Plain SQL backup
pg_dump -U postgres mydb > "C:\Backups\mydb.sql"

# Custom format (compressed, allows selective restore)
pg_dump -U postgres -Fc mydb > "C:\Backups\mydb.dump"

# Include schema only (no data)
pg_dump -U postgres --schema-only mydb > "C:\Backups\mydb_schema.sql"

# Include data only (no schema)
pg_dump -U postgres --data-only mydb > "C:\Backups\mydb_data.sql"
```

### pg_dumpall.exe

Backup all databases plus global objects (roles, tablespaces).

```powershell
# Full cluster backup
pg_dumpall -U postgres > "C:\Backups\full_cluster.sql"

# Roles and global objects only
pg_dumpall -U postgres --globals-only > "C:\Backups\globals.sql"
```

### pg_restore.exe

Restore from custom/tar format backups (not plain SQL).

```powershell
# Restore custom format backup
pg_restore -U postgres -d mydb "C:\Backups\mydb.dump"

# Restore to new database (must create first)
createdb -U postgres newdb
pg_restore -U postgres -d newdb "C:\Backups\mydb.dump"

# List contents of backup without restoring
pg_restore -l "C:\Backups\mydb.dump"
```

**For plain SQL backups, use psql instead:**

```powershell
psql -U postgres -d mydb -f "C:\Backups\mydb.sql"
```

### pg_ctl.exe

Control the PostgreSQL server (start, stop, status, reload).

```powershell
# Check status
pg_ctl status -D "C:\Program Files\PostgreSQL\17\data"

# Start server
pg_ctl start -D "C:\Program Files\PostgreSQL\17\data"

# Stop server (smart = wait for connections to close)
pg_ctl stop -D "C:\Program Files\PostgreSQL\17\data" -m smart

# Stop server (fast = rollback active transactions)
pg_ctl stop -D "C:\Program Files\PostgreSQL\17\data" -m fast

# Reload config without restart
pg_ctl reload -D "C:\Program Files\PostgreSQL\17\data"

# Restart
pg_ctl restart -D "C:\Program Files\PostgreSQL\17\data"
```

---

## 3. Port Configuration

### Default Port

PostgreSQL listens on port **5432** by default. Change when:

- Running multiple PostgreSQL clusters
- Port 5432 is blocked by firewall policies
- Avoiding conflicts with other installations

### Changing the Port

**Location:** `postgresql.conf` in the data directory

```powershell
# Open config file
notepad "C:\Program Files\PostgreSQL\17\data\postgresql.conf"
```

Find and modify:

```ini
# Default:
#port = 5432

# Change to:
port = 5433
```

**Restart required after port change:**

```powershell
# Via services.msc or:
Restart-Service postgresql-x64-17

# Or via pg_ctl:
pg_ctl restart -D "C:\Program Files\PostgreSQL\17\data"
```

### Windows Firewall Configuration

```powershell
# Allow PostgreSQL inbound (run as Administrator)
New-NetFirewallRule -DisplayName "PostgreSQL" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 5432 `
    -Action Allow

# For multiple clusters, allow additional ports
New-NetFirewallRule -DisplayName "PostgreSQL-Dev" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 5433 `
    -Action Allow

# Verify rules
Get-NetFirewallRule -DisplayName "PostgreSQL*" | Format-Table Name, Enabled, Direction
```

### Verify Port is Listening

```powershell
# Check if PostgreSQL is listening
netstat -an | findstr "5432"

# Expected output:
# TCP    0.0.0.0:5432           0.0.0.0:0              LISTENING
```

---

## 4. Critical psql Commands

### Connection

```powershell
# Standard connection
psql -U postgres

# With password prompt
psql -U postgres -W

# Connection string format
psql "host=localhost port=5432 dbname=mydb user=postgres"
```

### Essential Meta-Commands

Once inside psql, these backslash commands provide quick information:

```sql
-- List all databases
\l

-- Connect to different database
\c mydb

-- List tables in current database
\dt

-- List all tables including system tables
\dt *.*

-- Describe table structure
\d tablename

-- List users/roles
\du

-- List schemas
\dn

-- Show current connection info
\conninfo

-- Toggle expanded display (useful for wide tables)
\x

-- Show command history
\s

-- Run external SQL file
\i C:/scripts/setup.sql
-- NOTE: Use forward slashes in psql, even on Windows!

-- Quit psql
\q
```

### Essential SQL for Administration

#### Database Management

```sql
-- Create database
CREATE DATABASE myapp;

-- Create database with specific owner
CREATE DATABASE myapp OWNER myuser;

-- Drop database (CAREFUL!)
DROP DATABASE myapp;

-- Rename database (must disconnect all users first)
ALTER DATABASE oldname RENAME TO newname;
```

#### User/Role Management

```sql
-- Create user with password
CREATE USER appuser WITH PASSWORD 'SecurePass123!';

-- Create user with login + create database privileges
CREATE ROLE devuser WITH LOGIN CREATEDB PASSWORD 'DevPass456!';

-- Change password
ALTER USER appuser WITH PASSWORD 'NewPassword789!';

-- Grant superuser (use sparingly!)
ALTER USER appuser WITH SUPERUSER;

-- Remove superuser
ALTER USER appuser WITH NOSUPERUSER;

-- Drop user
DROP USER appuser;
```

#### Permissions

```sql
-- Grant all privileges on database
GRANT ALL PRIVILEGES ON DATABASE myapp TO appuser;

-- Grant connect only
GRANT CONNECT ON DATABASE myapp TO appuser;

-- Grant all on all tables in schema
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO appuser;

-- Grant read-only on specific table
GRANT SELECT ON tablename TO readonly_user;

-- Grant schema usage (required before table access)
GRANT USAGE ON SCHEMA public TO appuser;

-- Revoke privileges
REVOKE ALL PRIVILEGES ON DATABASE myapp FROM appuser;

-- Make user owner of all tables (run per-database)
-- Useful when migrating ownership
DO $$
DECLARE r RECORD;
BEGIN
    FOR r IN SELECT tablename FROM pg_tables WHERE schemaname = 'public'
    LOOP
        EXECUTE 'ALTER TABLE public.' || quote_ident(r.tablename) || ' OWNER TO newowner';
    END LOOP;
END $$;
```

---

## 5. Cluster Architecture

### What is a PostgreSQL Cluster?

A **cluster** is a single PostgreSQL server instance consisting of:

- One data directory containing all data files
- One configuration set (postgresql.conf, pg_hba.conf)
- One server process (postgres.exe)
- One port
- One Windows service

**Key points:**

- Multiple databases can exist within one cluster
- All databases in a cluster share the same:
    - Server process and memory
    - Configuration settings
    - User/role definitions (roles are cluster-wide)
    - Port number
    - Backup schedule (pg_dumpall backs up entire cluster)

### Cluster vs Database

```
┌─────────────────────────────────────────────────────────┐
│                   PostgreSQL Cluster                     │
│                   (One Windows Service)                  │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  postgres   │  │   myapp     │  │   testdb    │     │
│  │  (system)   │  │ (database)  │  │ (database)  │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                         │
│  Shared: Memory, CPU, Config, Roles, Port 5432         │
└─────────────────────────────────────────────────────────┘
```

### Single Cluster Limitations

|Limitation|Impact|
|---|---|
|Shared memory|One database hogging RAM affects all|
|Single config|Can't tune prod differently from dev|
|One failure domain|Crash affects all databases|
|Shared roles|Can't have same username with different passwords|
|Same port|All connections share same listener|

---

## 6. Dual Environment Setup (Production + Development)

### Why Separate Clusters?

**DON'T** run production and development as separate databases in one cluster.

**DO** run them as separate clusters because:

1. **Crash isolation**: Dev database corruption/crash won't touch production
2. **Resource isolation**: Dev queries can't starve production of memory/CPU
3. **Independent config**:
    - Production: performance tuning, minimal logging
    - Development: verbose logging, debug settings
4. **Independent restarts**: Restart dev without touching production
5. **Security boundaries**: Different pg_hba.conf rules per environment
6. **Independent upgrades**: Test PostgreSQL upgrades on dev cluster first

### Step-by-Step: Two-Cluster Setup

#### Step 1: Create Directory Structure

```powershell
# Run as Administrator
New-Item -ItemType Directory -Path "C:\PostgreSQL\data-prod" -Force
New-Item -ItemType Directory -Path "C:\PostgreSQL\data-dev" -Force
New-Item -ItemType Directory -Path "C:\PostgreSQL\backups\prod" -Force
New-Item -ItemType Directory -Path "C:\PostgreSQL\backups\dev" -Force
```

#### Step 2: Initialize Data Directories

```powershell
# Production cluster
& "C:\Program Files\PostgreSQL\17\bin\initdb.exe" `
    -D "C:\PostgreSQL\data-prod" `
    -U postgres `
    -W `
    -E UTF8 `
    --locale=en_US.UTF-8

# Development cluster
& "C:\Program Files\PostgreSQL\17\bin\initdb.exe" `
    -D "C:\PostgreSQL\data-dev" `
    -U postgres `
    -W `
    -E UTF8 `
    --locale=en_US.UTF-8
```

When prompted, enter passwords for each cluster's postgres superuser.

#### Step 3: Configure Ports

**Production** (`C:\PostgreSQL\data-prod\postgresql.conf`):

```ini
port = 5432
listen_addresses = 'localhost'    # Or '*' for remote access
```

**Development** (`C:\PostgreSQL\data-dev\postgresql.conf`):

```ini
port = 5433
listen_addresses = 'localhost'
```

#### Step 4: Configure Memory Allocation

**Production** (optimize for performance):

```ini
# C:\PostgreSQL\data-prod\postgresql.conf
shared_buffers = 2GB              # 25% of available RAM
effective_cache_size = 6GB        # 75% of available RAM
work_mem = 64MB
maintenance_work_mem = 512MB
max_connections = 200
log_min_duration_statement = 1000  # Log queries > 1 second
log_statement = 'none'             # Minimal logging
```

**Development** (optimize for debugging):

```ini
# C:\PostgreSQL\data-dev\postgresql.conf
shared_buffers = 512MB            # Modest allocation
effective_cache_size = 1GB
work_mem = 32MB
maintenance_work_mem = 128MB
max_connections = 50
log_min_duration_statement = 0     # Log all queries
log_statement = 'all'              # Verbose logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
```

#### Step 5: Register Windows Services

```powershell
# Run as Administrator

# Production service
& "C:\Program Files\PostgreSQL\17\bin\pg_ctl.exe" register `
    -N "postgresql-prod" `
    -D "C:\PostgreSQL\data-prod" `
    -S auto `
    -w

# Development service
& "C:\Program Files\PostgreSQL\17\bin\pg_ctl.exe" register `
    -N "postgresql-dev" `
    -D "C:\PostgreSQL\data-dev" `
    -S demand `
    -w

# -S auto = start automatically at boot (production)
# -S demand = manual start (development)
```

#### Step 6: Start Services

```powershell
Start-Service postgresql-prod
Start-Service postgresql-dev

# Verify both are running
Get-Service postgresql-prod, postgresql-dev | Format-Table Name, Status
```

#### Step 7: Verify Connections

```powershell
# Connect to production (port 5432)
psql -h localhost -p 5432 -U postgres -c "SELECT 'Production cluster' AS env;"

# Connect to development (port 5433)
psql -h localhost -p 5433 -U postgres -c "SELECT 'Development cluster' AS env;"
```

### Quick Reference: Dual Cluster Commands

|Action|Production|Development|
|---|---|---|
|Start|`Start-Service postgresql-prod`|`Start-Service postgresql-dev`|
|Stop|`Stop-Service postgresql-prod`|`Stop-Service postgresql-dev`|
|Connect|`psql -p 5432 -U postgres`|`psql -p 5433 -U postgres`|
|Backup|`pg_dump -p 5432 ...`|`pg_dump -p 5433 ...`|
|Config|`C:\PostgreSQL\data-prod\postgresql.conf`|`C:\PostgreSQL\data-dev\postgresql.conf`|
|Logs|`C:\PostgreSQL\data-prod\log\`|`C:\PostgreSQL\data-dev\log\`|

### Unregister Services (if needed)

```powershell
# Stop service first
Stop-Service postgresql-dev

# Unregister
& "C:\Program Files\PostgreSQL\17\bin\pg_ctl.exe" unregister -N "postgresql-dev"
```

---

## 7. Backup Strategy

### pg_dump Syntax for Windows

**Critical Windows path rules:**

- Use double quotes around paths with spaces
- Backslashes work but forward slashes are safer
- In PowerShell, escape special characters or use single quotes for literal strings

#### Basic Backups

```powershell
# Plain SQL (human-readable, larger file)
pg_dump -U postgres -p 5432 mydb > "C:\PostgreSQL\backups\prod\mydb_$(Get-Date -Format 'yyyyMMdd').sql"

# Custom format (compressed, recommended for large databases)
pg_dump -U postgres -p 5432 -Fc mydb -f "C:\PostgreSQL\backups\prod\mydb_$(Get-Date -Format 'yyyyMMdd').dump"

# Directory format (parallel backup for huge databases)
pg_dump -U postgres -p 5432 -Fd mydb -j 4 -f "C:\PostgreSQL\backups\prod\mydb_$(Get-Date -Format 'yyyyMMdd')_dir"
```

#### Full Cluster Backup

```powershell
pg_dumpall -U postgres -p 5432 > "C:\PostgreSQL\backups\prod\full_cluster_$(Get-Date -Format 'yyyyMMdd').sql"
```

### Scheduled Backups with Task Scheduler

#### Create Backup Script

Save as `C:\PostgreSQL\scripts\backup-prod.ps1`:

```powershell
# PostgreSQL Production Backup Script
$ErrorActionPreference = "Stop"

# Configuration
$pgBin = "C:\Program Files\PostgreSQL\17\bin"
$backupDir = "C:\PostgreSQL\backups\prod"
$database = "mydb"
$port = "5432"
$user = "postgres"
$retentionDays = 7

# Set PGPASSWORD environment variable (or use .pgpass file)
$env:PGPASSWORD = "YourSecurePassword"

# Generate filename with timestamp
$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$backupFile = Join-Path $backupDir "$database`_$timestamp.dump"

# Run backup
& "$pgBin\pg_dump.exe" -U $user -p $port -Fc $database -f $backupFile

if ($LASTEXITCODE -eq 0) {
    Write-Host "Backup successful: $backupFile"
    
    # Delete backups older than retention period
    Get-ChildItem $backupDir -Filter "*.dump" | 
        Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-$retentionDays) } | 
        Remove-Item -Force
    
    Write-Host "Cleaned up backups older than $retentionDays days"
} else {
    Write-Error "Backup failed with exit code $LASTEXITCODE"
    exit 1
}

# Clear password from environment
Remove-Item Env:\PGPASSWORD
```

#### Configure Task Scheduler

```powershell
# Create scheduled task (run as Administrator)
$action = New-ScheduledTaskAction `
    -Execute "powershell.exe" `
    -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\PostgreSQL\scripts\backup-prod.ps1"

$trigger = New-ScheduledTaskTrigger -Daily -At 2:00AM

$principal = New-ScheduledTaskPrincipal `
    -UserId "NT AUTHORITY\SYSTEM" `
    -LogonType ServiceAccount `
    -RunLevel Highest

$settings = New-ScheduledTaskSettingsSet `
    -StartWhenAvailable `
    -DontStopOnIdleEnd

Register-ScheduledTask `
    -TaskName "PostgreSQL-Prod-Backup" `
    -Action $action `
    -Trigger $trigger `
    -Principal $principal `
    -Settings $settings `
    -Description "Daily PostgreSQL production backup"
```

**Alternative: GUI setup**

1. Open Task Scheduler (`taskschd.msc`)
2. Create Basic Task → Name: "PostgreSQL-Prod-Backup"
3. Trigger: Daily at 2:00 AM
4. Action: Start a program
    - Program: `powershell.exe`
    - Arguments: `-NoProfile -ExecutionPolicy Bypass -File "C:\PostgreSQL\scripts\backup-prod.ps1"`
5. Finish → Check "Open Properties" → Change "Run whether user is logged on or not"

### Restoration Procedures

#### Restore Custom Format to Existing Database

```powershell
# WARNING: This will overwrite existing data!
pg_restore -U postgres -p 5432 -d mydb --clean --if-exists "C:\PostgreSQL\backups\prod\mydb_20250124.dump"
```

#### Restore to New Database

```powershell
# Create empty database first
psql -U postgres -p 5432 -c "CREATE DATABASE mydb_restored;"

# Restore
pg_restore -U postgres -p 5432 -d mydb_restored "C:\PostgreSQL\backups\prod\mydb_20250124.dump"
```

#### Restore Plain SQL Backup

```powershell
# Restore full cluster from SQL
psql -U postgres -p 5432 -f "C:\PostgreSQL\backups\prod\full_cluster_20250124.sql"

# Restore single database from SQL
psql -U postgres -p 5432 -d mydb -f "C:\PostgreSQL\backups\prod\mydb_20250124.sql"
```

#### Selective Restore (Tables Only)

```powershell
# List contents of backup
pg_restore -l "C:\PostgreSQL\backups\prod\mydb_20250124.dump" > C:\temp\backup_contents.txt

# Edit backup_contents.txt to comment out (;) items you don't want

# Restore using filtered list
pg_restore -U postgres -p 5432 -d mydb -L C:\temp\backup_contents.txt "C:\PostgreSQL\backups\prod\mydb_20250124.dump"
```

---

## 8. Windows-Specific Notes

### Service Management

#### Via services.msc

1. Press `Win + R`, type `services.msc`
2. Find "postgresql-x64-17" (or your custom service name)
3. Right-click for Start/Stop/Restart options
4. Properties → Startup type: Automatic/Manual/Disabled

#### Via PowerShell

```powershell
# Status
Get-Service postgresql*

# Start/Stop/Restart
Start-Service postgresql-x64-17
Stop-Service postgresql-x64-17
Restart-Service postgresql-x64-17

# Change startup type
Set-Service postgresql-x64-17 -StartupType Automatic
```

#### Via net commands (Command Prompt)

```cmd
net start postgresql-x64-17
net stop postgresql-x64-17
```

### Path Handling Differences

|Context|Path Format|Example|
|---|---|---|
|Windows Explorer|Backslashes|`C:\PostgreSQL\data`|
|postgresql.conf|Forward slashes preferred|`log_directory = 'C:/PostgreSQL/data/log'`|
|psql `\i` command|Forward slashes required|`\i C:/scripts/setup.sql`|
|pg_dump output file|Backslashes in quotes|`pg_dump -f "C:\backups\db.dump"`|
|PowerShell variables|Either, but escape backslashes in double quotes|`"C:\\PostgreSQL\\data"` or `'C:\PostgreSQL\data'`|

**Common Pitfall:** In `postgresql.conf`, backslashes must be escaped:

```ini
# WRONG - will fail
log_directory = 'C:\PostgreSQL\data\log'

# CORRECT - escaped backslashes
log_directory = 'C:\\PostgreSQL\\data\\log'

# CORRECT - forward slashes (recommended)
log_directory = 'C:/PostgreSQL/data/log'
```

### Authentication Configuration (pg_hba.conf)

Location: Same directory as postgresql.conf (e.g., `C:\PostgreSQL\data-prod\pg_hba.conf`)

**Format:** `TYPE DATABASE USER ADDRESS METHOD`

```ini
# Local connections via Windows named pipes
# TYPE     DATABASE  USER       METHOD
host     all       all        127.0.0.1/32    scram-sha-256

# IPv6 localhost
host     all       all        ::1/128         scram-sha-256

# Allow connections from local network (adjust subnet)
host     all       all        192.168.1.0/24  scram-sha-256

# Allow specific user to specific database from any IP (use with caution)
host     mydb      appuser    0.0.0.0/0       scram-sha-256

# Reject all other connections (default deny)
host     all       all        0.0.0.0/0       reject
```

**Authentication methods:**

- `scram-sha-256`: Secure password (recommended)
- `md5`: Legacy password hash (less secure)
- `trust`: No password required (NEVER use in production)
- `reject`: Explicitly deny connection

**Reload after changes:**

```powershell
# No restart needed, just reload
psql -U postgres -c "SELECT pg_reload_conf();"

# Or via pg_ctl
pg_ctl reload -D "C:\PostgreSQL\data-prod"
```

### Password File (.pgpass)

Avoid typing passwords by creating `%APPDATA%\postgresql\pgpass.conf`:

```
# Format: hostname:port:database:username:password
localhost:5432:*:postgres:ProdPassword123
localhost:5433:*:postgres:DevPassword456
```

**Secure the file:**

```powershell
# Create directory if needed
New-Item -ItemType Directory -Path "$env:APPDATA\postgresql" -Force

# Create pgpass.conf
Set-Content -Path "$env:APPDATA\postgresql\pgpass.conf" -Value @"
localhost:5432:*:postgres:ProdPassword123
localhost:5433:*:postgres:DevPassword456
"@
```

### Windows Event Log Integration

PostgreSQL can log to Windows Event Log. In `postgresql.conf`:

```ini
log_destination = 'eventlog'
event_source = 'PostgreSQL-Prod'
```

View logs in Event Viewer → Windows Logs → Application → Source: PostgreSQL-Prod

### Common Windows-Specific Issues

|Issue|Cause|Solution|
|---|---|---|
|"could not create lock file"|Permission denied|Run as Administrator or check folder permissions|
|Service won't start|Data directory ownership|Ensure service account has full control|
|"permission denied for schema"|Default privileges|Run `GRANT USAGE ON SCHEMA public TO user;`|
|Connection refused|Firewall or listen_addresses|Check `listen_addresses = '*'` and firewall rules|
|Slow performance|Antivirus scanning|Exclude PostgreSQL data directory from real-time scanning|

### Antivirus Exclusions

Add these paths to your antivirus exclusion list:

- `C:\PostgreSQL\data-prod\` (and data-dev)
- `C:\Program Files\PostgreSQL\17\`
- File extension: `*.dump`

---

## Quick Reference Card

### Connection Strings

```powershell
# Production
psql -h localhost -p 5432 -U postgres -d mydb

# Development  
psql -h localhost -p 5433 -U postgres -d mydb
```

### Daily Commands

```powershell
# Check what's running
Get-Service postgresql* | ft Name, Status

# View active connections
psql -U postgres -c "SELECT pid, usename, datname, state FROM pg_stat_activity;"

# Database sizes
psql -U postgres -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY pg_database_size(datname) DESC;"

# Kill a connection (replace PID)
psql -U postgres -c "SELECT pg_terminate_backend(12345);"
```

### Emergency Procedures

```powershell
# Force stop all connections to a database
psql -U postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'mydb' AND pid <> pg_backend_pid();"

# Force restart service
Stop-Service postgresql-prod -Force
Start-Service postgresql-prod

# Check logs for errors (PowerShell)
Get-Content "C:\PostgreSQL\data-prod\log\postgresql-*.log" -Tail 50
```