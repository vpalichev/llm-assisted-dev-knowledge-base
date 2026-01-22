This document provides a complete specification of the folder structure and YAML metadata format used by `Sync-SPOVersions-v04.ps1` for synchronizing SharePoint Online documents to a local file system.

---

## Overview

The script synchronizes SharePoint Online documents with:
- **YAML-based `.meta` files** for metadata storage
- **`__spo_store` folder** for version binary archives
- **Entity lifecycle tracking** (current, superseded, deleted)
- **Two-stage operation**: metadata-only or full download

---

## Folder Structure

```
<MirrorRoot>\
├── Folder\
│   ├── __spo_store\
│   │   └── <FileName>.<ext>.versions\
│   │       ├── <UniqueIdPrefix>_v<PaddedVersion>_<FileName>.<ext>
│   │       └── <UniqueIdPrefix>_v<PaddedVersion>_<FileName>.<ext>
│   ├── <FileName>.<ext>           (live copy - current version)
│   └── <FileName>.<ext>.meta      (YAML metadata file)
├── AnotherFolder\
│   ├── __spo_store\
│   │   └── ...
│   ├── ...
│   └── ...
└── sync-errors-<timestamp>.json   (error log if errors occurred)
```

### Components

| Component | Description |
|-----------|-------------|
| `<MirrorRoot>` | Local root folder for the mirror (configurable via `-MirrorRoot` param) |
| `<Folder>` | Mirrors SharePoint folder structure relative to the document library |
| `__spo_store\` | Hidden storage folder for version binaries (one per folder with files) |
| `<FileName>.<ext>.versions\` | Per-file folder containing all version blobs |
| `<FileName>.<ext>` | Live copy of the current version |
| `<FileName>.<ext>.meta` | YAML metadata file tracking entity lifecycle and versions |

### Version Binary Naming Convention

```
<UniqueIdPrefix>_v<PaddedVersion>_<FileLeafRef>
```

| Component | Format | Example |
|-----------|--------|---------|
| `UniqueIdPrefix` | First 8 characters of SharePoint GUID (lowercase) | `e4c3c2a1` |
| `PaddedVersion` | `v` + 3-digit padded major + `.` + minor | `v001.0`, `v012.0` |
| `FileLeafRef` | Original filename with extension | `Report.xlsx` |

**Full example:** `e4c3c2a1_v003.0_Report.xlsx`

---

## YAML Metadata Structure (`.meta` files)

Each `.meta` file is a YAML document with the following structure:

```yaml
FileRef: /sites/SiteName/LibraryName/Folder/FileName.xlsx
currentEntity: e4c3c2a1-b2d3-4e5f-6789-abcdef012345
currentVersion: "3.0"
LocalPathLength: 85
entities:
  - UniqueId: e4c3c2a1-b2d3-4e5f-6789-abcdef012345
    FileLeafRef: FileName.xlsx
    status: current
    versions:
      - number: "3.0"
        Modified: "2025-01-15T10:30:00.0000000Z"
        Editor: user@domain.com
        File_x0020_Size: 45678
      - number: "2.0"
        Modified: "2025-01-10T14:22:00.0000000Z"
        Editor: user@domain.com
        File_x0020_Size: 42100
      - number: "1.0"
        Modified: "2025-01-05T09:15:00.0000000Z"
        Editor: other@domain.com
        File_x0020_Size: 38000
  - UniqueId: old-guid-if-file-was-replaced
    FileLeafRef: FileName.xlsx
    status: superseded
    versions:
      - number: "2.0"
        ...
```

### Root-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| `FileRef` | string | Full SharePoint server-relative URL to the file |
| `currentEntity` | string \| null | GUID of the currently active entity, `null` if deleted |
| `currentVersion` | string \| null | Version number of current file (e.g., `"3.0"`), `null` if deleted |
| `LocalPathLength` | integer | Character length of the full local file path (for path length monitoring) |
| `entities` | array | List of entity records (current entity listed first) |

### Entity Record Fields

| Field | Type | Description |
|-------|------|-------------|
| `UniqueId` | string | SharePoint GUID uniquely identifying this file entity |
| `FileLeafRef` | string | Filename with extension |
| `status` | string | Entity lifecycle status: `current`, `superseded`, or `deleted` |
| `versions` | array | List of version records, sorted newest-first |

### Version Record Fields

| Field | Type | Description |
|-------|------|-------------|
| `number` | string | Version label (e.g., `"1.0"`, `"12.0"`) |
| `Modified` | string | ISO 8601 UTC timestamp when version was created |
| `Editor` | string | Email or display name of user who created the version |
| `File_x0020_Size` | integer | File size in bytes |

---

## Entity Lifecycle States

| Status | Description |
|--------|-------------|
| `current` | Active file entity at this path |
| `superseded` | A different file now occupies this path (same filename, different GUID) |
| `deleted` | File no longer exists at this path in SharePoint |

### State Transitions

```
[NEW] --> current
current --> superseded (when file replaced by different GUID)
current --> deleted (when file removed from SharePoint)
```

**Note:** Move detection is disabled. If a file is moved to a different path:
- Original location is marked as `deleted`
- New location creates a fresh `current` entity

---

## Sync Modes

| Mode | Description |
|------|-------------|
| `metadata-only` | Creates/updates `.meta` files without downloading binaries |
| `download-files` | Full sync including all version binaries to `__spo_store` |

### Version Limiting (download-files mode only)

The `VersionsToDownload` parameter controls how many versions are downloaded:

| Value | Behavior |
|-------|----------|
| `0` | Download all versions (default) |
| `1` | Download current version only |
| `N` | Download current + (N-1) most recent historical versions |

**Important:** Metadata always captures ALL versions regardless of this setting.

---

## Filtering Options (Script Flags)

| Flag | Type | Description |
|------|------|-------------|
| `$filterExtensions` | string[] | File extensions to include (e.g., `@('.xlsx', '.docx')`) |
| `$filterDateStart` | string | Items modified on or after this date (format: `'YYYY-MM-DD'`) |
| `$filterDateEnd` | string | Items modified on or before this date |
| `$filterMinSizeMB` | double | Minimum file size in MB |
| `$filterMaxSizeMB` | double | Maximum file size in MB |

---

## Error Handling

Errors are logged to a JSON file at `<MirrorRoot>/sync-errors-<timestamp>.json`:

```json
[
  {
    "Timestamp": "2025-01-15T10:30:00.0000000Z",
    "Type": "VersionDownload",
    "FileRef": "/sites/Site/Library/Folder/File.xlsx",
    "Version": "2.0",
    "Message": "Network timeout"
  }
]
```

### Error Types

| Type | Description |
|------|-------------|
| `MetaFileCorrupt` | YAML parse failed (corrupt file backed up) |
| `VersionFetch` | Failed to retrieve version history from SharePoint |
| `VersionDownload` | Failed to download a historical version blob |
| `CurrentVersionDownload` | Failed to download the current/live version |

---

## Dependencies

- **PnP.CustomUtilities** module (custom utilities for SharePoint connection)
- **powershell-yaml** module (`Install-Module powershell-yaml`)
- **PnP.PowerShell** module (SharePoint Online connectivity)
- **PowerShell 7.4+**

---

## Configuration Files

| File | Purpose |
|------|---------|
| `config/sharepoint-config.json` | SharePoint site and library configuration |
| `config/sharepoint-secrets.json` | Authentication secrets |
| `config/certs/SillenoProjectControl.pfx` | Certificate for app-only authentication |

---

## Example: Complete Metadata File

```yaml
FileRef: /sites/SillenoProjectControl/ProjectDocuments/2025/Q1/Budget.xlsx
currentEntity: a1b2c3d4-e5f6-7890-abcd-ef0123456789
currentVersion: "5.0"
LocalPathLength: 78
entities:
  - UniqueId: a1b2c3d4-e5f6-7890-abcd-ef0123456789
    FileLeafRef: Budget.xlsx
    status: current
    versions:
      - number: "5.0"
        Modified: "2025-01-20T16:45:00.0000000Z"
        Editor: finance@company.com
        File_x0020_Size: 125890
      - number: "4.0"
        Modified: "2025-01-18T11:30:00.0000000Z"
        Editor: finance@company.com
        File_x0020_Size: 118500
      - number: "3.0"
        Modified: "2025-01-15T09:00:00.0000000Z"
        Editor: manager@company.com
        File_x0020_Size: 95200
      - number: "2.0"
        Modified: "2025-01-10T14:20:00.0000000Z"
        Editor: analyst@company.com
        File_x0020_Size: 78400
      - number: "1.0"
        Modified: "2025-01-05T08:00:00.0000000Z"
        Editor: analyst@company.com
        File_x0020_Size: 45000
```

Corresponding folder structure:
```
d:\files-datastore\SillenoProjectControl-Mirror\
└── 2025\
    └── Q1\
        ├── __spo_store\
        │   └── Budget.xlsx.versions\
        │       ├── a1b2c3d4_v001.0_Budget.xlsx
        │       ├── a1b2c3d4_v002.0_Budget.xlsx
        │       ├── a1b2c3d4_v003.0_Budget.xlsx
        │       ├── a1b2c3d4_v004.0_Budget.xlsx
        │       └── a1b2c3d4_v005.0_Budget.xlsx
        ├── Budget.xlsx           (live copy)
        └── Budget.xlsx.meta      (YAML metadata)
```

---

## Quick Reference: Sync Operations

| SharePoint Event | Local Action | Entity Status |
|------------------|--------------|---------------|
| New file appears | Create `.meta`, download versions | `current` |
| Existing file updated (new version) | Update `.meta`, download new version | `current` (unchanged) |
| File replaced (same name, new GUID) | Add new entity, mark old as `superseded` | Old: `superseded`, New: `current` |
| File deleted | Update `.meta`, clear `currentEntity` | `deleted` |
| File moved | Old path: `deleted`, New path: `current` (new entity) | Treated as delete + create |

---

## Usage Examples

### Metadata-only sync (fast inventory)
```powershell
.\Sync-SPOVersions-v04.ps1 -Mode 'metadata-only'
```

### Full sync with all versions
```powershell
.\Sync-SPOVersions-v04.ps1 -Mode 'download-files' -VersionsToDownload 0
```

### Sync only last 3 versions per file
```powershell
.\Sync-SPOVersions-v04.ps1 -Mode 'download-files' -VersionsToDownload 3
```

### Custom mirror location
```powershell
.\Sync-SPOVersions-v04.ps1 -MirrorRoot 'E:\backup\sharepoint-mirror'
```

---

*Generated from `Sync-SPOVersions-v04.ps1` analysis*
