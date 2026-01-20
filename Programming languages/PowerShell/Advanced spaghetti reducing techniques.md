Perfect — here is the **same content, but PowerShell‑specific**, compressed for **human + LLM implementation**.

---

# Reducing Flag Soup in **PowerShell**

## Problem (PS Context)

Flag soup in PowerShell typically appears as:

- Many `[bool]$X` parameters

- `$PSBoundParameters.ContainsKey(...)`

- Nested `if ($A) { if ($B) { … } }`

- Script behavior controlled by switches deep inside logic

This destroys readability and makes scripts unsafe to extend.

---

## PowerShell‑Specific Patterns

### 1. **Replace Switch Parameters with Parameter Sets**

✅ *Primary PS‑native solution*

**Rule:** Mutually exclusive flags → **parameter sets**

```powershell

param(

[Parameter(ParameterSetName="Fast")]

[switch]$Fast,

[Parameter(ParameterSetName="Safe")]

[switch]$Safe

)

```

✅ Forces valid combinations

✅ Help + tab completion improve automatically

✅ Illegal flag combos impossible

---

### 2. **Replace Behavior Flags with ScriptBlocks**

✅ *PowerShell‑idiomatic Strategy Pattern*

```powershell

param(

[scriptblock]$RetryPolicy

)

& $RetryPolicy

```

Instead of:

```powershell

if ($Retry) { Retry-Logic }

```

✅ Behavior passed explicitly

✅ Zero branching inside core logic

---

### 3. **Use Enums Instead of Multiple Switches**

✅ *Best when flags represent modes*

```powershell

enum ExecutionMode {

Fast

Safe

DryRun

}

param(

[ExecutionMode]$Mode

)

```

✅ One variable instead of N switches

✅ Self‑documenting

✅ Easy `switch ($Mode) {}`

---

### 4. **Use Parameter Objects (Hashtable / PSCustomObject)**

✅ *Replace clusters of related switches*

```powershell

param(

[pscustomobject]$ExecutionPolicy

)

```

```powershell

$ExecutionPolicy = @{

Retry = $true

Timeout = 30

}

```

✅ Groups meaning

✅ Easier evolution than adding new switches

---

### 5. **Move Switch Logic to the Edge**

✅ *PowerShell best practice*

```powershell

# BEGIN block: interpret switches

$executor = if ($Fast) { $FastExecutor } else { $SafeExecutor }

# PROCESS block: no flags

$executor.Invoke()

```

✅ Core logic stays linear

✅ Flags only affect wiring, not behavior

---

### 6. **Use Guard Clauses, Not State Switches**

✅ *Especially for scripts*

```powershell

if (-not $IsAdmin) { throw "Admin required" }

```

❌ Instead of:

```powershell

$CanProceed = $true

if (-not $IsAdmin) { $CanProceed = $false }

```

✅ Fewer variables

✅ Shorter mental stack

---

### 7. **Use Capabilities Instead of Mode Checks**

✅ *RBAC‑friendly*

```powershell

if ($UserCapabilities -contains 'Edit') { ... }

```

❌ Not:

```powershell

if ($IsAdmin -or $IsEditor) { ... }

```

✅ Scales cleanly

✅ Maps well to security models

---

### 8. **Replace Feature Flags with Functions**

✅ *PowerShell favors verbs over flags*

```powershell

Enable-Logging

Disable-Logging

```

❌ Not:

```powershell

-EnableLogging:$true

```

✅ Discoverable via `Get-Command`

✅ Composable in pipelines

---

## Why This Works (PowerShell‑Specific)

- PowerShell already has **metadata‑rich parameters**

- Parameter sets encode constraints **without runtime logic**

- ScriptBlocks allow **behavior injection**

- Enums + objects reduce `$PSBoundParameters` abuse

- Verb‑based commands replace mode flags naturally

---

## Advantages

### Human

- Cmdlet help becomes accurate

- Tab completion guides correct usage

- Scripts read top‑down, not branch‑by‑branch

### Technical

- Fewer runtime checks

- Smaller test matrix

- Better error messages

- Safer refactors

---

## PowerShell Heuristics (Memorable)

> **If two switches can’t coexist → parameter sets**

> **If a switch changes behavior → scriptblock or enum**

> **If you check `$PSBoundParameters` → you’re already late**

---

If you want next:

- ✅ a **flag‑heavy PowerShell script refactored live**

- ✅ a **decision table: “flag shape → PowerShell pattern”**

- ✅ a **PowerShell linter rule set to detect flag soup**
