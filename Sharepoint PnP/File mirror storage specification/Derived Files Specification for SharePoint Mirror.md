## Preamble: Locating Source Files and Derived Output Paths

### Finding Office Files to Process

Office source files exist in two locations within the mirror structure:

1. **Live copies** reside in the main folder hierarchy at `<MirrorRoot>\<FolderPath>\<FileName>.<ext>`
2. **Historical versions** reside in `<MirrorRoot>\<FolderPath>\__spo_store\<FileName>.<ext>.versions\`

Version binaries follow the naming pattern:

```
<UniqueIdPrefix>_v<PaddedVersion>_<FileName>.<ext>
```

Target extensions for conversion: `.xlsx`, `.xls`, `.pptx`, `.ppt`

### Where to Place Derived Files

All derived files belong in `__spo_store`, in sibling folders alongside the `.versions` folder:

|Derived Type|Folder Suffix|Contents|
|---|---|---|
|Visual renderings|`.rendered`|PDF files and per-page JPEG bitmaps|
|Extracted content|`.extracted`|JSON files containing text and structure|

---

## Folder Structure Specification

```
<MirrorRoot>\
└── <FolderPath>\
    ├── __spo_store\
    │   ├── <FileName>.<ext>.versions\
    │   │   └── <UniqueIdPrefix>_v<PaddedVersion>_<FileName>.<ext>
    │   │
    │   ├── <FileName>.<ext>.rendered\
    │   │   ├── <UniqueIdPrefix>_v<PaddedVersion>_<FileName>.pdf
    │   │   ├── <UniqueIdPrefix>_v<PaddedVersion>_<FileName>_p001.jpg
    │   │   ├── <UniqueIdPrefix>_v<PaddedVersion>_<FileName>_p002.jpg
    │   │   └── ...
    │   │
    │   └── <FileName>.<ext>.extracted\
    │       └── <UniqueIdPrefix>_v<PaddedVersion>_<FileName>.json
    │
    ├── <FileName>.<ext>           (live copy)
    └── <FileName>.<ext>.meta      (YAML metadata)
```

---

## Naming Conventions

### Base Pattern (inherited from version binaries)

```
<UniqueIdPrefix>_v<PaddedVersion>_<FileName>.<derived-ext>
```

|Component|Format|Example|
|---|---|---|
|`UniqueIdPrefix`|First 8 characters of SharePoint GUID, lowercase|`a1b2c3d4`|
|`PaddedVersion`|`v` + 3-digit zero-padded major + `.` + minor|`v003.0`, `v012.0`|
|`FileName`|Original filename without extension|`Budget`|
|`derived-ext`|Extension appropriate to derived type|`.pdf`, `.jpg`, `.json`|

### Page Bitmap Extension

For per-page JPEG renderings, append `_p<PaddedPage>` before the extension:

```
<UniqueIdPrefix>_v<PaddedVersion>_<FileName>_p<PaddedPage>.jpg
```

|Component|Format|Example|
|---|---|---|
|`PaddedPage`|3-digit zero-padded page number, 1-indexed|`p001`, `p042`|

---

## File Type Summary

|File|Location|Naming Example|
|---|---|---|
|Source version|`.versions\`|`a1b2c3d4_v003.0_Budget.xlsx`|
|PDF rendering|`.rendered\`|`a1b2c3d4_v003.0_Budget.pdf`|
|Page bitmap|`.rendered\`|`a1b2c3d4_v003.0_Budget_p001.jpg`|
|Extracted content|`.extracted\`|`a1b2c3d4_v003.0_Budget.json`|

---

## Complete Example

Source file: `Budget.xlsx` with 3 versions, version 3 has 2 pages

```
D:\Mirror\
└── 2025\Q1\
    ├── __spo_store\
    │   ├── Budget.xlsx.versions\
    │   │   ├── a1b2c3d4_v001.0_Budget.xlsx
    │   │   ├── a1b2c3d4_v002.0_Budget.xlsx
    │   │   └── a1b2c3d4_v003.0_Budget.xlsx
    │   │
    │   ├── Budget.xlsx.rendered\
    │   │   ├── a1b2c3d4_v001.0_Budget.pdf
    │   │   ├── a1b2c3d4_v001.0_Budget_p001.jpg
    │   │   ├── a1b2c3d4_v002.0_Budget.pdf
    │   │   ├── a1b2c3d4_v002.0_Budget_p001.jpg
    │   │   ├── a1b2c3d4_v002.0_Budget_p002.jpg
    │   │   ├── a1b2c3d4_v003.0_Budget.pdf
    │   │   ├── a1b2c3d4_v003.0_Budget_p001.jpg
    │   │   └── a1b2c3d4_v003.0_Budget_p002.jpg
    │   │
    │   └── Budget.xlsx.extracted\
    │       ├── a1b2c3d4_v001.0_Budget.json
    │       ├── a1b2c3d4_v002.0_Budget.json
    │       └── a1b2c3d4_v003.0_Budget.json
    │
    ├── Budget.xlsx
    └── Budget.xlsx.meta
```

---

## Path Construction Reference

Given a source version binary at:

```
<MirrorRoot>\<FolderPath>\__spo_store\<FileName>.<ext>.versions\<VersionBinaryName>
```

Derived file paths are constructed by:

1. **PDF**: Replace `.versions` with `.rendered`, change extension to `.pdf`
2. **JPEG**: Replace `.versions` with `.rendered`, insert `_p###` before extension, change to `.jpg`
3. **JSON**: Replace `.versions` with `.extracted`, change extension to `.json`

PowerShell example:

```powershell
$versionPath = "D:\Mirror\2025\Q1\__spo_store\Budget.xlsx.versions\a1b2c3d4_v003.0_Budget.xlsx"

$baseName = [System.IO.Path]::GetFileNameWithoutExtension($versionPath)  # a1b2c3d4_v003.0_Budget
$parentFolder = Split-Path (Split-Path $versionPath)                      # ...\__spo_store
$sourceFileName = (Split-Path (Split-Path $versionPath) -Leaf) -replace '\.versions$', ''  # Budget.xlsx

$pdfPath  = Join-Path $parentFolder "$sourceFileName.rendered\$baseName.pdf"
$jpgPath  = Join-Path $parentFolder "$sourceFileName.rendered\$baseName_p001.jpg"  # per page
$jsonPath = Join-Path $parentFolder "$sourceFileName.extracted\$baseName.json"
```