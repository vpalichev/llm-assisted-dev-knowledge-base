## Core Terminology

**PnP.PowerShell Module**: SharePoint Patterns and Practices PowerShell module providing cmdlet wrappers for SharePoint Client-Side Object Model (CSOM) and REST API operations.

**CSOM (Client-Side Object Model)**: Microsoft's .NET library (`Microsoft.SharePoint.Client.dll`) providing programmatic SharePoint access through strongly-typed objects and methods.

**Server-Relative URL**: Absolute path from SharePoint server root (e.g., `/sites/ProjectSite/Shared Documents/File.xlsx`).

**Site-Relative URL**: Path relative to current site collection root (e.g., `Shared Documents/File.xlsx`).

**ClientContext**: CSOM connection object maintaining authentication state and request batching. Retrieved via `Get-PnPContext`.

---

## Object Hierarchy

```
PnPConnection (Session)
└── Web (Site)
    └── List (Document Library)
        ├── Folder
        │   ├── Folder (recursive)
        │   └── File
        │       ├── FileVersion
        │       └── ListItem (metadata container)
        └── ListItem
            └── File (reference)
```

---

## 1. Connection Object

### Type

`PnP.PowerShell.Commands.Base.PnPConnection`

### Establishment

powershell

```powershell
Connect-PnPOnline -Url <String> [-Interactive | -Credentials <PSCredential>] [-ReturnConnection]
```

**Parameters**:

- `-Url`: SharePoint site URL (required)
- `-Interactive`: Browser-based authentication (Modern Auth/MFA)
- `-Credentials`: Legacy credential object
- `-ReturnConnection`: Returns connection object for multi-site scenarios

**Context Retrieval**:

powershell

```powershell
$ctx = Get-PnPContext
```

Returns: `Microsoft.SharePoint.Client.ClientContext`

---

## 2. Web Object

### Type

`Microsoft.SharePoint.Client.Web`

### Retrieval

powershell

```powershell
Get-PnPWeb [-Identity <WebPipeBind>] [-Includes <String[]>]
```

**Properties**:

- `Title`: Site title
- `Url`: Absolute URL
- `ServerRelativeUrl`: Server-relative path
- `Lists`: Collection of List objects

---

## 3. List Object (Document Library)

### Type

`Microsoft.SharePoint.Client.List`

### Retrieval

powershell

```powershell
Get-PnPList [-Identity <ListPipeBind>] [-Includes <String[]>] [-ThrowExceptionIfListNotFound]
```

**Identity Parameter Types**:

- Title: `"Documents"`, `"Shared Documents"`
- GUID: `{12345678-1234-1234-1234-123456789012}`
- Server-relative URL: `/sites/Site/Lists/Documents`

**Key Properties**:

- `Title`: Display name
- `RootFolder`: Top-level Folder object
- `ItemCount`: Total items (files + folders)
- `EnableVersioning`: Boolean versioning state
- `MajorVersionLimit`: Maximum major versions (0 = unlimited)
- `EnableMinorVersions`: Draft version support
- `BaseType`: Enum (`DocumentLibrary = 1`)

**Version Configuration Query**:

powershell

```powershell
$list = Get-PnPList -Identity "Documents"
[PSCustomObject]@{
    Library = $list.Title
    VersioningEnabled = $list.EnableVersioning
    MajorVersionLimit = $list.MajorVersionLimit
    MinorVersionsEnabled = $list.EnableMinorVersions
    MajorWithMinorVersionsLimit = $list.MajorWithMinorVersionsLimit
}
```

---

## 4. Folder Object

### Type

`Microsoft.SharePoint.Client.Folder`

### Retrieval

powershell

```powershell
Get-PnPFolder -Url <String> [-Includes <String[]>]
Get-PnPFolderItem -FolderSiteRelativeUrl <String> [-ItemType <String>] [-Recursive]
```

**Key Properties**:

- `Name`: Folder name (leaf component)
- `ServerRelativeUrl`: Full server-relative path
- `ItemCount`: Direct child count (non-recursive)
- `TimeCreated`: Creation timestamp
- `TimeLastModified`: Last modification timestamp

**Enumeration Pattern**:

powershell

```powershell
# Non-recursive listing
$items = Get-PnPFolderItem -FolderSiteRelativeUrl "Shared Documents/Projects"

$folders = $items | Where-Object { $_.TypedObject.ToString() -eq "Microsoft.SharePoint.Client.Folder" }
$files = $items | Where-Object { $_.TypedObject.ToString() -eq "Microsoft.SharePoint.Client.File" }
```

**Recursive Enumeration**:

powershell

```powershell
function Get-PnPFolderRecursive {
    param([string]$FolderUrl)
    
    $items = Get-PnPFolderItem -FolderSiteRelativeUrl $FolderUrl
    
    foreach ($item in $items) {
        if ($item.TypedObject.ToString() -eq "Microsoft.SharePoint.Client.Folder") {
            $item  # Emit folder
            Get-PnPFolderRecursive -FolderUrl $item.ServerRelativeUrl  # Recurse
        } else {
            $item  # Emit file
        }
    }
}
```

**Folder Operations**:

powershell

```powershell
# Create folder
Add-PnPFolder -Name <String> -Folder <String>

# Move folder
Move-PnPFolder -Folder <String> -TargetFolder <String>

# Remove folder (recursive deletion)
Remove-PnPFolder -Name <String> -Folder <String> [-Recycle] [-Force]
```

---

## 5. File Object

### Type

`Microsoft.SharePoint.Client.File`

### Retrieval

powershell

```powershell
Get-PnPFile -Url <String> [-AsFile | -AsString | -AsMemoryStream | -AsListItem] [-Path <String>] [-Force]
```

**Key Properties**:

- `Name`: Filename with extension
- `ServerRelativeUrl`: Full server-relative path
- `Length`: File size (bytes)
- `TimeCreated`: Upload timestamp
- `TimeLastModified`: Last modification timestamp
- `MajorVersion`: Current major version number
- `MinorVersion`: Current minor version number
- `CheckOutType`: Enum (`None = 0`, `Online = 1`, `Offline = 2`)
- `CheckedOutByUser`: User object (if checked out)
- `Exists`: Boolean existence check

**Download Methods**:

|Method|Return Type|Use Case|
|---|---|---|
|`-AsFile`|None (side effect)|Save to disk|
|`-AsString`|String|Text file processing|
|`-AsMemoryStream`|System.IO.MemoryStream|Binary in-memory processing|
|`-AsListItem`|Microsoft.SharePoint.Client.ListItem|Metadata access|

**Upload Pattern**:

powershell

```powershell
Add-PnPFile -Path <String> -Folder <String> [-FileName <String>] [-Checkout] [-CheckInComment <String>] [-Publish] [-PublishComment <String>] [-ContentType <ContentTypePipeBind>]
```

**Upload with Metadata**:

powershell

```powershell
$file = Add-PnPFile -Path "C:\Local\Report.xlsx" -Folder "Shared Documents/Reports"
Set-PnPListItem -List "Documents" -Identity $file.ListItemAllFields.Id -Values @{
    "ProjectCode" = "PRJ-001"
    "Status" = "Draft"
}
```

**File Operations**:

powershell

```powershell
# Copy
Copy-PnPFile -SourceUrl <String> -TargetUrl <String> [-OverwriteIfAlreadyExists] [-Force]

# Move
Move-PnPFile -ServerRelativeUrl <String> -TargetUrl <String> [-OverwriteIfAlreadyExists] [-Force]

# Remove
Remove-PnPFile -ServerRelativeUrl <String> [-Recycle] [-Force]

# Rename (move to same folder with new name)
Move-PnPFile -ServerRelativeUrl "/sites/Site/Shared Documents/Old.xlsx" -TargetUrl "/sites/Site/Shared Documents/New.xlsx" -Force
```

**Check-in/Check-out**:

powershell

```powershell
Set-PnPFileCheckedOut -Url <String>
Set-PnPFileCheckedIn -Url <String> [-CheckInType <CheckInType>] [-Comment <String>]
```

**CheckInType Enum**: `MinorCheckIn = 0`, `MajorCheckIn = 1`, `OverwriteCheckIn = 2`

---

## 6. FileVersion Object

### Type

`Microsoft.SharePoint.Client.FileVersion`

### Retrieval

powershell

```powershell
Get-PnPFileVersion -Url <String> [-Identity <FileVersionPipeBind>] [-Path <String>]
```

**Key Properties**:

- `ID`: Unique version identifier (Int32)
- `VersionLabel`: Human-readable version (e.g., `"3.0"`, `"2.1"`)
- `IsCurrentVersion`: Boolean current version flag
- `Created`: Version creation timestamp
- `Size`: Version file size (bytes)
- `Url`: Version-specific server-relative URL (format: `/_vti_history/{ID}/path`)
- `CheckInComment`: Check-in comment text
- `CreatedBy`: User object

**Version Download (CSOM Method)**:

powershell

```powershell
$versions = Get-PnPFileVersion -Url "Shared Documents/File.xlsx"
$ver = $versions[0]

$ctx = Get-PnPContext
$stream = $ver.OpenBinaryStream()
$ctx.ExecuteQuery()

$fileStream = [System.IO.File]::Create("C:\Output\File_v$($ver.VersionLabel).xlsx")
$stream.Value.CopyTo($fileStream)
$fileStream.Close()
$stream.Value.Dispose()
```

**Version Deletion**:

powershell

```powershell
$ver.DeleteObject()
$ctx.ExecuteQuery()
```

---

## 7. ListItem Object (Metadata Container)

### Type

`Microsoft.SharePoint.Client.ListItem`

### Retrieval

powershell

```powershell
Get-PnPListItem -List <ListPipeBind> [-Id <Int32>] [-UniqueId <Guid>] [-Query <String>] [-Fields <String[]>] [-PageSize <Int32>]
```

**Key Properties**:

- `Id`: Integer list item ID
- `FileSystemObjectType`: Enum (`File = 0`, `Folder = 1`)
- `File`: Reference to associated File object
- `Folder`: Reference to associated Folder object
- `FieldValues`: Dictionary of metadata values

**Standard Fields**:

- `FileLeafRef`: Filename
- `FileRef`: Server-relative URL
- `File_x0020_Size`: File size (bytes, encoded space)
- `Modified`: Last modified timestamp
- `Created`: Creation timestamp
- `Author`: Creator (User lookup field)
- `Editor`: Last modifier (User lookup field)

**Field Value Access**:

powershell

```powershell
$item = Get-PnPListItem -List "Documents" -Id 123

# Method 1: Indexer
$item["Title"]
$item["ProjectCode"]

# Method 2: FieldValues dictionary
$item.FieldValues["Title"]
$item.FieldValues.Keys  # List all field internal names

# Lookup field access
$item["Author"].LookupValue  # User display name
$item["Author"].LookupId     # User ID
```

**CAML Query Pattern**:

powershell

```powershell
$query = @"
<View>
    <Query>
        <Where>
            <And>
                <Eq>
                    <FieldRef Name='Status'/>
                    <Value Type='Choice'>Active</Value>
                </Eq>
                <Geq>
                    <FieldRef Name='Modified'/>
                    <Value Type='DateTime' IncludeTimeValue='FALSE'>
                        <Today OffsetDays='-30'/>
                    </Value>
                </Geq>
            </And>
        </Where>
    </Query>
    <ViewFields>
        <FieldRef Name='FileLeafRef'/>
        <FieldRef Name='FileRef'/>
        <FieldRef Name='Modified'/>
        <FieldRef Name='ProjectCode'/>
    </ViewFields>
    <RowLimit>1000</RowLimit>
</View>
"@

$items = Get-PnPListItem -List "Documents" -Query $query
```

**Metadata Update**:

powershell

```powershell
Set-PnPListItem -List <ListPipeBind> -Identity <ListItemPipeBind> -Values <Hashtable> [-UpdateType <UpdateType>]
```

**UpdateType Enum**:

- `Update = 0`: Regular update (increments version)
- `SystemUpdate = 1`: Silent update (no version increment, no workflow trigger)
- `UpdateOverwriteVersion = 2`: Update without version increment (visible in audit)

powershell

```powershell
# Regular update (creates new version)
Set-PnPListItem -List "Documents" -Identity 123 -Values @{
    "Status" = "Approved"
    "ReviewDate" = (Get-Date)
}

# System update (no version change)
Set-PnPListItem -List "Documents" -Identity 123 -Values @{
    "InternalFlag" = "Processed"
} -UpdateType SystemUpdate
```

---

## 8. Batch Operations

### Batch Context

powershell

```powershell
$batch = New-PnPBatch
```

**Batch-Compatible Cmdlets**:

powershell

```powershell
# Add files to batch
Add-PnPFile -Path "C:\Files\File1.xlsx" -Folder "Documents" -Batch $batch
Add-PnPFile -Path "C:\Files\File2.xlsx" -Folder "Documents" -Batch $batch

# Update metadata in batch
Set-PnPListItem -List "Documents" -Identity 123 -Values @{"Status"="Done"} -Batch $batch
Set-PnPListItem -List "Documents" -Identity 124 -Values @{"Status"="Done"} -Batch $batch

# Execute all queued operations
Invoke-PnPBatch -Batch $batch
```

**Batch Benefits**:

- Single HTTP request for multiple operations
- Reduced latency (critical for high-latency connections)
- Transactional semantics (all-or-nothing execution)

---

## 9. Common Processing Patterns

### Pattern A: Enumerate All Files in Library

powershell

```powershell
function Get-AllFiles {
    param([string]$FolderUrl)
    
    $items = Get-PnPFolderItem -FolderSiteRelativeUrl $FolderUrl
    
    foreach ($item in $items) {
        $typeName = $item.TypedObject.ToString()
        
        if ($typeName -eq "Microsoft.SharePoint.Client.File") {
            [PSCustomObject]@{
                Name = $item.Name
                ServerRelativeUrl = $item.ServerRelativeUrl
                Size = $item.Length
                Modified = $item.TimeLastModified
                Type = "File"
            }
        } elseif ($typeName -eq "Microsoft.SharePoint.Client.Folder") {
            Get-AllFiles -FolderUrl $item.ServerRelativeUrl
        }
    }
}

$allFiles = Get-AllFiles -FolderUrl "Shared Documents"
```

### Pattern B: Download Library with Metadata

powershell

```powershell
$files = Get-AllFiles -FolderUrl "Shared Documents"

foreach ($file in $files) {
    # Get metadata via ListItem
    $listItem = Get-PnPFile -Url $file.ServerRelativeUrl -AsListItem
    
    # Construct local path
    $relativePath = $file.ServerRelativeUrl -replace "^/sites/[^/]+/Shared Documents/", ""
    $localPath = Join-Path "C:\Mirror" $relativePath
    $localDir = Split-Path $localPath -Parent
    
    # Create directory structure
    if (-not (Test-Path $localDir)) {
        New-Item -ItemType Directory -Path $localDir -Force | Out-Null
    }
    
    # Download file
    Get-PnPFile -Url $file.ServerRelativeUrl -Path $localPath -AsFile -Force
    
    # Set timestamp to match SharePoint
    (Get-Item $localPath).LastWriteTime = $file.Modified
    
    # Export metadata
    $metadata = [PSCustomObject]@{
        FilePath = $localPath
        SharePointUrl = $file.ServerRelativeUrl
        Title = $listItem["Title"]
        Modified = $listItem["Modified"]
        Author = $listItem["Author"].LookupValue
        CustomField1 = $listItem["CustomField1"]
    }
    
    $metadata | Export-Csv "C:\Mirror\metadata.csv" -Append -NoTypeInformation
}
```

### Pattern C: Version History Export

powershell

```powershell
$files = Get-AllFiles -FolderUrl "Shared Documents/Contracts"

foreach ($file in $files) {
    $versions = Get-PnPFileVersion -Url $file.ServerRelativeUrl
    
    foreach ($ver in $versions) {
        $versionDir = "C:\VersionArchive\$($file.Name -replace '\.[^.]+$', '')"
        if (-not (Test-Path $versionDir)) {
            New-Item -ItemType Directory -Path $versionDir -Force | Out-Null
        }
        
        $versionFile = Join-Path $versionDir "$($file.Name)_v$($ver.VersionLabel)"
        
        $ctx = Get-PnPContext
        $stream = $ver.OpenBinaryStream()
        $ctx.ExecuteQuery()
        
        $fs = [System.IO.File]::Create($versionFile)
        $stream.Value.CopyTo($fs)
        $fs.Close()
        $stream.Value.Dispose()
    }
}
```

### Pattern D: Bulk Metadata Update

powershell

```powershell
$items = Get-PnPListItem -List "Documents" -Query "<View><Query><Where><IsNull><FieldRef Name='Status'/></IsNull></Where></Query></View>"

$batch = New-PnPBatch

foreach ($item in $items) {
    Set-PnPListItem -List "Documents" -Identity $item.Id -Values @{
        "Status" = "Pending Review"
    } -Batch $batch
}

Invoke-PnPBatch -Batch $batch
```

---

## 10. Error Handling Patterns

### Standard Try-Catch

powershell

```powershell
try {
    $file = Get-PnPFile -Url "Shared Documents/MissingFile.xlsx" -ThrowExceptionIfFileNotFound -AsFile
} catch [System.Exception] {
    if ($_.Exception.Message -match "File Not Found") {
        Write-Warning "File does not exist"
    } elseif ($_.Exception.Message -match "Access Denied") {
        Write-Error "Insufficient permissions"
    } else {
        throw
    }
}
```

### Retry Logic with Exponential Backoff

powershell

```powershell
function Invoke-WithRetry {
    param(
        [ScriptBlock]$ScriptBlock,
        [int]$MaxAttempts = 3,
        [int]$InitialDelaySeconds = 3
    )
    
    $attempt = 1
    $delay = $InitialDelaySeconds
    
    while ($attempt -le $MaxAttempts) {
        try {
            & $ScriptBlock
            return
        } catch {
            if ($attempt -eq $MaxAttempts) {
                throw
            }
            
            Write-Warning "Attempt $attempt failed: $_. Retrying in $delay seconds..."
            Start-Sleep -Seconds $delay
            
            $attempt++
            $delay *= 2
        }
    }
}

# Usage
Invoke-WithRetry -ScriptBlock {
    Get-PnPFile -Url "Shared Documents/LargeFile.zip" -Path "C:\Downloads" -AsFile
}
```

---

## 11. Windows-Specific Considerations

### Path Handling

powershell

```powershell
# Windows backslash paths require escaping or single quotes
$localPath = 'C:\Files\Document.xlsx'  # Single quotes (verbatim)
$localPath = "C:\\Files\\Document.xlsx"  # Double quotes (escaped)

# SharePoint always uses forward slashes
$spUrl = "Shared Documents/Subfolder/File.xlsx"

# Path construction
$filename = "Report.xlsx"
$localPath = Join-Path "C:\Downloads" $filename  # Produces: C:\Downloads\Report.xlsx
```

### MAX_PATH Limitation (260 characters)

powershell

```powershell
# Enable long paths (Windows 10 1607+, requires admin + reboot)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1

# UNC path prefix bypass
$longPath = "\\?\C:\VeryLongPath\ToSomeDeep\Folder\Structure\File.xlsx"
Get-PnPFile -Url $spUrl -Path $longPath -AsFile
```

### Encoding Considerations

powershell

```powershell
# Set console encoding for non-ASCII characters
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'

# File system encoding is handled by .NET automatically
# PnP cmdlets handle SharePoint URL encoding internally
```

### Timestamp Synchronization

powershell

```powershell
# Windows file timestamps: LastWriteTime, LastAccessTime, CreationTime
$localFile = Get-Item "C:\Downloads\File.xlsx"
$localFile.LastWriteTime = $spFile.TimeLastModified
$localFile.CreationTime = $spFile.TimeCreated
```

---

## 12. Performance Optimization

### Throttling Awareness

SharePoint Online enforces throttling limits:

- **User Throttle**: 2,000 requests per user per 5 minutes
- **Tenant Throttle**: Variable based on tenant size

**Mitigation Strategy**:

powershell

```powershell
# Use batching for bulk operations
$batch = New-PnPBatch -BatchSize 100  # Process 100 items per batch

# Implement delays between batches
Invoke-PnPBatch -Batch $batch
Start-Sleep -Seconds 2
```

### Large Library Enumeration

powershell

```powershell
# Use pagination for libraries with >5,000 items
$pageSize = 500
$position = $null

do {
    if ($position) {
        $items = Get-PnPListItem -List "Documents" -PageSize $pageSize -ListItemCollectionPosition $position
    } else {
        $items = Get-PnPListItem -List "Documents" -PageSize $pageSize
    }
    
    $position = $items.ListItemCollectionPosition
    
    # Process $items
    
} while ($null -ne $position)
```

### Minimal Property Loading

powershell

```powershell
# Load only required properties to reduce payload
$list = Get-PnPList -Identity "Documents" -Includes "RootFolder", "ItemCount"

# Avoid loading entire File object when only URL needed
$folder = Get-PnPFolder -Url "Shared Documents"
$files = $folder.Files
$ctx = Get-PnPContext
$ctx.Load($files, "Include(Name,ServerRelativeUrl,Length)")
$ctx.ExecuteQuery()
```