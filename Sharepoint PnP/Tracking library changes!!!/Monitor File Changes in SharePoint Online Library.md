The problem is: **internal moves show as `Update` (ChangeType 2)**, not as a move type. There's no way to distinguish "file content edited" from "file moved between folders" using just the change log.

To detect internal moves, you **must** add `Update = $true` and then fetch the file path to see if it changed:

```powershell
$list = Get-PnPList -Identity "Shared Documents"

# Time range (last 7 days)
$startDate = [DateTime]::UtcNow.AddDays(-7)
$changeToken = "1;3;$($list.Id);$($startDate.Ticks);-1"

# Query for file changes (Update included for internal moves)
$body = @{
    query = @{
        Add              = $true
        Update           = $true
        DeleteObject     = $true
        Rename           = $true
        Move             = $true
        Item             = $true
        File             = $true
        RecursiveAll     = $true
        ChangeTokenStart = @{
            StringValue = $changeToken
        }
    }
}

# Get changes
$changes = Invoke-PnPSPRestMethod -Method Post `
    -Url "/_api/web/lists(guid'$($list.Id)')/getchanges" `
    -Content $body

# Display results with file names
$changeTypes = @{
    1 = "Added"
    2 = "Updated/Moved"
    3 = "Deleted"
    4 = "Renamed"
    5 = "Moved Out"
    6 = "Moved In"
}

$changes.value | ForEach-Object { 
    [PSCustomObject]@{
        Action   = $changeTypes[[int]$_.ChangeType]
        Time     = $_.Time
        ItemId   = $_.ItemId
        UniqueId = $_.UniqueId 
    } 
} | Format-Table -AutoSize
```

## The Limitation

`ChangeType = 2` means **either**:

- File content was edited
- File was moved between folders (path changed)
- File metadata was updated

The change log doesn't tell you which. You'd need to track paths yourself to know if it was a move vs edit.