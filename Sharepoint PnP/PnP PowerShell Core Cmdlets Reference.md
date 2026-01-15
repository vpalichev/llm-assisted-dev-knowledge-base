# PnP PowerShell Core Cmdlets for SharePoint Online

## Module Overview

**PnP PowerShell** (formerly SharePointPnPPowerShellOnline) is a cross-platform PowerShell module for managing SharePoint Online environments. It wraps the SharePoint REST API and CSOM (Client-Side Object Model) into PowerShell cmdlets.

**Installation:**
```powershell
Install-Module -Name PnP.PowerShell -Scope CurrentUser
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

---

## Connection Management

### Connect-PnPOnline

Establishes an authenticated session to a SharePoint Online site collection. Must execute before any other PnP operations.

```powershell
Connect-PnPOnline -Url "https://tenant.sharepoint.com/sites/sitename" -Interactive
```

**Parameters:**
- `-Url`: Target site collection URL (mandatory)
- `-Interactive`: Browser-based modern authentication
- `-ClientId` / `-ClientSecret`: Application-based authentication
- `-UseWebLogin`: Legacy web-based authentication

The connection persists for the PowerShell session duration. Multiple connections to different sites require `Disconnect-PnPOnline` or new connection invocation.

---

## Get-PnPList

Retrieves list object instances from the current SharePoint site collection context. Returns **`Microsoft.SharePoint.Client.List`** objects representing SharePoint list containers (document libraries, custom lists, system lists, hidden lists).

### Parameter Sets

**Identity Parameter Set:**
```powershell
Get-PnPList -Identity <ListPipeBind>
```

The `-Identity` parameter accepts:
- **List Title** (string): Display name (e.g., "Documents")
- **List URL** (string): Server-relative URL (e.g., "Shared Documents")
- **List GUID** (Guid): `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`

**Retrieval Without Parameters:**
```powershell
Get-PnPList  # Returns all lists including hidden system lists
```

**Includes Parameter:**
```powershell
Get-PnPList -Identity "Documents" -Includes RootFolder,Fields,ContentTypes
```

Pre-loads related objects not retrieved by default due to CSOM's demand-loading architecture.

Common include targets:
- `RootFolder`: Folder object reference with path information
- `Fields`: Complete field (column) schema collection
- `ContentTypes`: Content type definitions
- `Views`: View definition collection
- `DefaultView`: Default view configuration

### Critical Properties

**Identification:**

| Property | Type | Description |
|----------|------|-------------|
| `Id` | Guid | Immutable unique identifier |
| `Title` | String | Display name (mutable) |
| `RootFolder.ServerRelativeUrl` | String | Server-relative path (requires `-Includes RootFolder`) |

**Template and Type:**

| Property | Type | Description |
|----------|------|-------------|
| `BaseTemplate` | Integer | List template type identifier |
| `BaseType` | Enum | Fundamental list classification |

**BaseTemplate Values:**
- `100`: Generic List
- `101`: Document Library
- `106`: Events List
- `107`: Tasks List
- `108`: Discussion Board
- `109`: Picture Library
- `119`: Wiki Page Library
- `171`: Tasks List with Timeline

**BaseType Values:**
- `GenericList` (0): Standard list
- `DocumentLibrary` (1): File storage container
- `Survey` (4): Survey list
- `Issue` (5): Issue tracking list

**Content Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `ItemCount` | Integer | Item count (files for document libraries, excluding folders) |
| `Hidden` | Boolean | Hidden from SharePoint navigation |
| `NoCrawl` | Boolean | Prevents search indexing |

**Versioning Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `EnableVersioning` | Boolean | Version history enabled |
| `MajorVersionLimit` | Integer | Max major versions (0 = unlimited) |
| `EnableMinorVersions` | Boolean | Draft versions enabled |
| `HasUniqueRoleAssignments` | Boolean | Unique permissions (not inherited) |

### Examples

```powershell
# Retrieve by title
$docLibrary = Get-PnPList -Identity "Documents"

# Retrieve by server-relative URL (more reliable for special characters)
$library = Get-PnPList -Identity "Shared Documents"

# Retrieve by GUID (most reliable - immutable)
$list = Get-PnPList -Identity "{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}"

# Get all document libraries
$documentLibraries = Get-PnPList | Where-Object { $_.BaseTemplate -eq 101 }

# Pre-load properties for direct access
$list = Get-PnPList -Identity "Documents" -Includes RootFolder,Fields,DefaultView
$folderPath = $list.RootFolder.ServerRelativeUrl
$fieldCount = $list.Fields.Count
$defaultViewTitle = $list.DefaultView.Title

# Filter non-hidden lists with items
Get-PnPList | Where-Object {
    -not $_.Hidden -and $_.ItemCount -gt 0
} | Select-Object Title, ItemCount, BaseType
```

### Performance: Demand-Loading

CSOM objects use demand-loading. Accessing properties without `-Includes` triggers server requests:

```powershell
# Inefficient: Server call per iteration
foreach ($list in Get-PnPList) {
    $path = $list.RootFolder.ServerRelativeUrl
}

# Efficient: Single server round-trip
foreach ($list in Get-PnPList -Includes RootFolder) {
    $path = $list.RootFolder.ServerRelativeUrl
}
```

---

## Get-PnPListItem

Retrieves list item instances from a SharePoint list or document library. Returns **`Microsoft.SharePoint.Client.ListItem`** objects.

### List Items vs. Files

- **List Item** = Data row with field values and metadata
- **File** = Binary content referenced by a list item

For document libraries:
- Every file has an underlying ListItem storing metadata
- ListItem contains `.File` property reference to file content
- `Get-PnPListItem` retrieves metadata, not file binary
- Use `Get-PnPFile` to download file content

### Parameter Sets

**Basic Retrieval:**
```powershell
Get-PnPListItem -List <ListPipeBind>
```

**List Parameter Types:**
- List Title: "Documents"
- List URL: "Shared Documents"
- List GUID: `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`
- List Object: Result from `Get-PnPList`

**Single Item by ID:**
```powershell
Get-PnPListItem -List <ListPipeBind> -Id <Int32>
```
Most efficient retrieval method (direct index lookup).

**Single Item by GUID:**
```powershell
Get-PnPListItem -List <ListPipeBind> -UniqueId <Guid>
```

**CAML Query:**
```powershell
Get-PnPListItem -List <ListPipeBind> -Query <String>
```

**Field Projection:**
```powershell
Get-PnPListItem -List <ListPipeBind> -Fields <String[]>
```
Server-side column filtering reduces payload size.

**Pagination:**
```powershell
Get-PnPListItem -List <ListPipeBind> -PageSize <Int32>
```
- Default: 100 items
- Maximum: 5000 (list view threshold)
- Continuation tokens handled automatically

### System Properties

| Property | Type | Description |
|----------|------|-------------|
| `Id` | Int32 | Auto-incrementing integer (1 to 2,147,483,647) |
| `GUID` | Guid | Globally unique identifier |
| `ContentTypeId` | String | Hierarchical content type ID (e.g., `0x0101` = Document) |
| `FileSystemObjectType` | Enum | `File` (0), `Folder` (1), `Web` (2), `Invalid` (-1) |
| `Created` / `Modified` | DateTime | Timestamps (UTC, converted to local) |
| `Author` / `Editor` | FieldUserValue | User references |

### Field Value Access

**Indexer Syntax (Primary):**
```powershell
$title = $item["Title"]
$customField = $item["CustomColumnName"]
```

**FieldValues Collection:**
```powershell
$title = $item.FieldValues["Title"]
$allFields = $item.FieldValues  # Hashtable
```

**Internal Name vs. Display Name:**
```powershell
# Get internal names
$list = Get-PnPList -Identity "Documents"
Get-PnPField -List $list | Select-Object Title, InternalName
```

### Document Library Fields

| Field | Description |
|-------|-------------|
| `FileRef` | Server-relative URL (`/sites/sitename/Documents/file.docx`) |
| `FileLeafRef` | File name with extension (`file.docx`) |
| `FileDirRef` | Parent folder path |
| `File_x0020_Size` | File size in bytes |

### Lookup and User Fields

**Lookup:**
```powershell
$lookupValue = $item["ProjectLookup"]
$lookupValue.LookupId      # Integer ID
$lookupValue.LookupValue   # Display text
```

**Multi-Value Lookup:**
```powershell
$multiLookup = $item["RelatedProjects"]
foreach ($lookup in $multiLookup) {
    $lookup.LookupId
    $lookup.LookupValue
}
```

**User/Person:**
```powershell
$user = $item["AssignedTo"]
$user.LookupId      # User ID
$user.LookupValue   # Display name
$user.Email         # Email (when available)
```

---

## CAML Query Reference

CAML (Collaborative Application Markup Language) is XML-based query syntax for SharePoint list data.

### Structure

```xml
<View>
  <Query>
    <Where>
      <!-- Filter conditions -->
    </Where>
    <OrderBy>
      <!-- Sort specification -->
    </OrderBy>
  </Query>
  <ViewFields>
    <!-- Column selection -->
  </ViewFields>
  <RowLimit>N</RowLimit>
</View>
```

### Comparison Operators

| Operator | Description |
|----------|-------------|
| `<Eq>` | Equal |
| `<Neq>` | Not Equal |
| `<Gt>` / `<Lt>` | Greater Than / Less Than |
| `<Geq>` / `<Leq>` | Greater or Equal / Less or Equal |
| `<IsNull>` / `<IsNotNull>` | Null check |
| `<Contains>` | Substring match |
| `<BeginsWith>` | Prefix match |

### Logical Operators

`<And>` and `<Or>` accept exactly two child elements. Nest for multiple conditions:

```xml
<And>
  <Eq>
    <FieldRef Name='Status' />
    <Value Type='Text'>Active</Value>
  </Eq>
  <And>
    <Gt>
      <FieldRef Name='Priority' />
      <Value Type='Number'>5</Value>
    </Gt>
    <IsNotNull>
      <FieldRef Name='AssignedTo' />
    </IsNotNull>
  </And>
</And>
```

### Field Types

| Type | Usage |
|------|-------|
| `Text` | Single line text, choice fields |
| `Note` | Multiple lines of text |
| `Number` | Numeric (integer or decimal) |
| `Currency` | Currency fields |
| `DateTime` | Date and date/time |
| `Boolean` | Yes/No (values: 0 or 1) |
| `User` | Person or Group |
| `Lookup` | Lookup column |
| `Integer` | Explicit integer |
| `Counter` | Auto-incrementing ID |

### DateTime Queries

**UTC Format:**
```xml
<Value Type='DateTime'>2024-01-15T14:30:00Z</Value>
```

**Date-Only:**
```xml
<Value Type='DateTime' IncludeTimeValue='FALSE'>2024-01-15</Value>
```

**Relative Tokens:**
```xml
<Value Type='DateTime'><Today /></Value>
```
Available: `<Today />`, `<Now />`, `<Month />`, `<Year />`

### OrderBy

```xml
<OrderBy>
  <FieldRef Name='Modified' Ascending='FALSE' />
  <FieldRef Name='Title' Ascending='TRUE' />
</OrderBy>
```

---

## Query Examples

### Basic Equality Filter

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <Eq>
        <FieldRef Name='Status' />
        <Value Type='Text'>Approved</Value>
      </Eq>
    </Where>
  </Query>
</View>
"@

$items = Get-PnPListItem -List "Documents" -Query $query
```

### Date Range

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <And>
        <Geq>
          <FieldRef Name='Modified' />
          <Value Type='DateTime'>2024-01-01T00:00:00Z</Value>
        </Geq>
        <Leq>
          <FieldRef Name='Modified' />
          <Value Type='DateTime'>2024-01-31T23:59:59Z</Value>
        </Leq>
      </And>
    </Where>
  </Query>
</View>
"@

$items = Get-PnPListItem -List "Documents" -Query $query
```

### Multiple Conditions

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <And>
        <Eq>
          <FieldRef Name='Category' />
          <Value Type='Text'>Finance</Value>
        </Eq>
        <Or>
          <Eq>
            <FieldRef Name='Priority' />
            <Value Type='Text'>High</Value>
          </Eq>
          <Eq>
            <FieldRef Name='Priority' />
            <Value Type='Text'>Critical</Value>
          </Eq>
        </Or>
      </And>
    </Where>
  </Query>
</View>
"@

$items = Get-PnPListItem -List "Tasks" -Query $query
```

### Sort and Limit

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <Gt>
        <FieldRef Name='Amount' />
        <Value Type='Number'>1000</Value>
      </Gt>
    </Where>
    <OrderBy>
      <FieldRef Name='Amount' Ascending='FALSE' />
    </OrderBy>
  </Query>
  <RowLimit>10</RowLimit>
</View>
"@

$items = Get-PnPListItem -List "Expenses" -Query $query
```

### Null Check

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <IsNull>
        <FieldRef Name='ApprovalDate' />
      </IsNull>
    </Where>
  </Query>
</View>
"@

$items = Get-PnPListItem -List "Requests" -Query $query
```

### Substring Match

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <Contains>
        <FieldRef Name='Title' />
        <Value Type='Text'>Budget</Value>
      </Contains>
    </Where>
  </Query>
</View>
"@

$items = Get-PnPListItem -List "Documents" -Query $query
```

### Files Only (Exclude Folders)

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <Eq>
        <FieldRef Name='FSObjType' />
        <Value Type='Integer'>0</Value>
      </Eq>
    </Where>
  </Query>
</View>
"@

$files = Get-PnPListItem -List "Documents" -Query $query
```

### Items Modified Today

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <Geq>
        <FieldRef Name='Modified' />
        <Value Type='DateTime'>
          <Today />
        </Value>
      </Geq>
    </Where>
  </Query>
</View>
"@

$items = Get-PnPListItem -List "Documents" -Query $query
```

### Query by User ID

```powershell
$query = @"
<View>
  <Query>
    <Where>
      <Eq>
        <FieldRef Name='AssignedTo' LookupId='TRUE' />
        <Value Type='Integer'>15</Value>
      </Eq>
    </Where>
  </Query>
</View>
"@

$items = Get-PnPListItem -List "Tasks" -Query $query
```

---

## Other Essential Cmdlets

### Get-PnPFolder

```powershell
Get-PnPFolder -Url "/sites/sitename/Shared Documents/FolderName"
```

### Get-PnPFile

```powershell
# Metadata
Get-PnPFile -Url "/sites/sitename/Shared Documents/file.docx"

# Download
Get-PnPFile -Url "/sites/sitename/Shared Documents/file.docx" -Path "C:\LocalFolder" -AsFile
```

### Add-PnPFile

```powershell
Add-PnPFile -Path "C:\LocalFolder\document.docx" -Folder "Shared Documents/SubFolder"
Add-PnPFile -Path "C:\file.pdf" -Folder "Documents" -Values @{Title="Custom Title"; Category="Report"}
```

### Remove-PnPFile

```powershell
Remove-PnPFile -ServerRelativeUrl "/sites/sitename/Shared Documents/file.docx" -Force
Remove-PnPFile -ServerRelativeUrl "/path/to/file.docx" -Recycle
```

### Copy-PnPFile / Move-PnPFile

```powershell
Copy-PnPFile -SourceUrl "Documents/source.docx" -TargetUrl "Archive/source.docx"
Move-PnPFile -SourceUrl "Documents/file.docx" -TargetUrl "NewLibrary/file.docx" -Force
```

### Add-PnPFolder / Remove-PnPFolder

```powershell
Add-PnPFolder -Name "NewFolder" -Folder "Shared Documents"
Add-PnPFolder -Name "Parent/Child/Grandchild" -Folder "Documents"  # Nested structure
Remove-PnPFolder -Name "FolderName" -Folder "Shared Documents" -Force
```

### Set-PnPListItem

```powershell
Set-PnPListItem -List "Documents" -Identity 5 -Values @{Title="Updated Title"; Status="Approved"}
```

### Batch Operations

```powershell
$batch = New-PnPBatch

$items = Get-PnPListItem -List "Documents" -Fields "Title"
foreach ($item in $items) {
    Set-PnPListItem -List "Documents" -Identity $item.Id -Values @{
        ProcessedDate = (Get-Date)
    } -Batch $batch
}

Invoke-PnPBatch -Batch $batch
```

---

## Performance Optimization

### Field Projection

```powershell
# Without projection: ~5KB per item
$items = Get-PnPListItem -List "Documents"

# With projection: ~1KB per item (80% reduction)
$items = Get-PnPListItem -List "Documents" -Fields "Title","Modified","FileLeafRef"
```

### Server-Side vs. Client-Side Filtering

**Server-side (CAML):** SharePoint filters before transmission.

```powershell
$query = "<View><Query><Where><Eq><FieldRef Name='Status' /><Value Type='Text'>Approved</Value></Eq></Where></Query></View>"
$items = Get-PnPListItem -List "Documents" -Query $query
```

**Client-side (PowerShell):** All items transmitted, filtered locally.

```powershell
$items = Get-PnPListItem -List "Documents"
$filtered = $items | Where-Object { $_["Status"] -eq "Approved" }
```

Always prefer CAML queries for filtering.

### List View Threshold

SharePoint Online enforces a **5000-item threshold** for non-indexed columns.

**Mitigation:**
1. Index columns used in WHERE clauses
2. Use time-based filtering to reduce result sets
3. Use pagination with `-PageSize`

---

## Common Patterns

### Pipeline Processing

```powershell
Get-PnPListItem -List "Documents" -Fields "Title","FileLeafRef","Modified","Author" |
    Where-Object { $_.FileSystemObjectType -eq "File" } |
    Sort-Object { $_["Modified"] } -Descending |
    Select-Object -First 10 |
    ForEach-Object {
        [PSCustomObject]@{
            FileName = $_["FileLeafRef"]
            Title = $_["Title"]
            Modified = $_["Modified"]
            ItemId = $_.Id
        }
    }
```

### Error Handling

```powershell
try {
    $items = Get-PnPListItem -List "Documents" -Query $query -ErrorAction Stop
    Write-Host "Retrieved $($items.Count) items"
}
catch [System.Net.WebException] {
    Write-Error "Network error: $_"
}
catch {
    Write-Error "SharePoint error: $($_.Exception.Message)"
}
```

### CSV Export

```powershell
$items = Get-PnPListItem -List "Documents" -Fields "Title","FileLeafRef","Modified","Author"

$export = $items | ForEach-Object {
    [PSCustomObject]@{
        FileName = $_["FileLeafRef"]
        Title = $_["Title"]
        Modified = $_["Modified"]
        Author = $_["Author"].LookupValue
        ItemId = $_.Id
    }
}

$export | Export-Csv -Path "C:\Export\DocumentList.csv" -NoTypeInformation -Encoding UTF8
```

### Download Files from Query Results

```powershell
$items = Get-PnPListItem -List "Documents" -Fields "FileRef","FileLeafRef"

foreach ($item in $items) {
    if ($item.FileSystemObjectType -eq "File") {
        Get-PnPFile -Url $item["FileRef"] -Path "C:\Downloads" -FileName $item["FileLeafRef"] -AsFile
    }
}
```

---

## URL Path Format

SharePoint PnP cmdlets require **server-relative URLs**: `/sites/sitename/library/file.docx`

---

## Permissions

File operations require **Contribute** or higher permission levels on target libraries.