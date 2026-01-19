# SharePoint Online Document Library Backup Script — Technical Reference

## Purpose

This document provides a complete PnP PowerShell script for downloading files and version history from SharePoint Online document libraries. The script includes metadata extraction, incremental sync capability, and workarounds for known PnP.PowerShell module limitations. This reference is intended for LLM-assisted modification, extension, or troubleshooting.

---

## Environment Requirements

| Requirement | Specification |
|-------------|---------------|
| PowerShell | Version 7.0+ (mandatory — PnP.PowerShell does not run on Windows PowerShell 5.1) |
| Module | PnP.PowerShell (latest version from PSGallery) |
| Authentication | Must be connected via `Connect-PnPOnline` before execution |
| Platform | Windows (paths use backslash notation; adjust for cross-platform if needed) |

---

## Known PnP PowerShell Pitfalls

The PnP.PowerShell module has documented reliability issues that this script addresses:

### 1. Locale-Dependent Library Names

SharePoint localizes display names but preserves internal URL names:

| Property | English | Russian | German |
|----------|---------|---------|--------|
| Display Name | Documents | Документы | Dokumente |
| Internal URL Name | Shared Documents | Shared Documents | Shared Documents |

**Rule:** Always reference libraries by internal URL name or GUID, never display name. The script includes a commented discovery block for environments where the internal name is unknown.

### 2. CAML Date Query Failures

CAML queries silently return empty results without proper date formatting:
```xml
<!-- INCORRECT: time component ignored -->
<Value Type='DateTime'>2024-01-15T09:00:00Z</Value>

<!-- CORRECT: explicit time handling -->
<Value Type='DateTime' IncludeTimeValue='TRUE' StorageTZ='TRUE'>2024-01-15T09:00:00Z</Value>
```

**Requirements:**
- Format: ISO 8601 (`yyyy-MM-ddTHH:mm:ssZ`)
- Timezone: Must convert to UTC before formatting
- Attributes: `IncludeTimeValue='TRUE'` and `StorageTZ='TRUE'` are mandatory for reliable filtering

### 3. File Version API Limitations

`Get-PnPFileVersion` fails for files exceeding 2GB due to 32-bit integer overflow in the Size property. The script wraps this call in try/catch.

`Get-PnPFile -Url $version.Url` does not reliably handle `_vti_history` URLs. The recommended method is CSOM's `OpenBinaryStream()`:
```powershell
$ctx = Get-PnPContext
$versionStream = $version.OpenBinaryStream()
$ctx.ExecuteQuery()
$fileStream = [System.IO.File]::Create($targetPath)
$versionStream.Value.CopyTo($fileStream)
$fileStream.Close()
```

### 4. Token Expiry

Access tokens expire after 60-75 minutes. For libraries with thousands of files, implement periodic `Connect-PnPOnline` calls (not included in base script — add if processing time exceeds 45 minutes).

### 5. Large List Threshold

SharePoint enforces a 5,000-item view threshold. Always use `-PageSize` parameter with `Get-PnPListItem`. The script uses `-PageSize 2000`.

---

## Script Configuration Parameters
```powershell
# === MODE & DEBUG FLAGS ===
$mode = "download"              # "download" | "metadata" | "dry-run"
$debugNoCaml = $false           # bypass CAML, filter client-side instead
$useCSOMVersionDownload = $true # $true = CSOM OpenBinaryStream, $false = Get-PnPFile legacy

# === CONFIGURATION ===
$libraryName = "Shared Documents"   # Internal URL name, NOT display name
$localRoot = "C:\Downloads\SharePointBackup"
$metaFolderName = ".metadata"
$downloadPreviousVersions = $true
$maxVersions = 5                    # 0 = all versions

# Date range: $null disables that bound
$modifiedAfter = [DateTime]"2024-01-01"
$modifiedBefore = $null
```

### Parameter Definitions

| Parameter | Type | Description |
|-----------|------|-------------|
| `$mode` | String | `"download"` — full backup; `"metadata"` — JSON sidecar only; `"dry-run"` — console output only |
| `$debugNoCaml` | Boolean | When `$true`, fetches all items and filters client-side (useful when CAML returns unexpected empty results) |
| `$useCSOMVersionDownload` | Boolean | When `$true`, uses reliable CSOM `OpenBinaryStream()`; when `$false`, uses legacy `Get-PnPFile -AsMemoryStream` |
| `$libraryName` | String | Internal URL name of document library (not localized display name) |
| `$localRoot` | String | Local filesystem root for backup |
| `$metaFolderName` | String | Subdirectory name for metadata sidecar files |
| `$downloadPreviousVersions` | Boolean | Whether to download historical versions |
| `$maxVersions` | Integer | Maximum historical versions per file (0 = unlimited) |
| `$modifiedAfter` | DateTime/null | Lower bound for Modified date filter |
| `$modifiedBefore` | DateTime/null | Upper bound for Modified date filter |

---

## Output Structure
```
C:\Downloads\SharePointBackup\
├── Subfolder\
│   └── Report.xlsx                    # Current file version
└── .metadata\
    └── Subfolder\
        ├── Report.xlsx.meta.json      # Metadata + version history
        └── Report.xlsx.versions\      # Historical versions
            ├── v1.0_Report.xlsx
            └── v2.0_Report.xlsx
```

### Metadata JSON Schema
```json
{
  "FileName": "Report.xlsx",
  "ServerPath": "/sites/MySite/Shared Documents/Subfolder/Report.xlsx",
  "LocalPath": "C:\\Downloads\\SharePointBackup\\Subfolder\\Report.xlsx",
  "Created": "2024-01-15T10:30:00Z",
  "Modified": "2024-06-20T14:22:00Z",
  "Author": "john@contoso.com",
  "Editor": "jane@contoso.com",
  "Size": 245760,
  "ContentType": "Document",
  "VersionCount": 4,
  "CurrentVersion": "3.0",
  "Versions": [
    {
      "VersionLabel": "2.0",
      "Created": "2024-05-10T09:15:00Z",
      "CreatedBy": "jane@contoso.com",
      "Size": 230400
    },
    {
      "VersionLabel": "1.0",
      "Created": "2024-01-15T10:30:00Z",
      "CreatedBy": "john@contoso.com",
      "Size": 215040
    }
  ],
  "ExportedAt": "2024-06-21T08:00:00+00:00"
}
```

---

## Incremental Sync Logic

The script implements skip logic for unchanged files:

1. Check if `$metaFilePath` and `$localFilePath` both exist
2. Parse existing metadata JSON
3. Compare `CurrentVersion` and `Modified` timestamp
4. Skip download if both match

This enables efficient incremental backups without re-downloading unchanged content.

---

## Session Cache Behavior

The `$spFileItems` variable persists in PowerShell session scope. On subsequent script executions within the same session:

1. Script detects populated `$spFileItems` array
2. Prompts user: `"Re-fetch from SharePoint? (y/N)"`
3. If declined, reuses cached item list (avoids slow `Get-PnPListItem` call)

This is useful when adjusting `$mode` or other parameters without re-querying SharePoint.

---

## Complete Script
```powershell
#Requires -Version 7.2

  

<#

.SYNOPSIS

    U??????????????

  

.NOTES

    This script expects:

    - Config file: ..\config\sharepoint-config.json

    - Secrets file: ..\config\sharepoint-secrets.json

    - Certificate file: ..\config\certs\SillenoProjectControl.pfx

#>

  

Import-Module PnP.PowerShell

  

$config = Get-Content (Join-Path $PSScriptRoot "..\config\sharepoint-config.json") | ConvertFrom-Json

$secrets = Get-Content (Join-Path $PSScriptRoot "..\config\sharepoint-secrets.json") | ConvertFrom-Json

  

$params = $config.main_production

$secrets.main_production.PSObject.Properties | ForEach-Object { $params | Add-Member -NotePropertyName $_.Name -NotePropertyValue $_.Value }

  

Connect-PnPOnline `

    -Url $params.siteUrl `

    -ClientId $params.clientId `

    -Tenant $params.tenant `

    -CertificatePath (Join-Path $PSScriptRoot "..\config\certs\SillenoProjectControl.pfx") `

    -CertificatePassword (ConvertTo-SecureString $params.clientSecret -AsPlainText -Force)

  

Write-Host "Connected to SharePoint" -ForegroundColor Green

  
  

# ==============================================================

# ==============================================================

# ==============================================================

# ==============================================================

  

# === MODE & DEBUG FLAGS ===

$mode = "download"              # "download" | "metadata" | "dry-run"

$debugNoCaml = $true           # bypass CAML, filter client-side instead

$useCSOMVersionDownload = $true # $true = CSOM OpenBinaryStream (reliable), $false = Get-PnPFile (legacy)

  

$downloadPreviousVersions = $true      # Set to $false to skip version history entirely

$verboseItemProgress = $true           # $true = detailed messages, $false = dot every 200 items

  
  
  

# === CONFIGURATION ===

$libraryName = "Документы"   # Use internal URL name, not display name (e.g., not "Документы")

$localRoot = "d:\files-datastore\SillenoProjectControl-Shared_Documents"

$metaFolderName = ".metadata"

$downloadPreviousVersions = $true

$maxVersions = 2                    # 0 = all versions

  

# Date range: set to $null to disable that bound

$modifiedAfter = [DateTime]"2026-01-01"

$modifiedBefore = [DateTime]"2026-01-16"

  

# === LIBRARY DISCOVERY (locale-safe) ===

# Uncomment to auto-discover library internal name:

# $docLib = Get-PnPList -Includes RootFolder | Where-Object { $_.BaseTemplate -eq 101 }

# $libraryName = $docLib.RootFolder.Name

# Write-Host "Using library: $libraryName (Display: $($docLib.Title))" -ForegroundColor Cyan

  

# === CAML BUILDER ===

function Build-DateFilterCaml {

    param(

        [Parameter()][AllowNull()][DateTime]$After,

        [Parameter()][AllowNull()][DateTime]$Before

    )

    if (-not $After -and -not $Before) {

        return "<View Scope='RecursiveAll'><Query></Query></View>"

    }

    # IncludeTimeValue='TRUE' StorageTZ='TRUE' are CRITICAL for reliable date filtering

    $afterClause = if ($After) {

        $afterUtc = $After.ToUniversalTime().ToString('yyyy-MM-ddTHH:mm:ssZ')

        "<Geq><FieldRef Name='Modified'/><Value Type='DateTime' IncludeTimeValue='TRUE' StorageTZ='TRUE'>$afterUtc</Value></Geq>"

    }

    else { $null }

    $beforeClause = if ($Before) {

        $beforeUtc = $Before.ToUniversalTime().ToString('yyyy-MM-ddTHH:mm:ssZ')

        "<Leq><FieldRef Name='Modified'/><Value Type='DateTime' IncludeTimeValue='TRUE' StorageTZ='TRUE'>$beforeUtc</Value></Leq>"

    }

    else { $null }

    $whereContent = if ($After -and $Before) {

        "<And>$afterClause$beforeClause</And>"

    }

    elseif ($After) {

        $afterClause

    }

    else {

        $beforeClause

    }

    return "<View Scope='RecursiveAll'><Query><Where>$whereContent</Where></Query></View>"

}

  

# === CACHE CHECK ===

$shouldFetch = $true

if ($null -ne $spFileItems -and

    $spFileItems.Count -gt 0 -and

    $spFileItems[0] -is [Microsoft.SharePoint.Client.ListItem]) {

    Write-Host "Cache: $($spFileItems.Count) items already loaded." -ForegroundColor Yellow

    $response = Read-Host "Re-fetch from SharePoint? (y/N)"

    $shouldFetch = ($response.ToLower() -eq 'y')

}

  

# ============================================

# ===   SHAREPOINT ONLINE: FETCH ITEMS     ===

# ============================================

if ($shouldFetch) {

    if ($debugNoCaml) {

        Write-Host "[DEBUG] CAML bypassed - fetching all items" -ForegroundColor Magenta

        $spFileItems = Get-PnPListItem -List $libraryName -PageSize 2000 -ScriptBlock {

            param($items)

            Write-Host "." -NoNewline -ForegroundColor Cyan

        }

    }

    else {

        $camlQuery = Build-DateFilterCaml -After $modifiedAfter -Before $modifiedBefore

        Write-Host "Fetching from SharePoint..." -ForegroundColor Cyan

        Write-Host "[CAML] $camlQuery" -ForegroundColor DarkGray

        $spFileItems = Get-PnPListItem -List $libraryName -Query $camlQuery -PageSize 2000 -ScriptBlock {

            param($items)

            Write-Host "." -NoNewline -ForegroundColor Cyan

        }

    }

  

    Write-Host ""  # New line after dots

}

# ============================================

  

# Client-side date filter (fallback when debugNoCaml = $true)

if ($debugNoCaml -and $spFileItems) {

    $preFilterCount = $spFileItems.Count

    $spFileItems = @($spFileItems | Where-Object {

            $mod = [DateTime]$_["Modified"]

            ((-not $modifiedAfter) -or ($mod -ge $modifiedAfter)) -and

            ((-not $modifiedBefore) -or ($mod -le $modifiedBefore))

        })

    Write-Host "[DEBUG] Filtered: $preFilterCount -> $($spFileItems.Count) items" -ForegroundColor Magenta

}

  

# Date range description

$rangeDesc = if (-not $modifiedAfter -and -not $modifiedBefore) {

    "all time"

}

elseif ($modifiedAfter -and $modifiedBefore) {

    "$($modifiedAfter.ToString('yyyy-MM-dd')) to $($modifiedBefore.ToString('yyyy-MM-dd'))"

}

elseif ($modifiedAfter) {

    "after $($modifiedAfter.ToString('yyyy-MM-dd'))"

}

else {

    "before $($modifiedBefore.ToString('yyyy-MM-dd'))"

}

  

$versionMethod = if ($useCSOMVersionDownload) { "CSOM" } else { "Legacy" }

Write-Host "[$mode] Processing $($spFileItems.Count) files ($rangeDesc) [Version method: $versionMethod]" -ForegroundColor Cyan

  

# === MAIN LOOP ===

$stats = @{ Downloaded = 0; Skipped = 0; Versions = 0; VersionsFailed = 0; MetadataOnly = 0; Processed = 0 }

  

foreach ($item in $spFileItems) {

    if ($item.FileSystemObjectType -ne "File") { continue }

    $stats.Processed++

    # Progress indicator for non-verbose mode

    if (-not $verboseItemProgress -and $stats.Processed % 5 -eq 0) {

        Write-Host "." -NoNewline -ForegroundColor Cyan

    }

    $fileUrl = $item["FileRef"]

    $fileName = $item["FileLeafRef"]

    $folderPath = $item["FileDirRef"] -replace "^.*?$([regex]::Escape($libraryName))", ""

    $currentVersion = $item["_UIVersionString"]

    $modifiedDate = [DateTime]$item["Modified"]

    $fileSize = $item["File_x0020_Size"]

    $localFolder = Join-Path $localRoot $folderPath

    $metaFolder = Join-Path $localRoot (Join-Path $metaFolderName $folderPath)

    $localFilePath = Join-Path $localFolder $fileName

    $metaFilePath = Join-Path $metaFolder "$fileName.meta.json"

    # === SKIP CHECK ===

    if ($mode -eq "download" -and (Test-Path $metaFilePath) -and (Test-Path $localFilePath)) {

        try {

            $existingMeta = Get-Content $metaFilePath -Raw | ConvertFrom-Json

            if ($existingMeta.CurrentVersion -eq $currentVersion -and

                [DateTime]$existingMeta.Modified -eq $modifiedDate) {

                if ($verboseItemProgress) {

                    Write-Host "Skipped (unchanged): $fileName" -ForegroundColor DarkGray

                }

                $stats.Skipped++

                continue

            }

        }

        catch {

            if ($verboseItemProgress) {

                Write-Host "Warning: Could not parse metadata for $fileName - re-downloading" -ForegroundColor Yellow

            }

        }

    }

    # === FILE VERSIONS (only if downloadPreviousVersions = $true) ===

    $versions = @()

    if ($downloadPreviousVersions) {

        try {

            $versions = Get-PnPFileVersion -Url $fileUrl -ErrorAction Stop

        }

        catch {

            if ($verboseItemProgress) {

                Write-Host "Warning: Could not get versions for $fileName (file may be >2GB)" -ForegroundColor Yellow

            }

        }

    }

    # Build metadata object

    $metadata = [ordered]@{

        FileName       = $fileName

        ServerPath     = $fileUrl

        LocalPath      = $localFilePath

        Created        = $item["Created"]

        Modified       = $modifiedDate

        Author         = if ($item["Author"]) { $item["Author"].Email } else { $null }

        Editor         = if ($item["Editor"]) { $item["Editor"].Email } else { $null }

        Size           = $fileSize

        ContentType    = if ($item["ContentType"]) { $item["ContentType"].Name } else { $null }

        VersionCount   = $versions.Count + 1

        CurrentVersion = $currentVersion

        Versions       = @($versions | ForEach-Object {

                [ordered]@{

                    VersionLabel = $_.VersionLabel

                    Created      = $_.Created

                    CreatedBy    = $_.CreatedByEmail

                    Size         = $_.Size

                }

            })

        ExportedAt     = (Get-Date -Format "o")

    }

    # === DRY-RUN MODE ===

    if ($mode -eq "dry-run") {

        if ($verboseItemProgress) {

            Write-Host "`n--- $fileName (v$currentVersion) ---" -ForegroundColor Yellow

            $metadata | ConvertTo-Json -Depth 4 | Write-Host

        }

        $stats.MetadataOnly++

        continue

    }

    # Create metadata folder & save JSON

    if (-not (Test-Path $metaFolder)) {

        New-Item -ItemType Directory -Path $metaFolder -Force | Out-Null

    }

    $metadata | ConvertTo-Json -Depth 4 | Out-File -LiteralPath $metaFilePath -Encoding UTF8

    # === METADATA MODE ===

    if ($mode -eq "metadata") {

        if ($verboseItemProgress) {

            Write-Host "Metadata saved: $fileName" -ForegroundColor Blue

        }

        $stats.MetadataOnly++

        continue

    }

    # === DOWNLOAD FILE ===

    if (-not (Test-Path $localFolder)) {

        New-Item -ItemType Directory -Path $localFolder -Force | Out-Null

    }

    try {

        Get-PnPFile -Url $fileUrl -Path $localFolder -Filename $fileName -AsFile -Force

        if ($verboseItemProgress) {

            Write-Host "Downloaded: $fileName (v$currentVersion)" -ForegroundColor Green

        }

        $stats.Downloaded++

    }

    catch {

        if ($verboseItemProgress) {

            Write-Host "Failed to download: $fileName - $_" -ForegroundColor Red

        }

        continue

    }

    # === VERSION HISTORY (only if downloadPreviousVersions = $true) ===

    if ($downloadPreviousVersions -and $versions.Count -gt 0) {

        $versionFolder = Join-Path $metaFolder "$fileName.versions"

        if (-not (Test-Path $versionFolder)) {

            New-Item -ItemType Directory -Path $versionFolder -Force | Out-Null

        }

        $versionsToDl = if ($maxVersions -gt 0) {

            $versions | Select-Object -First $maxVersions

        }

        else {

            $versions

        }

        foreach ($ver in $versionsToDl) {

            $versionFileName = "v$($ver.VersionLabel)_$fileName"

            $versionFilePath = Join-Path $versionFolder $versionFileName

            if (Test-Path $versionFilePath) { continue }

            try {

                if ($useCSOMVersionDownload) {

                    $ctx = Get-PnPContext

                    $versionStream = $ver.OpenBinaryStream()

                    $ctx.ExecuteQuery()

                    $fileStream = [System.IO.File]::Create($versionFilePath)

                    $versionStream.Value.CopyTo($fileStream)

                    $fileStream.Close()

                    $versionStream.Value.Dispose()

                }

                else {

                    $stream = Get-PnPFile -Url $ver.Url -AsMemoryStream

                    [System.IO.File]::WriteAllBytes($versionFilePath, $stream.ToArray())

                    $stream.Dispose()

                }

                if ($verboseItemProgress) {

                    Write-Host "  + v$($ver.VersionLabel)" -ForegroundColor DarkGray

                }

                $stats.Versions++

            }

            catch {

                if ($verboseItemProgress) {

                    Write-Host "  ! v$($ver.VersionLabel) failed: $_" -ForegroundColor Red

                }

                $stats.VersionsFailed++

            }

        }

    }

}

  

# Add newline after dots

if (-not $verboseItemProgress) {

    Write-Host ""

}

  

# === SUMMARY ===

Write-Host "`n=== Complete [$mode] ===" -ForegroundColor Cyan

Write-Host "Processed: $($stats.Processed) files"

switch ($mode) {

    "download" {

        Write-Host "Downloaded: $($stats.Downloaded) | Skipped: $($stats.Skipped) | Versions: $($stats.Versions) | Version failures: $($stats.VersionsFailed)"

    }

    "metadata" { Write-Host "Metadata files written: $($stats.MetadataOnly)" }

    "dry-run" { Write-Host "Files inspected: $($stats.MetadataOnly) (no writes)" }

}

Write-Host "Location: $localRoot"
```

---

## Extension Points

When modifying this script, consider:

1. **Token refresh for large libraries:** Add `Connect-PnPOnline` call every N items if processing exceeds 45 minutes.

2. **Progress reporting:** Add `Write-Progress` with percentage based on `$spFileItems.Count`.

3. **Parallel downloads:** PowerShell 7 supports `ForEach-Object -Parallel`, but PnP context is not thread-safe — requires separate connections per runspace.

4. **Retry logic:** Wrap SharePoint calls in retry loop with exponential backoff for 429 (throttling) responses.

5. **Filtering by file type:** Add CAML `<Contains>` clause on `FileLeafRef` or filter `$spFileItems` client-side by extension.

6. **Delta sync from last run:** Store last successful `ExportedAt` timestamp and use as `$modifiedAfter` for subsequent runs.

---

## Troubleshooting

| Symptom                              | Cause                    | Resolution                                                |
| ------------------------------------ | ------------------------ | --------------------------------------------------------- |
| Empty results from CAML              | Missing date attributes  | Verify `IncludeTimeValue='TRUE' StorageTZ='TRUE'` present |
| Empty results from CAML              | Wrong library name       | Use internal URL name, not localized display name         |
| "List view threshold exceeded"       | Missing PageSize         | Always include `-PageSize` parameter                      |
| Version download fails               | Legacy method limitation | Set `$useCSOMVersionDownload = $true`                     |
| "Operation is not valid" on versions | File >2GB                | Known bug — no workaround for version metadata            |
| Script hangs                         | Token expiry             | Add periodic reconnection for long operations             |
| Wrong files downloaded               | Date filtering mismatch  | Set `$debugNoCaml = $true` to verify CAML behavior        |
