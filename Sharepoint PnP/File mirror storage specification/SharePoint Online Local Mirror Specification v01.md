**Version:** 0.1.0-draft  
**Date:** 2026-01-20  
**Status:** Draft

---

## 1. Overview

This specification defines a local filesystem structure for mirroring a SharePoint Online (SPO) document library, including full version history and metadata preservation. The design prioritizes:

- Accurate representation of file version histories
- Disambiguation of multiple file entities that have occupied the same path over time
- Human-readable metadata files
- Compatibility with Windows filesystems

---

## 2. Terminology

|Term|Definition|
|---|---|
|**Entity**|A discrete file object in SharePoint, uniquely identified by its `UniqueId` (GUID). An entity persists through renames and moves.|
|**Path**|The location of a file within the library hierarchy (e.g., `/Finance/Letter to lenders.docx`). A path may be occupied by different entities at different points in time.|
|**Version**|A specific revision of an entity, identified by a version number (e.g., `1.0`, `2.0`). SharePoint Online permits up to 512 major versions per entity.|
|**Timeline**|The complete version history of a single entity. Multiple timelines may exist for a single path if different entities have occupied that path over time.|
|**Meta File**|A YAML file containing metadata for all entities that have occupied a specific path.|
|**Sidecar Store**|A hidden folder (`__spo_store`) containing the actual file binaries organized by version.|

---

## 3. Folder Structure

### 3.1 General Layout

The local mirror replicates the folder hierarchy of the SharePoint library. Each folder in the mirror contains:

1. **Meta files** (`.meta` suffix) representing each file path
2. **Sidecar store** (`__spo_store` subfolder) containing versioned file binaries

```
<MirrorRoot>\
├── Engineering\
│   ├── __spo_store\
│   │   ├── General drawings list.xlsx.versions\
│   │   │   ├── e4c3c2a1_v001.0_General drawings list.xlsx
│   │   │   ├── e4c3c2a1_v002.0_General drawings list.xlsx
│   │   │   └── e4c3c2a1_v003.0_General drawings list.xlsx
│   │   └── Letter to lead engineer.docx.versions\
│   │       ├── c2b4e5a1_v001.0_Letter to lead engineer.docx
│   │       └── ...
│   ├── General drawings list.xlsx.meta
│   └── Letter to lead engineer.docx.meta
├── Finance\
│   ├── __spo_store\
│   │   └── ...
│   ├── Letter to lenders.docx.meta
│   └── Queries to banks.xlsx.meta
└── ...
```

### 3.2 Sidecar Store (`__spo_store`)

Each folder containing files SHALL have a corresponding `__spo_store` subfolder. This folder:

- Contains one `.versions` subfolder per file path
- Is named with double underscore prefix to indicate infrastructure status
- SHOULD have the Windows "hidden" attribute set

### 3.3 Versions Subfolder

Each `.versions` subfolder:

- Is named `<FileLeafRef>.versions` (e.g., `Letter to lenders.docx.versions`)
- Contains all version binaries for all entities that have occupied the corresponding path
- Groups files by entity via the UniqueId prefix in filenames

---

## 4. File Naming Conventions

### 4.1 Meta File Naming

```
<FileLeafRef>.meta
```

**Example:** `Letter to lenders.docx.meta`

The meta file name corresponds exactly to the SharePoint `FileLeafRef` field value with `.meta` appended.

### 4.2 Version Binary Naming

```
<UniqueIdPrefix>_v<PaddedVersionNumber>_<FileLeafRef>
```

**Components:**

|Component|Format|Description|
|---|---|---|
|`UniqueIdPrefix`|8 hexadecimal characters|First 8 characters of the entity's `UniqueId` GUID|
|`PaddedVersionNumber`|`vNNN.N`|Version number zero-padded to 3 digits for major, 1 digit for minor|
|`FileLeafRef`|Original filename|The file's name as stored in SharePoint|

**Examples:**

- `e4c3c2a1_v001.0_General drawings list.xlsx`
- `c2b4e5a1_v012.0_Letter to lead engineer.docx`
- `11c3b5e5_v003.5_Draft proposal.docx` (minor version example)

**Rationale:**

- 8-character UniqueId prefix provides sufficient uniqueness to disambiguate entities sharing a path while keeping filenames manageable
- Zero-padded version numbers ensure correct alphabetical sorting
- Including the original filename aids human identification when browsing the store directly

---

## 5. Meta File Structure

Meta files use YAML format for human readability and ease of parsing.

### 5.1 Schema

```yaml
# Top-level current state indicators
currentEntity: <UniqueId | null>
currentVersion: <VersionNumber | null>

# Complete history of all entities at this path
entities:
  - UniqueId: <GUID>
    FileLeafRef: <string>
    status: <EntityStatus>
    versions:
      - number: <VersionNumber>
        Modified: <ISO8601Timestamp>
        Editor: <string>
        File_x0020_Size: <integer>
      # Additional versions, ordered newest to oldest
  # Additional entities, current entity first
```

### 5.2 Field Definitions

#### Top-Level Fields

|Field|Type|Required|Description|
|---|---|---|---|
|`currentEntity`|GUID or `null`|Yes|The `UniqueId` of the entity currently occupying this path in SharePoint, or `null` if no file currently exists at this path.|
|`currentVersion`|String or `null`|Yes|The version number of the current version of the current entity, or `null` if no file currently exists at this path.|
|`entities`|Array|Yes|List of all entities that have occupied this path, ordered with current/most recent entity first.|

#### Entity Fields

|Field|Type|Required|Description|
|---|---|---|---|
|`UniqueId`|GUID|Yes|SharePoint's unique identifier for this file entity. Persists through moves and renames.|
|`FileLeafRef`|String|Yes|The filename (without path) as recorded in SharePoint.|
|`status`|EntityStatus|Yes|Current status of this entity relative to this path. See §5.3.|
|`versions`|Array|Yes|List of all versions of this entity, ordered newest to oldest.|

#### Version Fields

|Field|Type|Required|Description|
|---|---|---|---|
|`number`|String|Yes|Version number as reported by SharePoint (e.g., `"1.0"`, `"2.0"`, `"3.5"`).|
|`Modified`|ISO8601 Timestamp|Yes|Timestamp when this version was created/modified. Sourced from SharePoint `Modified` field.|
|`Editor`|String|Yes|Display name or identifier of the user who created this version. Sourced from SharePoint `Editor` field.|
|`File_x0020_Size`|Integer|Yes|File size in bytes. Sourced from SharePoint `File_x0020_Size` field.|

### 5.3 Entity Status Values

|Status|Description|
|---|---|
|`current`|This entity currently occupies this path in SharePoint. Only one entity per meta file may have this status.|
|`superseded`|This entity previously occupied this path but was deleted or replaced by another entity at the same path.|
|`moved`|This entity previously occupied this path but was moved to a different path.|
|`deleted`|This entity previously occupied this path and has been permanently deleted from SharePoint.|

### 5.4 Ordering Requirements

1. **Entities:** The entity with `status: current` (if any) SHALL appear first. Remaining entities SHOULD be ordered by most recent activity (most recent first).
    
2. **Versions:** Within each entity, versions SHALL be ordered from newest to oldest (descending by version number).
    

**Rationale:** Newest-first ordering optimizes for the common case of examining recent activity without parsing the entire file.

---

## 6. Complete Example

### 6.1 Folder Structure

```
D:\SPOMirror\Finance\
├── __spo_store\
│   └── Letter to lenders.docx.versions\
│       ├── d8ebc5c1_v001.0_Letter to lenders.docx
│       ├── d8ebc5c1_v002.0_Letter to lenders.docx
│       ├── d8ebc5c1_v003.0_Letter to lenders.docx
│       ├── d8ebc5c1_v004.0_Letter to lenders.docx
│       ├── 11c3b5e5_v001.0_Letter to lenders.docx
│       ├── 11c3b5e5_v002.0_Letter to lenders.docx
│       ├── 11c3b5e5_v003.0_Letter to lenders.docx
│       └── 11c3b5e5_v004.0_Letter to lenders.docx
└── Letter to lenders.docx.meta
```

### 6.2 Meta File Content

**File:** `Letter to lenders.docx.meta`

```yaml
currentEntity: d8ebc5c1-1234-5678-9abc-def012345678
currentVersion: "4.0"

entities:
  - UniqueId: d8ebc5c1-1234-5678-9abc-def012345678
    FileLeafRef: "Letter to lenders.docx"
    status: current
    versions:
      - number: "4.0"
        Modified: 2024-07-15T09:20:00Z
        Editor: Carol Smith
        File_x0020_Size: 19456
      - number: "3.0"
        Modified: 2024-07-01T14:00:00Z
        Editor: Carol Smith
        File_x0020_Size: 18432
      - number: "2.0"
        Modified: 2024-06-15T11:30:00Z
        Editor: Carol Smith
        File_x0020_Size: 17408
      - number: "1.0"
        Modified: 2024-06-01T10:00:00Z
        Editor: Carol Smith
        File_x0020_Size: 15360

  - UniqueId: 11c3b5e5-aaaa-bbbb-cccc-ddddeeeeaaaa
    FileLeafRef: "Letter to lenders.docx"
    status: superseded
    versions:
      - number: "4.0"
        Modified: 2024-02-20T14:30:00Z
        Editor: Alice Johnson
        File_x0020_Size: 28160
      - number: "3.0"
        Modified: 2024-02-10T16:45:00Z
        Editor: Bob Williams
        File_x0020_Size: 27648
      - number: "2.0"
        Modified: 2024-01-22T11:00:00Z
        Editor: Alice Johnson
        File_x0020_Size: 26112
      - number: "1.0"
        Modified: 2024-01-15T09:00:00Z
        Editor: Alice Johnson
        File_x0020_Size: 24576
```

### 6.3 Vacated Path Example

When no file currently exists at a path:

```yaml
currentEntity: null
currentVersion: null

entities:
  - UniqueId: 11c3b5e5-aaaa-bbbb-cccc-ddddeeeeaaaa
    FileLeafRef: "Letter to lenders.docx"
    status: moved
    versions:
      - number: "4.0"
        Modified: 2024-02-20T14:30:00Z
        Editor: Alice Johnson
        File_x0020_Size: 28160
      # ... remaining version history
```

---

## 7. Edge Cases and Behaviors

### 7.1 File Moved to Different Path

When a file entity is moved from Path A to Path B:

1. **Path A meta file:** Update the entity's `status` from `current` to `moved`. Set `currentEntity` and `currentVersion` to `null`.
2. **Path B meta file:** Create if not exists. Add the entity with `status: current`. Update `currentEntity` and `currentVersion`.
3. **Version binaries:** Binaries MAY be relocated from `PathA\__spo_store\` to `PathB\__spo_store\`, OR MAY remain in place with cross-reference mechanism (implementation-dependent).

### 7.2 File Renamed

When a file entity is renamed (same folder, different `FileLeafRef`):

1. **Old meta file:** Update entity `status` to `moved`. Set top-level fields to `null`.
2. **New meta file:** Create with entity having `status: current`.
3. **Versions folder:** A new `.versions` folder is created with the new name. Existing version binaries MAY be renamed or left in place (implementation-dependent).

### 7.3 File Deleted

When a file entity is permanently deleted from SharePoint:

1. **Meta file:** Update entity `status` to `deleted`. Set `currentEntity` and `currentVersion` to `null` if this was the current entity.
2. **Version binaries:** SHOULD be retained for archival purposes unless explicitly purged.

### 7.4 New File at Previously Occupied Path

When a new file is created at a path that previously held a different entity:

1. **Meta file:** Add new entity with `status: current` at the beginning of the `entities` array. Update `currentEntity` and `currentVersion`. Previous entity retains its status (`superseded`, `moved`, or `deleted`).

### 7.5 Path Length Considerations

Windows default path limit is 260 characters. Full version file paths may approach this limit in deeply nested structures.

**Mitigations:**

- Enable long path support in Windows (registry setting `LongPathsEnabled`)
- Use PowerShell 7+ which handles long paths natively
- Consider mirror root placement to minimize base path length

---

## 8. Future Considerations

The following topics are identified for future specification revisions:

1. **Database Layer:** Index/query system for cross-file metadata queries
2. **Sync Protocol:** Mechanism for detecting and reconciling changes with SharePoint
3. **Custom Metadata:** Extension schema for SharePoint custom columns and content types
4. **Conflict Resolution:** Handling concurrent modifications during sync
5. **Integrity Verification:** Checksums or hashes for version binary validation

---

## Appendix A: SharePoint Field Reference

|Field Name|Type|Description|
|---|---|---|
|`UniqueId`|GUID|Immutable unique identifier for the file item|
|`FileLeafRef`|String|Filename without path|
|`FileRef`|String|Server-relative path to the file|
|`FileDirRef`|String|Server-relative path to the containing folder|
|`EncodedAbsUrl`|String|Full URL-encoded absolute URL to the file|
|`File_x0020_Size`|Integer|File size in bytes|
|`Created`|DateTime|Timestamp of initial creation|
|`Modified`|DateTime|Timestamp of last modification|
|`Author`|User|User who created the item|
|`Editor`|User|User who last modified the item|
|`FSObjType`|Integer|Object type (0 = file, 1 = folder)|

---

## Appendix B: Version Number Format

SharePoint Online uses a `Major.Minor` versioning scheme:

- **Major versions** (e.g., `1.0`, `2.0`, `3.0`): Published versions
- **Minor versions** (e.g., `1.1`, `1.2`, `2.5`): Draft versions (if minor versioning is enabled)

For filename encoding, version numbers are zero-padded:

- Major: 3 digits (e.g., `001`, `012`, `512`)
- Minor: 1 digit (e.g., `0`, `5`)

**Format:** `vMMM.m` where `MMM` is zero-padded major and `m` is minor.

**Examples:**

- Version `1.0` → `v001.0`
- Version `12.0` → `v012.0`
- Version `3.5` → `v003.5`
- Version `512.0` → `v512.0` (maximum major version)