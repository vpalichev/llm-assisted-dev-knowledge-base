The core principle: **always control all three variables**—the datetime value format, the timezone interpretation flag, and the time inclusion flag.

---

## The Three Pillars of Deterministic CAML Dates

```powershell
<Value Type='DateTime' IncludeTimeValue='TRUE' StorageTZ='TRUE'>2025-01-15T14:30:00Z</Value>
```

|Attribute|Omitted Behavior|Explicit Setting|
|---|---|---|
|`IncludeTimeValue`|Compares **date only** (midnight boundaries)|`TRUE` = include time component|
|`StorageTZ`|Interprets datetime per **site timezone**|`TRUE` = interpret as **UTC**|
|DateTime format|Ambiguous parsing|ISO 8601 with `Z` suffix|

---

## Bulletproof DateTime Helper Function

```powershell
function Get-CamlDateTimeValue {
    <#
    .SYNOPSIS
        Generates a deterministic CAML datetime Value element in UTC.
    .PARAMETER DateTime
        Input datetime. Converted to UTC if not already.
    .PARAMETER IncludeTime
        Include time component in comparison. Default: $true
    .PARAMETER DateOnly
        Compare date only (midnight to midnight). Overrides IncludeTime.
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [DateTime]$DateTime,
        
        [switch]$DateOnly
    )
    
    # Normalize to UTC
    $utc = switch ($DateTime.Kind) {
        'Utc'         { $DateTime }
        'Local'       { $DateTime.ToUniversalTime() }
        'Unspecified' { [DateTime]::SpecifyKind($DateTime, [DateTimeKind]::Utc) }
    }
    
    if ($DateOnly) {
        # Date-only: use midnight, no time component
        $formatted = $utc.ToString("yyyy-MM-dd")
        return "<Value Type='DateTime' StorageTZ='TRUE'>$formatted</Value>"
    }
    else {
        # Full precision with time
        $formatted = $utc.ToString("yyyy-MM-ddTHH:mm:ssZ")
        return "<Value Type='DateTime' IncludeTimeValue='TRUE' StorageTZ='TRUE'>$formatted</Value>"
    }
}
```

---

## Building Deterministic Range Queries

```powershell
function New-CamlDateRangeQuery {
    <#
    .SYNOPSIS
        Creates a deterministic CAML query for a datetime range.
    .PARAMETER FieldName
        The internal name of the DateTime field.
    .PARAMETER StartUtc
        Range start (inclusive) in UTC.
    .PARAMETER EndUtc
        Range end (exclusive) in UTC.
    #>
    param(
        [Parameter(Mandatory)][string]$FieldName,
        [Parameter(Mandatory)][DateTime]$StartUtc,
        [Parameter(Mandatory)][DateTime]$EndUtc
    )
    
    $startValue = Get-CamlDateTimeValue -DateTime $StartUtc
    $endValue   = Get-CamlDateTimeValue -DateTime $EndUtc
    
    return @"
<View>
    <Query>
        <Where>
            <And>
                <Geq>
                    <FieldRef Name='$FieldName' />
                    $startValue
                </Geq>
                <Lt>
                    <FieldRef Name='$FieldName' />
                    $endValue
                </Lt>
            </And>
        </Where>
    </Query>
</View>
"@
}
```

### Usage Examples

```powershell
# Query items created in the last 24 hours (UTC)
$end   = [DateTime]::UtcNow
$start = $end.AddHours(-24)

$query = New-CamlDateRangeQuery -FieldName "Created" -StartUtc $start -EndUtc $end
$items = Get-PnPListItem -List "Documents" -Query $query
```

```powershell
# Query a specific calendar day in Kazakhstan (UTC+5)
# January 15, 2025 local = Jan 14 19:00 UTC to Jan 15 19:00 UTC
$localDate = [DateTime]::Parse("2025-01-15")
$startUtc  = $localDate.AddHours(-5)  # Convert KZ midnight to UTC
$endUtc    = $startUtc.AddDays(1)

$query = New-CamlDateRangeQuery -FieldName "EventDate" -StartUtc $startUtc -EndUtc $endUtc
```

---

## Common Margin Patterns

### "Today" (Deterministic UTC Day)

```powershell
function Get-CamlTodayUtcQuery {
    param([string]$FieldName = "Created")
    
    $todayStart = [DateTime]::UtcNow.Date
    $todayEnd   = $todayStart.AddDays(1)
    
    return New-CamlDateRangeQuery -FieldName $FieldName -StartUtc $todayStart -EndUtc $todayEnd
}
```

### "Today" (Deterministic Local Day in Kazakhstan)

```powershell
function Get-CamlTodayKazakhstanQuery {
    param([string]$FieldName = "Created")
    
    # Kazakhstan = UTC+5
    $kzOffset   = [TimeSpan]::FromHours(5)
    $utcNow     = [DateTime]::UtcNow
    $kzNow      = $utcNow.Add($kzOffset)
    $kzMidnight = $kzNow.Date
    
    # Convert KZ midnight back to UTC
    $startUtc = $kzMidnight.Add(-$kzOffset)
    $endUtc   = $startUtc.AddDays(1)
    
    return New-CamlDateRangeQuery -FieldName $FieldName -StartUtc $startUtc -EndUtc $endUtc
}
```

### Rolling Windows

```powershell
# Last N hours
function Get-CamlLastHoursQuery {
    param(
        [string]$FieldName = "Modified",
        [int]$Hours = 24
    )
    
    $endUtc   = [DateTime]::UtcNow
    $startUtc = $endUtc.AddHours(-$Hours)
    
    return New-CamlDateRangeQuery -FieldName $FieldName -StartUtc $startUtc -EndUtc $endUtc
}
```

---

## Avoiding the `<Today/>` Token Trap

CAML's built-in `<Today/>` token is **site-timezone-dependent** and unpredictable:

```xml
<!-- AVOID: Ambiguous, depends on site timezone -->
<Value Type='DateTime'><Today/></Value>

<!-- AVOID: Offset also relative to site timezone -->
<Value Type='DateTime'><Today OffsetDays='-7'/></Value>
```

**Always replace with explicit UTC values** using the helper functions above.

---

## Quick Reference: Deterministic Query Checklist

|Requirement|Solution|
|---|---|
|Predictable timezone|`StorageTZ='TRUE'` on every `<Value>`|
|Predictable precision|`IncludeTimeValue='TRUE'` when time matters|
|Predictable format|ISO 8601: `yyyy-MM-ddTHH:mm:ssZ`|
|Predictable "today"|Calculate UTC bounds explicitly, not `<Today/>`|
|Predictable local day|Convert local midnight → UTC, then query|
|Predictable comparisons|Use `Geq`/`Lt` pairs for ranges (half-open intervals)|

---

## Complete Working Example

```powershell
# Connect
Connect-PnPOnline -Url "https://contoso.sharepoint.com/sites/ops" -Interactive

# Define: Items modified in the last 7 days, Kazakhstan business hours only (9 AM - 6 PM local)
$days = 7
$results = @()

for ($i = 0; $i -lt $days; $i++) {
    $kzDate = [DateTime]::UtcNow.AddHours(5).Date.AddDays(-$i)
    
    # 9 AM KZ = 04:00 UTC, 6 PM KZ = 13:00 UTC
    $startUtc = $kzDate.AddHours(-5).AddHours(9)   # 9 AM KZ in UTC
    $endUtc   = $kzDate.AddHours(-5).AddHours(18)  # 6 PM KZ in UTC
    
    $query = New-CamlDateRangeQuery -FieldName "Modified" -StartUtc $startUtc -EndUtc $endUtc
    $items = Get-PnPListItem -List "Projects" -Query $query
    $results += $items
}

Write-Host "Found $($results.Count) items modified during KZ business hours"
```

This approach guarantees identical results regardless of site timezone, user locale, or server location.