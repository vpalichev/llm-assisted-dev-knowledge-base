# Two history systems, one file: why SharePoint and Excel disagree

**SharePoint Online's version history and Excel for the web's "Show Changes" pane are entirely separate systems that track different things at different granularity levels.** SharePoint records periodic whole-file snapshots attributed to a single user, while Excel maintains a cell-level revision log from the co-authoring data stream that captures every individual edit by every user for up to 60 days. This architectural split explains why Excel's in-app history consistently shows more editors and more granular changes than the SharePoint document library — and why PnP PowerShell cmdlets like `Get-PnPFileVersion` return the sparser SharePoint data rather than the richer Excel data.

The discrepancy is not a bug. These systems serve fundamentally different purposes: SharePoint versioning exists for file-level recovery and storage management, while Excel's revision log exists for collaborative transparency. Understanding where each stores its data, what triggers entries, and how to access each programmatically is essential for anyone building audit workflows or investigating edit attribution in shared workbooks.

## SharePoint coalesces dozens of saves into periodic snapshots

SharePoint Online's version history operates at the **file level**. Each version is a complete, restorable snapshot of the entire workbook stored in SharePoint's shredded storage infrastructure (content database). When AutoSave is enabled — which is the default for all files opened from SharePoint Online or OneDrive — Excel continuously pushes differential changes to the server every few seconds. But SharePoint does **not** create a new numbered version on every micro-save.

Instead, SharePoint uses **version coalescing**. According to Microsoft's AutoSave FAQ, the first edit in a session creates an immediate new version, but subsequent versions are only created "periodically (about every 10 minutes)" for the remainder of the editing session. During co-authoring sessions, the interval stretches to roughly **30 minutes**, controlled by an internal `CoauthoringVersionPeriod` parameter that is not configurable in SharePoint Online. In practice, practitioners report the actual interval varies — sometimes every few minutes during heavy editing, sometimes longer during light activity — suggesting Microsoft uses a heuristic algorithm rather than a strict fixed timer.

The critical consequence for edit attribution: **each SharePoint version is attributed to exactly one user** — whoever last committed changes at the moment the version snapshot was captured. If three people are co-authoring simultaneously, a given version entry might credit only User A, even though Users B and C also made edits during that coalescing window. Their contributions are baked into the file content but invisible in the version metadata. This single-user attribution is the primary reason SharePoint's version history appears to show fewer editors than Excel's in-app history.

PnP PowerShell cmdlets — `Get-PnPFileVersion` for file versions, or loading the `Versions` property on a list item via `Get-PnPProperty` — return exactly the same data visible in the SharePoint UI: version label, timestamp, single "Modified By" user, file size, and check-in comments. There is no hidden richer dataset behind these cmdlets.

## Excel's revision log tracks every cell edit independently

Excel for the web's "Show Changes" feature (accessible via the **Review tab** or by right-clicking a cell) draws from an entirely different data source: a **server-side co-authoring revision log** maintained by Microsoft's collaboration service. This log captures every individual cell-level operation — who changed which cell, when, and the old and new values — with full per-user attribution regardless of how SharePoint's versioning coalesces those changes.

This revision log is a byproduct of the same real-time synchronization stream that powers live co-authoring. As one technical analysis noted: "Rather than saving the entire workbook, Excel is tracking that a particular person changed a particular cell, at a particular date and time, with a particular new value or formula. While this tiny data stream was originally created to allow collaboration, it now also powers the Show Changes feature." The data is stored **server-side in Microsoft's cloud infrastructure**, not embedded in the .xlsx file's OOXML structure and not in the SharePoint content database.

Key characteristics of the Excel revision log:

- **Retention**: Up to **60 days** of cell-level changes, a hard limit set by Microsoft that cannot be configured
- **Scope**: Tracks cell values, formulas, moves, sorts, insertions, and deletions — but **not** chart edits, PivotTable operations, formatting changes, shapes, or filter operations
- **Fragility**: Opening the workbook in older Excel versions (Excel 2019 or earlier) clears the entire change history; the file owner can also manually reset it via File → Info
- **Attribution**: Every edit is individually attributed to its actual author, solving the co-authoring attribution problem that plagues SharePoint's version history

Microsoft's documentation explicitly distinguishes "revision logs" from "version history" as separate data stores, confirming they are architecturally independent systems. The revision log is not an extension of SharePoint versioning — it is a parallel system operating at a different layer of the stack.

## How the protocols and architecture create the split

The separation traces back to the protocol architecture. When Excel for the web communicates with SharePoint, it uses the **WOPI (Web Application Open Platform Interface)** protocol — a REST-based protocol whose `PutFile` operation writes updated file content back to SharePoint. SharePoint then decides whether to create a new version based on its coalescing logic. Desktop Excel clients use **MS-FSSHTTP (File Synchronization via SOAP over HTTP)**, which includes explicit `CoauthVersioning` attributes and `CoalesceHResult` responses — direct protocol-level evidence of the coalescing mechanism.

Both protocols ultimately write to the same SharePoint version store. But **Excel's internal co-authoring engine maintains its own change stream independently** of what it writes to SharePoint. This stream powers both real-time co-authoring (showing other users' cursors and edits live) and the Show Changes pane. It is an application-layer log that exists above and separate from the storage-layer versioning in SharePoint.

The "Activity" section visible in the SharePoint details pane (the ⓘ icon) represents yet a third data source — a file-level activity feed tracking events like views, edits, and shares. This is closer to SharePoint's version history in granularity (file-level, not cell-level) and should not be confused with Excel's cell-level Show Changes feature.

## Neither system is a complete audit trail

**For file recovery**, SharePoint's version history is authoritative — each version is a complete, restorable file snapshot with configurable retention (default 500 major versions). **For understanding who changed what**, Excel's Show Changes is far more accurate and granular, but its 60-day limit, clearability, and incomplete scope (no formatting or chart tracking) make it unreliable as a permanent audit record.

For a true compliance-grade audit trail, the **Microsoft 365 Unified Audit Log** (accessible via the Security & Compliance Center or `Search-UnifiedAuditLog` in PowerShell) is the most authoritative source. It independently logs all file operations — `FileModified`, `FileAccessed`, `FileDeleted` — with user attribution and timestamps, retained for **180 days** (E3) or **365 days** (E5). However, even the Unified Audit Log operates at file-level granularity; it records that User X modified file Y at time Z, not which cells changed.

## Programmatic access to the cell-level history does not exist

**Excel's cell-level Show Changes data cannot be accessed through any public API.** After exhaustive review of Microsoft Graph (v1.0 and beta), the SharePoint REST API, PnP PowerShell, and the Office JavaScript API, no endpoint returns the who-changed-which-cell data that the Excel UI displays. The available programmatic options, ranked by usefulness:

- **Microsoft Graph `activities` endpoint** (beta only, limited preview): Returns file-level edit/comment/share events via `GET /drives/{drive-id}/items/{item-id}/activities`. Shows that an edit happened but not which cells changed. Not available to all tenants.
- **`getActivitiesByInterval`** (v1.0): Returns only aggregate daily counts — "3 edits by 1 user on January 2" — with no content details. The `Activities` property is frequently null for OneDrive for Business items.
- **SharePoint/Graph versions API**: `GET /drives/{drive-id}/items/{item-id}/versions` returns the same version list as the SharePoint UI and PnP PowerShell. You can download each version's binary content, but extracting cell-level diffs requires programmatically comparing .xlsx files using libraries like OpenXML SDK.
- **Unified Audit Log**: File-level operation records with 180–365 day retention. Best available programmatic source for tracking who edited when, but no cell-level detail.

The only workaround for approximating cell-level change history programmatically is to periodically download version snapshots via the versions API and diff them using an XML/spreadsheet parsing library. This approach is resource-intensive, misses changes that occurred between version snapshots (due to coalescing), and cannot attribute individual cell changes to specific users during co-authoring windows.

## Conclusion

The SharePoint-versus-Excel history discrepancy is a deliberate architectural consequence, not an oversight. SharePoint's version history is a storage-layer system optimized for file recovery with periodic snapshots and single-user attribution. Excel's Show Changes is an application-layer system built atop the co-authoring synchronization stream, designed for collaborative transparency with cell-level, per-user granularity. They share no underlying data store. The practical implications are significant: anyone relying on `Get-PnPFileVersion` or the SharePoint UI for edit auditing during co-authoring sessions will see an incomplete picture of who contributed what. Excel's richer history fills that gap but is ephemeral (60 days), fragile (clearable), and — critically — **locked behind the UI with no programmatic access**. For organizations requiring durable cell-level audit trails, the current Microsoft 365 architecture offers no complete solution; the closest approximation is combining Unified Audit Log data with periodic version snapshots and programmatic file diffing.