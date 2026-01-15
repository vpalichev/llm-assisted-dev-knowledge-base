
```powershell
Get-PnPListItem -List $params.libraryName -PageSize 5000 -Fields "FileLeafRef","FileRef","File_x0020_Size","Created","Modified","Author","Editor"
```

Based on my research, I've compiled comprehensive information about the fields that `Get-PnPListItem` can accept in the `-Fields` parameter.

## How the -Fields Parameter Works

The `-Fields` parameter in `Get-PnPListItem` accepts an array of **internal field names** (not display names). If you don't specify fields, all fields are loaded by default.

**Syntax:**

```powershell
Get-PnPListItem -List "Documents" -Fields "Title","FileRef","Modified"
```

## Complete Reference: SharePoint Internal Field Names

### Common Document Library Fields

Based on the official Microsoft documentation, here are the internal field names you can use:

| Display Name                | Internal Name                | Type          | Description                                |
| --------------------------- | ---------------------------- | ------------- | ------------------------------------------ |
| **File & Path**             |                              |               |                                            |
| Name                        | `FileLeafRef`                | File          | Filename with extension                    |
| URL Path                    | `FileRef`                    | Lookup        | Server-relative URL path                   |
| Path                        | `FileDirRef`                 | Lookup        | Folder path                                |
| Server Relative URL         | `ServerUrl`                  | Computed      | Full server-relative URL                   |
| Encoded Absolute URL        | `EncodedAbsUrl`              | Computed      | Encoded absolute URL                       |
| Base Name                   | `BaseName`                   | Computed      | Filename without extension                 |
| **File Properties**         |                              |               |                                            |
| File Size                   | `File_x0020_Size`            | Lookup        | File size in bytes                         |
| File Size Display           | `FileSizeDisplay`            | Computed      | Formatted file size                        |
| File Type                   | `File_x0020_Type`            | Text          | File extension                             |
| HTML File Type              | `HTML_x0020_File_x0020_Type` | Text/Computed | HTML file type                             |
| Type Icon                   | `DocIcon`                    | Computed      | Document icon                              |
| **Metadata**                |                              |               |                                            |
| ID                          | `ID`                         | Counter       | Unique item ID                             |
| Title                       | `Title`                      | Text          | Item title                                 |
| GUID                        | `GUID`                       | Guid          | Globally unique identifier                 |
| Unique Id                   | `UniqueId`                   | Lookup        | Unique identifier                          |
| **Dates & Users**           |                              |               |                                            |
| Created                     | `Created`                    | DateTime      | Creation date/time                         |
| Created Date                | `Created_x0020_Date`         | Lookup        | Creation date lookup                       |
| Created By                  | `Author`                     | User          | User who created                           |
| Document Created By         | `Created_x0020_By`           | Text          | Creator name as text                       |
| Modified                    | `Modified`                   | DateTime      | Last modified date/time                    |
| Last Modified               | `Last_x0020_Modified`        | Lookup        | Last modified lookup                       |
| Modified By                 | `Editor`                     | User          | User who last modified                     |
| Document Modified By        | `Modified_x0020_By`          | Text          | Modifier name as text                      |
| **Check-out Status**        |                              |               |                                            |
| Checked Out To              | `CheckoutUser`               | User          | User with checkout                         |
| Checked Out Title           | `CheckedOutTitle`            | Lookup        | Checkout user title                        |
| Link Checked Out Title      | `LinkCheckedOutTitle`        | Computed      | Checkout user link                         |
| Check In Comment            | `_CheckinComment`            | Lookup        | Latest check-in comment                    |
| Checked Out User Id         | `CheckedOutUserId`           | Lookup        | User ID with checkout                      |
| Is Checked out to Local     | `IsCheckedoutToLocal`        | Lookup        | Local checkout status                      |
| **Content Management**      |                              |               |                                            |
| Content Type                | `ContentType`                | Text          | Content type name                          |
| Content Type ID             | `ContentTypeId`              | ContentTypeId | Content type identifier                    |
| Approval Status             | `_ModerationStatus`          | ModStat       | Content approval status                    |
| Approver Comments           | `_ModerationComments`        | Note          | Approval comments                          |
| **Versioning**              |                              |               |                                            |
| Version                     | `_UIVersionString`           | Text          | Version string (e.g., "1.0")               |
| UI Version                  | `_UIVersion`                 | Integer       | Version number                             |
| owshiddenversion            | `owshiddenversion`           | Integer       | Hidden version number                      |
| Is Current Version          | `_IsCurrentVersion`          | Boolean       | Current version flag                       |
| **Other Properties**        |                              |               |                                            |
| Item Type                   | `FSObjType`                  | Lookup        | File system object type (0=file, 1=folder) |
| Attachments                 | `Attachments`                | Attachments   | Has attachments flag                       |
| Level                       | `_Level`                     | Integer       | Hierarchy level                            |
| Order                       | `Order`                      | Number        | Sort order                                 |
| Instance ID                 | `InstanceID`                 | Integer       | Instance identifier                        |
| Scope Id                    | `ScopeId`                    | Lookup        | Scope identifier                           |
| Prog Id                     | `ProgId`                     | Lookup        | Program ID                                 |
| Property Bag                | `MetaInfo`                   | Lookup        | Metadata property bag                      |
| Effective Permissions Mask  | `PermMask`                   | Computed      | User permissions                           |
| Virus Status                | `VirusStatus`                | Lookup        | Virus scan status                          |
| **Workflow**                |                              |               |                                            |
| Workflow Version            | `WorkflowVersion`            | Integer       | Workflow version                           |
| Workflow Instance ID        | `WorkflowInstanceID`         | Guid          | Workflow instance                          |
| **Copy & Source**           |                              |               |                                            |
| Copy Source                 | `_CopySource`                | Text          | Copy source location                       |
| Has Copy Destinations       | `_HasCopyDestinations`       | Boolean       | Has copy destinations                      |
| Source Url                  | `_SourceUrl`                 | Text          | Original source URL                        |
| **Computed/Display Fields** |                              |               |                                            |
| Edit                        | `Edit`                       | Computed      | Edit link                                  |
| Select                      | `SelectTitle`                | Computed      | Selection checkbox                         |
| Select Filename             | `SelectFilename`             | Computed      | Filename selector                          |
| Link Filename               | `LinkFilename`               | Computed      | Filename with link                         |
| Link Filename No Menu       | `LinkFilenameNoMenu`         | Computed      | Filename link without menu                 |

### Custom List Fields (Additional)

|Display Name|Internal Name|Type|
|---|---|---|
|Title|`LinkTitle`|Computed|
|Title|`LinkTitleNoMenu`|Computed|

## How to Discover Fields in Your Specific List

Since SharePoint lists can have custom columns, use this PowerShell command to discover all available fields:

```powershell
# Connect to your site
Connect-PnPOnline -Url "https://yourtenant.sharepoint.com/sites/yoursite" -Interactive

# Get all fields from a specific list
$fields = Get-PnPField -List "YourLibraryName"

# Display internal names and types
$fields | Select-Object Title, InternalName, TypeAsString, Hidden | 
    Where-Object { $_.Hidden -eq $false } | 
    Format-Table -AutoSize

# Export to CSV for reference
$fields | Select-Object Title, InternalName, TypeAsString, Required, ReadOnlyField | 
    Export-Csv -Path "fields.csv" -NoTypeInformation
```

## Important Notes

1. **Internal Names vs Display Names**: Always use internal names (case-sensitive) in the `-Fields` parameter
2. **Spaces in Names**: Spaces are encoded as `_x0020_` (e.g., "File Size" → `File_x0020_Size`)
3. **Performance**: Only request the fields you need to improve performance
4. **Access via FieldValues**: Access field data using `$item.FieldValues.InternalName`
5. **Cannot Use with -Query**: When using `-Query` parameter, specify fields in CAML `<ViewFields>` instead





Let me explain the main field types you'll encounter:

## SharePoint Field Types

### **Lookup**

- **Purpose**: References data from another list or calculates values from the current item
- **Characteristics**: Read-only for most fields, computed by SharePoint
- **Examples**:
    - `FileRef` - URL path (looks up file location)
    - `File_x0020_Size` - File size (looks up from file system)
    - `UniqueId` - Unique identifier (system-generated)
    - `CheckedOutUserId` - User who checked out (references user list)

### **Computed**

- **Purpose**: Dynamically calculated/generated values, often for display purposes
- **Characteristics**: Always read-only, calculated on-the-fly
- **Examples**:
    - `DocIcon` - Document type icon
    - `ServerUrl` - Full server-relative URL
    - `FileSizeDisplay` - Formatted file size (e.g., "2.5 MB")
    - `LinkFilename` - Clickable filename link
    - `Edit` - Edit button/link in list views

### **Text**

- **Purpose**: Simple text strings
- **Characteristics**: Editable, stores plain text (up to 255 characters)
- **Examples**:
    - `Title` - Item title
    - `File_x0020_Type` - File extension (e.g., "docx", "pdf")
    - `_UIVersionString` - Version as text (e.g., "1.0")
    - `Created_x0020_By` - Creator name as text

## Other Common Field Types

|Type|Description|Examples|
|---|---|---|
|**DateTime**|Date and time values|`Created`, `Modified`|
|**User**|Reference to SharePoint user|`Author`, `Editor`, `CheckoutUser`|
|**Integer**|Whole numbers|`_UIVersion`, `_Level`|
|**Number**|Decimal numbers|`Order`|
|**Boolean**|True/False values|`_IsCurrentVersion`, `IsCheckedoutToLocal`|
|**Counter**|Auto-incrementing ID|`ID`|
|**Guid**|Globally unique identifier|`GUID`, `WorkflowInstanceID`|
|**File**|File reference|`FileLeafRef`|
|**Note**|Multi-line text|`_ModerationComments`|
|**Attachments**|Has attachments flag|`Attachments`|
|**Choice**|Dropdown selection|Custom fields|
|**ModStat**|Moderation status|`_ModerationStatus` (Approved/Pending/Rejected)|
|**ContentTypeId**|Content type identifier|`ContentTypeId`|

## Practical Implications

### When Using Get-PnPListItem:

```powershell
$items = Get-PnPListItem -List "Documents" -Fields "FileRef","Title","File_x0020_Size"

foreach ($item in $items) {
    # Lookup type - read-only, system-calculated
    $path = $item.FieldValues.FileRef  # e.g., "/sites/mysite/Documents/file.pdf"
    
    # Text type - editable by users
    $title = $item.FieldValues.Title  # e.g., "My Document"
    
    # Lookup type - system-calculated from file system
    $size = $item.FieldValues.File_x0020_Size  # e.g., 2048576 (bytes)
}
```

### Why It Matters:

1. **Read-Only vs Editable**:
    
    - **Text** fields can be updated with `Set-PnPListItem`
    - **Lookup/Computed** fields are read-only, calculated by SharePoint
2. **Performance**:
    
    - **Computed** fields are calculated on-demand (may be slower)
    - **Lookup** fields reference stored data
    - **Text** fields are directly stored
3. **Data Access**:
    
    - **User** types require `.LookupValue` to get the name: `$item.FieldValues.Author.LookupValue`
    - **Lookup** user fields work similarly
    - **Text/DateTime/Integer** can be accessed directly

## Example from Your Codebase

In [export-sharepoint-files-to-sqlite.ps1:65-85](vscode-webview://16rkfkgf77vh70hu4jofnr27e51ln002l5acl0go7ha0520h7nt4/src/export-sharepoint-files-to-sqlite.ps1#L65-L85):

```powershell
Get-PnPListItem -List $params.libraryName -PageSize 500 `
    -Fields "FileLeafRef","FileRef","File_x0020_Size","Created","Modified","Author","Editor"
```

Field types being used:

- `FileLeafRef` - **File** type (filename)
- `FileRef` - **Lookup** type (file path)
- `File_x0020_Size` - **Lookup** type (file size in bytes)
- `Created`, `Modified` - **DateTime** type (dates)
- `Author`, `Editor` - **User** type (people)

The User fields require special handling:

```powershell
created_by = if ($_.FieldValues.Author) { $_.FieldValues.Author.LookupValue } else { "" }
```