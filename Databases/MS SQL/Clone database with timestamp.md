Good call — pulling the prod DB name and data folder up as variables means there's exactly one place to change anything per environment.

```sql
DECLARE @SourceDb  sysname       = N'YourProdDb';
DECLARE @BackupDir nvarchar(260) = N'C:\Temp\';
DECLARE @DataDir   nvarchar(260) = N'C:\Data\';

DECLARE @BakFile nvarchar(520) = @BackupDir + @SourceDb + N'.bak';

DECLARE @name sysname = @SourceDb
                      + N'_' + FORMAT(SYSDATETIME(),'yyyyMMdd_HHmmss')
                      + N'_throwaway_sandbox_noprod';

DECLARE @mdf nvarchar(260) = @DataDir + @name + N'.mdf';
DECLARE @ldf nvarchar(260) = @DataDir + @name + N'.ldf';

DECLARE @sqlBackup nvarchar(max) =
    N'BACKUP DATABASE ' + QUOTENAME(@SourceDb) + N'
        TO DISK = N''' + @BakFile + N'''
        WITH COPY_ONLY, INIT, COMPRESSION;';
EXEC (@sqlBackup);

DECLARE @sqlRestore nvarchar(max) =
    N'RESTORE DATABASE ' + QUOTENAME(@name) + N'
        FROM DISK = N''' + @BakFile + N'''
        WITH MOVE ''' + @SourceDb + N'''     TO N''' + @mdf + N''',
             MOVE ''' + @SourceDb + N'_log'' TO N''' + @ldf + N''';';
EXEC (@sqlRestore);
```

---

### Line 1: `DECLARE @SourceDb sysname = N'YourProdDb';`

- **`DECLARE`** — T-SQL keyword that creates a local variable. Variables live only for the duration of the current batch (until `GO` or end of script).
- **`@SourceDb`** — the variable's name. **The `@` prefix is mandatory** for local variables. Without it, the parser reads `SourceDb` as a column or object name.
- **`sysname`** — a built-in SQL Server data type used by convention for variables that hold **SQL identifiers** (names of databases, tables, columns, indexes, logins, etc.). Under the hood it's defined as **`nvarchar(128) NOT NULL`**. The `128` matches SQL Server's maximum identifier length, so you can't accidentally declare a variable too small for any valid object name. It's Unicode (`n` prefix) because identifiers can contain non-ASCII characters — you can name a table `[Café]` if you want. The `NOT NULL` part doesn't really constrain anything at variable declaration (variables start as `NULL` until assigned), but it signals intent: identifiers shouldn't be null. Functionally, `sysname` and `nvarchar(128)` are interchangeable — `sysname` is a readability hint that says "this variable holds an object name." SQL Server's own system catalogs use it everywhere (e.g., `sys.databases.name` is typed as `sysname`). If your guru asks "why not just `nvarchar(128)`?" — pure convention; both work identically.
- **`=`** — combined declare-and-assign in one statement. Older T-SQL required two lines (`DECLARE @x type;` then `SET @x = value;`); modern versions allow this shortcut.
- **`N'YourProdDb'`** — **the `N` prefix marks a Unicode (UTF-16) string literal**. Without `N`, the literal is interpreted using the database's non-Unicode codepage, which can corrupt non-ASCII characters. Always use `N` when assigning to `nvarchar` or `sysname`. The `N` stands for "National character set."

### Line 2: `DECLARE @BackupDir nvarchar(260) = N'C:\Temp\';`

- **`nvarchar(260)`** — variable-length Unicode string up to 260 characters. The number 260 isn't random: it matches Windows's historical `MAX_PATH` (drive letter + colon + 256 path chars + null terminator), so any legal Windows path fits. If your guru asks "why 260 not 4000?" — `nvarchar` non-`max` types can go up to 4000; 260 is sized for a Windows path, it's documentation of intent.
- **`C:\Temp\`** — the trailing backslash matters. Later lines concatenate filenames onto this; without the slash you'd get `C:\Tempprod.bak`.

### Line 3: `DECLARE @DataDir nvarchar(260) = N'C:\Data\';`

Same shape as `@BackupDir`. This is the folder where the _new_ `.mdf` and `.ldf` files for the sandbox will be created. It must exist and be writable by the SQL Server service account.

### Line 4: `DECLARE @BakFile nvarchar(520) = @BackupDir + @SourceDb + N'.bak';`

- **`+`** is **the string concatenation operator in T-SQL**. (Standard SQL uses `||`, MySQL uses `CONCAT()`; SQL Server overloads `+`.)
- **`520`** — generous size in case the directory portion is long. Could be 260; doesn't matter much.
- Result: `C:\Temp\YourProdDb.bak`. Tying the backup filename to `@SourceDb` means you can change one variable at the top and everything downstream tracks it.

### Lines 5–7: building the sandbox name

```sql
DECLARE @name sysname = @SourceDb
                      + N'_' + FORMAT(SYSDATETIME(),'yyyyMMdd_HHmmss')
                      + N'_throwaway_sandbox_noprod';
```

- **`SYSDATETIME()`** — returns the current date and time as `datetime2(7)`, precise to 100 nanoseconds. Compare with `GETDATE()`, which returns plain `datetime` (~3ms precision). For a timestamp in a filename, either works; `SYSDATETIME()` is the modern choice.
- **`FORMAT(value, 'pattern')`** — applies .NET-style format strings (it calls into the .NET runtime under the hood). Format pattern characters:
    - `yyyy` = 4-digit year
    - `MM` = 2-digit month — **capital M**. **Lowercase `mm` means minutes**. Extremely common bug.
    - `dd` = 2-digit day
    - `HH` = 2-digit hour, 24-hour clock. Lowercase `hh` would be 12-hour.
    - `mm` = 2-digit minute (after the date portion).
    - `ss` = 2-digit second.
- For May 21, 2026 at 14:30:22, the result is the string `20260521_143022`.
- The whole expression yields e.g. `YourProdDb_20260521_143022_throwaway_sandbox_noprod`. The "throwaway" tag is at the **end** (postfix), keeping the source DB name at the front for sorting.

If your guru asks "why not `CONVERT(varchar, GETDATE(), 112)`?" — that's the older way and only offers fixed style codes (112 = `yyyymmdd`). `FORMAT` is more flexible but slightly slower because of the .NET round-trip. For a script that runs once per sandbox, negligible.

### Lines 8–9: physical file paths

```sql
DECLARE @mdf nvarchar(260) = @DataDir + @name + N'.mdf';
DECLARE @ldf nvarchar(260) = @DataDir + @name + N'.ldf';
```

Just string concatenation. **`.mdf`** is the conventional extension for SQL Server's primary data file; **`.ldf`** is the conventional extension for the transaction log file. Convention, not enforcement — SQL Server wouldn't object to `.banana`, but every DBA on earth would.

### Lines 10–13: building the backup command

```sql
DECLARE @sqlBackup nvarchar(max) =
    N'BACKUP DATABASE ' + QUOTENAME(@SourceDb) + N'
        TO DISK = N''' + @BakFile + N'''
        WITH COPY_ONLY, INIT, COMPRESSION;';
```

- **`nvarchar(max)`** — variable-length Unicode string up to ~2 GB. The `(max)` flavor exists because regular `nvarchar(n)` caps at 4000. For dynamic SQL we use `(max)` so we never truncate.
    
- **Multi-line string literal** — T-SQL strings can span lines; the newlines become part of the string content. No continuation character needed.
    
- **`QUOTENAME(@SourceDb)`** — wraps a string in `[brackets]` and safely escapes any `]` inside as `]]`. Output: a guaranteed-valid SQL identifier like `[YourProdDb]`. We use it because the database name appears in a position that requires a literal identifier, not a variable.
    
- **`N'''`** — the part that hurts the eyes. Inside a single-quoted SQL string, **a single quote is escaped by doubling it**: `''` represents one `'` character in the resulting string. Reading `N'''` character by character: opening `N'` starts the literal, `''` adds one `'` to the content, the final `'` closes the literal. So `N'''` is a Unicode string containing exactly one apostrophe.
    
    Applied here: the fragment `N'... TO DISK = N'''` produces the runtime string `... TO DISK = N'` — with a trailing apostrophe ready to wrap the path. Then concatenate `@BakFile`. Then `N'''` adds the closing apostrophe. Then `WITH COPY_ONLY...;'` is the rest.
    
    After concatenation, `@sqlBackup` holds:
    
    ```
    BACKUP DATABASE [YourProdDb]
          TO DISK = N'C:\Temp\YourProdDb.bak'
          WITH COPY_ONLY, INIT, COMPRESSION;
    ```
    
- **`COPY_ONLY`** — tells SQL Server "this is a side backup; don't update the log chain markers." Without it, this backup would reset prod's differential base and confuse the regular backup schedule. Critical for production.
    
- **`INIT`** — overwrite the `.bak` file if it exists. Without `INIT`, SQL Server _appends_ to the file (a `.bak` can hold multiple backups), which is rarely what you want for a throwaway temp file.
    
- **`COMPRESSION`** — backup compression. Smaller file, less I/O, often faster overall. Available in Standard Edition and up.
    

### `EXEC (@sqlBackup);`

- **`EXEC` with parentheses around a string variable** = "execute the contents of this string as a SQL statement." Without parentheses (`EXEC sp_something`) it would try to invoke a stored procedure. This is the canonical **dynamic SQL** invocation.
- Why dynamic SQL at all? Because `RESTORE DATABASE @var` isn't valid syntax — that position requires a literal identifier. So we build the literal as a string and execute it. Using the same pattern for backup and restore keeps the script consistent.

### Lines 14–18: the restore

```sql
DECLARE @sqlRestore nvarchar(max) =
    N'RESTORE DATABASE ' + QUOTENAME(@name) + N'
        FROM DISK = N''' + @BakFile + N'''
        WITH MOVE ''' + @SourceDb + N'''     TO N''' + @mdf + N''',
             MOVE ''' + @SourceDb + N'_log'' TO N''' + @ldf + N''';';
EXEC (@sqlRestore);
```

- **`QUOTENAME(@name)`** — same trick as before: produces `[YourProdDb_20260521_143022_throwaway_sandbox_noprod]` for the target database name in the generated SQL.
- **`MOVE 'logical_name' TO 'physical_path'`** — RESTORE clause meaning "the logical file called `logical_name` inside the backup should be written to `physical_path` on disk." Logical names are stored _inside_ the `.bak` file and were assigned when prod was created. The convention is that the data file's logical name equals the database name, and the log's logical name is `<dbname>_log`. **That convention isn't guaranteed** — if prod was restored or renamed at some point, the logical names may differ. Confirm once with:
    
    ```sql
    RESTORE FILELISTONLY FROM DISK = N'C:\Temp\YourProdDb.bak';
    ```
    
    If they differ, you'll need to either fix the actual logical names with `ALTER DATABASE ... MODIFY FILE` or change the two `MOVE` clauses to match what's in the backup.
- The `'''` escape pattern is the same as in the backup section: each `'''` in source code becomes one `'` in the generated SQL. There's more of it here because we're concatenating multiple variables in a row.

After concatenation, `@sqlRestore` holds:

```
RESTORE DATABASE [YourProdDb_20260521_143022_throwaway_sandbox_noprod]
        FROM DISK = N'C:\Temp\YourProdDb.bak'
        WITH MOVE 'YourProdDb'     TO N'C:\Data\YourProdDb_20260521_143022_throwaway_sandbox_noprod.mdf',
             MOVE 'YourProdDb_log' TO N'C:\Data\YourProdDb_20260521_143022_throwaway_sandbox_noprod.ldf';
```

---

### Likely guru pushback and your replies

**"Why dynamic SQL instead of just hardcoding the name?"** Because the database name contains a runtime timestamp from `FORMAT(SYSDATETIME(),...)`. You can't put an expression in `RESTORE DATABASE`'s name position — it demands a literal identifier. Dynamic SQL is the standard workaround.

**"This is vulnerable to SQL injection."** Not meaningfully here: every value being concatenated (`@SourceDb`, `@name`, `@BakFile`, `@mdf`, `@ldf`) is built from string constants and `SYSDATETIME()`, none from external input. If you later parameterize `@SourceDb` from user input, switch to `sp_executesql` with typed parameters.

**"Hand-escaping quotes is ugly."** True — `sp_executesql` is cleaner where parameters are accepted. But `BACKUP DATABASE` and `RESTORE DATABASE` don't accept parameter binding for the database name or file paths, so you have to build a string anyway. Hand-escaping is unavoidable for the parts that can't be parameterized.

**"Add `PRINT @sqlBackup` and `PRINT @sqlRestore` before the `EXEC`s while testing."** Good advice — it shows the exact generated text, which makes any quote-escaping mistake or syntax error immediately obvious. Drop the `PRINT`s once you're confident the script works in your environment.

**"What if prod has more than one data file?"** Then you need additional `MOVE` clauses, one per file, and the logical names won't follow the simple `<dbname>` / `<dbname>_log` convention. Run `RESTORE FILELISTONLY` once to see what's in the backup and add `MOVE` clauses to match.