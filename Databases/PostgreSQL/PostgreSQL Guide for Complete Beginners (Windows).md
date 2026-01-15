# Table of Contents

- [[#Table of Contents]]
  - [[#What is a Database?]]
  - [[#What is PostgreSQL?]]
  - [[#Installation on Windows]]
  - [[#Starting PostgreSQL]]
  - [[#Connecting to PostgreSQL]]
  - [[#Fundamental SQL Commands]]
  - [[#Inserting Data]]
  - [[#Retrieving Data]]
  - [[#Updating Data]]
  - [[#Deleting Data]]
  - [[#Understanding Relationships Between Tables]]
  - [[#Joining Tables]]
  - [[#Aggregate Functions]]
  - [[#Grouping Data]]
  - [[#Transactions]]
  - [[#Basic Indexes]]
  - [[#Data Export and Import]]
  - [[#User Management]]
  - [[#Common psql Meta-Commands Reference]]
  - [[#Error Handling]]
  - [[#Auto-Increment Primary Keys]]
  - [[#Connection Configuration Files]]
  - [[#Data Integrity Concepts]]
  - [[#Best Practices for Beginners]]
  - [[#Next Steps for Learning]]

---

## What is a Database?

[[#Table of Contents|Back to TOC]]

**Database**: A structured collection of data stored electronically in a computer system, designed for efficient retrieval, modification, and management of information.

**Database Management System (DBMS)**: Software that provides an interface between the database and users or applications. The DBMS handles data storage, retrieval, security, and integrity.

**Relational Database**: A database organized into **tables** (also called **relations**) consisting of rows and columns, where relationships between data items are explicitly defined. PostgreSQL is a relational database management system.

**Table**: A collection of related data organized in rows and columns, similar to a spreadsheet. Each table represents a specific entity type (customers, products, orders).

**Row** (also called **record** or **tuple**): A single entry in a table containing data for one instance of the entity. Example: one customer's information.

**Column** (also called **field** or **attribute**): A vertical component of a table defining a single property of the entity. Example: customer email address.

**Primary Key**: A column or combination of columns that uniquely identifies each row in a table. No two rows can have the same primary key value, and the value cannot be NULL (empty).

**Foreign Key**: A column in one table that references the primary key of another table, establishing a relationship between the two tables.

## What is PostgreSQL?

[[#Table of Contents|Back to TOC]]

**PostgreSQL**: An open-source, object-relational database management system emphasizing extensibility and SQL standards compliance. Originally developed at the University of California, Berkeley in 1986.

**SQL (Structured Query Language)**: The standardized programming language used to communicate with relational databases. SQL statements perform operations such as retrieving data, inserting records, updating values, and deleting entries.

**Client-Server Architecture**: PostgreSQL operates using a client-server model:

- **Server**: The PostgreSQL database service running continuously on the computer, managing data storage and processing requests
- **Client**: A program (such as psql) that connects to the server to submit SQL statements and receive results

**Port**: A numbered communication endpoint on a computer. PostgreSQL's default port is 5432. When connecting to PostgreSQL, clients specify the port number to reach the database server.

## Installation on Windows

[[#Table of Contents|Back to TOC]]

**Installation Package**: Download the Windows installer from the official PostgreSQL website (postgresql.org). The installer includes:

- PostgreSQL database server
- pgAdmin (graphical administration tool)
- psql (command-line interface)
- Additional utilities

**Installation Process**:

1. Execute the installer executable file
2. Select installation directory (default: `C:\Program Files\PostgreSQL\[version]\`)
3. Choose components (recommend selecting all)
4. Set data directory location (default: `C:\Program Files\PostgreSQL\[version]\data\`)
5. Create **superuser password**: The password for the `postgres` user, which has complete control over the database system. Record this password securely.
6. Select port number (default: 5432)
7. Choose locale (character set and sorting rules for text data)

**postgres User**: The default administrative user account created during installation with full privileges to create databases, users, and modify all data.

**Windows Service**: After installation, PostgreSQL runs as a Windows service named `postgresql-x64-[version]`, automatically starting when the computer boots.

## Starting PostgreSQL

### Verification of Service Status

Open Command Prompt (cmd.exe) as Administrator:

```cmd
sc query postgresql-x64-15
```

Replace `15` with your installed version number. Output displays service state:

- `RUNNING`: Service is operational
- `STOPPED`: Service is not running

### Starting the Service Manually

```cmd
net start postgresql-x64-15
```

### Stopping the Service

```cmd
net stop postgresql-x64-15
```

**Service Manager**: Alternative method using Windows Services application:

1. Press `Windows + R`, type `services.msc`, press Enter
2. Locate `postgresql-x64-[version]` in the list
3. Right-click and select Start, Stop, or Restart

## Connecting to PostgreSQL

### Using psql (Command-Line Interface)

**psql**: The interactive terminal-based front-end to PostgreSQL, allowing direct execution of SQL statements and special commands.

**Opening psql**:

Method 1 - Start Menu:

1. Navigate to Start Menu → PostgreSQL [version] → SQL Shell (psql)
2. Press Enter to accept default values for server, database, port, and username
3. Enter the superuser password created during installation

Method 2 - Command Prompt:

```cmd
"C:\Program Files\PostgreSQL\15\bin\psql.exe" -U postgres
```

Enter password when prompted.

**Connection Parameters**:

- `-U username`: Specifies the database user account (default: `postgres`)
- `-d database_name`: Specifies which database to connect to (default: `postgres`)
- `-h hostname`: Specifies the server address (default: `localhost` for same computer)
- `-p port`: Specifies the port number (default: `5432`)

**Prompt Appearance**: After successful connection, psql displays a prompt:

```
postgres=#
```

Format: `database_name=#` or `database_name=>`

- `#` indicates superuser privileges
- `>` indicates normal user privileges

### Using pgAdmin (Graphical Interface)

**pgAdmin**: A graphical administration tool for PostgreSQL providing a visual interface for database management tasks.

**Opening pgAdmin**:

1. Start Menu → PostgreSQL [version] → pgAdmin 4
2. Web browser opens displaying the pgAdmin interface
3. Expand "Servers" in the left panel
4. Click "PostgreSQL [version]"
5. Enter the superuser password

**Browser-Based Interface**: pgAdmin operates as a local web application, running a web server on your computer and displaying the interface in your default browser.

## Fundamental SQL Commands

### Creating a Database

**Database**: A container holding collections of tables and other database objects. Each database is independent and isolated from other databases on the same server.

**Syntax**:

```sql
CREATE DATABASE database_name;
```

**Example**:

```sql
CREATE DATABASE bookstore;
```

**Semicolon Requirement**: All SQL statements must terminate with a semicolon (`;`). The semicolon signals the end of the command and instructs PostgreSQL to execute it.

**Naming Rules**:

- Database names must begin with a letter or underscore
- Names can contain letters, numbers, and underscores
- Names are case-insensitive (PostgreSQL converts them to lowercase unless enclosed in double quotes)
- Maximum length: 63 characters

### Listing Databases

**psql Meta-Command**: Special commands in psql beginning with a backslash (`\`) that perform administrative tasks without using SQL.

```
\l
```

Output displays all databases with their owners and encoding settings.

**Equivalent SQL Statement**:

```sql
SELECT datname FROM pg_database;
```

### Connecting to a Database

```
\c database_name
```

**Example**:

```
\c bookstore
```

Prompt changes to reflect the new database:

```
bookstore=#
```

### Creating a Table

**Table Definition**: Specification of the table structure including:

- Table name
- Column names
- Data types for each column
- Constraints (rules governing data validity)

**Syntax**:

```sql
CREATE TABLE table_name (
    column_name1 data_type constraints,
    column_name2 data_type constraints,
    ...
);
```

**Example**:

```sql
CREATE TABLE books (
    book_id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    publication_year INTEGER,
    price NUMERIC(10, 2)
);
```

**Data Type Definitions**:

- **INTEGER**: Whole numbers from -2,147,483,648 to 2,147,483,647
- **TEXT**: Variable-length character strings with unlimited length
- **NUMERIC(precision, scale)**: Exact decimal numbers where `precision` is total digits and `scale` is digits after decimal point
- **VARCHAR(n)**: Variable-length character string with maximum length `n`
- **DATE**: Calendar date (year, month, day)
- **TIMESTAMP**: Date and time value
- **BOOLEAN**: True or false value

**Constraint Definitions**:

- **PRIMARY KEY**: Designates the column as the unique identifier for each row
- **NOT NULL**: Prohibits NULL (empty/unknown) values in the column
- **UNIQUE**: Ensures all values in the column are distinct
- **DEFAULT value**: Specifies a default value when no value is provided during insertion
- **CHECK (condition)**: Enforces a boolean condition on column values

### Listing Tables

```
\dt
```

Displays all tables in the current database.

### Viewing Table Structure

```
\d table_name
```

**Example**:

```
\d books
```

Output shows:

- Column names
- Data types
- Nullable status (whether NULL values are permitted)
- Default values
- Indexes and constraints

## Inserting Data

**INSERT Statement**: SQL command for adding new rows to a table.

**Syntax**:

```sql
INSERT INTO table_name (column1, column2, column3)
VALUES (value1, value2, value3);
```

**Example**:

```sql
INSERT INTO books (book_id, title, author, publication_year, price)
VALUES (1, 'The Great Gatsby', 'F. Scott Fitzgerald', 1925, 15.99);
```

**Multiple Row Insertion**:

```sql
INSERT INTO books (book_id, title, author, publication_year, price)
VALUES 
    (2, '1984', 'George Orwell', 1949, 13.99),
    (3, 'To Kill a Mockingbird', 'Harper Lee', 1960, 14.99),
    (4, 'Pride and Prejudice', 'Jane Austen', 1813, 12.99);
```

**String Literals**: Text values must be enclosed in single quotes (`'`). To include a single quote within the text, use two single quotes: `'O''Brien'`.

**NULL Values**: Represent missing or unknown data. To insert a NULL value, use the keyword `NULL` without quotes:

```sql
INSERT INTO books (book_id, title, author, publication_year, price)
VALUES (5, 'Unknown Title', 'Unknown Author', NULL, NULL);
```

## Retrieving Data

**SELECT Statement**: SQL command for retrieving data from tables.

**Basic Syntax**:

```sql
SELECT column1, column2 FROM table_name;
```

**Retrieve All Columns**:

```sql
SELECT * FROM books;
```

The asterisk (`*`) represents all columns in the table.

**Retrieve Specific Columns**:

```sql
SELECT title, author FROM books;
```

### Filtering Results with WHERE

**WHERE Clause**: Specifies conditions that rows must satisfy to be included in the result set.

**Syntax**:

```sql
SELECT columns FROM table_name WHERE condition;
```

**Examples**:

```sql
-- Books published after 1900
SELECT title, publication_year FROM books WHERE publication_year > 1900;

-- Books by specific author
SELECT title, price FROM books WHERE author = 'George Orwell';

-- Books with price less than 15
SELECT title, price FROM books WHERE price < 15.00;
```

**Comparison Operators**:

- `=`: Equal to
- `!=` or `<>`: Not equal to
- `>`: Greater than
- `<`: Less than
- `>=`: Greater than or equal to
- `<=`: Less than or equal to

**Logical Operators**:

**AND**: Both conditions must be true

```sql
SELECT title FROM books 
WHERE publication_year > 1900 AND price < 15.00;
```

**OR**: At least one condition must be true

```sql
SELECT title FROM books 
WHERE author = 'George Orwell' OR author = 'Jane Austen';
```

**NOT**: Negates a condition

```sql
SELECT title FROM books WHERE NOT author = 'George Orwell';
```

**LIKE Operator**: Pattern matching for text values

```sql
-- Titles containing "the" (case-insensitive in PostgreSQL)
SELECT title FROM books WHERE title ILIKE '%the%';
```

- `%`: Matches zero or more characters
- `_`: Matches exactly one character
- `ILIKE`: Case-insensitive pattern matching (PostgreSQL extension)
- `LIKE`: Case-sensitive pattern matching

### Sorting Results

**ORDER BY Clause**: Specifies the sort order for result rows.

**Syntax**:

```sql
SELECT columns FROM table_name ORDER BY column [ASC|DESC];
```

- **ASC**: Ascending order (smallest to largest, A to Z) - default if not specified
- **DESC**: Descending order (largest to smallest, Z to A)

**Examples**:

```sql
-- Sort by price, lowest first
SELECT title, price FROM books ORDER BY price ASC;

-- Sort by publication year, newest first
SELECT title, publication_year FROM books ORDER BY publication_year DESC;

-- Multiple column sorting
SELECT title, author, price FROM books 
ORDER BY author ASC, price DESC;
```

Multiple column sorting applies subsequent columns as tiebreakers when previous columns have identical values.

### Limiting Results

**LIMIT Clause**: Restricts the number of rows returned.

**Syntax**:

```sql
SELECT columns FROM table_name LIMIT number;
```

**Example**:

```sql
-- Get the 5 most expensive books
SELECT title, price FROM books ORDER BY price DESC LIMIT 5;
```

**OFFSET Clause**: Skips a specified number of rows before returning results.

```sql
-- Skip first 2 results, then return next 5
SELECT title, price FROM books ORDER BY price DESC LIMIT 5 OFFSET 2;
```

## Updating Data

**UPDATE Statement**: Modifies existing rows in a table.

**Syntax**:

```sql
UPDATE table_name 
SET column1 = value1, column2 = value2 
WHERE condition;
```

**WARNING**: Omitting the WHERE clause updates ALL rows in the table.

**Example**:

```sql
-- Update price for a specific book
UPDATE books 
SET price = 16.99 
WHERE book_id = 1;

-- Update multiple columns
UPDATE books 
SET price = 18.99, publication_year = 1926 
WHERE book_id = 1;

-- Update based on condition
UPDATE books 
SET price = price * 1.10 
WHERE publication_year < 1950;
```

The last example increases prices by 10% for books published before 1950.

## Deleting Data

**DELETE Statement**: Removes rows from a table.

**Syntax**:

```sql
DELETE FROM table_name WHERE condition;
```

**WARNING**: Omitting the WHERE clause deletes ALL rows in the table.

**Example**:

```sql
-- Delete specific book
DELETE FROM books WHERE book_id = 5;

-- Delete based on condition
DELETE FROM books WHERE price > 20.00;
```

**TRUNCATE Statement**: Removes all rows from a table more efficiently than DELETE, but cannot include a WHERE clause.

```sql
TRUNCATE TABLE books;
```

## Understanding Relationships Between Tables

### One-to-Many Relationship

**Definition**: One row in Table A can relate to multiple rows in Table B, but each row in Table B relates to only one row in Table A.

**Example**: One author can write multiple books, but each book has one primary author.

**Implementation Using Foreign Keys**:

```sql
-- Create authors table
CREATE TABLE authors (
    author_id INTEGER PRIMARY KEY,
    author_name TEXT NOT NULL,
    birth_year INTEGER
);

-- Create books table with foreign key
CREATE TABLE books (
    book_id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author_id INTEGER REFERENCES authors(author_id),
    publication_year INTEGER,
    price NUMERIC(10, 2)
);
```

**REFERENCES Constraint**: Establishes a foreign key relationship. The `author_id` column in the `books` table must contain values that exist in the `author_id` column of the `authors` table (or NULL if permitted).

**Referential Integrity**: The database enforces that foreign key values must reference existing primary key values, preventing orphaned records (child records without corresponding parent records).

### Inserting Related Data

```sql
-- Insert authors first
INSERT INTO authors (author_id, author_name, birth_year)
VALUES 
    (1, 'F. Scott Fitzgerald', 1896),
    (2, 'George Orwell', 1903);

-- Insert books referencing authors
INSERT INTO books (book_id, title, author_id, publication_year, price)
VALUES 
    (1, 'The Great Gatsby', 1, 1925, 15.99),
    (2, '1984', 2, 1949, 13.99);
```

**Constraint Violation**: Attempting to insert a book with a non-existent `author_id` produces an error:

```sql
-- This will fail
INSERT INTO books (book_id, title, author_id, publication_year, price)
VALUES (3, 'Some Book', 999, 2020, 10.00);
```

Error message: `ERROR: insert or update on table "books" violates foreign key constraint`

## Joining Tables

**JOIN Operation**: Combines rows from multiple tables based on related columns.

**INNER JOIN**: Returns only rows where matching values exist in both tables.

**Syntax**:

```sql
SELECT columns
FROM table1
INNER JOIN table2 ON table1.column = table2.column;
```

**Example**:

```sql
SELECT books.title, authors.author_name, books.price
FROM books
INNER JOIN authors ON books.author_id = authors.author_id;
```

**Table Aliases**: Shortened names for tables used to simplify queries:

```sql
SELECT b.title, a.author_name, b.price
FROM books AS b
INNER JOIN authors AS a ON b.author_id = a.author_id;
```

The `AS` keyword is optional: `FROM books b` is equivalent to `FROM books AS b`.

**LEFT JOIN** (or **LEFT OUTER JOIN**): Returns all rows from the left table and matching rows from the right table. If no match exists, NULL values appear for right table columns.

```sql
SELECT a.author_name, b.title
FROM authors AS a
LEFT JOIN books AS b ON a.author_id = b.author_id;
```

This query returns all authors, including those with no books in the database.

## Aggregate Functions

**Aggregate Function**: A function that performs a calculation on multiple rows and returns a single result value.

**COUNT()**: Returns the number of rows

```sql
-- Count total books
SELECT COUNT(*) FROM books;

-- Count books by specific author
SELECT COUNT(*) FROM books WHERE author_id = 1;
```

**SUM()**: Calculates the total of numeric values

```sql
-- Total value of all books
SELECT SUM(price) FROM books;
```

**AVG()**: Calculates the average of numeric values

```sql
-- Average book price
SELECT AVG(price) FROM books;
```

**MAX()**: Returns the maximum value

```sql
-- Highest book price
SELECT MAX(price) FROM books;
```

**MIN()**: Returns the minimum value

```sql
-- Lowest book price
SELECT MIN(price) FROM books;
```

**Multiple Aggregates**:

```sql
SELECT 
    COUNT(*) AS total_books,
    AVG(price) AS average_price,
    MIN(price) AS lowest_price,
    MAX(price) AS highest_price
FROM books;
```

**AS Keyword**: Assigns an alias (alternative name) to a column or table in the result set, improving readability.

## Grouping Data

**GROUP BY Clause**: Divides rows into groups based on column values, allowing aggregate functions to operate on each group independently.

**Syntax**:

```sql
SELECT column, aggregate_function(column)
FROM table_name
GROUP BY column;
```

**Example**:

```sql
-- Count books by each author
SELECT author_id, COUNT(*) AS book_count
FROM books
GROUP BY author_id;
```

**With JOIN**:

```sql
-- Count books by each author with author names
SELECT a.author_name, COUNT(b.book_id) AS book_count
FROM authors AS a
LEFT JOIN books AS b ON a.author_id = b.author_id
GROUP BY a.author_name;
```

**HAVING Clause**: Filters groups after aggregation (WHERE filters rows before aggregation).

```sql
-- Authors with more than 2 books
SELECT a.author_name, COUNT(b.book_id) AS book_count
FROM authors AS a
LEFT JOIN books AS b ON a.author_id = b.author_id
GROUP BY a.author_name
HAVING COUNT(b.book_id) > 2;
```

## Transactions

**Transaction**: A sequence of one or more SQL operations executed as a single unit. Transactions ensure data consistency by following ACID properties:

- **Atomicity**: All operations complete successfully or none take effect
- **Consistency**: Database transitions from one valid state to another
- **Isolation**: Concurrent transactions do not interfere with each other
- **Durability**: Completed transactions persist even after system failure

**Transaction Commands**:

**BEGIN**: Starts a new transaction

```sql
BEGIN;
```

**COMMIT**: Saves all changes made during the transaction permanently

```sql
COMMIT;
```

**ROLLBACK**: Discards all changes made during the transaction

```sql
ROLLBACK;
```

**Example Transaction**:

```sql
BEGIN;

INSERT INTO authors (author_id, author_name, birth_year)
VALUES (3, 'Ernest Hemingway', 1899);

INSERT INTO books (book_id, title, author_id, publication_year, price)
VALUES (10, 'The Old Man and the Sea', 3, 1952, 12.99);

COMMIT;
```

If any statement fails between BEGIN and COMMIT, execute ROLLBACK to undo changes:

```sql
BEGIN;

DELETE FROM books WHERE book_id = 1;

-- Realized this was a mistake
ROLLBACK;
```

**Autocommit Mode**: By default, psql automatically commits each statement immediately. BEGIN disables autocommit until COMMIT or ROLLBACK.

## Basic Indexes

**Index**: A data structure that improves the speed of data retrieval operations at the cost of additional storage space and slower write operations.

**Analogy**: Similar to a book index allowing direct navigation to specific topics rather than reading sequentially.

**Creating an Index**:

```sql
CREATE INDEX index_name ON table_name (column_name);
```

**Example**:

```sql
-- Index on author_id for faster lookups
CREATE INDEX idx_books_author ON books (author_id);

-- Index on title for text searches
CREATE INDEX idx_books_title ON books (title);
```

**Index Naming Convention**: Prefix with `idx_`, followed by table name and column name.

**When to Create Indexes**:

- Columns frequently used in WHERE clauses
- Columns used in JOIN conditions
- Columns used in ORDER BY clauses
- Foreign key columns

**When NOT to Create Indexes**:

- Small tables (few hundred rows)
- Columns with few distinct values (low cardinality)
- Tables with frequent INSERT, UPDATE, DELETE operations where read speed is not critical

**Viewing Indexes**:

```
\di
```

**Removing an Index**:

```sql
DROP INDEX index_name;
```

## Data Export and Import

### Exporting Data to CSV

**COPY Command**: PostgreSQL command for bulk data transfer between files and tables.

**Syntax for Export**:

```sql
\copy table_name TO 'C:\path\to\file.csv' WITH CSV HEADER;
```

**Example**:

```sql
\copy books TO 'C:\Users\username\Documents\books.csv' WITH CSV HEADER;
```

**Options**:

- `CSV`: Specifies comma-separated values format
- `HEADER`: Includes column names in the first row
- `DELIMITER ','`: Specifies the character separating values (comma is default for CSV)

**Windows Path Requirement**: Use forward slashes (`/`) or double backslashes (`\\`) in paths:

```sql
-- Forward slashes (recommended)
\copy books TO 'C:/Users/username/Documents/books.csv' WITH CSV HEADER;

-- Double backslashes
\copy books TO 'C:\\Users\\username\\Documents\\books.csv' WITH CSV HEADER;
```

### Importing Data from CSV

**Syntax for Import**:

```sql
\copy table_name FROM 'C:\path\to\file.csv' WITH CSV HEADER;
```

**Example**:

```sql
\copy books FROM 'C:/Users/username/Documents/books.csv' WITH CSV HEADER;
```

**Prerequisites**:

- Table must exist before importing
- Column names in CSV must match table column names (when using HEADER)
- Data types in CSV must be compatible with table column types

### Database Backup

**pg_dump**: Command-line utility for creating backups of PostgreSQL databases.

**Basic Syntax**:

```cmd
"C:\Program Files\PostgreSQL\15\bin\pg_dump.exe" -U postgres -d database_name -f "C:\backups\database_backup.sql"
```

**Example**:

```cmd
"C:\Program Files\PostgreSQL\15\bin\pg_dump.exe" -U postgres -d bookstore -f "C:\backups\bookstore_backup.sql"
```

Enter password when prompted.

**Output**: SQL text file containing all commands necessary to recreate the database structure and data.

### Database Restore

**psql Import**:

```cmd
"C:\Program Files\PostgreSQL\15\bin\psql.exe" -U postgres -d database_name -f "C:\backups\database_backup.sql"
```

**Steps for Complete Restore**:

1. Create new database:

```cmd
"C:\Program Files\PostgreSQL\15\bin\psql.exe" -U postgres -c "CREATE DATABASE bookstore_restored;"
```

2. Restore backup into new database:

```cmd
"C:\Program Files\PostgreSQL\15\bin\psql.exe" -U postgres -d bookstore_restored -f "C:\backups\bookstore_backup.sql"
```

## User Management

**Database User** (also called **Role**): An account with specific privileges for accessing and manipulating databases and their objects.

### Creating a User

```sql
CREATE USER username WITH PASSWORD 'password123';
```

**Example**:

```sql
CREATE USER librarian WITH PASSWORD 'secure_password';
```

**User Attributes**:

```sql
CREATE USER username WITH 
    PASSWORD 'password'
    CREATEDB           -- Can create databases
    CREATEROLE         -- Can create other users
    LOGIN;             -- Can connect to database
```

### Granting Permissions

**GRANT Statement**: Assigns specific privileges to users.

**Syntax**:

```sql
GRANT privilege ON object TO username;
```

**Common Privileges**:

- **SELECT**: Read data
- **INSERT**: Add new rows
- **UPDATE**: Modify existing rows
- **DELETE**: Remove rows
- **ALL PRIVILEGES**: All available privileges

**Examples**:

```sql
-- Grant read-only access to books table
GRANT SELECT ON books TO librarian;

-- Grant full access to books table
GRANT ALL PRIVILEGES ON books TO librarian;

-- Grant access to all tables in database
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO librarian;

-- Grant database connection permission
GRANT CONNECT ON DATABASE bookstore TO librarian;
```

**Schema**: A namespace within a database containing tables and other objects. The `public` schema is the default schema created in every database.

### Revoking Permissions

```sql
REVOKE privilege ON object FROM username;
```

**Example**:

```sql
REVOKE DELETE ON books FROM librarian;
```

### Removing a User

```sql
DROP USER username;
```

**Prerequisite**: User must not own any database objects or have active connections.

## Common psql Meta-Commands Reference

**Meta-Command**: psql-specific command beginning with backslash (`\`) for administrative tasks.

|Command|Description|
|---|---|
|`\l`|List all databases|
|`\c database_name`|Connect to database|
|`\dt`|List all tables in current database|
|`\d table_name`|Describe table structure|
|`\du`|List all users (roles)|
|`\di`|List all indexes|
|`\df`|List all functions|
|`\dn`|List all schemas|
|`\q`|Quit psql|
|`\?`|Display help for meta-commands|
|`\h SQL_COMMAND`|Display SQL command syntax help|
|`\timing`|Toggle query execution time display|
|`\x`|Toggle expanded table display (vertical format)|

**Usage Example**:

```
\h CREATE TABLE
```

Displays syntax documentation for the CREATE TABLE command.

## Error Handling

### Common Errors and Solutions

**Error**: `database "bookstore" does not exist`

**Solution**: Create the database first using `CREATE DATABASE bookstore;`

**Error**: `relation "books" does not exist`

**Solution**: Verify you are connected to the correct database (`\c database_name`) and the table exists (`\dt`)

**Error**: `column "titel" does not exist`

**Solution**: Check spelling of column names. Column names are case-insensitive unless quoted.

**Error**: `syntax error at or near "SELECT"`

**Solution**: Verify previous statement ended with semicolon. Check for missing parentheses, commas, or quotes.

**Error**: `duplicate key value violates unique constraint`

**Solution**: Attempting to insert a primary key value that already exists. Use different value or let PostgreSQL generate it automatically (see Auto-Increment section).

**Error**: `permission denied for table books`

**Solution**: Current user lacks necessary privileges. Requires GRANT statement from database owner or superuser.

**Error**: `password authentication failed for user "postgres"`

**Solution**: Incorrect password. Use password set during installation. If forgotten, requires password reset procedure.

## Auto-Increment Primary Keys

**SERIAL Data Type**: Special integer type that automatically generates sequential numbers for new rows.

**Definition**: `SERIAL` is equivalent to creating an INTEGER column with a DEFAULT value from a sequence generator.

**Syntax**:

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT NOT NULL
);
```

**Insertion Without Specifying ID**:

```sql
INSERT INTO books (title, author)
VALUES ('New Book', 'New Author');
```

PostgreSQL automatically assigns the next available `book_id` value.

**Explicit ID Specification** (overrides automatic generation):

```sql
INSERT INTO books (book_id, title, author)
VALUES (100, 'Specific ID Book', 'Author Name');
```

**BIGSERIAL**: Similar to SERIAL but accommodates larger numbers (maximum: 9,223,372,036,854,775,807).

**Sequence**: The underlying database object generating sequential numbers. Created automatically with SERIAL columns.

**Viewing Current Sequence Value**:

```sql
SELECT currval('books_book_id_seq');
```

## Connection Configuration Files

**pgpass.conf**: File storing passwords to avoid manual entry when connecting.

**Location on Windows**: `%APPDATA%\postgresql\pgpass.conf`

Typical path: `C:\Users\username\AppData\Roaming\postgresql\pgpass.conf`

**Format**: Each line contains connection parameters and password:

```
hostname:port:database:username:password
```

**Example Content**:

```
localhost:5432:bookstore:postgres:your_password
localhost:5432:*:postgres:your_password
*:*:*:librarian:librarian_password
```

- `*`: Wildcard matching any value
- Lines are evaluated top-to-bottom, first match is used

**Security**: File should have restricted permissions. Only the file owner should have read access.

**Environment Variables**:

Set default connection parameters using environment variables (session-specific):

```cmd
SET PGHOST=localhost
SET PGPORT=5432
SET PGDATABASE=bookstore
SET PGUSER=postgres
```

After setting these, `psql` command without parameters uses these defaults:

```cmd
psql
```

## Data Integrity Concepts

**NULL**: Special marker representing missing, unknown, or inapplicable data. NULL is not equal to zero, empty string, or any other value.

**Testing for NULL**:

```sql
-- Find books with unknown publication year
SELECT title FROM books WHERE publication_year IS NULL;

-- Find books with known publication year
SELECT title FROM books WHERE publication_year IS NOT NULL;
```

**Note**: Use `IS NULL` and `IS NOT NULL` operators, not `= NULL` or `!= NULL`.

**CHECK Constraint**: Enforces a boolean condition on column values.

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    price NUMERIC(10, 2) CHECK (price > 0),
    publication_year INTEGER CHECK (publication_year >= 1450)
);
```

Attempts to insert invalid data produce constraint violation errors:

```sql
-- This fails due to CHECK constraint
INSERT INTO books (title, price) VALUES ('Invalid Book', -5.00);
```

**UNIQUE Constraint**: Ensures all values in a column (or combination of columns) are distinct.

```sql
CREATE TABLE authors (
    author_id SERIAL PRIMARY KEY,
    author_name TEXT NOT NULL,
    email TEXT UNIQUE
);
```

**Composite Unique Constraint**: Uniqueness across multiple columns:

```sql
CREATE TABLE enrollments (
    student_id INTEGER,
    course_id INTEGER,
    semester TEXT,
    UNIQUE (student_id, course_id, semester)
);
```

This allows the same student to enroll in different courses or the same course in different semesters, but prevents duplicate enrollments in the same course during the same semester.

## Best Practices for Beginners

### SQL Formatting

**Readability**: Write SQL statements across multiple lines for complex queries:

```sql
SELECT 
    b.title,
    a.author_name,
    b.publication_year,
    b.price
FROM 
    books AS b
INNER JOIN 
    authors AS a ON b.author_id = a.author_id
WHERE 
    b.publication_year > 1950
    AND b.price < 20.00
ORDER BY 
    b.publication_year DESC;
```

**Capitalization Convention**: Write SQL keywords in UPPERCASE and user-defined names (tables, columns) in lowercase for visual distinction. PostgreSQL is case-insensitive for keywords and unquoted identifiers.

### Naming Conventions

**Table Names**:

- Use plural nouns: `books`, `authors`, `customers`
- Use lowercase with underscores for multiple words: `book_reviews`, `customer_orders`

**Column Names**:

- Use singular descriptive names: `title`, `price`, `publication_year`
- Use lowercase with underscores: `first_name`, `last_name`

**Primary Key Columns**:

- Pattern: `table_name_id` (e.g., `book_id`, `author_id`)
- Alternative: Simply `id` (less explicit but common)

**Foreign Key Columns**:

- Use same name as referenced primary key: `author_id` in books table references `author_id` in authors table

**Index Names**:

- Pattern: `idx_table_column` (e.g., `idx_books_title`)

### Data Backup Routine

**Recommendation**: Perform regular database backups before making structural changes or bulk data modifications.

**Simple Backup Strategy**:

1. Create backups directory:

```cmd
mkdir C:\PostgreSQL_Backups
```

2. Regular backup command:

```cmd
"C:\Program Files\PostgreSQL\15\bin\pg_dump.exe" -U postgres -d bookstore -f "C:\PostgreSQL_Backups\bookstore_%date:~-4,4%%date:~-10,2%%date:~-7,2%.sql"
```

This creates a backup file with the current date in the filename.

**Backup Schedule**: Daily backups for actively modified databases, weekly backups for stable databases.

### Testing Queries

**Development Practice**: Test queries with SELECT before executing UPDATE or DELETE:

```sql
-- Test which rows will be affected
SELECT * FROM books WHERE publication_year < 1900;

-- If results are correct, proceed with update
UPDATE books SET price = price * 0.90 WHERE publication_year < 1900;
```

### Transaction Usage

**Critical Operations**: Wrap important operations in transactions for safety:

```sql
BEGIN;

-- Execute changes
DELETE FROM books WHERE author_id = 5;
DELETE FROM authors WHERE author_id = 5;

-- Verify results
SELECT * FROM books WHERE author_id = 5;
SELECT * FROM authors WHERE author_id = 5;

-- If correct, commit; otherwise rollback
COMMIT;
```

## Next Steps for Learning

### Skills to Develop

1. **Subqueries**: Queries nested within other queries
2. **Views**: Saved queries accessible as virtual tables
3. **Stored Procedures**: Reusable code blocks executed by the database
4. **Triggers**: Automatic actions executed in response to data changes
5. **Performance Optimization**: Query analysis and index tuning
6. **Normalization**: Database design principles for reducing redundancy

### Resources for Further Study

**PostgreSQL Official Documentation**: https://www.postgresql.org/docs/

**Tutorial Sections**:

- SQL Tutorial (Part II of documentation)
- Data Definition (Part II, Chapter 5)
- Data Manipulation (Part II, Chapter 6)
- Queries (Part II, Chapter 7)

**Practice Recommendations**:

- Create sample databases for personal projects (recipe collection, movie library, expense tracking)
- Experiment with different table designs and relationships
- Practice writing queries before looking at solutions
- Review PostgreSQL error messages carefully for learning opportunities

This guide provides foundational knowledge for beginning PostgreSQL database work on Windows systems. Mastery requires consistent practice with progressively complex database scenarios.
