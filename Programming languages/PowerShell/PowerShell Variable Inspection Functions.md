## Terminology and Concepts

**Collection**: In PowerShell, any object that implements `IEnumerable` and contains multiple elements. This includes arrays (`[]`), `ArrayList`, generic collections (`List<T>`, `Dictionary<K,V>`), and custom objects with enumerable properties.

**Member**: A property, method, or other attribute accessible on a PowerShell object, retrievable via `Get-Member`.

**Type Accelerator**: A shorthand notation for .NET type names (e.g., `[string]` for `System.String`).

## Inspection Function Set

powershell

```powershell
function Get-VariableInfo {
    <#
    .SYNOPSIS
    Provides comprehensive information about a variable's structure and contents.
    
    .PARAMETER InputObject
    The variable to inspect. Pass by reference using $variable.
    #>
    param(
        [Parameter(Mandatory=$true, ValueFromPipeline=$true)]
        [AllowNull()]
        $InputObject
    )
    
    process {
        $info = [PSCustomObject]@{
            IsNull = $null -eq $InputObject
            BaseType = if ($InputObject) { $InputObject.GetType().FullName } else { "Null" }
            IsCollection = $false
            Count = 0
            ItemTypes = @()
            SelectableProperties = @()
        }
        
        # Determine if collection
        if ($InputObject -is [Array] -or 
            $InputObject -is [System.Collections.IEnumerable] -and 
            $InputObject -isnot [string]) {
            
            $info.IsCollection = $true
            $info.Count = @($InputObject).Count
            
            # Get unique types of items
            $info.ItemTypes = @($InputObject | ForEach-Object { 
                if ($null -ne $_) { $_.GetType().FullName } else { "Null" }
            } | Select-Object -Unique)
            
            # Get common selectable properties (intersection of all item properties)
            if ($info.Count -gt 0) {
                $firstNonNull = @($InputObject) | Where-Object { $_ -ne $null } | Select-Object -First 1
                if ($firstNonNull) {
                    $info.SelectableProperties = @($firstNonNull | Get-Member -MemberType Properties | 
                        Select-Object -ExpandProperty Name)
                }
            }
        } else {
            # Single object
            $info.Count = 1
            if ($InputObject) {
                $info.ItemTypes = @($InputObject.GetType().FullName)
                $info.SelectableProperties = @($InputObject | Get-Member -MemberType Properties | 
                    Select-Object -ExpandProperty Name)
            }
        }
        
        return $info
    }
}

function Get-CollectionCount {
    <#
    .SYNOPSIS
    Returns the count of items in a collection or 1 for scalar objects.
    
    .DESCRIPTION
    Uses Measure-Object for accurate counting. Handles edge cases including
    null values, empty collections, and single-item implicit arrays.
    #>
    param(
        [Parameter(Mandatory=$true, ValueFromPipeline=$true)]
        [AllowNull()]
        $InputObject
    )
    
    process {
        if ($null -eq $InputObject) {
            return 0
        }
        
        # Force array conversion to handle single items
        $measured = @($InputObject) | Measure-Object
        return $measured.Count
    }
}

function Get-CollectionTypes {
    <#
    .SYNOPSIS
    Enumerates the distinct .NET types present in a collection.
    
    .DESCRIPTION
    Returns a hashtable with type names as keys and occurrence counts as values.
    Useful for heterogeneous collections with mixed types.
    #>
    param(
        [Parameter(Mandatory=$true, ValueFromPipeline=$true)]
        [AllowNull()]
        $InputObject
    )
    
    process {
        $typeStats = @{}
        
        @($InputObject) | ForEach-Object {
            $typeName = if ($null -eq $_) { 
                "[Null]" 
            } else { 
                $_.GetType().FullName 
            }
            
            if ($typeStats.ContainsKey($typeName)) {
                $typeStats[$typeName]++
            } else {
                $typeStats[$typeName] = 1
            }
        }
        
        return $typeStats
    }
}

function Get-SelectableProperties {
    <#
    .SYNOPSIS
    Retrieves property names available for Select-Object operations.
    
    .DESCRIPTION
    Returns properties with their types. For collections, analyzes the first
    non-null element. Filters to include only properties (excludes methods,
    events, and other member types).
    #>
    param(
        [Parameter(Mandatory=$true, ValueFromPipeline=$true)]
        [AllowNull()]
        $InputObject,
        
        [switch]$IncludeMethods
    )
    
    process {
        if ($null -eq $InputObject) {
            return @()
        }
        
        # Get sample object
        $sample = if ($InputObject -is [Array] -or 
                     ($InputObject -is [System.Collections.IEnumerable] -and 
                      $InputObject -isnot [string])) {
            @($InputObject) | Where-Object { $_ -ne $null } | Select-Object -First 1
        } else {
            $InputObject
        }
        
        if ($null -eq $sample) {
            return @()
        }
        
        # Determine member types to retrieve
        $memberTypes = if ($IncludeMethods) {
            @('Property', 'NoteProperty', 'ScriptProperty', 'Method')
        } else {
            @('Property', 'NoteProperty', 'ScriptProperty')
        }
        
        $members = $sample | Get-Member -MemberType $memberTypes | 
            Select-Object Name, MemberType, 
                @{Name='PropertyType'; Expression={ $_.Definition -replace '^(\S+).*','$1' }}
        
        return $members
    }
}

function Show-VariableStructure {
    <#
    .SYNOPSIS
    Displays formatted summary of variable structure and contents.
    
    .DESCRIPTION
    Combines all inspection functions into a single comprehensive output.
    Provides formatted display optimized for console readability.
    #>
    param(
        [Parameter(Mandatory=$true, ValueFromPipeline=$true)]
        [AllowNull()]
        $InputObject,
        
        [string]$VariableName = "InputObject"
    )
    
    process {
        Write-Host "`n=== Variable: $VariableName ===" -ForegroundColor Cyan
        
        $info = Get-VariableInfo -InputObject $InputObject
        
        Write-Host "`nBase Type: " -NoNewline
        Write-Host $info.BaseType -ForegroundColor Yellow
        
        Write-Host "Is Collection: " -NoNewline
        Write-Host $info.IsCollection -ForegroundColor $(if($info.IsCollection){"Green"}else{"Gray"})
        
        Write-Host "Count: " -NoNewline
        Write-Host $info.Count -ForegroundColor Magenta
        
        if ($info.ItemTypes.Count -gt 0) {
            Write-Host "`nItem Types:"
            $info.ItemTypes | ForEach-Object {
                Write-Host "  - $_" -ForegroundColor Yellow
            }
        }
        
        if ($info.SelectableProperties.Count -gt 0) {
            Write-Host "`nSelectable Properties ($($info.SelectableProperties.Count)):"
            $info.SelectableProperties | Sort-Object | ForEach-Object {
                Write-Host "  - $_" -ForegroundColor Green
            }
        }
        
        Write-Host ""
    }
}
```

## Usage Examples

powershell

```powershell
# Example 1: Inspect a heterogeneous array
$mixedArray = @(1, "string", (Get-Date), $null, @{Key="Value"})
Get-VariableInfo $mixedArray
Show-VariableStructure $mixedArray -VariableName "mixedArray"

# Example 2: Count items
Get-CollectionCount $mixedArray  # Returns: 5

# Example 3: Analyze type distribution
Get-CollectionTypes $mixedArray
# Output: @{ "System.Int32"=1; "System.String"=1; "System.DateTime"=1; "[Null]"=1; "System.Collections.Hashtable"=1 }

# Example 4: Get selectable properties from process objects
$processes = Get-Process | Select-Object -First 5
Get-SelectableProperties $processes
Show-VariableStructure $processes -VariableName "processes"

# Example 5: Inspect single object
$date = Get-Date
Get-VariableInfo $date
```

## Technical Considerations

**Type Resolution**: The `GetType()` method returns the runtime type. For generic collections like `List<T>`, this includes type parameters (e.g., `System.Collections.Generic.List'1[[System.String]]`).

**Pipeline Behavior**: PowerShell's pipeline may unwrap collections. Use `@()` array subexpression operator to force enumeration and accurate counting.

**Null Handling**: The `[AllowNull()]` attribute permits null input without parameter binding errors. Functions explicitly test for null using `$null -eq $variable` (preferred left-hand side placement for type safety).

**Property vs. Member Distinction**: `Get-Member` returns multiple member types. Properties are data-holding members; methods are executable. `Select-Object` operates on properties, note properties, and script properties—not methods.

**Homogeneous vs. Heterogeneous Collections**: Strongly-typed collections (e.g., `[int[]]`) guarantee uniform types. Weakly-typed arrays (`@()`) permit mixed types, requiring per-element type inspection.