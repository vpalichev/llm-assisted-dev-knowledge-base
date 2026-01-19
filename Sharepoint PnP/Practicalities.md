# Table of Contents

- [[#How to get list of all list items for library, folders, files, etc]]
- [[#Export to CSV]]
- [[#Connect with Certificate Authentication]]
- [[#Parameters]]

---

## How to get list of all list items for library, folders, files, etc

[[#Table of Contents|Back to TOC]]

```powershell
Measure-Command {
  $all_list_items_for_library =  Get-PnPListItem -List $params.libraryName -PageSize 5000 -Fields "FileLeafRef","FileRef","File_x0020_Size","Created","Modified","Author","Editor"
}
```

## Export to CSV

[[#Table of Contents|Back to TOC]]

```powershell
$items = Get-PnPListItem -List $params.libraryName -PageSize 5000 `
    -Fields "FileLeafRef","FileRef","File_x0020_Size","Created","Modified","Author","Editor","FSObjType","UniqueId"

$csvData = $items | ForEach-Object {
    [PSCustomObject]@{
        FileLeafRef    = $_.FieldValues.FileLeafRef
        FileRef        = $_.FieldValues.FileRef
        File_x0020_Size = $_.FieldValues.File_x0020_Size
        Created        = $_.FieldValues.Created.ToUniversalTime().ToString("o")
        Modified       = $_.FieldValues.Modified.ToUniversalTime().ToString("o")
        Author         = $_.FieldValues.Author.Email
        Editor         = $_.FieldValues.Editor.Email
        FSObjType = $_.FieldValues.FSObjType
        UniqueId = if ($_.FieldValues.UniqueId) { $_.FieldValues.UniqueId.ToString() } else { $null }
    }
}

$csvData | Export-Csv -Path "sharepoint_items.csv" -NoTypeInformation -Encoding UTF8 -UseQuotes AsNeeded
```

**Note:** If using Windows PowerShell 5.1 (not PowerShell 7), remove `-UseQuotes AsNeeded` — it's not supported. The default quoting should work fine for this data.

**PostgreSQL table (with timezone-aware timestamp):**

```sql
CREATE TABLE sharepoint_files (
    file_name    TEXT,
    file_path    TEXT,
    file_size    BIGINT,
    created_at   TIMESTAMPTZ,  -- stores as UTC
    modified_at  TIMESTAMPTZ,
    created_by   TEXT,
    modified_by  TEXT
);

\COPY sharepoint_files FROM 'sharepoint_items.csv' WITH (FORMAT csv, HEADER true);
```

# SharePoint PnP Connection

## Connect with Certificate Authentication

[[#Table of Contents|Back to TOC]]

```powershell
Connect-PnPOnline `
    -Url $params.siteUrl `
    -ClientId $params.clientId `
    -Tenant $params.tenant `
    -CertificatePath (Join-Path $PSScriptRoot "..\config\certs\SillenoProjectControl.pfx") `
    -CertificatePassword (ConvertTo-SecureString $params.clientSecret -AsPlainText -Force)
```

## Parameters

[[#Table of Contents|Back to TOC]]

### `siteUrl`

SharePoint site URL.

**Example:** `https://exampledotorg.sharepoint.com/sites/TheSiteLikeProjectControl`

### `clientId`

Azure AD Application (Client) ID — a GUID from your app registration.

**How to obtain:**
- **Azure Portal:** App registrations → your app → Overview → Application (client) ID
- **PowerShell:**
  ```powershell
  Connect-AzAccount
  Get-AzADApplication | Select-Object DisplayName, AppId
  ```

**Example:** `a1b2c3d4-e5f6-7890-abcd-ef1234567890`

### `tenant`

Microsoft 365 tenant domain.

**Example:** `exampledotorg.onmicrosoft.com`


## Errors encountered:

  + v2.0
Failed to download: BOE_r3 & BOM_r2 - Based on 30%MR.xlsx - Exception calling "ExecuteQuery" with "0" argument(s): "Файл не найден."
Downloaded: Status and Open Questions for TCM-SSEC as of November 2025.docx (v114.0)