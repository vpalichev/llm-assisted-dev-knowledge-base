# PowerShell Array and Object Operations

Techniques for inspecting, manipulating, and transforming arrays and objects, with SharePoint Client Object Model patterns.

## Variable Type Inspection

PowerShell methods for examining variable types:

### Basic Type Checking

```powershell
# Get type of variable
$var = "hello"
$var.GetType()

# Just the type name (shorter)
$var.GetType().Name

# Full type name with namespace (most complete)
$var.GetType().FullName
```

### Inspecting Array Elements

When working with arrays (Object[]), you need to inspect individual elements since arrays themselves have limited type information.

```powershell
# See what's inside an array with detailed formatting
$var | Format-List *

# Get type of first element
$var[0].GetType()

# Get types of all elements (useful for heterogeneous arrays)
$var | ForEach-Object { $_.GetType().Name }

# Basic array information
$var.Count    # Number of elements
$var.Length   # Same as Count for arrays

# Display actual values
$var

# Deep inspection of object members
$var | Get-Member
```

### SharePoint ListItem Type Names

SharePoint Client objects often show abbreviated type names. Here's how to get complete type information:

```powershell
# Get full type name (preferred for SharePoint objects)
$var[0].GetType().FullName

# Alternative using Get-Member's TypeName property
# This works because all member objects share the same TypeName
$var[0] | Get-Member | Select-Object -First 1 -ExpandProperty TypeName
```

**Understanding the Get-Member TypeName approach:**
- `Get-Member` returns objects representing each property/method
- Each member object has a `TypeName` property containing the source object's full type
- All members from the same object share identical `TypeName` values
- `Select-Object -First 1` grabs any single member to extract the shared type name

## Array Operations

### Length vs Count Properties

Differences between Length and Count properties by object type:

```powershell
# Arrays: both work identically
$arr = 1,2,3
$arr.Length  # 3
$arr.Count   # 3

# Collections/Generic Lists: Count is preferred
$list = [System.Collections.ArrayList]@(1,2,3)
$list.Count   # 3
$list.Length  # May not exist or behave differently

# Strings: Length for character count
$str = "hello"
$str.Length  # 5 (characters)
$str.Count   # 1 (single string object)

# Best Practice: Use .Count for collections, .Length for arrays/strings
```

### Array Type Statistics

When working with mixed-type arrays, you often need to understand the composition:

```powershell
# Group by short type name and count occurrences
$var | Group-Object { $_.GetType().Name } | Select-Object Name, Count

# Group by full type name (more precise for .NET objects)
$var | Group-Object { $_.GetType().FullName } | Select-Object Name, Count

# Sort by frequency (most common types first)
$var | Group-Object { $_.GetType().Name } | Sort-Object Count -Descending
```

### Array Slicing

PowerShell offers multiple ways to extract array subsets:

```powershell
# Get first 30 elements
$var | Select-Object -First 30

# Get elements from index 100 to 200 (inclusive, 101 elements total)
$var[100..200]

# Alternative using Select-Object (more flexible for complex filtering)
$var | Select-Object -Skip 100 -First 101
```

**Note:** Array slicing with `[100..200]` is efficient and direct, while `Select-Object -Skip/-First` provides more pipeline flexibility.

## Field Value Extraction

### Basic Field Access

```powershell
# Extract single field from all objects
$var | Select-Object -ExpandProperty FieldName
$var.FieldName  # Shorter syntax for simple cases

# Extract multiple fields as objects
$var | Select-Object Field1, Field2, Field3

# Display as formatted table
$var | Format-Table Field1, Field2, Field3

# Access nested properties
$var | Select-Object -ExpandProperty "ParentProperty.ChildProperty"
```

### Array Slice Field Extraction

Combine slicing with field extraction for targeted data access:

```powershell
# First 30 elements, specific fields only
$var | Select-Object -First 30 | Select-Object Field1, Field2

# Index range 100-200, specific fields
$var[100..200] | Select-Object Field1, Field2

# Combined approach (single pipeline)
$var | Select-Object -Skip 100 -First 101 Field1, Field2

# Single field from slice
$var[100..200].FieldName
$var[100..200] | Select-Object -ExpandProperty FieldName
```

### Indexer-Based Field Access (SharePoint Pattern)

SharePoint Client objects use indexer syntax (like `["FieldName"]`) for field access. This is common in CSOM (Client-Side Object Model):

```powershell
# Access single indexer field from array slice
$var[100..200] | ForEach-Object { $_["FieldName"] }

# Multiple indexer fields as custom objects
$var[100..200] | ForEach-Object {
    [PSCustomObject]@{
        Field1 = $_["Field1"]
        Field2 = $_["Field2"]
        Field3 = $_["Field3"]
    }
}

# Display indexer fields in table format
$var[100..200] | ForEach-Object {
    [PSCustomObject]@{
        Title = $_["Title"]
        Author = $_["Author"]
    }
} | Format-Table
```

### Accessing All Indexer Fields

For objects with comprehensive field collections (like SharePoint FieldValues):

```powershell
# Access all fields via FieldValues property
$var[100..200] | ForEach-Object { $_.FieldValues }

# Alternative if using Item property
$var[100..200] | ForEach-Object { $_.Item }

# Convert all fields to custom objects (preserves structure)
$var[100..200] | ForEach-Object {
    $item = $_
    $props = @{}
    $_.FieldValues.Keys | ForEach-Object { $props[$_] = $item.FieldValues[$_] }
    [PSCustomObject]$props
}

# Direct access if FieldValues is a hashtable
$var[100..200].FieldValues
```

### Field Type Inspection

```powershell
# Get type of specific indexer field
$var[0]["FieldName"].GetType()
$var[0]["FieldName"].GetType().FullName

# Check type from slice element
$var[100]["FieldName"].GetType()

# Type checking across multiple elements
$var[100..200] | ForEach-Object {
    $_["FieldName"].GetType().FullName
} | Select-Object -Unique

# Safe type checking (handles null values)
$var[0]["FieldName"]?.GetType().FullName

# Traditional null check approach
if ($var[0]["FieldName"] -ne $null) {
    $var[0]["FieldName"].GetType()
}
```

## Data Transformation and Export

### Converting to Structured Tables

Transform SharePoint ListItem arrays into tabular data with specific columns:

```powershell
# Convert to table with specific SharePoint columns
$var | ForEach-Object {
    [PSCustomObject]@{
        FileLeafRef = $_["FileLeafRef"]      # File name
        FileRef = $_["FileRef"]              # Full file path
        File_x0020_Size = $_["File_x0020_Size"]  # File size
        FSObjType = $_["FSObjType"]          # File/folder indicator
    }
} | Format-Table

# Dynamic conversion using all available fields
$var | ForEach-Object {
    $item = $_
    $props = @{}
    $item.FieldValues.Keys | ForEach-Object { $props[$_] = $item.FieldValues[$_] }
    [PSCustomObject]$props
} | Format-Table
```

### CSV Export

Export transformed data for external analysis:

```powershell
# Export specific columns to CSV
$var | ForEach-Object {
    [PSCustomObject]@{
        FileLeafRef = $_["FileLeafRef"]
        FileRef = $_["FileRef"]
        File_x0020_Size = $_["File_x0020_Size"]
        FSObjType = $_["FSObjType"]
    }
} | Export-Csv -Path "output.csv" -NoTypeInformation
```

### Understanding the Transformation Pipeline

The SharePoint object transformation follows this data flow:

#### Step 1: Pipeline Input
```powershell
$var | ForEach-Object {
```
- `$var` contains array of SharePoint ListItem objects
- `|` (pipeline operator) passes each ListItem to `ForEach-Object`
- `ForEach-Object` processes one item at a time

#### Step 2: Object Creation
```powershell
    [PSCustomObject]@{
        FileLeafRef = $_["FileLeafRef"]      # File name
        FileRef = $_["FileRef"]              # Full file path
        File_x0020_Size = $_["File_x0020_Size"]  # File size
        FSObjType = $_["FSObjType"]          # File/folder indicator
    }
```
- `[PSCustomObject]` creates structured object with named properties
- `$_` represents current ListItem in pipeline
- `$_["FieldName"]` accesses SharePoint field values via indexer
- Each property extracts specific field data from the ListItem

#### Step 3: Pipeline Output
```powershell
} | Format-Table
```
- `ForEach-Object` outputs PSCustomObject for each input item
- Result: Array of structured objects with consistent property names
- `Format-Table` displays results in readable tabular format

#### Key Components Explained

**Pipeline Flow:**
```
SharePoint ListItem[] → ForEach-Object → PSCustomObject[] → Format-Table → Display
```

**SharePoint Field Access:**
- `$_["FileLeafRef"]` gets filename from ListItem.FieldValues
- SharePoint stores field data in FieldValues collection
- Indexer syntax `["FieldName"]` retrieves values by internal field name

**PSCustomObject Benefits:**
- Structured data with named properties
- Enables `Format-Table`, `Export-Csv`, and property access
- Replaces loosely-typed SharePoint objects with predictable structure

### Error Handling with Get-Member

When working with arrays, Get-Member can sometimes fail unexpectedly:

```powershell
# Direct Get-Member may fail on arrays in some contexts
$var | Get-Member  # Can throw "You must specify an object" error

# Safer approaches:
# Check individual elements
$var[0] | Get-Member

# Get unique members across all elements
$var | ForEach-Object { $_ | Get-Member } | Select-Object -Unique
