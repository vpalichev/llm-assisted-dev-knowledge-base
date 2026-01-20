Current folder for PS Modules: 
c:\\Users\\USERNAME\\Documents\\WindowsPowerShell\\Modules\\

### Force Reload Module After Update
```powershell
# Remove and reimport
Remove-Module "PnP.CustomUtilities" -Force
Import-Module "PnP.CustomUtilities" -Force

# Or single command
Import-Module "PnP.CustomUtilities" -Force
```
The `-Force` parameter removes the module if loaded, then imports the fresh version.
**Verify reload:**
```powershell
Get-Command -Module "PnP.CustomUtilities"
```

# Creating a PowerShell Module Repository

## Conceptual Foundation

**PowerShell Module**: A package containing reusable PowerShell commands (functions, cmdlets, variables) organized in a structured format that the PowerShell runtime can discover and load.

**Module Repository**: A directory structure containing one or more PowerShell modules, stored in a location where PowerShell's module autoloading mechanism can locate them.

## Module Storage Locations

PowerShell searches for modules in directories specified by the `$env:PSModulePath` environment variable. Standard locations include:

```powershell
# View current module paths
$env:PSModulePath -split ';'
```

**System-level locations:**

- `C:\Program Files\WindowsPowerShell\Modules` (all users)
- `C:\Windows\System32\WindowsPowerShell\v1.0\Modules` (system modules)

**User-level location:**

- `C:\Users\<Username>\Documents\WindowsPowerShell\Modules` (current user)

## Module Structure Specification

A properly structured PowerShell module consists of:

```
ModuleName\
├── ModuleName.psm1          # Module script file (required)
├── ModuleName.psd1          # Module manifest (recommended)
└── Public\                  # Optional: organizational subdirectory
    ├── Function1.ps1
    └── Function2.ps1
```

## Implementation Steps

### Step 1: Create Module Directory Structure

```powershell
# Define module name
$ModuleName = "ServerUtilities"

# Create module directory in user module path
$ModulePath = "$env:USERPROFILE\Documents\WindowsPowerShell\Modules\$ModuleName"
New-Item -Path $ModulePath -ItemType Directory -Force

# Create subdirectories for organization
New-Item -Path "$ModulePath\Public" -ItemType Directory -Force
New-Item -Path "$ModulePath\Private" -ItemType Directory -Force
```

**Terminology:**

- **Public functions**: Exported functions accessible to users importing the module
- **Private functions**: Internal helper functions not exposed outside the module

### Step 2: Create Individual Function Files

Example function file (`Public\Get-ServerStatus.ps1`):

```powershell
function Get-ServerStatus {
    <#
    .SYNOPSIS
        Retrieves operational status of specified server.
    
    .DESCRIPTION
        Queries target server for CPU usage, memory consumption, and disk space.
        Returns structured object containing status metrics.
    
    .PARAMETER ComputerName
        Target server hostname or IP address.
    
    .EXAMPLE
        Get-ServerStatus -ComputerName "SERVER01"
    #>
    
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [string]$ComputerName
    )
    
    process {
        # Function implementation
        $Result = [PSCustomObject]@{
            ComputerName = $ComputerName
            Status       = "Online"
            CPUUsage     = 45
            MemoryUsage  = 62
        }
        
        return $Result
    }
}
```

### Step 3: Create Module Script File (.psm1)

The `.psm1` file aggregates all function definitions and controls export behavior.

**Method A: Dot-Sourcing Individual Files**

```powershell
# ServerUtilities.psm1

# Import all public functions
$PublicFunctions = Get-ChildItem -Path "$PSScriptRoot\Public\*.ps1" -ErrorAction SilentlyContinue

foreach ($Function in $PublicFunctions) {
    . $Function.FullName
}

# Import all private functions (not exported)
$PrivateFunctions = Get-ChildItem -Path "$PSScriptRoot\Private\*.ps1" -ErrorAction SilentlyContinue

foreach ($Function in $PrivateFunctions) {
    . $Function.FullName
}

# Export only public functions
Export-ModuleMember -Function $PublicFunctions.BaseName
```

**Method B: Direct Function Definition**

```powershell
# ServerUtilities.psm1

function Get-ServerStatus {
    # Function implementation here
}

function Restart-ServerService {
    # Function implementation here
}

# Explicitly export public functions
Export-ModuleMember -Function Get-ServerStatus, Restart-ServerService
```

**Technical Note**: The `$PSScriptRoot` automatic variable contains the directory path of the executing module file, enabling relative path resolution.

### Step 4: Create Module Manifest (.psd1)

The manifest file provides metadata and configuration for the module:

```powershell
# Generate manifest
New-ModuleManifest -Path "$ModulePath\$ModuleName.psd1" `
    -RootModule "$ModuleName.psm1" `
    -ModuleVersion "1.0.0" `
    -Author "Your Name" `
    -Description "Collection of server administration utilities" `
    -PowerShellVersion "7.2" `
    -FunctionsToExport @('Get-ServerStatus', 'Restart-ServerService')
    #FunctionsToExport is wrong, should be all functions
```

**Manifest Properties:**

- `RootModule`: Specifies the `.psm1` file containing module logic
- `FunctionsToExport`: Explicitly declares exported functions (performance optimization)
- `PowerShellVersion`: Minimum PowerShell version requirement

### Step 5: Verify Module Availability

```powershell
# Force module discovery refresh
Get-Module -ListAvailable -Name $ModuleName

# Display module information
Get-Module -ListAvailable -Name $ModuleName | Format-List
```

## Module Usage Procedures

### Importing Modules

**Explicit Import:**

```powershell
Import-Module -Name ServerUtilities
```

**Automatic Import** (PowerShell 3.0+): Functions are auto-imported when invoked if module exists in `$env:PSModulePath`.

### Executing Module Functions

```powershell
# Direct invocation
Get-ServerStatus -ComputerName "SERVER01"

# Pipeline usage
"SERVER01", "SERVER02" | ForEach-Object { Get-ServerStatus -ComputerName $_ }
```

### Listing Module Functions

```powershell
# Display all exported functions
Get-Command -Module ServerUtilities

# Display function definitions
Get-Command -Module ServerUtilities | Get-Help -Full
```

### Module Removal

```powershell
# Unload module from current session
Remove-Module -Name ServerUtilities
```

## Version Control Integration

For enterprise environments, implement version control:

```powershell
# Initialize Git repository in module directory
cd $ModulePath
git init
git add .
git commit -m "Initial module commit"

# Configure remote repository
git remote add origin https://github.com/organization/ServerUtilities.git
git push -u origin main
```

## Advanced Configuration: Custom Repository Path

To add a custom module repository location:

```powershell
# Append custom path to PSModulePath
$CustomPath = "\\FileServer\SharedModules"
$CurrentPath = [Environment]::GetEnvironmentVariable("PSModulePath", "User")
$NewPath = "$CurrentPath;$CustomPath"
[Environment]::SetEnvironmentVariable("PSModulePath", $NewPath, "User")

# Verify modification
$env:PSModulePath -split ';'
```

**Persistence Scope:**

- `"User"`: Current user only
- `"Machine"`: All users (requires administrative privileges)

## Execution Policy Considerations

PowerShell's execution policy may prevent module loading. Verify and adjust if necessary:

```powershell
# Check current policy
Get-ExecutionPolicy

# Set policy (requires administrative privileges for system-wide change)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**Execution Policy Values:**

- `Restricted`: No scripts execute
- `RemoteSigned`: Locally-created scripts execute unsigned; downloaded scripts require signature
- `Unrestricted`: All scripts execute with confirmation prompt for downloaded scripts

This configuration establishes a maintainable, discoverable repository of PowerShell functions suitable for server administration tasks.




```powershell
# ============================================================================
# STEP 1: View Module Search Paths
# ============================================================================

# $env:PSModulePath = Environment variable containing semicolon-separated paths
# -split ';' = Splits string into array using semicolon as delimiter
# Displays each directory where PowerShell searches for modules
$env:PSModulePath -split ';'


# ============================================================================
# STEP 2: Create Module Directory Structure
# ============================================================================

# Define string variable containing module name
$ModuleName = "ServerUtilities"

# $env:USERPROFILE = Environment variable pointing to C:\Users\<Username>
# Combines path segments into full module directory path
$ModulePath = "$env:USERPROFILE\Documents\WindowsPowerShell\Modules\$ModuleName"

# New-Item = Creates filesystem objects
# -Path = Target location to create
# -ItemType Directory = Specifies creating a folder (not file)
# -Force = Creates parent directories if missing, doesn't error if exists
New-Item -Path $ModulePath -ItemType Directory -Force

# $ModulePath\ = Uses previously defined path as base
# Creates 'Public' subfolder for exported functions
New-Item -Path "$ModulePath\Public" -ItemType Directory -Force

# Creates 'Private' subfolder for internal helper functions
New-Item -Path "$ModulePath\Private" -ItemType Directory -Force


# ============================================================================
# STEP 3: Create Function File
# ============================================================================

# Save this as: Public\Get-ServerStatus.ps1

function Get-ServerStatus {
    # <# #> = Comment-based help block for Get-Help cmdlet
    # .SYNOPSIS = One-line function description
    <#
    .SYNOPSIS
        Retrieves operational status of specified server.
    
    # .DESCRIPTION = Detailed function description
    .DESCRIPTION
        Queries target server for CPU usage, memory consumption, and disk space.
        Returns structured object containing status metrics.
    
    # .PARAMETER = Documents function parameters
    .PARAMETER ComputerName
        Target server hostname or IP address.
    
    # .EXAMPLE = Usage example displayed in help
    .EXAMPLE
        Get-ServerStatus -ComputerName "SERVER01"
    #>
    
    # [CmdletBinding()] = Enables advanced function features (verbose, debug, error handling)
    [CmdletBinding()]
    # param() = Declares function parameters
    param(
        # [Parameter()] = Configures parameter behavior
        # Mandatory = $true = Makes parameter required (function fails without it)
        [Parameter(Mandatory = $true)]
        # [string] = Type constraint, only accepts string values
        # $ComputerName = Parameter variable name
        [string]$ComputerName
    )
    
    # process{} = Block that executes once per pipeline input
    process {
        # [PSCustomObject]@{} = Creates custom object with named properties
        # Property = Value pairs define object structure
        $Result = [PSCustomObject]@{
            ComputerName = $ComputerName  # Assigns parameter value to property
            Status       = "Online"        # Hardcoded value (example placeholder)
            CPUUsage     = 45              # Integer value representing percentage
            MemoryUsage  = 62              # Integer value representing percentage
        }
        
        # return = Exits function and outputs $Result to pipeline
        return $Result
    }
}


# ============================================================================
# STEP 4A: Module File - Dot-Sourcing Method
# ============================================================================

# Save this as: ServerUtilities.psm1

# $PSScriptRoot = Automatic variable containing directory of current script
# Get-ChildItem = Lists files/folders (like dir command)
# -Path = Target directory and filter pattern
# \*.ps1 = Wildcard selecting all PowerShell script files
# -ErrorAction SilentlyContinue = Suppresses errors if directory doesn't exist
$PublicFunctions = Get-ChildItem -Path "$PSScriptRoot\Public\*.ps1" -ErrorAction SilentlyContinue

# foreach = Loop construct, iterates through collection
# $Function = Loop variable holding current item
foreach ($Function in $PublicFunctions) {
    # . (dot operator) = Dot-sources script (runs in current scope)
    # .FullName = Property containing complete file path
    # Effect: Loads function definition into module scope
    . $Function.FullName
}

# Same pattern for private functions (not exported)
$PrivateFunctions = Get-ChildItem -Path "$PSScriptRoot\Private\*.ps1" -ErrorAction SilentlyContinue

foreach ($Function in $PrivateFunctions) {
    . $Function.FullName  # Loads but won't be exported
}

# Export-ModuleMember = Controls which functions are publicly accessible
# -Function = Specifies functions to export (not variables/aliases)
# .BaseName = Property containing filename without extension
# Effect: Only public functions become available when module is imported
Export-ModuleMember -Function $PublicFunctions.BaseName


# ============================================================================
# STEP 4B: Module File - Direct Definition Method
# ============================================================================

# Alternative approach: define functions directly in .psm1 file
function Get-ServerStatus {
    # Function implementation
}

function Restart-ServerService {
    # Function implementation
}

# Explicitly lists function names to export (comma-separated)
Export-ModuleMember -Function Get-ServerStatus, Restart-ServerService


# ============================================================================
# STEP 5: Create Module Manifest
# ============================================================================

# New-ModuleManifest = Generates .psd1 manifest file
# Backtick (`) = Line continuation character for readability
New-ModuleManifest -Path "$ModulePath\$ModuleName.psd1" `
    # -RootModule = Points to .psm1 file containing actual code
    -RootModule "$ModuleName.psm1" `
    # -ModuleVersion = Semantic version string
    -ModuleVersion "1.0.0" `
    # -Author = Module creator name (metadata)
    -Author "Your Name" `
    # -Description = Human-readable module purpose
    -Description "Collection of server administration utilities" `
    # -PowerShellVersion = Minimum PS version required to load module
    -PowerShellVersion "5.1" `
    # -FunctionsToExport = Array of function names to expose
    # @() = Array literal syntax
    -FunctionsToExport @('Get-ServerStatus', 'Restart-ServerService')


# ============================================================================
# STEP 6: Verify Module Installation
# ============================================================================

# Get-Module = Queries module information
# -ListAvailable = Searches $env:PSModulePath (doesn't load module)
# -Name = Filters to specific module name
# Effect: Confirms module is discoverable by PowerShell
Get-Module -ListAvailable -Name $ModuleName

# | (pipe) = Passes output to next cmdlet
# Format-List = Displays properties in vertical list format
# Effect: Shows all module properties (version, path, exported commands)
Get-Module -ListAvailable -Name $ModuleName | Format-List


# ============================================================================
# USING THE MODULE
# ============================================================================

# Import-Module = Explicitly loads module into current session
# -Name = Module name or path
Import-Module -Name ServerUtilities

# Call function with named parameter
# -ComputerName = Parameter name from function definition
Get-ServerStatus -ComputerName "SERVER01"

# Pipeline example: processes multiple servers
# | (pipe) = Passes each string to ForEach-Object
# ForEach-Object = Executes script block for each input
# $_ = Automatic variable representing current pipeline object
"SERVER01", "SERVER02" | ForEach-Object { Get-ServerStatus -ComputerName $_ }

# Get-Command = Lists available commands
# -Module = Filters to specific module's commands
Get-Command -Module ServerUtilities

# Get-Help = Displays function documentation
# -Full = Shows complete help including examples
Get-Command -Module ServerUtilities | Get-Help -Full

# Remove-Module = Unloads module from memory (doesn't delete files)
Remove-Module -Name ServerUtilities


# ============================================================================
# ADVANCED: Custom Module Path
# ============================================================================

# UNC path to network share
$CustomPath = "\\FileServer\SharedModules"

# [Environment]::GetEnvironmentVariable() = .NET method to read env vars
# "PSModulePath" = Variable name
# "User" = Scope (User vs Machine)
$CurrentPath = [Environment]::GetEnvironmentVariable("PSModulePath", "User")

# String concatenation with semicolon separator
$NewPath = "$CurrentPath;$CustomPath"

# SetEnvironmentVariable() = Persists change to registry
# Effect: New path available in future sessions
[Environment]::SetEnvironmentVariable("PSModulePath", $NewPath, "User")


# ============================================================================
# EXECUTION POLICY
# ============================================================================

# Get-ExecutionPolicy = Displays current script execution restrictions
Get-ExecutionPolicy

# Set-ExecutionPolicy = Changes script execution permissions
# -ExecutionPolicy RemoteSigned = Allows local scripts, requires signatures for downloaded
# -Scope CurrentUser = Applies to current user only (no admin rights needed)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

------------------------
# Check Module Functions

## Primary Method

```powershell
# List all commands exported by module
Get-Command -Module ModuleName

# Filter to functions only
Get-Command -Module ModuleName -CommandType Function
```

## Alternative Methods

```powershell
# View module object properties
Get-Module ModuleName | Select-Object -ExpandProperty ExportedFunctions

# Direct property access
(Get-Module ModuleName).ExportedFunctions

# See all exported commands (functions, cmdlets, aliases)
(Get-Module ModuleName).ExportedCommands
```

## Detailed Information

```powershell
# Full function details including parameters
Get-Command -Module ModuleName | Get-Help -Full

# List with syntax
Get-Command -Module ModuleName | Format-List Name, CommandType, Source
```

## Example

```powershell
Import-Module ServerUtilities

# Quick list
Get-Command -Module ServerUtilities

# Output:
# CommandType     Name                           Version    Source
# -----------     ----                           -------    ------
# Function        Get-ServerStatus               1.0.0      ServerUtilities
# Function        Restart-ServerService          1.0.0      ServerUtilities
```

**Note**: Use `Get-Module -ListAvailable` to see module info _before_ importing.

-------------

# Troubleshooting Empty Module

## Check if Module Actually Loaded

```powershell
# Verify module is loaded
Get-Module "PnP.CustomUtilities"

# If nothing shows, it didn't load. Check available modules:
Get-Module -ListAvailable -Name "PnP.CustomUtilities"
```

If `Get-Module` (without `-ListAvailable`) shows nothing, the import failed silently.

## Common Causes

### 1. Module Has No Exported Functions

```powershell
# Check the .psm1 file for Export-ModuleMember
Get-Content "$env:USERPROFILE\Documents\WindowsPowerShell\Modules\PnP.CustomUtilities\PnP.CustomUtilities.psm1"
```

**Fix**: Add to end of `.psm1` file:

```powershell
Export-ModuleMember -Function * 
# Or specify functions explicitly:
Export-ModuleMember -Function Get-CustomData, Set-CustomConfig
```

### 2. Manifest Restricts Exports

```powershell
# Check manifest FunctionsToExport
$Manifest = Import-PowerShellDataFile "$env:USERPROFILE\Documents\WindowsPowerShell\Modules\PnP.CustomUtilities\PnP.CustomUtilities.psd1"
$Manifest.FunctionsToExport
```

If it shows `@()` (empty array), no functions are exported.

**Fix**: Edit `.psd1` file:

```powershell
FunctionsToExport = '*'  # Export all
# Or list specific functions
FunctionsToExport = @('Get-CustomData', 'Set-CustomConfig')
```

### 3. Functions Not Defined in Module

```powershell
# Check what's actually in the .psm1
Select-String -Path "$env:USERPROFILE\Documents\WindowsPowerShell\Modules\PnP.CustomUtilities\PnP.CustomUtilities.psm1" -Pattern "^function "
```

If no matches, the `.psm1` contains no functions.

## Quick Diagnostic

```powershell
# Import with verbose output
Import-Module "PnP.CustomUtilities" -Verbose -Force

# Check for errors
Import-Module "PnP.CustomUtilities" -Force -ErrorAction Stop
```

**Most likely issue**: Missing `Export-ModuleMember` or `FunctionsToExport = @()` in manifest.
