

### File Identifiers in SharePoint

SharePoint assigns multiple identifiers to each file:

|Identifier|Type|Scope|Persistence|
|---|---|---|---|
|**UniqueId**|GUID|Site collection|Survives renames, moves within site collection|
|**ID**|Integer|List/Library|Sequential, reused after deletion|
|**ETag**|String|Version-specific|Changes with every modification|
|**ctag**|String|Item-specific|Changes when content or metadata changes|

The **UniqueId** (also called **GUID** or **UniqueIdentifier**) is the primary persistent identifier for a file's lifecycle.

---

### Replacement Scenarios and UniqueId Behavior

#### Scenario 1: Overwrite via Upload (Same Filename)

**Method:** Upload a file with identical name to existing file, confirm overwrite.

**Result:** UniqueId is **preserved**. SharePoint treats this as a new version of the existing item.

```
Before: Document.docx  →  UniqueId: a1b2c3d4-...
After:  Document.docx  →  UniqueId: a1b2c3d4-...  (same)
                          Version: 2.0 (incremented)
```

#### Scenario 2: OneDrive Sync Client — File Modification

**Method:** Edit file locally in synced folder; sync client uploads changes.

**Result:** UniqueId is **preserved**. The sync client performs an in-place update.

#### Scenario 3: OneDrive Sync Client — Delete and Add New File

**Method:** Delete file in synced folder, add different file with same name.

**Result:** UniqueId **changes**. The sync client interprets this as deletion + creation.

```
Before: Report.xlsx    →  UniqueId: a1b2c3d4-...
(delete locally)
(add new Report.xlsx)
After:  Report.xlsx    →  UniqueId: e5f6g7h8-...  (NEW)
```

#### Scenario 4: SharePoint UI — "Replace" Option

**Method:** Select file → Upload → Replace existing file.

**Result:** UniqueId is **preserved**. Explicit replacement maintains identity.

#### Scenario 5: Delete Then Upload

**Method:** Delete the original file, then upload a new file (even with same name).

**Result:** UniqueId **changes**. SharePoint creates a new item entry.

#### Scenario 6: Rename Replacement Trick

**Method:** Rename original to `file_old.docx`, upload new `file.docx`, delete old.

**Result:** New file gets **new UniqueId**. The original UniqueId is now associated with `file_old.docx` (until deleted).

---

### Summary Table

|Replacement Method|UniqueId Preserved?|Version Incremented?|
|---|---|---|
|Upload overwrite (same name)|Yes|Yes|
|OneDrive sync (edit in place)|Yes|Yes|
|OneDrive sync (delete + add)|**No**|N/A (new item)|
|SharePoint UI "Replace"|Yes|Yes|
|Delete then upload|**No**|N/A (new item)|
|Move within site collection|Yes|No|
|Copy|**No**|N/A (new item)|

---

### Verification Method

To check a file's UniqueId before and after replacement:

**PowerShell (PnP Module):**

```powershell
Connect-PnPOnline -Url "https://tenant.sharepoint.com/sites/SiteName" -Interactive

# Get UniqueId
$file = Get-PnPFile -Url "/sites/SiteName/Shared Documents/Document.docx" -AsListItem
$file["UniqueId"]
```

**REST API:**

```
GET https://tenant.sharepoint.com/sites/SiteName/_api/web/GetFileByServerRelativeUrl('/sites/SiteName/Shared Documents/Document.docx')?$select=UniqueId
```

---

### Implication

If your workflow depends on stable file identity (e.g., external systems referencing SharePoint files by GUID), ensure replacement is performed via **overwrite** or **explicit replace**, not delete-and-recreate. OneDrive sync client behavior can inadvertently break references if users delete and re-add files locally.




## Distinction: Scenario 3 vs Scenario 5

### Core Difference: Operation Context

|Aspect|Scenario 3|Scenario 5|
|---|---|---|
|**Interface**|Local file system (OneDrive synced folder)|SharePoint directly (browser, API, mapped drive)|
|**Actor**|OneDrive Sync Client interprets file system events|User or application acts on SharePoint|
|**Operation visibility**|Implicit — user manipulates local files|Explicit — user interacts with SharePoint|

### Scenario 3: OneDrive Sync Client — Delete and Add New File

**Context:** User operates within a locally synced OneDrive/SharePoint folder (e.g., `C:\Users\Name\OneDrive - Company\Documents\`).

**Sequence:**

```
1. User deletes "Report.xlsx" from local synced folder
   └─► Sync client queues: DELETE /sites/Site/Documents/Report.xlsx

2. User copies/creates new "Report.xlsx" in same local folder
   └─► Sync client queues: UPLOAD /sites/Site/Documents/Report.xlsx (new file)

3. Sync client executes queued operations against SharePoint
   └─► SharePoint: Item deleted, new item created
```

**Behavioral nuance:** The OneDrive sync client does not inherently understand user intent. It observes file system events (delete, create) and translates them into discrete SharePoint operations. Even if the user's intent was "replace," the sync client sees two separate events.

**Timing dependency:** If the delete and create occur in rapid succession, the sync client _may_ optimize this into a single replace operation (preserving UniqueId). However, this is not guaranteed and depends on:

- Sync client version
- Time delta between operations
- File content similarity heuristics

### Scenario 5: Delete Then Upload

**Context:** User operates directly on SharePoint via web browser, REST API, PowerShell, or WebDAV-mapped drive.

**Sequence:**

```
1. User deletes "Report.xlsx" via SharePoint UI or API
   └─► SharePoint: Item moved to Recycle Bin, UniqueId retained there

2. User uploads new "Report.xlsx" via SharePoint UI or API
   └─► SharePoint: New item created with new UniqueId
```

**Behavioral nuance:** These are explicit, discrete operations executed directly against SharePoint. No intermediary (sync client) interprets or potentially optimizes them.

### Why Both Exist as Separate Scenarios

The distinction matters for troubleshooting and documentation:

|Scenario|User Perception|Actual Behavior|
|---|---|---|
|3|"I replaced the file locally"|Sync client may or may not preserve identity|
|5|"I deleted then uploaded in SharePoint"|Always results in new identity|

**Scenario 3** highlights the **unpredictability** introduced by the sync client as an intermediary. Users often assume local file replacement equals SharePoint file replacement — this is false.

**Scenario 5** represents the **deterministic** baseline: explicit delete + explicit upload always yields a new UniqueId.

### Outcome Equivalence

From SharePoint's perspective, the **end result is identical** in both scenarios when the sync client does not optimize:

```
DELETE item (UniqueId: A) → Item removed (or sent to Recycle Bin)
CREATE item (same name)   → New item (UniqueId: B)
```

The distinction lies in **how the user arrives at that outcome** and **whether they are aware** that identity will be lost.