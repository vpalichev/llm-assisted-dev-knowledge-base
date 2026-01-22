Original Plan: Sync-SPOVersions-v04.ps1 (Reference)
## Summary

Create a new SharePoint document sync script implementing YAML-based metadata (`.meta` files) and a dedicated version store (`__spo_store`) with proper entity lifecycle tracking.

---
## Key Design Decisions

| Decision        | Choice                                                       |
| --------------- | ------------------------------------------------------------ |
| Move detection  | Disabled - if UniqueId disappears, mark as `deleted`         |
| Tombstones      | Preserve metadata and blobs when files disappear             |
| Renames         | Treated as "old deleted + new created"                       |
| Current version | Live copy at real path AND duplicate in `__spo_store`        |
| Operation mode  | Flag: `metadata-only` or `download-files` (independent runs) |
| Sync detection  | Compare `.meta` files to preserve tombstones                 |
---
## Target Folder Structure

```
<MirrorRoot>\
├── Engineering\
│   ├── __spo_store\
│   │   └── Report.xlsx.versions\
│   │       ├── e4c3c2a1_v001.0_Report.xlsx
│   │       ├── e4c3c2a1_v002.0_Report.xlsx
│   │       └── e4c3c2a1_v003.0_Report.xlsx  ← current version copy
│   ├── Report.xlsx                          ← live copy
│   └── Report.xlsx.meta                     ← YAML metadata
```

---

## Implementation Steps

### Step 1: Create Script Skeleton

**File:** `src/Sync-SPOVersions-v04.ps1`

- Script parameters: `$Mode` ('metadata-only' | 'download-files'), `$MirrorRoot`, filtering params
- Import modules: `PnP.CustomUtilities`, `powershell-yaml`
- Connect to SharePoint using existing `Connect-SharePointWithConfig`

### Step 2: Implement Helper Functions

| Function                   | Purpose                                                   |
| -------------------------- | --------------------------------------------------------- |
| `Get-UniqueIdPrefix`       | Extract first 8 chars of GUID                             |
| `Format-VersionBinaryName` | Generate `e4c3c2a1_v003.0_Report.xlsx` format             |
| `Read-MetaFile`            | Parse YAML `.meta` file, return `$null` if missing        |
| `Write-MetaFile`           | Write YAML `.meta` file with ordered keys                 |
| `New-EntityRecord`         | Create entity hashtable with versions array               |
| `Get-CurrentVersionNumber` | Infer current version from `Get-PnPFileVersion` (max + 1) |

### Step 3: Implement Version Download Functions

|Function|Purpose|
|---|---|
|`Get-FileVersionsFromSharePoint`|Retrieve version history via `Get-PnPFileVersion`|
|`Download-VersionBlob`|Download historical version using CSOM `OpenBinaryStream()`|
|`Download-CurrentVersion`|Download current file using CSOM `GetFileByServerRelativeUrl` (TODO: no, download with usual get-pnpfile)|
|`Ensure-VersionStoreFolder`|Create `__spo_store/<filename>.versions/` structure|

### Step 4: Implement Sync Comparison Algorithm

**Function:** `Invoke-FolderSync`

1. Build lookup of SharePoint files by UniqueId and filename
2. Scan existing `.meta` files in local folder
3. For each SharePoint file:
    - If `.meta` exists with same UniqueId + version → **skip**
    - If `.meta` exists with same UniqueId but different version → **update** (add new versions)
    - If `.meta` exists with different UniqueId → **supersede** old entity, create new
    - If no `.meta` → **create new**
4. For each `.meta` where UniqueId NOT in SharePoint:
    - Mark entity as `deleted`, set `currentEntity: null`
5. Execute actions based on `$Mode`

### Step 5: Implement Main Loop

1. Fetch all library items via `Get-SharePointAllListItems`
2. Transform via `ConvertTo-ReadablePnPListItem`
3. Filter to files only (exclude folders)
4. Group by local folder path (using `ConvertTo-LocalFolderPath`)
5. For each folder group: call `Invoke-FolderSync`
6. Display progress and final summary

### Step 6: Error Handling & Reporting

- Track errors in `$script:errorLog` collection
- Retry downloads with existing `Invoke-WithRetry`
- Handle corrupt `.meta` files: backup and recreate
- Export error log to JSON at end of run
- Display summary with created/updated/deleted/skipped counts

---

## Meta File Schema (YAML)

```yaml
currentEntity: d8ebc5c1-1234-5678-9abc-def012345678  # or null
currentVersion: "4.0"  # or null
entities:
  - UniqueId: d8ebc5c1-1234-5678-9abc-def012345678
    FileLeafRef: "Report.xlsx"
    status: current  # current | superseded | deleted
    versions:
      - number: "4.0"
        Modified: 2024-07-15T09:20:00Z
        Editor: carol.smith@company.com
        File_x0020_Size: 19456
      - number: "3.0"
        Modified: 2024-07-01T14:00:00Z
        Editor: carol.smith@company.com
        File_x0020_Size: 18432
```

**Note:** Versions ordered newest-first within each entity.

### Entity Status Values

|Status|Condition|currentEntity|currentVersion|
|---|---|---|---|
|`current`|This entity is the active file at this path|Set to this UniqueId|Set to version|
|`superseded`|A **different file** (different UniqueId) now occupies this path|Set to new file's UniqueId|Set to new file's version|
|`deleted`|Path is **empty** in SharePoint (file deleted or moved away)|`null`|`null`|

**Detection logic:**

```
if (local .meta has currentEntity X) and (SharePoint has UniqueId Y at this path where X ≠ Y):
    → mark X as 'superseded', add Y as 'current'

if (local .meta has currentEntity X) and (SharePoint has NO file at this path):
    → mark X as 'deleted', set currentEntity/currentVersion to null
```

---

## Version Binary Naming

Format: `<UniqueIdPrefix>_v<PaddedVersion>_<FileLeafRef>`

|Component|Format|Example|
|---|---|---|
|UniqueIdPrefix|8 hex chars|`e4c3c2a1`|
|PaddedVersion|`vNNN.N`|`v003.0`, `v012.0`|
|FileLeafRef|Original name|`Report.xlsx`|

**Result:** `e4c3c2a1_v003.0_Report.xlsx`

---

## Critical Files to Modify/Create

|File|Action|
|---|---|
|`src/Sync-SPOVersions-v04.ps1`|**CREATE** - New main script|
|`src/Modules/PnP.CustomUtilities/`|No changes needed - reuse existing functions|

---

## Dependencies

- **PowerShell 7.4+** (already required)
- **PnP.PowerShell module** (already used)
- **powershell-yaml module** - needs to be installed: `Install-Module powershell-yaml`
- **Existing module:** `PnP.CustomUtilities` with `Connect-SharePointWithConfig`, `Get-SharePointAllListItems`, `ConvertTo-ReadablePnPListItem`, `ConvertTo-LocalFolderPath`, `Invoke-WithRetry`

---

## Verification Plan

### Test 1: Metadata-Only Mode

```powershell
.\Sync-SPOVersions-v04.ps1 -Mode 'metadata-only' -MirrorRoot 'D:\TestMirror'
```

- Verify `.meta` files created with correct YAML structure
- Verify `__spo_store` folders NOT created (no blobs)
- Verify tombstone detection works (remove file from SP, re-run, check status changes to `deleted`)

### Test 2: Download-Files Mode

```powershell
.\Sync-SPOVersions-v04.ps1 -Mode 'download-files' -MirrorRoot 'D:\TestMirror'
```

- Verify live copy at real path
- Verify all versions in `__spo_store/<filename>.versions/`
- Verify version filenames match format `e4c3c2a1_v001.0_filename.ext`
- Verify file timestamps synced to SharePoint Modified date

### Test 3: Incremental Sync

1. Run full sync
2. Add new version to a file in SharePoint
3. Re-run sync
4. Verify only new version downloaded, `.meta` updated with new version entry

### Test 4: Supersede Scenario

1. Run sync with File A at path `/Docs/Report.xlsx`
2. In SharePoint: delete File A, upload new File B at same path
3. Re-run sync
4. Verify `.meta` has two entities: File B as `current`, File A as `superseded`

---

## Estimated Scope

- ~400-500 lines of PowerShell code
- Single new file: `Sync-SPOVersions-v04.ps1`
- No modifications to existing module