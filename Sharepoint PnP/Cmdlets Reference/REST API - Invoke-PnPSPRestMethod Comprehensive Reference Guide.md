# Invoke-PnPSPRestMethod: Comprehensive Reference Guide

**Version:** 1.0  
**Date:** 2026-01-21  
**Applies to:** PnP PowerShell (PnP.PowerShell module)

---

## 1. Overview

`Invoke-PnPSPRestMethod` is a PnP PowerShell cmdlet that executes HTTP requests against the SharePoint REST API. It serves as a bridge between PowerShell scripts and SharePoint Online's REST endpoints, handling authentication automatically through the existing PnP connection context.

**Why use it over native `Invoke-RestMethod`?**

The cmdlet abstracts away the complexities of SharePoint authentication. When you use `Connect-PnPOnline`, the session acquires an access token (OAuth2) or authentication cookies. `Invoke-PnPSPRestMethod` automatically injects these credentials into every request, so you never need to manually manage Bearer tokens, cookie containers, or authentication headers.

Additionally, PnP PowerShell implements built-in throttling protection with exponential backoff retry logic, which native `Invoke-RestMethod` does not provide.

---

## 2. Syntax

```powershell
Invoke-PnPSPRestMethod 
    -Url <String>
    [-Method <HttpRequestMethod>]
    [-Content <Object>]
    [-ContentType <String>]
    [-Accept <String>]
    [-Raw]
    [-ResponseHeadersVariable <String>]
    [-Connection <PnPConnection>]
    [-Batch <PnPBatch>]
```

---

## 3. Parameters

### 3.1 -Url (Required)

The REST API endpoint to invoke.

**Accepts two formats:**

|Format|Example|Behavior|
|---|---|---|
|**Relative URL**|`/_api/web/lists`|Appended to the current connection's site URL|
|**Absolute URL**|`https://tenant.sharepoint.com/sites/site/_api/web/lists`|Used as-is (must match authenticated tenant)|

**Relative URLs are preferred** because they work regardless of which site you're connected to and are more portable across environments.

**Important:** When using OData query parameters (`$select`, `$filter`, `$expand`, etc.) in URLs, you must handle PowerShell's variable interpolation:

```powershell
# CORRECT: Use single quotes to prevent $ interpretation
$result = Invoke-PnPSPRestMethod -Url '/_api/web/lists?$select=Id,Title'

# CORRECT: Use backtick to escape $ in double quotes
$result = Invoke-PnPSPRestMethod -Url "/_api/web/lists?`$select=Id,Title"

# WRONG: PowerShell interprets $select as a variable (likely $null)
$result = Invoke-PnPSPRestMethod -Url "/_api/web/lists?$select=Id,Title"
```

### 3.2 -Method

The HTTP method to execute. Defaults to `GET`.

|Value|Use Case|
|---|---|
|`Get`|Retrieve data (default)|
|`Post`|Create items, invoke actions, call methods like `getchanges`|
|`Put`|Full replacement of a resource|
|`Patch`|Partial update of a resource|
|`Merge`|SharePoint-specific partial update (legacy, equivalent to PATCH)|
|`Delete`|Remove a resource|
|`Head`|Retrieve headers only|
|`Options`|Query supported methods|

**Note on MERGE:** SharePoint historically used `MERGE` for partial updates before `PATCH` became standard. Both work, but `PATCH` is preferred for new development.

### 3.3 -Content

The request body to send. Accepts:

|Input Type|Behavior|
|---|---|
|**Hashtable** (`@{}`)|Automatically serialized to JSON|
|**String**|Sent as-is (must be valid JSON if ContentType is application/json)|
|**PSObject**|Serialized to JSON|

**Hashtable example (recommended for simple payloads):**

```powershell
$item = @{
    Title = "New Document"
    Status = "Draft"
}
Invoke-PnPSPRestMethod -Method Post `
    -Url "/_api/web/lists/getbytitle('Documents')/items" `
    -Content $item
```

**JSON string example (for complex or nested structures):**

```powershell
$body = @{
    query = @{
        Add = $true
        Update = $true
        DeleteObject = $true
        Item = $true
        File = $true
        RecursiveAll = $true
    }
} | ConvertTo-Json -Depth 5

Invoke-PnPSPRestMethod -Method Post `
    -Url "/_api/web/lists(guid'$listGuid')/getchanges" `
    -Content $body
```

**Critical:** When using `ConvertTo-Json`, always specify `-Depth` for nested objects. The default depth is 2, which truncates deeper structures silently.

### 3.4 -ContentType

The `Content-Type` HTTP header. Defaults to `application/json`.

|Value|When to Use|
|---|---|
|`application/json`|Default, works for most modern SharePoint REST calls|
|`application/json;odata=verbose`|Legacy format, returns full metadata including `__metadata`|
|`application/json;odata=minimalmetadata`|Returns some metadata (entity types, edit links)|
|`application/json;odata=nometadata`|Minimal response, no OData metadata|
|`application/atom+xml`|Legacy XML format (rarely needed)|

**When you need verbose metadata:**

Some operations require the `__metadata` type information in the request body:

```powershell
$item = @"
{
    "__metadata": { "type": "SP.Data.DocumentsItem" },
    "Title": "Test Document"
}
"@

Invoke-PnPSPRestMethod -Method Post `
    -Url "/_api/web/lists/getbytitle('Documents')/items" `
    -Content $item `
    -ContentType "application/json;odata=verbose"
```

**Finding the correct type name:** Query the list's `ListItemEntityTypeFullName`:

```powershell
$list = Invoke-PnPSPRestMethod -Url "/_api/web/lists/getbytitle('Documents')?`$select=ListItemEntityTypeFullName"
$list.ListItemEntityTypeFullName  # e.g., "SP.Data.DocumentsItem"
```

### 3.5 -Accept

The `Accept` HTTP header controlling response format. Defaults to `application/json;odata=nometadata`.

Changing this affects the verbosity of the response:

```powershell
# Get verbose response with full metadata
$result = Invoke-PnPSPRestMethod -Url "/_api/web" `
    -Accept "application/json;odata=verbose"
```

### 3.6 -Raw

When specified, returns the response as a raw JSON string instead of deserializing to a PSObject.

```powershell
# Returns PSObject (default)
$result = Invoke-PnPSPRestMethod -Url "/_api/web"
$result.Title  # Access as property

# Returns JSON string
$jsonString = Invoke-PnPSPRestMethod -Url "/_api/web" -Raw
$jsonString  # Raw JSON text
```

**Use cases for -Raw:**

- Debugging response structure
- Passing response to external tools expecting JSON
- Avoiding PowerShell's JSON depth limitations during deserialization

### 3.7 -ResponseHeadersVariable

Captures response headers into a variable for inspection. Useful for debugging, pagination tokens, or throttling information.

```powershell
$result = Invoke-PnPSPRestMethod -Url "/_api/web/lists" -ResponseHeadersVariable "headers"

# Access captured headers
$headers["X-SharePointHealthScore"]  # Server load indicator (0-10)
$headers["X-RequestDigest"]          # Form digest for subsequent POST requests
```

**Note:** Provide the variable name as a string without the `$` prefix.

### 3.8 -Connection

Specifies which PnP connection to use. By default, uses the most recent connection established by `Connect-PnPOnline`.

```powershell
# Connect to multiple sites
$conn1 = Connect-PnPOnline -Url "https://tenant.sharepoint.com/sites/Site1" -Interactive -ReturnConnection
$conn2 = Connect-PnPOnline -Url "https://tenant.sharepoint.com/sites/Site2" -Interactive -ReturnConnection

# Query specific site
$lists = Invoke-PnPSPRestMethod -Url "/_api/web/lists" -Connection $conn1
```

### 3.9 -Batch

Adds the request to a batch for combined execution. Batching reduces round trips and can improve performance for multiple operations.

```powershell
$batch = New-PnPBatch -RetainRequests

# Queue multiple requests
Invoke-PnPSPRestMethod -Method Get -Url "/_api/web/lists" -Batch $batch
Invoke-PnPSPRestMethod -Method Get -Url "/_api/web/webs" -Batch $batch

# Execute all at once
$response = Invoke-PnPBatch $batch -Details
```

**Important:** When using batching with `Invoke-PnPSPRestMethod`:

- Use `-RetainRequests` on `New-PnPBatch` if you need to process responses
- Absolute URLs are required (relative URLs don't work in batch mode)
- Each request in the batch is evaluated individually for throttling

---

## 4. Response Handling

### 4.1 Standard Response Structure

SharePoint REST API responses typically follow this pattern:

```powershell
$result = Invoke-PnPSPRestMethod -Url "/_api/web/lists"

# Single item endpoints return the object directly
$result.Title

# Collection endpoints return items in .value array
$result.value | ForEach-Object { $_.Title }
```

**Case sensitivity warning:** PowerShell property access is case-insensitive, but the actual JSON properties are case-sensitive. `$result.value` and `$result.Value` work the same in PowerShell, but when using `-Raw` and parsing manually, case matters.

### 4.2 Pagination

Large result sets are paginated. Check for `odata.nextLink` (or `__next` in verbose mode):

```powershell
$allItems = @()
$url = "/_api/web/lists/getbytitle('LargeList')/items?`$top=1000"

do {
    $result = Invoke-PnPSPRestMethod -Url $url
    $allItems += $result.value
    $url = $result.'odata.nextLink'  # Note: property name contains a dot
} while ($url)
```

### 4.3 Error Handling

Errors throw terminating exceptions. Use try/catch for graceful handling:

```powershell
try {
    $result = Invoke-PnPSPRestMethod -Url "/_api/web/lists/getbytitle('NonExistent')"
}
catch {
    $errorDetails = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorDetails.'odata.error'.message.value)"
    
    # Or access the raw exception
    Write-Host "Status: $($_.Exception.Response.StatusCode)"
}
```

---

## 5. Throttling and Retry Behavior

### 5.1 Built-in Throttling Protection

PnP PowerShell implements automatic retry logic for HTTP 429 (Too Many Requests) responses:

|Retry|Wait Time|
|---|---|
|1|0.5 seconds|
|2|1 second|
|3|2 seconds|
|4|4 seconds|
|5|8 seconds|
|6|16 seconds|
|7|32 seconds|
|8|64 seconds|
|9|128 seconds|
|10|256 seconds|

**Maximum total wait:** ~512 seconds (8.5 minutes) before giving up.

This exponential backoff is automatic — you don't need to implement retry logic yourself. However, it's still important to design efficient queries to minimize throttling in the first place.

### 5.2 Simulating Throttling for Testing

SharePoint provides a test parameter to trigger a mock 429 response:

```powershell
# This will trigger throttling response for testing error handling
Invoke-PnPSPRestMethod -Url "/_api/web/lists?test429=true"
```

### 5.3 Monitoring Server Load

The `X-SharePointHealthScore` header indicates server load (0 = idle, 10 = overloaded):

```powershell
Invoke-PnPSPRestMethod -Url "/_api/web" -ResponseHeadersVariable "headers"
$healthScore = [int]$headers["X-SharePointHealthScore"]

if ($healthScore -gt 5) {
    Write-Warning "Server under load (score: $healthScore), consider slowing down"
    Start-Sleep -Seconds 2
}
```

---

## 6. Common Patterns and Examples

### 6.1 GetChanges API (Change Tracking)

```powershell
$list = Get-PnPList -Identity "Documents"
$startDate = [DateTime]::UtcNow.AddDays(-7)
$changeToken = "1;3;$($list.Id.Guid);$($startDate.Ticks);-1"

$body = @{
    query = @{
        Add = $true
        Update = $true  
        DeleteObject = $true
        Rename = $true
        Move = $true
        Item = $true
        File = $true
        Folder = $true
        RecursiveAll = $true
        ChangeTokenStart = @{
            StringValue = $changeToken
        }
    }
} | ConvertTo-Json -Depth 5

$changes = Invoke-PnPSPRestMethod -Method Post `
    -Url "/_api/web/lists(guid'$($list.Id.Guid)')/getchanges" `
    -Content $body

$changes.value | ForEach-Object {
    [PSCustomObject]@{
        ChangeType = $_.ChangeType
        Time = $_.Time
        UniqueId = $_.UniqueId
        ItemId = $_.ItemId
    }
}
```

### 6.2 Creating List Items

```powershell
# Simple creation (modern endpoint)
$newItem = @{
    Title = "New Item"
    Description = "Created via REST"
}

$result = Invoke-PnPSPRestMethod -Method Post `
    -Url "/_api/web/lists/getbytitle('Tasks')/items" `
    -Content $newItem

Write-Host "Created item with ID: $($result.Id)"
```

### 6.3 Updating List Items

```powershell
$itemId = 42
$updates = @{
    Title = "Updated Title"
    Status = "Completed"
}

# PATCH for partial update
Invoke-PnPSPRestMethod -Method Patch `
    -Url "/_api/web/lists/getbytitle('Tasks')/items($itemId)" `
    -Content $updates
```

**Note:** Some update operations require an `If-Match` header for concurrency control. If you get a 412 Precondition Failed error, you may need to fetch the item's `@odata.etag` first.

### 6.4 Querying with OData Filters

```powershell
# Select specific fields
$items = Invoke-PnPSPRestMethod `
    -Url "/_api/web/lists/getbytitle('Documents')/items?`$select=Id,Title,Modified,Editor/Title&`$expand=Editor"

# Filter results
$recentItems = Invoke-PnPSPRestMethod `
    -Url "/_api/web/lists/getbytitle('Documents')/items?`$filter=Modified gt datetime'2024-01-01T00:00:00Z'"

# Order and limit
$topItems = Invoke-PnPSPRestMethod `
    -Url "/_api/web/lists/getbytitle('Documents')/items?`$orderby=Modified desc&`$top=10"
```

### 6.5 Getting Current Change Token

```powershell
$list = Get-PnPList -Identity "Documents"
$token = Invoke-PnPSPRestMethod -Url "/_api/web/lists(guid'$($list.Id.Guid)')/CurrentChangeToken"
$token.StringValue | Out-File ".\lastToken.txt"
```

### 6.6 File Operations

```powershell
# Get file metadata
$file = Invoke-PnPSPRestMethod `
    -Url "/_api/web/getfilebyserverrelativeurl('/sites/site/Shared Documents/file.docx')"

# Get file properties (list item fields)
$fileItem = Invoke-PnPSPRestMethod `
    -Url "/_api/web/getfilebyserverrelativeurl('/sites/site/Shared Documents/file.docx')/ListItemAllFields"
```

---

## 7. Windows-Specific Considerations

### 7.1 Path Length in URLs

When constructing URLs programmatically from file paths, be aware of encoding requirements:

```powershell
$filePath = "/sites/site/Shared Documents/Very Long Folder Name/Document.docx"
$encodedPath = [System.Uri]::EscapeDataString($filePath)

# Use encoded path in URL
$file = Invoke-PnPSPRestMethod -Url "/_api/web/getfilebyserverrelativeurl('$encodedPath')"
```

### 7.2 Character Encoding in Content

When sending content with special characters:

```powershell
$body = @{
    Title = "Document with émojis 🎉"
} | ConvertTo-Json

# Ensure UTF-8 encoding
Invoke-PnPSPRestMethod -Method Post `
    -Url "/_api/web/lists/getbytitle('Documents')/items" `
    -Content $body `
    -ContentType "application/json; charset=utf-8"
```

### 7.3 PowerShell 5.1 vs 7+ JSON Handling

PowerShell 7+ handles JSON differently than 5.1:

```powershell
# PowerShell 5.1: May have issues with large JSON or deep nesting
# PowerShell 7+: Better JSON handling, use -Depth with ConvertTo-Json

$body = $data | ConvertTo-Json -Depth 10 -Compress
```

---

## 8. Troubleshooting

### 8.1 Common Errors

|Error|Cause|Solution|
|---|---|---|
|401 Unauthorized|Token expired or insufficient permissions|Re-run `Connect-PnPOnline`, check app permissions|
|403 Forbidden|Access denied to resource|Verify user/app has access to the target site/list|
|404 Not Found|Invalid URL or resource doesn't exist|Check URL spelling, verify resource exists|
|400 Bad Request|Malformed request body or invalid parameters|Validate JSON structure, check OData syntax|
|429 Too Many Requests|Throttled|Wait and retry (automatic), reduce request rate|
|500 Internal Server Error|Server-side issue|Check request body, retry later|

### 8.2 Debugging Requests

```powershell
# Capture both response and headers
try {
    $result = Invoke-PnPSPRestMethod -Url "/_api/web/lists" `
        -ResponseHeadersVariable "headers"
    
    Write-Host "Request ID: $($headers['request-id'])"
    Write-Host "Health Score: $($headers['X-SharePointHealthScore'])"
}
catch {
    Write-Host "Error: $($_.Exception.Message)"
    Write-Host "Status: $($_.Exception.Response.StatusCode)"
}
```

### 8.3 Validating JSON Before Sending

```powershell
$body = @{
    query = @{
        Add = $true
        Item = $true
    }
} | ConvertTo-Json -Depth 5

# Validate it's parseable
try {
    $null = $body | ConvertFrom-Json
    Write-Host "JSON is valid"
}
catch {
    Write-Error "Invalid JSON: $_"
}
```

---

## 9. Best Practices

1. **Use relative URLs** for portability across environments and sites.
    
2. **Always use single quotes** for URLs with OData parameters, or escape the `$` character.
    
3. **Specify `-Depth`** when using `ConvertTo-Json` for nested objects.
    
4. **Handle pagination** for any endpoint that might return large datasets.
    
5. **Minimize requests** by using `$select` to retrieve only needed fields and `$expand` to include related data in a single call.
    
6. **Use batching** for multiple independent operations to reduce round trips.
    
7. **Monitor health scores** in high-volume scenarios to proactively slow down before hitting throttling limits.
    
8. **Store and reuse change tokens** for incremental sync operations rather than querying all data repeatedly.
    

---

## 10. Quick Reference Card

```powershell
# GET request (default)
Invoke-PnPSPRestMethod -Url '/_api/web'

# GET with OData query
Invoke-PnPSPRestMethod -Url '/_api/web/lists?$select=Id,Title&$top=10'

# POST with hashtable body
Invoke-PnPSPRestMethod -Method Post -Url '/_api/endpoint' -Content @{Key="Value"}

# POST with JSON string body
Invoke-PnPSPRestMethod -Method Post -Url '/_api/endpoint' -Content '{"Key":"Value"}'

# POST with verbose metadata
Invoke-PnPSPRestMethod -Method Post -Url '/_api/endpoint' -Content $body -ContentType 'application/json;odata=verbose'

# Capture response headers
Invoke-PnPSPRestMethod -Url '/_api/web' -ResponseHeadersVariable 'headers'

# Use specific connection
Invoke-PnPSPRestMethod -Url '/_api/web' -Connection $savedConnection

# Get raw JSON string
Invoke-PnPSPRestMethod -Url '/_api/web' -Raw

# Batch requests
$batch = New-PnPBatch -RetainRequests
Invoke-PnPSPRestMethod -Url 'https://tenant.sharepoint.com/sites/site/_api/web' -Batch $batch
Invoke-PnPBatch $batch -Details
```

---

## Appendix A: HTTP Methods and SharePoint Operations

|Operation|HTTP Method|Example Endpoint|
|---|---|---|
|Read item|GET|`/_api/web/lists/getbytitle('List')/items(1)`|
|Read collection|GET|`/_api/web/lists/getbytitle('List')/items`|
|Create item|POST|`/_api/web/lists/getbytitle('List')/items`|
|Update item (full)|PUT|`/_api/web/lists/getbytitle('List')/items(1)`|
|Update item (partial)|PATCH or MERGE|`/_api/web/lists/getbytitle('List')/items(1)`|
|Delete item|DELETE|`/_api/web/lists/getbytitle('List')/items(1)`|
|Invoke method|POST|`/_api/web/lists/getbytitle('List')/getchanges`|

---

## Appendix B: Useful SharePoint REST Endpoints

|Purpose|Endpoint|
|---|---|
|Current web properties|`/_api/web`|
|Current site properties|`/_api/site`|
|All lists|`/_api/web/lists`|
|List by title|`/_api/web/lists/getbytitle('ListName')`|
|List by GUID|`/_api/web/lists(guid'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx')`|
|List items|`/_api/web/lists/getbytitle('ListName')/items`|
|Current change token|`/_api/web/lists/getbytitle('ListName')/CurrentChangeToken`|
|Get changes|`/_api/web/lists/getbytitle('ListName')/getchanges` (POST)|
|File by path|`/_api/web/getfilebyserverrelativeurl('/path/to/file.docx')`|
|Folder contents|`/_api/web/getfolderbyserverrelativeurl('/path')/files`|
|Current user|`/_api/web/currentuser`|
|Site users|`/_api/web/siteusers`|