# PostgreSQL Usage Guide

Techniques for connecting to PostgreSQL, managing databases, and importing data from various sources.

## Connecting to PostgreSQL

### Default User Login

```powershell
# Login to PostgreSQL as default 'postgres' user
psql -U postgres

# Or specify database explicitly
psql -U postgres -d postgres

# If psql not in PATH, use full path (adjust version as needed)
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" -U postgres
```

**Windows note:** You'll be prompted for password after running the command. Use `-W` flag to force password prompt explicitly: `psql -U postgres -W`

## Database Management

### Creating Databases

```sql
-- Create database for SharePoint file metadata
CREATE DATABASE sharepoint_metadata;

-- Connect to the new database
\c sharepoint_metadata

-- Optional: verify you're connected to the right database
SELECT current_database();
```

**Common naming conventions:**
- `sharepoint_metadata` - descriptive and clear
- `sp_file_metadata` - shorter alternative
- `document_library` - more generic

**Windows psql tip:** The `\c` command works the same as on Linux. To reconnect from outside psql:

```powershell
psql -U postgres -d sharepoint_metadata
```



## Bulk Data Loading

### COPY Command with CSV

```sql
-- Create sample table for SharePoint metadata
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    file_name VARCHAR(255),
    file_path TEXT,
    file_size BIGINT,
    modified_date TIMESTAMP,
    created_by VARCHAR(100)
);

-- Import from CSV (fastest method)
COPY files(file_name, file_path, file_size, modified_date, created_by)
FROM 'C:\data\sharepoint_files.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',', ENCODING 'UTF8');

-- Alternative: psql meta-command (same performance)
\copy files(file_name, file_path, file_size, modified_date, created_by) FROM 'C:\data\sharepoint_files.csv' WITH CSV HEADER
```

## Line-by-Line Breakdown

### Line 1: Command and Target

```sql
COPY files(file_name, file_path, file_size, modified_date, created_by)
```

- **COPY** - Bulk load command for fast data import
- **files** - Target table (database object)
- **Column list** - Specifies which table columns receive data and their mapping order from the CSV

### Line 2: Source File

```sql
FROM 'C:\data\sharepoint_files.csv'
```

- **FROM** - Indicates data source location
- **File path** - Windows absolute path to CSV file

**Windows note**: Single backslashes work in PostgreSQL/DuckDB string literals. If using escape syntax (`E'...'`), you'd need `C:\\data\\sharepoint_files.csv`

### Line 3: Import Options

```sql
WITH (FORMAT csv, HEADER true, DELIMITER ',', ENCODING 'UTF8');
```

- **FORMAT csv** - Treat file as CSV with standard parsing rules
- **HEADER true** - Skip first row (contains column names, not data)
- **DELIMITER ','** - Field separator character
- **ENCODING 'UTF8'** - Character encoding for interpreting file bytes

## How It Works

The database reads the CSV file, skips the header row, splits each line by commas, and inserts values into the specified columns in order. COPY is much faster than INSERT statements because it bypasses per-row parsing overhead and uses optimized batch loading.

**Sample CSV format** (`sharepoint_files.csv`):

```csv
file_name,file_path,file_size,modified_date,created_by
report.docx,/Documents/Reports/report.docx,524288,2024-01-15 10:30:00,john.doe
data.xlsx,/Documents/Data/data.xlsx,1048576,2024-01-14 15:45:00,jane.smith
```

```powershell
# Windows path handling: Use forward slashes or escaped backslashes
# This works:
psql -U postgres -d sharepoint_metadata -c "\copy files FROM 'C:/data/sharepoint_files.csv' CSV HEADER"

# Or escape backslashes (PowerShell requires extra escaping):
psql -U postgres -d sharepoint_metadata -c "\copy files FROM 'C:\\data\\sharepoint_files.csv' CSV HEADER"
```

**Performance comparison:**
- COPY/\copy: 50,000+ rows/sec
- INSERT: ~1,000 rows/sec
- Use CSV for bulk imports

## CSV Format Specification

### SharePoint Metadata Schema

```csv
# CSV Format Specification for SharePoint File Metadata
# Encoding: UTF-8
# Delimiter: comma (,)
# Quote char: " (for fields containing commas/newlines)
# Null values: empty string or explicit NULL

# Column definitions:
file_id,file_name,file_path,file_extension,file_size_bytes,mime_type,created_date,modified_date,created_by,modified_by,version,is_checked_out,checkout_user,document_id,parent_folder,tags

# Example data rows:
1,Q4_Report.docx,/Shared Documents/Reports/Q4_Report.docx,.docx,2097152,application/vnd.openxmlformats-officedocument.wordprocessingml.document,2024-01-10 09:15:00,2024-01-15 14:30:00,john.doe@company.com,jane.smith@company.com,3.0,false,,{ABC-123-DEF},/Shared Documents/Reports,"finance,quarterly"
2,Budget_2024.xlsx,/Shared Documents/Finance/Budget_2024.xlsx,.xlsx,524288,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet,2024-01-05 11:20:00,2024-01-05 11:20:00,finance@company.com,finance@company.com,1.0,true,mike.jones@company.com,{XYZ-456-GHI},/Shared Documents/Finance,"budget,2024"
3,Meeting_Notes.txt,/Team Site/Notes/Meeting_Notes.txt,.txt,8192,text/plain,2024-01-12 13:45:00,2024-01-14 16:10:00,sarah.williams@company.com,sarah.williams@company.com,2.0,false,,{JKL-789-MNO},/Team Site/Notes,meeting

# PostgreSQL table schema:
CREATE TABLE sharepoint_files (
    file_id INTEGER PRIMARY KEY,
    file_name VARCHAR(255) NOT NULL,
    file_path TEXT NOT NULL UNIQUE,
    file_extension VARCHAR(50),
    file_size_bytes BIGINT,
    mime_type VARCHAR(100),
    created_date TIMESTAMP,
    modified_date TIMESTAMP,
    created_by VARCHAR(255),
    modified_by VARCHAR(255),
    version VARCHAR(20),
    is_checked_out BOOLEAN DEFAULT false,
    checkout_user VARCHAR(255),
    document_id VARCHAR(100),
    parent_folder TEXT,
    tags TEXT
);

# Import command:
# \copy sharepoint_files FROM 'C:/data/sharepoint_metadata.csv' WITH (FORMAT csv, HEADER true, NULL '')
```

## SharePoint Export Schema

### Matching SharePoint CSV Fields

```sql
-- Table schema matching SharePoint export fields
CREATE TABLE sharepoint_files (
    id SERIAL PRIMARY KEY,
    file_leaf_ref VARCHAR(255) NOT NULL,      -- Filename
    file_ref TEXT NOT NULL UNIQUE,            -- Full path
    file_size BIGINT,                         -- File size in bytes
    created TIMESTAMP,
    modified TIMESTAMP,
    author VARCHAR(255),
    editor VARCHAR(255)
);

-- Import from CSV
\copy sharepoint_files(file_leaf_ref, file_ref, file_size, created, modified, author, editor) FROM 'C:/data/sharepoint.csv' WITH (FORMAT csv, HEADER true, NULL '')
```

**Expected CSV format:**

```csv
"FileLeafRef","FileRef","File_x0020_Size","Created","Modified","Author","Editor"
"report.docx","/sites/team/Documents/report.docx",524288,"2024-01-15 10:30:00","2024-01-15 14:20:00","John Doe","Jane Smith"
"data.xlsx","/sites/team/Documents/data.xlsx",1048576,"2024-01-10 09:15:00","2024-01-12 16:45:00","Mike Jones","Mike Jones"
```

```powershell
# If dates in different format, adjust during import:
# psql -U postgres -d sharepoint_metadata -c "SET datestyle TO 'ISO, MDY';"
```

## PowerShell Integration

### Loading PSCustomObject Arrays

**Method 1: CSV Export Pipeline (Recommended)**

```powershell
# Given array of PSCustomObject
$objects = @(
    [PSCustomObject]@{
        FileLeafRef = "report.docx"
        FileRef = "/sites/team/Documents/report.docx"
        File_x0020_Size = 524288
        Created = "2024-01-15 10:30:00"
        Modified = "2024-01-15 14:20:00"
        Author = "John Doe"
        Editor = "Jane Smith"
    }
)

# Export to temporary CSV file
$csvPath = "$env:TEMP\sharepoint_import.csv"
$objects | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8

# Import via psql \copy command
$copyCommand = "\copy sharepoint_files(file_leaf_ref, file_ref, file_size, created, modified, author, editor) FROM '$($csvPath -replace '\\', '/')' WITH (FORMAT csv, HEADER true)"

psql -U postgres -d sharepoint_metadata -c $copyCommand

# Cleanup
Remove-Item $csvPath
```

**Method 2: Npgsql .NET Driver (Native Integration)**

```powershell
# Install Npgsql package (one-time setup)
# Install-Package Npgsql -ProviderName NuGet -Scope CurrentUser

# Load assembly
Add-Type -Path "C:\Users\YourUser\.nuget\packages\npgsql\8.0.1\lib\net8.0\Npgsql.dll"

# Connection configuration
$connectionString = "Host=localhost;Database=sharepoint_metadata;Username=postgres;Password=yourpassword"
$connection = New-Object Npgsql.NpgsqlConnection($connectionString)
$connection.Open()

# Batch insert using prepared statements
$sql = "INSERT INTO sharepoint_files (file_leaf_ref, file_ref, file_size, created, modified, author, editor) VALUES (@p1, @p2, @p3, @p4, @p5, @p6, @p7)"
$cmd = New-Object Npgsql.NpgsqlCommand($sql, $connection)

foreach ($obj in $objects) {
    $cmd.Parameters.Clear()
    $cmd.Parameters.AddWithValue("p1", $obj.FileLeafRef)
    $cmd.Parameters.AddWithValue("p2", $obj.FileRef)
    $cmd.Parameters.AddWithValue("p3", [long]$obj.File_x0020_Size)
    $cmd.Parameters.AddWithValue("p4", [DateTime]::Parse($obj.Created))
    $cmd.Parameters.AddWithValue("p5", [DateTime]::Parse($obj.Modified))
    $cmd.Parameters.AddWithValue("p6", $obj.Author)
    $cmd.Parameters.AddWithValue("p7", $obj.Editor)
    
    [void]$cmd.ExecuteNonQuery()
}

$connection.Close()
```

**Method 3: SQL Generation with Transaction Wrapping**

```powershell
# Generate multi-row INSERT statement
$sqlValues = $objects | ForEach-Object {
    $fileLeafRef = $_.FileLeafRef -replace "'", "''"
    $fileRef = $_.FileRef -replace "'", "''"
    $author = $_.Author -replace "'", "''"
    $editor = $_.Editor -replace "'", "''"
    
    "('$fileLeafRef', '$fileRef', $($_.File_x0020_Size), '$($_.Created)', '$($_.Modified)', '$author', '$editor')"
}

$insertSql = @"
BEGIN;
INSERT INTO sharepoint_files (file_leaf_ref, file_ref, file_size, created, modified, author, editor) VALUES
$($sqlValues -join ",`n");
COMMIT;
"@

# Execute via psql with proper escaping
$insertSql | psql -U postgres -d sharepoint_metadata
```

**Performance characteristics:**
- **Method 1 (CSV)**: 50,000+ rows/second, optimal for bulk operations
- **Method 2 (Npgsql)**: 5,000-10,000 rows/second, type-safe with parameterization
- **Method 3 (SQL generation)**: 1,000-5,000 rows/second, simple but SQL injection risk

**Windows-specific considerations:**
1. **Path separator normalization**: Convert backslashes to forward slashes for PostgreSQL file paths
2. **Quote escaping**: PowerShell requires doubling single quotes within SQL strings
3. **Encoding specification**: UTF8 encoding prevents character corruption in file names
4. **Password handling**: Use `$env:PGPASSWORD` environment variable to avoid interactive prompts

```powershell
# Set password environment variable (session-scoped)
$env:PGPASSWORD = "yourpassword"
```
