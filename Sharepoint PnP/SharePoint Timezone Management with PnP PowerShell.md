## Core Concepts

SharePoint stores datetime values internally in **UTC**, but displays them according to the **site's regional settings timezone**. The conundrum arises because:

1. **CAML queries** interpret datetime literals according to the site's configured timezone unless explicitly instructed otherwise
2. **PnP PowerShell** returns `DateTime` objects that may be converted to local time by .NET
3. The **`IncludeTimeValue`** and **`StorageTZ`** CAML attributes control interpretation behavior
---
## Retrieving the Current Site Timezone

```powershell
# Connect to your site
Connect-PnPOnline -Url "https://yourtenant.sharepoint.com/sites/yoursite" -Interactive

# Method 1: Get the regional settings timezone
$web = Get-PnPWeb -Includes RegionalSettings, RegionalSettings.TimeZone
$tz = $web.RegionalSettings.TimeZone

Write-Host "Timezone ID: $($tz.Id)"
Write-Host "Description: $($tz.Description)"
Write-Host "Bias (minutes from UTC): $($tz.Bias)"
```

```powershell
# Method 2: List all available SharePoint timezones
$web = Get-PnPWeb -Includes RegionalSettings.TimeZones
$web.RegionalSettings.TimeZones | Select-Object Id, Description, Bias | Sort-Object Bias
```

**UTC timezone in SharePoint** has **ID = 39** (displayed as "(UTC) Coordinated Universal Time").

---

## Changing the Site Timezone to UTC

```powershell
# Set timezone to UTC (ID 39)
Set-PnPWeb -LocaleId 1033 -TimeZone 39

# Verify the change
$web = Get-PnPWeb -Includes RegionalSettings.TimeZone
$web.RegionalSettings.TimeZone.Description
```

> **Caveat**: Changing the site timezone affects **display only**—existing stored values remain in UTC internally. However, users will see datetime values shift in the UI.

---

## CAML Query UTC Handling

The critical attributes for UTC-aware CAML queries:

|Attribute|Purpose|
|---|---|
|`IncludeTimeValue='TRUE'`|Include time component in comparison (not just date)|
|`StorageTZ='TRUE'`|Interpret the provided datetime as **UTC** (storage timezone)|

### Correct UTC CAML Query Pattern

```powershell
# Generate UTC timestamp in ISO 8601 format
$utcNow = [DateTime]::UtcNow.ToString("yyyy-MM-ddTHH:mm:ssZ")

$camlQuery = @"
<View>
    <Query>
        <Where>
            <Geq>
                <FieldRef Name='Created' />
                <Value Type='DateTime' IncludeTimeValue='TRUE' StorageTZ='TRUE'>$utcNow</Value>
            </FieldRef>
        </Where>
    </Query>
</View>
"@

$items = Get-PnPListItem -List "YourList" -Query $camlQuery
```

**Without `StorageTZ='TRUE'`**, SharePoint interprets the datetime literal according to the site's regional timezone, causing offset errors.

---

## Ensuring PnP PowerShell Returns UTC

PnP returns .NET `DateTime` objects with `Kind = Unspecified` or `Local`. Force UTC handling:

```powershell
$items = Get-PnPListItem -List "Documents" -Fields "Created", "Modified"

foreach ($item in $items) {
    # SharePoint returns UTC but .NET may not mark it as such
    $createdRaw = $item["Created"]
    
    # Explicitly specify the value is UTC
    $createdUtc = [DateTime]::SpecifyKind($createdRaw, [DateTimeKind]::Utc)
    
    Write-Host "Created (UTC): $($createdUtc.ToString('u'))"
}
```

---

## Pitfalls and Quirks (Windows PowerShell)

|Issue|Mitigation|
|---|---|
|String interpolation in CAML with special characters|Use `@""@` here-strings or escape properly|
|`Get-Date` returns local time by default|Use `[DateTime]::UtcNow` or `(Get-Date).ToUniversalTime()`|
|Daylight Saving Time transitions|Always use UTC internally; convert only for display|
|PowerShell 5.1 vs 7.x datetime handling differences|Test explicitly; PS 7.x has improved ISO 8601 parsing|

---

## Recommended Practice

Set your site to UTC (ID 39) **only if** all users accept UTC display. Otherwise, keep the site timezone user-friendly and enforce `StorageTZ='TRUE'` in all CAML queries to guarantee consistent UTC interpretation regardless of site configuration.


# Correcting SharePoint Timezone: US Pacific → Kazakhstan

## Diagnose Current Configuration

```powershell
Connect-PnPOnline -Url "https://yourtenant.sharepoint.com/sites/yoursite" -Interactive

# Confirm current timezone setting
$web = Get-PnPWeb -Includes RegionalSettings, RegionalSettings.TimeZone
Write-Host "Current: $($web.RegionalSettings.TimeZone.Description)"
Write-Host "Bias: $($web.RegionalSettings.TimeZone.Bias) minutes from UTC"
```

Expected output for US Pacific: `(UTC-08:00) Pacific Time (US and Canada)` with Bias = 480.

---

## Find Kazakhstan Timezone ID

```powershell
# List all available timezones, filter for relevant options
$web = Get-PnPWeb -Includes RegionalSettings.TimeZones
$web.RegionalSettings.TimeZones | 
    Where-Object { $_.Description -match "Astana|Almaty|UTC\+05|UTC\+06" -or $_.Id -eq 39 } |
    Select-Object Id, Description, Bias |
    Format-Table -AutoSize
```

**Relevant SharePoint Timezone IDs:**

|ID|Description|UTC Offset|
|---|---|---|
|**47**|(UTC+05:00) Astana|UTC+5|
|**46**|(UTC+06:00) Almaty (legacy)|UTC+6|
|**39**|(UTC) Coordinated Universal Time|UTC+0|

> **Note**: Kazakhstan unified on UTC+5 in March 2024. SharePoint's "Astana" timezone (ID 47) is the correct choice for most of Kazakhstan. Almaty region historically used UTC+6, but this changed.

---

## Change to Kazakhstan Time (Astana, UTC+5)

```powershell
# Set timezone to Astana (UTC+05:00)
Set-PnPWeb -TimeZone 47

# Verify
$web = Get-PnPWeb -Includes RegionalSettings.TimeZone
Write-Host "Updated to: $($web.RegionalSettings.TimeZone.Description)"
```

---

## Alternative: Set to Pure UTC

If you prefer timezone-neutral storage and display (recommended for multinational teams or API-heavy workflows):

```powershell
# Set timezone to UTC
Set-PnPWeb -TimeZone 39

# Verify
$web = Get-PnPWeb -Includes RegionalSettings.TimeZone
Write-Host "Updated to: $($web.RegionalSettings.TimeZone.Description)"
```

---

## Impact Assessment

|Aspect|Effect of Timezone Change|
|---|---|
|**Stored data**|No change—internal storage remains UTC|
|**UI display**|All datetime columns shift by offset difference (Pacific → Astana = +13 hours)|
|**CAML queries without `StorageTZ`**|Interpretation baseline changes to new timezone|
|**Existing alerts/workflows**|May trigger at unexpected times if based on display time|
|**Audit logs**|Unaffected (logged in UTC)|

---

## Recommendation

Given you're in Kazakhstan with a site misconfigured to US Pacific:

1. **For user-facing sites**: Set to **ID 47 (Astana)** so timestamps display correctly for local users
2. **For integration/API-heavy sites**: Set to **ID 39 (UTC)** and handle display conversion client-side
3. **For CAML queries**: Always use `StorageTZ='TRUE'` regardless of site timezone to ensure predictable UTC interpretation

```powershell
# Final recommended command for Kazakhstan
Set-PnPWeb -TimeZone 47
```




---------------------------

### PowerShell DateTimeKind Values

- `Unspecified` - No timezone info (default for parsed strings)
- `Utc` - Explicitly UTC time
- `Local` - Local computer's timezone

### What Happens Without SpecifyKind()

```powershell
# SharePoint returns: 2025-02-01 14:30:00 (Kind=Unspecified, actually UTC)
$spDate = $item.FieldValues.Modified

# PowerShell assumes it's local time!
$spDate.ToLocalTime()  # Converts Unspecified→Local (WRONG!)
# Result: 2025-02-01 16:30:00 (adds 2 hours on UTC+2 system)

# With SpecifyKind
$utcDate = [DateTime]::SpecifyKind($spDate, [DateTimeKind]::Utc)
$utcDate.ToLocalTime()  # Converts UTC→Local (CORRECT!)
# Result: 2025-02-01 16:30:00 (correct conversion)
```

---

## Sources

- [Get-PnPListItem returns wrong value for DateTime properties · Issue #934](https://github.com/SharePoint/PnP-PowerShell/issues/934)
- [SharePoint Online: PowerShell to Get-Set Date Field Value](https://www.sharepointdiary.com/2018/01/sharepoint-online-powershell-to-get-set-date-field-value.html)
- [How to work with SharePoint datetime localization/timezones](https://julieturner.net/post/sharepoint-datetime/)
- [Convert DateTime Field's UTC Value To The Correct Time Zone](https://www.c-sharpcorner.com/article/convert-datetime-fields-utc-value-to-the-correct-time-zone-in-an-spfx-webpart/)

---

## Summary

**Your current implementation in `ConvertTo-ReadablePnPListItem.ps1` is CORRECT!**

- SharePoint Online **does return UTC** in FieldValues
- The issue is that PowerShell doesn't always know it's UTC
- Use `[DateTime]::SpecifyKind($dateTime, [DateTimeKind]::Utc)` to mark it
- This prevents incorrect timezone conversions
- Your codebase already follows this best practice

User approved the plan

Perfect! I've completed my research on SharePoint Online DateTime handling. Here's the comprehensive answer to your question:

## Answer: Yes, SharePoint Online Returns UTC ✅

**SharePoint Online DOES return DateTime values in UTC** when you use `Get-PnPListItem`, but there's an important catch: the `DateTimeKind` property is often not explicitly set, which can cause PowerShell to misinterpret the values.

## Your Current Implementation is CORRECT!

The code you already have in [ConvertTo-ReadablePnPListItem.ps1:70-75](vscode-webview://16rkfkgf77vh70hu4jofnr27e51ln002l5acl0go7ha0520h7nt4/index.html?id=ec230be1-ef29-4158-97bf-1a0b3af0efd6&parentId=3&origin=15d247ee-ad8c-4607-a11b-ebe4a0aa9930&swVersion=4&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&session=cf4c8a0d-b7ad-47a3-9834-a554a1e2a27c) is the **best practice**:

```powershell
Created  = if ($item.FieldValues.Created) {
    [DateTime]::SpecifyKind($item.FieldValues.Created, [DateTimeKind]::Utc)
} else { $null }
```

## Why Use `SpecifyKind()` Instead of `ToUniversalTime()`?

This is the key insight:

- **SharePoint values are ALREADY in UTC** - no conversion needed
- `SpecifyKind()` just **marks** the existing value as UTC
- `ToUniversalTime()` would try to **convert** from local → UTC (which would be incorrect!)

## The Problem Without It

When SharePoint returns a DateTime like `2025-02-01 14:30:00` with `Kind=Unspecified`:

- PowerShell might assume it's local time
- Converting to local time would add hours again (double conversion!)
- On a UTC+2 system, you'd see `16:30:00` instead of `14:30:00`

## Summary

Your implementation is already following best practices. I've created a detailed research document in the plan file that includes:

- Evidence from GitHub issues and community sources
- Comparison of different DateTime handling patterns in your codebase
- Best practice examples for different scenarios
- Technical explanation of why this matters