# PowerShell Function Structure Progression

## Level 1: Minimal Function

```powershell
function Get-Greeting {
    "Hello, World!"
}
```

**Explanation:**

- `function` keyword declares function
- `Get-Greeting` follows Verb-Noun naming convention
- No parameters
- Returns string implicitly (last unassigned value)

**Usage:** `Get-Greeting` → outputs "Hello, World!"

---

## Level 2: Basic Parameters

```powershell
function Get-Greeting {
    param(
        $Name
    )
    
    "Hello, $Name!"
}
```

**New elements:**

- `param()` block declares parameters
- `$Name` untyped parameter (accepts anything)
- String interpolation with `$Name`

**Usage:** `Get-Greeting -Name "Alice"` → "Hello, Alice!"

---

## Level 3: Typed Parameters with Defaults

```powershell
function Get-Greeting {
    param(
        [string]$Name = "World",
        [int]$Count = 1
    )
    
    for ($i = 0; $i -lt $Count; $i++) {
        "Hello, $Name!"
    }
}
```

**New elements:**

- `[string]` type constraint on `$Name`
- `= "World"` default value
- `[int]$Count` typed integer parameter
- Logic using parameters

**Usage:**

- `Get-Greeting` → "Hello, World!"
- `Get-Greeting -Name "Bob" -Count 2` → outputs greeting twice

---

## Level 4: Advanced Function with Validation

```powershell
function Get-Greeting {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true,
                   Position = 0,
                   ValueFromPipeline = $true,
                   HelpMessage = "Enter a name")]
        [ValidateNotNullOrEmpty()]
        [ValidateLength(1, 50)]
        [string]$Name,
        
        [Parameter()]
        [ValidateRange(1, 10)]
        [int]$Count = 1,
        
        [Parameter()]
        [ValidateSet("Formal", "Casual", "Enthusiastic")]
        [string]$Style = "Casual"
    )
    
    begin {
        Write-Verbose "Starting greeting function"
        $Prefix = switch ($Style) {
            "Formal"       { "Good day" }
            "Casual"       { "Hey" }
            "Enthusiastic" { "Hello!!!" }
        }
    }
    
    process {
        for ($i = 0; $i -lt $Count; $i++) {
            "$Prefix, $Name!"
        }
    }
    
    end {
        Write-Verbose "Greeting function complete"
    }
}
```

**New elements:**

- `[CmdletBinding()]` enables advanced features (Verbose, Debug, ErrorAction, etc.)
- `[Parameter()]` attribute configures parameter behavior:
    - `Mandatory = $true` requires parameter
    - `Position = 0` allows positional input (no parameter name needed)
    - `ValueFromPipeline = $true` accepts pipeline input
    - `HelpMessage` provides interactive help prompt
- Validation attributes:
    - `[ValidateNotNullOrEmpty()]` rejects null/empty
    - `[ValidateLength(1, 50)]` enforces string length
    - `[ValidateRange(1, 10)]` enforces numeric range
    - `[ValidateSet()]` restricts to specific values
- `begin{}` block executes once before pipeline processing
- `process{}` block executes once per pipeline input
- `end{}` block executes once after pipeline processing
- `Write-Verbose` outputs only when `-Verbose` switch used

**Usage:**

```powershell
Get-Greeting "Alice"  # Positional parameter
Get-Greeting -Name "Bob" -Style "Formal" -Verbose
"Alice", "Bob" | Get-Greeting  # Pipeline input
```

---

## Level 5: Highly Contrived Complex Function

```powershell
function Invoke-ServerOperation {
    [CmdletBinding(DefaultParameterSetName = 'ByName',
                   SupportsShouldProcess = $true,
                   ConfirmImpact = 'High')]
    [OutputType([PSCustomObject])]
    param(
        # Parameter Set: ByName
        [Parameter(Mandatory = $true,
                   Position = 0,
                   ParameterSetName = 'ByName',
                   ValueFromPipeline = $true,
                   ValueFromPipelineByPropertyName = $true)]
        [ValidateNotNullOrEmpty()]
        [ValidatePattern('^[A-Z0-9-]+$')]
        [Alias('Server', 'Host')]
        [string[]]$ComputerName,
        
        # Parameter Set: ByIP
        [Parameter(Mandatory = $true,
                   ParameterSetName = 'ByIP')]
        [ValidateScript({
            $_ -match '^((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$'
        })]
        [string[]]$IPAddress,
        
        # Common parameters
        [Parameter(Mandatory = $true)]
        [ValidateSet('Restart', 'Shutdown', 'Status', 'Backup')]
        [string]$Operation,
        
        [Parameter()]
        [ValidateRange(1, 3600)]
        [int]$TimeoutSeconds = 300,
        
        [Parameter()]
        [PSCredential]
        [System.Management.Automation.Credential()]
        $Credential = [PSCredential]::Empty,
        
        [Parameter()]
        [switch]$Force,
        
        [Parameter()]
        [switch]$AsJob,
        
        [Parameter(DontShow = $true)]
        [string]$InternalDebugPath
    )
    
    begin {
        # Initialization
        Write-Verbose "Function started at $(Get-Date)"
        Write-Debug "Parameter set: $($PSCmdlet.ParameterSetName)"
        
        $ErrorActionPreference = 'Stop'
        $Results = [System.Collections.ArrayList]::new()
        
        # Validation logic
        if ($PSBoundParameters.ContainsKey('InternalDebugPath')) {
            Write-Warning "Internal debug mode enabled"
            Start-Transcript -Path $InternalDebugPath
        }
        
        # Dynamic parameter validation
        if ($Operation -eq 'Backup' -and -not $PSBoundParameters.ContainsKey('Credential')) {
            Write-Warning "Backup operations typically require credentials"
        }
    }
    
    process {
        # Determine target from parameter set
        $Targets = switch ($PSCmdlet.ParameterSetName) {
            'ByName' { $ComputerName }
            'ByIP'   { $IPAddress }
        }
        
        foreach ($Target in $Targets) {
            # ShouldProcess check for confirmation
            if ($PSCmdlet.ShouldProcess($Target, "Execute $Operation operation")) {
                
                try {
                    Write-Progress -Activity "Processing $Operation" `
                                   -Status "Target: $Target" `
                                   -PercentComplete (($Results.Count / $Targets.Count) * 100)
                    
                    # Simulated operation
                    $OperationResult = [PSCustomObject]@{
                        PSTypeName   = 'ServerOperation.Result'
                        Target       = $Target
                        Operation    = $Operation
                        Status       = 'Success'
                        Timestamp    = Get-Date
                        Duration     = New-TimeSpan -Seconds (Get-Random -Min 1 -Max 5)
                        Message      = "Operation completed successfully"
                    }
                    
                    # Error simulation
                    if (-not $Force -and (Get-Random -Maximum 10) -eq 0) {
                        throw "Simulated random failure on $Target"
                    }
                    
                    # Add custom type for formatting
                    $OperationResult.PSObject.TypeNames.Insert(0, 'Custom.ServerOperation.Result')
                    
                    [void]$Results.Add($OperationResult)
                    
                    # Output to pipeline
                    if (-not $AsJob) {
                        Write-Output $OperationResult
                    }
                    
                } catch {
                    $ErrorRecord = [PSCustomObject]@{
                        PSTypeName = 'ServerOperation.Error'
                        Target     = $Target
                        Operation  = $Operation
                        Status     = 'Failed'
                        Timestamp  = Get-Date
                        Error      = $_.Exception.Message
                    }
                    
                    Write-Error "Operation failed on ${Target}: $_"
                    [void]$Results.Add($ErrorRecord)
                    
                    # Continue or stop based on ErrorActionPreference
                    if ($ErrorActionPreference -eq 'Stop') {
                        throw
                    }
                }
            } else {
                Write-Verbose "Operation cancelled by user for $Target"
            }
        }
    }
    
    end {
        Write-Progress -Activity "Processing $Operation" -Completed
        
        if ($AsJob) {
            Write-Output $Results
        }
        
        # Summary
        $SuccessCount = ($Results | Where-Object Status -eq 'Success').Count
        $FailCount = ($Results | Where-Object Status -eq 'Failed').Count
        
        Write-Verbose "Operations complete: $SuccessCount succeeded, $FailCount failed"
        
        if ($PSBoundParameters.ContainsKey('InternalDebugPath')) {
            Stop-Transcript
        }
        
        Write-Verbose "Function ended at $(Get-Date)"
    }
}
```

**Advanced elements explained:**

### CmdletBinding attributes:

- `DefaultParameterSetName = 'ByName'` specifies default when multiple parameter sets exist
- `SupportsShouldProcess = $true` enables `-WhatIf` and `-Confirm` support
- `ConfirmImpact = 'High'` determines when confirmation prompts appear

### Parameter sets:

- `ParameterSetName = 'ByName'` creates mutually exclusive parameter groups
- Prevents using `-ComputerName` and `-IPAddress` simultaneously
- `$PSCmdlet.ParameterSetName` identifies which set is active

### Advanced parameter attributes:

- `ValueFromPipelineByPropertyName = $true` binds pipeline object properties to parameters
- `[Alias('Server', 'Host')]` creates alternative parameter names
- `[string[]]` array type accepts multiple values
- `[ValidateScript({})]` custom validation logic
- `[ValidatePattern()]` regex validation
- `[PSCredential]` secure credential type
- `[switch]` boolean flag (present = `$true`)
- `DontShow = $true` hides parameter from tab completion and help

### Automatic variables:

- `$PSBoundParameters` hashtable of provided parameters
- `$PSCmdlet` provides cmdlet context and methods
- `$ErrorActionPreference` controls error handling behavior

### Pipeline blocks:

- `begin{}` runs once: initialization, validation
- `process{}` runs per pipeline item: main logic
- `end{}` runs once: cleanup, summary

### Methods and techniques:

- `$PSCmdlet.ShouldProcess()` implements `-WhatIf`/`-Confirm` logic
- `Write-Progress` displays progress bar
- `[System.Collections.ArrayList]::new()` efficient collection for accumulation
- `PSTypeName` for custom formatting/type system
- `Write-Error` writes to error stream
- `throw` terminates with exception
- `try/catch` structured error handling
- `[void]` suppresses output (ArrayList.Add() returns index)

**Usage examples:**

```powershell
# Basic usage
Invoke-ServerOperation -ComputerName "SERVER01" -Operation Status

# Pipeline by name
"SERVER01", "SERVER02" | Invoke-ServerOperation -Operation Restart -Verbose

# Pipeline by property
$Servers = @(
    [PSCustomObject]@{ComputerName="SRV1"; Environment="Prod"}
    [PSCustomObject]@{ComputerName="SRV2"; Environment="Dev"}
)
$Servers | Invoke-ServerOperation -Operation Status

# Using IP parameter set
Invoke-ServerOperation -IPAddress "192.168.1.10" -Operation Shutdown

# With confirmation
Invoke-ServerOperation -ComputerName "PROD-DB" -Operation Restart -Confirm

# WhatIf preview
Invoke-ServerOperation -ComputerName "SERVER01" -Operation Shutdown -WhatIf

# With credentials
$Cred = Get-Credential
Invoke-ServerOperation -ComputerName "SERVER01" -Operation Backup -Credential $Cred

# Force execution
Invoke-ServerOperation -ComputerName "SERVER01" -Operation Restart -Force

# Return as collection
Invoke-ServerOperation -ComputerName "SERVER01","SERVER02" -Operation Status -AsJob
```

This progression demonstrates PowerShell functions from basic string output to enterprise-grade cmdlets with full validation, error handling, pipeline support, and user safety features.

----------------------------

# PowerShell Function Grammar Pattern

```
Function ::= 'function' Name '{' FunctionBody '}'

FunctionBody ::= [Attributes] [ParamBlock] [BeginBlock] [ProcessBlock] [EndBlock] Code

Name ::= Verb '-' Noun
       | AnyIdentifier

Attributes ::= '[CmdletBinding(' AttributeParams ')]'
             | '[OutputType(' TypeName ')]'
             | Both

ParamBlock ::= 'param(' ParameterList ')'

ParameterList ::= Parameter [',' Parameter]*
                | Empty

Parameter ::= [ParameterAttributes]* [TypeConstraint] [Validation]* Variable [DefaultValue]

ParameterAttributes ::= '[Parameter(' ParamConfig ')]'

ParamConfig ::= ['Mandatory = ' Boolean]
              | ['Position = ' Integer]  
              | ['ValueFromPipeline = ' Boolean]
              | ['ValueFromPipelineByPropertyName = ' Boolean]
              | ['ParameterSetName = ' String]
              | ['HelpMessage = ' String]
              | ['DontShow = ' Boolean]
              | Combination

TypeConstraint ::= '[' TypeName ']'
                 | '[' TypeName '[]' ']'    # Array

Validation ::= '[ValidateNotNull()]'
             | '[ValidateNotNullOrEmpty()]'
             | '[ValidateLength(' Min ',' Max ')]'
             | '[ValidateRange(' Min ',' Max ')]'
             | '[ValidateSet(' StringList ')]'
             | '[ValidatePattern(' Regex ')]'
             | '[ValidateScript({' ScriptBlock '})]'
             | '[ValidateCount(' Min ',' Max ')]'
             | '[Alias(' StringList ')]'

Variable ::= '$' Identifier

DefaultValue ::= '=' Expression

BeginBlock ::= 'begin {' Code '}'

ProcessBlock ::= 'process {' Code '}'

EndBlock ::= 'end {' Code '}'

Code ::= Statement*

# Common patterns
Statement ::= Assignment
            | FunctionCall  
            | ControlFlow
            | PipelineExpression
            | Return

ControlFlow ::= 'if (' Condition ') {' Code '}' ['else {' Code '}']
              | 'foreach (' Item 'in' Collection ') {' Code '}'
              | 'while (' Condition ') {' Code '}'
              | 'switch (' Value ') {' Cases '}'

PipelineExpression ::= Expression ['|' Expression]*

Return ::= 'return' Expression
         | Expression              # Implicit return
         | 'Write-Output' Expression
```

## Minimal Example

```
function Get-Data {
    param($Name)
    "Result: $Name"
}
```

## Full-Featured Example Pattern

```
function Verb-Noun {
    [CmdletBinding(DefaultParameterSetName='SetA')]
    [OutputType([PSCustomObject])]
    param(
        [Parameter(Mandatory=$true, Position=0, ValueFromPipeline=$true)]
        [ValidateNotNullOrEmpty()]
        [string[]]$Name,
        
        [switch]$Force
    )
    
    begin {
        # Initialize once
    }
    
    process {
        # Execute per pipeline item
        foreach ($Item in $Name) {
            # Logic
        }
    }
    
    end {
        # Cleanup
    }
}
```

## Common Type Constraints

```
[string]         # Single string
[string[]]       # String array
[int]            # Integer
[bool]           # Boolean
[switch]         # Switch parameter (flag)
[PSCredential]   # Credential object
[datetime]       # Date/time
[hashtable]      # Hash table
[PSCustomObject] # Custom object
[object]         # Any type
```

## Execution Flow

```
Pipeline Input → begin{} → process{} (per item) → end{}
                    ↓           ↓                    ↓
                  Once       Multiple              Once
```