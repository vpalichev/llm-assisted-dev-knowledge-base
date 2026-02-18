```powershell
function Get-FileVersionHistory {
    <#
    .SYNOPSIS
        Retrieves complete version history metadata for a SharePoint file
        via a single REST round-trip.

    .PARAMETER ListName
        Display name of the document library.

    .PARAMETER UniqueId
        The file's UniqueId (GUID). Bypasses the 5,000-item list view threshold.

    .PARAMETER ItemId
        The list item integer ID. Use when UniqueId is unavailable.

    .OUTPUTS
        Array of PSCustomObject with version metadata.
    #>
    [CmdletBinding(DefaultParameterSetName = 'ByUniqueId')]
    param(
        [Parameter(Mandatory)]
        [string]$ListName,

        [Parameter(Mandatory, ParameterSetName = 'ByUniqueId')]
        [guid]$UniqueId,

        [Parameter(Mandatory, ParameterSetName = 'ById')]
        [int]$ItemId
    )

    # Build the item path based on identifier type
    $itemPath = switch ($PSCmdlet.ParameterSetName) {
        'ByUniqueId' { "GetItemByUniqueId(guid'$UniqueId')" }
        'ById'       { "items($ItemId)" }
    }

    # Single REST call: all versions + expanded Editor in one round-trip
    $url = "/_api/web/lists/getbytitle('$ListName')/$itemPath/Versions" +
           '?$select=VersionId,VersionLabel,Created,IsCurrentVersion,' +
           'Editor/Title,Editor/EMail' +
           '&$expand=Editor'

    $response = Invoke-PnPSPRestMethod -Url $url -ContentType 'application/json;odata=nometadata'

    # Normalize: REST returns .value array or a single object
    $versions = if ($response.value) { $response.value } else { @($response) }

    foreach ($v in $versions) {
        [PSCustomObject]@{
            VersionId   = $v.VersionId
            Label       = $v.VersionLabel
            IsCurrent   = $v.IsCurrentVersion
            Created     = [datetime]$v.Created
            EditorName  = $v.Editor.Title
            EditorEmail = $v.Editor.EMail
        }
    }
}
```

**Usage:**

```powershell
# By UniqueId (threshold-safe)
Get-FileVersionHistory -ListName 'Documents' -UniqueId 'a1b2c3d4-e5f6-7890-abcd-ef0123456789'

# By integer ID
Get-FileVersionHistory -ListName 'Documents' -ItemId 42

# Export to CSV
Get-FileVersionHistory -ListName 'Documents' -UniqueId $guid |
    Export-Csv -Path 'C:\temp\versions.csv' -NoTypeInformation
```

**Key design decisions:**

- **`odata=nometadata`** in `ContentType` strips OData type annotations from the response, reducing payload size and producing cleaner `PSCustomObject` properties (no `odata.type`, `odata.editLink` noise).
- **`$expand=Editor`** inlines the author object server-side, eliminating the N+1 round-trip problem entirely. `Editor` is the user who created that version (the field name is counterintuitive — SharePoint names the version author `Editor`, not `Author`).
- **`GetItemByUniqueId`** path is the default parameter set because it bypasses the list view threshold, consistent with the REST pattern discussed earlier.
- **`[guid]` type constraint** on `$UniqueId` validates the GUID format at parameter binding time rather than failing silently at the API layer.
- **Total round-trips: 1.** The `Invoke-PnPSPRestMethod` call is the only HTTP request.