# SharePoint Online: File Change Tracking — `Modified` vs `_UIVersionString`

## Core Fact

When a file is stored in SharePoint Online with AutoSave enabled (the default), two metadata fields on the list item update at **different rates**:

|Field|Updates when|Latency|
|---|---|---|
|`Modified` (datetime)|Every AutoSave sync|Seconds|
|`_UIVersionString` (e.g. "19.0")|SharePoint creates a new version snapshot|~10–30 minutes|

## Why They Differ

AutoSave pushes file content to SharePoint every few seconds. The **live file on the server is always current**.

However, SharePoint does **not** create a new numbered version on every sync. It uses **version coalescing**: the first edit in a session creates a version immediately, then subsequent versions are created only periodically (~10 min solo editing, ~30 min during co-authoring). The exact algorithm is undocumented and not configurable in SharePoint Online.

## What This Means

- `Modified` timestamp = reliable indicator of "file changed." Updates within seconds of any edit.
- `_UIVersionString` = unreliable for detecting recent changes. May stay at "19.0" for 30 minutes while the live file content has changed many times.
- `Get-PnPFile` (file download) = always returns the **live file**, not the latest version snapshot. Content is current.
- `Get-PnPFileVersion` / version history = returns **frozen snapshots**. All past versions are precise and immutable. The _latest_ version number may lag behind the live file.

## For Change Monitoring

Use `Modified`, not `_UIVersionString`:

```powershell
$item = Get-PnPListItem -List "Documents" -Id $itemId
$item.FieldValues["Modified"]            # current, updates every few seconds
$item.FieldValues["_UIVersionString"]    # may be stale for up to 30 minutes
$item.FieldValues["Editor"]              # last user whose sync hit the server
```

## Version Attribution During Co-Authoring

Each version snapshot records a **single user** in "Modified By" — whichever user's sync triggered the snapshot. Other co-authors' edits are included in the file content but not credited in that version's metadata.

## Summary

- **Live file content**: always current (seconds latency)
- **`Modified` timestamp**: always current (seconds latency)
- **Version number**: lags behind (10–30 minute coalescing window)
- **Frozen past versions**: precise and immutable snapshots
- **For monitoring: poll `Modified`, not version number**