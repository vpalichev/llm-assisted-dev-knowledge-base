# SharePoint Online special character handling: why CSOM fails and Get-PnPFile succeeds

Files with `#` and `%` characters in SharePoint Online filenames **are fully supported** since Microsoft enabled tenant-level support in 2017—but they create critical ambiguity for legacy APIs that interpret these characters as URL encoding signals. The root cause of download failures lies in how different API methods handle the fundamental difference between **decoded paths** (actual filenames) and **encoded URLs** (percent-encoded representations).

Microsoft introduced the **ResourcePath API** to resolve this ambiguity, and `Get-PnPFile` internally uses this modern API while raw CSOM methods like `GetFileByServerRelativeUrl` cannot distinguish between a literal `#` character and a URL fragment delimiter. This explains why identical download operations succeed with one method and fail with another.

## How # and % characters exist despite documented "restrictions"

The characters `#` and `%` were historically blocked in SharePoint filenames but **are no longer restricted** in SharePoint Online. Microsoft officially enabled support in April 2017 through a tenant-level configuration that is now enabled by default for all tenants created after June 2017.

The tenant setting `SpecialCharactersStateInFileFolderNames` controls this behavior and can be verified via PowerShell:

```powershell
Get-SPOTenant | Format-List SpecialCharactersStateInFileFolderNames
# Allowed = support enabled, Disallowed = legacy restrictions
```

All upload methods—browser drag-and-drop, OneDrive sync client, REST API, and CSOM—support `#` and `%` characters when the tenant setting is enabled. The confusion arises because **different API layers process these characters differently**, not because upload restrictions vary by method.

The only permanently blocked characters in SharePoint Online are: `" * : < > ? / \ |` plus leading/trailing spaces and reserved names like `CON`, `PRN`, and files starting with `~$`.

## SharePoint stores decoded filenames but serves encoded URLs

SharePoint's internal database stores the **literal, decoded filename**—a file named `Report#2024.docx` is stored exactly as `Report#2024.docx`. However, when SharePoint constructs URLs for browser navigation or API responses, it percent-encodes special characters according to RFC 3986 standards:

|Actual filename|URL representation|
|---|---|
|`File#1.docx`|`File%231.docx`|
|`Q1 50%.xlsx`|`Q1%2050%25.xlsx`|
|`Test%20File.pdf`|`Test%2520File.pdf`|

This creates the **core ambiguity problem**: when a string-based API receives the path `/Documents/Report%232024.docx`, it cannot determine whether `%23` represents an encoded `#` (meaning the actual filename is `Report#2024.docx`) or whether the literal filename contains `%23` (actual filename is `Report%232024.docx`).

Legacy APIs like `GetFileByServerRelativeUrl` assume any `%` or `#` indicates URL encoding. After Microsoft added support for these literal characters in filenames, this automatic interpretation became fundamentally broken for ambiguous cases.

## The critical difference between CSOM and Get-PnPFile methods

The `Get-PnPFile` cmdlet internally uses SharePoint's modern **ResourcePath API** which was designed specifically to eliminate the encoding ambiguity. When you examine the PnP PowerShell source code, the difference becomes clear:

**Get-PnPFile implementation:**

```csharp
// Uses ResourcePath for SharePoint Online
var file = web.GetFileByServerRelativePath(ResourcePath.FromDecodedUrl(serverRelativeUrl));
```

**Direct CSOM (legacy approach):**

```csharp
// Old method - CANNOT reliably handle # or %
File file = web.GetFileByServerRelativeUrl(serverRelativeUrl);
```

The `ResourcePath.FromDecodedUrl()` method **expects decoded URLs as input** and handles encoding internally. This eliminates ambiguity—you pass the actual filename `Report#2024.docx`, not an encoded version, and the API handles the rest.

When using manual CSOM with `GetFileByServerRelativeUrl` followed by `OpenBinaryStream`:

- The `#` character is interpreted as a URL fragment delimiter, truncating everything after it
- The `%` character triggers URL decoding logic, causing `%25` to become `%`, leading to file-not-found errors or corrupted paths
- Double-encoding can occur if you pre-encode the URL before passing it to methods that also encode

The `OpenBinaryStream` method itself works correctly once you have a valid file reference—**the failure occurs during file resolution**, not stream operations.

## Server-relative URL processing across API layers

Different SharePoint API layers have distinct encoding expectations, which is the source of most developer confusion:

**CSOM with ResourcePath (recommended for SharePoint Online):** Pass decoded paths directly. The API handles all encoding internally.

```powershell
$path = "/sites/MySite/Shared Documents/Report#2024.docx"  # Decoded
$resourcePath = [Microsoft.SharePoint.Client.ResourcePath]::FromDecodedUrl($path)
$file = $ctx.Web.GetFileByServerRelativePath($resourcePath)
```

**REST API with ResourcePath endpoints:** The newer REST endpoints accept a `decodedUrl` parameter but still require encoding `#` and `%` to prevent URL parsing issues:

```
GET /_api/web/GetFileByServerRelativePath(decodedUrl='/Shared Documents/Report%232024.docx')
```

**Microsoft Graph API:** Requires full RFC 3986 percent-encoding in path segments:

```
GET /drives/{id}/root:/Documents/Report%232024.docx
```

**Legacy CSOM string methods:** Cannot reliably support `#` or `%` in filenames—avoid entirely for files with these characters.

The PnP PowerShell cmdlets abstract this complexity by automatically selecting the appropriate API based on SharePoint version, which is why `Get-PnPFile` generally succeeds where manual CSOM code fails.

## Known PnP PowerShell issues with special characters

Despite using ResourcePath internally, `Get-PnPFile` has documented bugs related to URL parameter handling that occur **before** the file is passed to the underlying API:

- **GitHub Issue #1766**: Filenames with `#` get truncated (e.g., `1#1.jpg` becomes `1`) because the cmdlet's parameter parsing treats `#` as a fragment delimiter
- **GitHub Issue #1700**: Files with `%` return "file not found" because the cmdlet double-encodes the percent sign
- **GitHub Issue #1864**: The `+` character is decoded to a space, causing path mismatches
- **GitHub Issue #5010**: Files with literal `%20` in the name fail because the cmdlet decodes it to a space

These issues occur in the **cmdlet's parameter processing layer**, not in the underlying ResourcePath API. The workaround is to bypass URL-based access entirely and retrieve files via list item queries.

## Best practices for reliable downloads with special characters

**Approach 1: Query by list item instead of URL path**

This completely avoids URL encoding issues by retrieving files through their list item reference:

```powershell
# Get file via CAML query by filename
$item = Get-PnPListItem -List "Documents" -Query @"
<View Scope='RecursiveAll'>
    <Query>
        <Where>
            <Eq>
                <FieldRef Name='FileLeafRef'/>
                <Value Type='Text'>Report#2024.docx</Value>
            </Eq>
        </Where>
    </Query>
</View>
"@

# Access file through item's File property
$file = Get-PnPProperty -ClientObject $item -Property File
$stream = $file.OpenBinaryStream()
Invoke-PnPQuery

# Manually save stream to disk with sanitized local filename
$memStream = New-Object System.IO.MemoryStream
$stream.Value.CopyTo($memStream)
$localName = $item["FileLeafRef"] -replace '[#%]', '_'
[System.IO.File]::WriteAllBytes("C:\Downloads\$localName", $memStream.ToArray())
```

**Approach 2: Use CSOM with ResourcePath directly**

For SharePoint Online, use the ResourcePath API with decoded URLs:

```powershell
$decodedPath = "/sites/MySite/Shared Documents/File#Name.docx"
$resourcePath = [Microsoft.SharePoint.Client.ResourcePath]::FromDecodedUrl($decodedPath)
$file = $ctx.Web.GetFileByServerRelativePath($resourcePath)
$ctx.Load($file)
$ctx.ExecuteQuery()

$binaryStream = $file.OpenBinaryStream()
$ctx.ExecuteQuery()
```

**Approach 3: Sanitize filenames and maintain a mapping**

When downloading files in bulk, replace problematic characters locally while preserving a mapping to original names:

```powershell
$items = Get-PnPListItem -List "Documents" -Fields "FileRef","FileLeafRef","ID"

foreach ($item in $items) {
    $originalName = $item["FileLeafRef"]
    $sanitizedName = $originalName -replace '[#%]', '_'
    
    try {
        Get-PnPFile -Url $item["FileRef"] -Path "C:\Downloads" -Filename $sanitizedName -AsFile
        Write-Host "Downloaded: $originalName -> $sanitizedName"
    }
    catch {
        # Fall back to list item approach
        $file = Get-PnPProperty -ClientObject $item -Property File
        # Manual stream download as above
    }
}
```

**Encoding helper for manual REST/CSOM operations:**

When you must construct URLs manually, encode each path segment individually:

```powershell
Function Get-EncodedSharePointPath($path) {
    $segments = $path -split '/'
    $encoded = $segments | ForEach-Object { 
        if ($_ -ne '') { [uri]::EscapeDataString($_) }
    }
    return '/' + ($encoded -join '/')
}
```

## Conclusion

The failure of CSOM `OpenBinaryStream` operations for files with `#` and `%` characters is not a limitation of the stream operations themselves—it's a file resolution failure caused by legacy string-based APIs that cannot distinguish between literal special characters and URL encoding sequences.

**Key technical insights:**

- **ResourcePath API** is the solution Microsoft designed for this exact problem—always use `GetFileByServerRelativePath` with `ResourcePath.FromDecodedUrl()` for SharePoint Online
- **Get-PnPFile succeeds** because it uses ResourcePath internally, but has its own parameter parsing bugs documented in GitHub issues
- **List item queries** bypass URL encoding entirely and are the most reliable approach for files with any special characters
- **Double-encoding** is the most common developer mistake—never pre-encode URLs when using ResourcePath methods

For SharePoint on-premises (2016/2019), the ResourcePath API is not available, making list item queries or ID-based retrieval the only reliable options for files with `#` and `%` in their names.