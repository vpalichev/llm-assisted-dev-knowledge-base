# Diagnostic Algorithm: Excel Hang with GTB DLP + SharePoint AutoSave

## Objective

Identify the blocking component in the chain: **Excel → GTB Injector → Network/DLP Scan → SharePoint WOPI** using measurable data, not assumptions.

---

## Phase 1: Baseline Measurement

### 1.1 Establish Normal Behavior

**Tool:** PowerShell + Stopwatch

```powershell
# Measure AutoSave round-trip without GTB (if test machine available)
$file = "C:\Users\$env:USERNAME\OneDrive - Company\TestFile.xlsx"
$excel = New-Object -ComObject Excel.Application
$wb = $excel.Workbooks.Open($file)
$sw = [System.Diagnostics.Stopwatch]::StartNew()
$wb.Sheets[1].Cells[1,1].Value = (Get-Date).ToString()
$wb.Save()
$sw.Stop()
Write-Host "Save duration: $($sw.ElapsedMilliseconds) ms"
$wb.Close()
$excel.Quit()
```

**Expected baseline:** 500–2000 ms for a small file over decent network. **Hang threshold:** >10,000 ms indicates blocking.

---

## Phase 2: Instrumentation Setup

### 2.1 Process Monitor Configuration

**Tool:** `Procmon64.exe` (Sysinternals)

**Filter configuration (set BEFORE capture):**

|Column|Relation|Value|Action|
|---|---|---|---|
|Process Name|contains|`EXCEL`|Include|
|Process Name|contains|`OneDrive`|Include|
|Process Name|contains|`GTB`|Include|
|Process Name|contains|`MIP`|Include|
|Operation|is|`Process Create`|Include|
|Operation|is|`Thread Create`|Include|
|Operation|is|`IRP_MJ`|Include|
|Result|contains|`TIMEOUT`|Include|
|Result|contains|`PENDING`|Include|
|Duration|more than|`1`|Include|

**Critical columns to enable:** `Duration`, `Thread ID`, `Detail`, `Time of Day`

**Export format:** PML (native) for analysis, CSV for scripting.

---

### 2.2 Fiddler Configuration

**Tool:** Fiddler Classic or Fiddler Everywhere

**Setup steps:**

1. `Tools > Options > HTTPS > Decrypt HTTPS traffic` — enable
2. Trust root certificate when prompted
3. Filter: `Filters > Request Headers > Show only if URL contains`:
    
    ```
    sharepoint.comwopi.ashxoffice.comgtbprotection.outlook.com
    ```
    

**Columns to observe:**

- `#` (sequence)
- `Result` (HTTP status)
- `Host`
- `URL`
- `Time` (request timestamp)
- `Duration` (total round-trip)

**Capture flags:** Look for requests with `Duration > 5000 ms` or `Result = 0` (incomplete).

---

### 2.3 Windows Performance Recorder (Optional, Deep Dive)

**Tool:** `wpr.exe` (built into Windows)

```cmd
:: Start trace
wpr -start CPU -start FileIO -start Network

:: Reproduce hang

:: Stop and save
wpr -stop HangTrace.etl
```

**Analysis tool:** Windows Performance Analyzer (`wpa.exe`)

Useful when ProcMon doesn't reveal the blocking call — WPR captures kernel-level wait chains.

---

## Phase 3: Capture Procedure

### 3.1 Controlled Reproduction

**Sequence:**

1. Close all Office applications
2. Start ProcMon (paused)
3. Start Fiddler (capturing)
4. Open Task Manager → Details tab, sort by CPU
5. Resume ProcMon capture (`Ctrl+E`)
6. Open target Excel file from SharePoint (via browser "Open in Desktop App" or OneDrive sync folder)
7. Make trivial edit (type single character in any cell)
8. Observe: AutoSave triggers in 3–10 seconds
9. **If hang occurs:** Wait for resolution or timeout
10. Immediately pause ProcMon (`Ctrl+E`)
11. Stop Fiddler capture
12. Save both captures with timestamp

---

## Phase 4: Analysis Decision Tree

### 4.1 Network Layer Analysis (Fiddler)

```
Q1: Are there pending requests (Result = 0) during hang window?
    │
    ├─ YES → Identify target host
    │        │
    │        ├─ sharepoint.com / wopi.ashx
    │        │   → Network latency or SharePoint throttling
    │        │   → Measure: Check `Timeline` tab for TCP connect vs. server response
    │        │
    │        ├─ gtb* or internal DLP endpoint
    │        │   → GTB cloud/server-side scan blocking
    │        │   → Escalate to GTB admin with request URL + timing
    │        │
    │        └─ protection.outlook.com
    │            → Microsoft MIP layer (separate from GTB)
    │            → Check for duplicate DLP enforcement
    │
    └─ NO → Hang is client-side (proceed to ProcMon analysis)
```

### 4.2 Process Layer Analysis (ProcMon)

**Step 1: Identify hang window**

Find the timestamp range where Excel becomes unresponsive. Correlate with:

- Last successful `WriteFile` operation
- First `ReadFile` or `IRP_MJ_CREATE` after resume

**Step 2: Filter to hang window**

`Tools > Count Occurrences > Column: Process Name`

Identify which process has highest activity during hang. Expected suspects:

|Process|Implication|
|---|---|
|`EXCEL.EXE` only|Internal Excel processing or waiting on IPC|
|`GTBInjector64.exe`|DLP hook is blocking|
|`GTBOCRWorker64.exe`|Content scan (image/embedded object)|
|`OneDrive.exe`|Sync client file lock or upload stall|

**Step 3: Thread-level inspection**

Filter by `EXCEL.EXE`, add `Thread ID` column. Identify thread(s) with:

- `IRP_MJ_CREATE` followed by long gap
- `ReadFile` or `WriteFile` with `Duration > 1 sec`
- Operations on paths containing `GTB` or `pipe` (named pipe IPC)

**Step 4: Cross-process IPC detection**

Search for:

```
Path contains: \Device\NamedPipe
Path contains: GTB
Operation: CreateFile, ReadFile, WriteFile
```

Long-duration pipe operations indicate Excel waiting on GTB response.

---

## Phase 5: Quantitative Diagnosis

### 5.1 Metric Collection Script

Run after capture to extract key metrics:

```powershell
# Parse ProcMon CSV export
$log = Import-Csv "ProcmonLog.csv"

# Total time Excel spent waiting on GTB-related operations
$gtbOps = $log | Where-Object { 
    $_.'Process Name' -eq 'EXCEL.EXE' -and 
    $_.Path -match 'GTB|NamedPipe' 
}

$totalWait = ($gtbOps | Measure-Object -Property 'Duration' -Sum).Sum
Write-Host "Excel time blocked on GTB IPC: $totalWait seconds"

# Count OCR worker invocations during window
$ocrEvents = $log | Where-Object { $_.'Process Name' -like 'GTBOCRWorker*' }
Write-Host "OCR worker events: $($ocrEvents.Count)"
```

### 5.2 Decision Criteria

|Metric|Threshold|Conclusion|
|---|---|---|
|Excel→GTB IPC wait|> 5 sec|GTB injector blocking|
|OCR worker events|> 50 per save|Aggressive content scan|
|Fiddler WOPI `PutFile` duration|> 10 sec|Network/SharePoint issue|
|No IPC, no network delay|—|Excel internal hang (different issue)|

---

## Phase 6: Evidence Package for Escalation

### 6.1 Required Artifacts

|Artifact|Source|Purpose|
|---|---|---|
|`HangCapture.pml`|ProcMon|Full syscall trace|
|`HangCapture.saz`|Fiddler|Network trace|
|`Timeline.xlsx`|Manual|Annotated timestamps: edit, hang start, hang end|
|`ProcessMetrics.txt`|PowerShell script|Aggregated wait times|
|`GTBVersion.txt`|`wmic product where "name like 'GTB%'" get name,version`|Exact agent version|
|`OfficeVersion.txt`|`winver` + Office About|Build numbers|

### 6.2 Escalation Paths

|Conclusion|Escalation Target|Key Evidence|
|---|---|---|
|GTB injector blocking|GTB/Fortra support or internal security team|IPC wait times, pipe operations|
|GTB OCR overload|GTB policy admin|OCR event count, CPU during scan|
|SharePoint throttling|M365 admin|WOPI call latency, HTTP 429/503 codes|
|OneDrive sync stall|M365 admin|OneDrive.exe file lock duration|
|Dual DLP conflict|Security architecture|Both GTB and MIP activity in same window|

---

## Phase 7: Validation Test

After remediation (policy change, exclusion, upgrade), repeat Phase 1 baseline measurement. Compare:

```powershell
# Before fix
Save duration: 14320 ms  # Hang

# After fix
Save duration: 1120 ms   # Normal
```

Document delta as proof of resolution.



----------------------------------------------

# Diagnostic Algorithm Extension: Intermittent Hang Conditions

## Problem Characteristics

Intermittent failures introduce non-determinism. Manual capture (Phase 3 in prior algorithm) becomes impractical—you cannot predict when the hang will occur. The diagnostic strategy must shift from **reactive observation** to **continuous instrumentation with post-hoc correlation**.

---

## Strategy: Ambient Data Collection + Trigger-Based Capture

### Core Principle

Collect lightweight telemetry continuously. When hang occurs, correlate against ambient data to identify differentiating conditions between hang vs. no-hang events.

---

## Phase A: Continuous Background Instrumentation

### A.1 Performance Counter Logging

**Tool:** `logman.exe` (built-in) or `typeperf.exe`

Creates persistent, low-overhead performance log that captures system state over days/weeks.

```cmd
:: Create data collector set
logman create counter ExcelHangBaseline ^
  -c "\Process(EXCEL*)\% Processor Time" ^
     "\Process(EXCEL*)\IO Read Bytes/sec" ^
     "\Process(EXCEL*)\IO Write Bytes/sec" ^
     "\Process(EXCEL*)\Thread Count" ^
     "\Process(GTB*)\% Processor Time" ^
     "\Process(OneDrive*)\IO Write Bytes/sec" ^
     "\Memory\Available MBytes" ^
     "\Network Interface(*)\Bytes Sent/sec" ^
     "\PhysicalDisk(_Total)\Avg. Disk Queue Length" ^
  -si 5 ^
  -f csv ^
  -o "C:\Logs\ExcelBaseline.csv" ^
  -rf 168:00:00

:: Start collection
logman start ExcelHangBaseline
```

**Parameters explained:**

|Flag|Value|Purpose|
|---|---|---|
|`-si`|`5`|Sample interval: 5 seconds|
|`-rf`|`168:00:00`|Run for 168 hours (1 week)|
|`-f`|`csv`|Output format for analysis|

**Overhead:** <1% CPU. Safe for production workstations.

---

### A.2 Event Log Subscription

**Tool:** Task Scheduler + PowerShell

Create a persistent log of application events that may correlate with hangs.

```powershell
# Save as C:\Scripts\Log-ExcelEvents.ps1

$logPath = "C:\Logs\ExcelEvents.csv"

$events = Get-WinEvent -FilterHashtable @{
    LogName = 'Application'
    ProviderName = 'Microsoft Office*', 'ESENT', 'Application Hang', 'Application Error'
    StartTime = (Get-Date).AddMinutes(-5)
} -ErrorAction SilentlyContinue

$events | Select-Object TimeCreated, ProviderName, Id, Message | 
    Export-Csv -Path $logPath -Append -NoTypeInformation
```

**Schedule:** Every 5 minutes via Task Scheduler.

---

### A.3 GTB-Specific Telemetry

GTB logs typically reside in:

```
C:\ProgramData\GTB\Logs\
C:\Program Files\GTB\Logs\
```

**Identify exact path:**

```powershell
Get-ChildItem -Path "C:\ProgramData", "C:\Program Files", "C:\Program Files (x86)" `
  -Recurse -Filter "*.log" -ErrorAction SilentlyContinue | 
  Where-Object { $_.FullName -match "GTB" } |
  Select-Object FullName, LastWriteTime
```

**Continuous monitoring:**

```powershell
# Tail GTB logs for scan duration entries
Get-Content "C:\ProgramData\GTB\Logs\GTBAgent.log" -Tail 100 -Wait |
  Select-String -Pattern "scan|duration|block|timeout" |
  ForEach-Object { 
    "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') $_" | 
    Out-File "C:\Logs\GTBActivity.log" -Append 
  }
```

---

## Phase B: Hang Detection Trigger

### B.1 Automated Hang Detection Script

**Tool:** PowerShell scheduled task (runs every 30 seconds)

```powershell
# Save as C:\Scripts\Detect-ExcelHang.ps1

$threshold = 10  # seconds unresponsive = hang

$excel = Get-Process EXCEL -ErrorAction SilentlyContinue
if (-not $excel) { exit }

Add-Type @"
using System;
using System.Runtime.InteropServices;
public class Win32 {
    [DllImport("user32.dll")]
    public static extern bool IsHungAppWindow(IntPtr hWnd);
    
    [DllImport("user32.dll")]
    public static extern IntPtr GetForegroundWindow();
}
"@

foreach ($proc in $excel) {
    $hwnd = $proc.MainWindowHandle
    if ($hwnd -ne [IntPtr]::Zero) {
        $isHung = [Win32]::IsHungAppWindow($hwnd)
        
        if ($isHung) {
            $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
            
            # Log hang event
            $hangRecord = [PSCustomObject]@{
                Timestamp     = (Get-Date)
                ProcessId     = $proc.Id
                WorkingSetMB  = [math]::Round($proc.WorkingSet64 / 1MB, 2)
                ThreadCount   = $proc.Threads.Count
                HandleCount   = $proc.HandleCount
                CPUTime       = $proc.TotalProcessorTime
            }
            $hangRecord | Export-Csv "C:\Logs\HangEvents.csv" -Append -NoTypeInformation
            
            # Trigger deep capture (see B.2)
            & "C:\Scripts\Capture-HangState.ps1" -Timestamp $timestamp -Pid $proc.Id
        }
    }
}
```

**API used:** `IsHungAppWindow` — Windows User32 function. Returns `$true` if window message queue has not been serviced for ~5 seconds. This is the same heuristic Windows uses to display "(Not Responding)".

---

### B.2 Triggered Deep Capture

**Tool:** Automated ProcMon + dump collection

```powershell
# Save as C:\Scripts\Capture-HangState.ps1
param(
    [string]$Timestamp,
    [int]$Pid
)

$captureDir = "C:\Logs\HangCaptures\$Timestamp"
New-Item -ItemType Directory -Path $captureDir -Force | Out-Null

# 1. Process dump (full memory)
$dumpPath = "$captureDir\EXCEL_$Pid.dmp"
& "C:\Tools\procdump64.exe" -ma $Pid $dumpPath -accepteula

# 2. Mini ProcMon capture (30 seconds around event)
$pmlPath = "$captureDir\Procmon.pml"
& "C:\Tools\Procmon64.exe" /Quiet /Minimized /BackingFile $pmlPath
Start-Sleep -Seconds 30
& "C:\Tools\Procmon64.exe" /Terminate

# 3. Network state snapshot
netstat -ano | Out-File "$captureDir\netstat.txt"

# 4. GTB process state
Get-Process GTB* | Select-Object Name, Id, CPU, WorkingSet64, Threads | 
    Out-File "$captureDir\GTBProcesses.txt"

# 5. Pending IO snapshot
$proc = Get-Process -Id $Pid
$threads = $proc.Threads | Where-Object { $_.WaitReason -eq 'Executive' -or $_.ThreadState -eq 'Wait' }
$threads | Select-Object Id, ThreadState, WaitReason | 
    Out-File "$captureDir\WaitingThreads.txt"

# 6. Copy recent GTB logs
Copy-Item "C:\ProgramData\GTB\Logs\*.log" "$captureDir\GTBLogs\" -Recurse -ErrorAction SilentlyContinue

Write-Host "Capture complete: $captureDir"
```

**Dependencies:**

- `procdump64.exe` — Sysinternals, must be pre-staged in `C:\Tools\`
- `Procmon64.exe` — Sysinternals, same location

---

## Phase C: Statistical Correlation Analysis

### C.1 Hang vs. No-Hang Comparison

After collecting multiple hang events (minimum 5–10 for statistical validity), compare environmental conditions.

```powershell
# Load datasets
$hangs = Import-Csv "C:\Logs\HangEvents.csv"
$perf = Import-Csv "C:\Logs\ExcelBaseline.csv"

# Convert timestamps
$hangs | ForEach-Object { 
    $_.Timestamp = [DateTime]::Parse($_.Timestamp) 
}

# For each hang, extract perf counters from ±60 seconds
$correlations = foreach ($hang in $hangs) {
    $window = $perf | Where-Object {
        $ts = [DateTime]::Parse($_.'(PDH-CSV 4.0) (W. Europe Standard Time)(0)')
        [Math]::Abs(($ts - $hang.Timestamp).TotalSeconds) -le 60
    }
    
    [PSCustomObject]@{
        HangTime           = $hang.Timestamp
        AvgGTBCpu          = ($window.'\\*\Process(GTB*)\% Processor Time' | Measure-Object -Average).Average
        AvgExcelIO         = ($window.'\\*\Process(EXCEL*)\IO Write Bytes/sec' | Measure-Object -Average).Average
        AvgDiskQueue       = ($window.'\\*\PhysicalDisk(_Total)\Avg. Disk Queue Length' | Measure-Object -Average).Average
        AvgAvailableMemMB  = ($window.'\\*\Memory\Available MBytes' | Measure-Object -Average).Average
    }
}

$correlations | Format-Table -AutoSize
```

### C.2 Pattern Identification Matrix

|Condition|Hang Events (avg)|Normal Operation (avg)|Significant?|
|---|---|---|---|
|GTB CPU %|?|?|Δ > 20%|
|Excel IO Write bytes/sec|?|?|Δ > 50%|
|Disk Queue Length|?|?|> 2|
|Available Memory MB|?|?|< 1000|
|GTBOCRWorker count|?|?|Δ > 5|
|Network bytes/sec|?|?|Near zero during hang|

Fill in measured values. Statistically significant deltas indicate causal factors.

---

## Phase D: Temporal Pattern Analysis

### D.1 Time-of-Day Correlation

```powershell
$hangs = Import-Csv "C:\Logs\HangEvents.csv"

$hangs | 
    ForEach-Object { [DateTime]::Parse($_.Timestamp).Hour } |
    Group-Object |
    Sort-Object Name |
    Select-Object @{N='Hour';E={$_.Name}}, Count |
    Format-Table
```

**Interpretation:**

|Pattern|Implication|
|---|---|
|Clustered 9–10 AM|Login storm, GPO processing, GTB policy sync|
|Clustered 12–1 PM|Lunch hour backup jobs|
|Uniform distribution|No temporal trigger — content-dependent|

### D.2 File-Specific Correlation

Track which files trigger hangs:

```powershell
# Add to Detect-ExcelHang.ps1 capture routine:
$excel = New-Object -ComObject Excel.Application
$activeFile = $excel.ActiveWorkbook.FullName
# Log to HangEvents.csv
```

If specific files consistently trigger hangs:

- Large file size
- Embedded images (triggers GTB OCR)
- External data connections
- Macros invoking network calls

---

## Phase E: Controlled Variable Testing

Once correlation identifies candidate causes, validate via controlled experiment.

### E.1 Test Matrix

|Test|Condition|Expected Outcome|
|---|---|---|
|A|Disable GTB (if permitted)|No hangs → GTB confirmed|
|B|Disable AutoSave only|Hangs stop → AutoSave frequency is trigger|
|C|File without images|Hangs stop → GTB OCR is cause|
|D|File on local disk (no SharePoint)|Hangs stop → Network/WOPI is cause|
|E|Different time of day|Hangs stop → Temporal system load factor|

### E.2 A/B Logging

```powershell
# Log test condition alongside hang detection
$testCondition = "GTB_DISABLED"  # Change per test run
$hangRecord | Add-Member -NotePropertyName "TestCondition" -NotePropertyValue $testCondition
```

---

## Phase F: Probabilistic Diagnosis Output

After sufficient data collection, produce likelihood assessment:

```
HANG ROOT CAUSE PROBABILITY ASSESSMENT
======================================
Data points collected: 47 hang events over 14 days

Primary suspect: GTB OCR scan on embedded images
  Evidence: 
    - 89% of hangs correlate with files containing ≥3 embedded images
    - GTBOCRWorker CPU spike precedes hang by 2–5 seconds in 91% of cases
    - Files without images: 0 hangs in 312 save operations
  Confidence: HIGH (>90%)

Secondary factor: Network latency during scan
  Evidence:
    - 34% of hangs show concurrent WOPI PutFile timeout
    - May be downstream effect (GTB holds lock → SharePoint times out)
  Confidence: MEDIUM (60–70%)

Recommended remediation:
  1. Request GTB policy change: disable OCR for Office documents
  2. Alternative: exclude SharePoint sync folder from GTB real-time scan
```

---

## Summary: Intermittent vs. Consistent Hang Approach

|Aspect|Consistent Hang|Intermittent Hang|
|---|---|---|
|Capture method|Manual, on-demand|Automated, triggered|
|Data volume|Single session|Days/weeks of telemetry|
|Analysis|Direct observation|Statistical correlation|
|Tooling|ProcMon + Fiddler|Performance counters + scheduled scripts|
|Diagnosis|Deterministic|Probabilistic|
|Time to diagnosis|Hours|Days to weeks|



----------------------------------------------------

# Prompt


# System Prompt: Excel Hang Diagnostic Agent (GTB DLP + SharePoint Environment)

```
You are a Windows systems diagnostic specialist focused on identifying root causes of Microsoft Excel application hangs in enterprise environments running GTB Technologies (Fortra) Data Loss Prevention software with SharePoint Online / OneDrive file synchronization.

## Environment Context

Target systems exhibit the following configuration:
- Microsoft Excel (Microsoft 365 / Office 2019+)
- GTB DLP agent with injector-based monitoring (GTBInjector32.exe, GTBInjector64.exe)
- GTB OCR content scanning (GTBOCRWorker64.exe processes)
- SharePoint Online document libraries accessed via:
  - OneDrive sync client (Files On-Demand)
  - WOPI protocol (Open in Desktop App)
- AutoSave enabled for cloud-hosted documents

## Failure Mode Definition

"Hang" is defined as: Excel main window fails to process Windows messages for >5 seconds, triggering "(Not Responding)" state as detected by Win32 API `IsHungAppWindow()`. The hang may resolve spontaneously or require forced termination.

## Diagnostic Toolchain

You have expertise in the following Windows diagnostic tools:

| Tool | Purpose | Data Produced |
|------|---------|---------------|
| Process Monitor (Procmon64.exe) | Syscall-level file, registry, network, process activity | PML/CSV trace files |
| Fiddler Classic | HTTPS traffic inspection including WOPI and DLP endpoints | SAZ session archives |
| Performance Monitor (logman.exe, typeperf.exe) | Continuous performance counter collection | CSV time-series data |
| Process Dump (procdump64.exe) | Full memory dumps of hung processes | DMP files for WinDbg analysis |
| Windows Performance Recorder (wpr.exe) | Kernel-level ETW tracing for wait chain analysis | ETL files |
| PowerShell | Scripted data collection, correlation analysis, automation | Custom telemetry |

## Diagnostic Methodology

### Classification: Consistent vs. Intermittent

First, determine hang reproducibility:

**Consistent hang**: Occurs on every or nearly every AutoSave operation.
- Approach: Real-time manual capture with ProcMon + Fiddler
- Timeline: Diagnosis achievable in single session (hours)

**Intermittent hang**: Occurs unpredictably, frequency ranges from multiple times daily to weekly.
- Approach: Continuous ambient telemetry with automated trigger-based deep capture
- Timeline: Requires data collection over days/weeks for statistical correlation

### Analysis Framework

For any hang event, identify the blocking component in this chain:

```

Excel.exe → GTBInjector64.exe (DLP hook intercepts I/O) → GTBOCRWorker64.exe (content scan, if images present) → GTB policy server (cloud/on-prem validation) → OneDrive.exe / WOPI endpoint (SharePoint upload)

```

The hang occurs when any component in this chain blocks without returning control to Excel's main thread.

### Decision Tree: Locating the Block

```

1. Is there network activity during hang? │ ├─ YES: Check Fiddler │ ├─ Pending request to sharepoint.com/wopi.ashx → SharePoint/network issue │ ├─ Pending request to GTB endpoint → GTB server-side scan delay │ └─ Pending request to protection.outlook.com → Microsoft MIP (separate from GTB) │ └─ NO: Block is client-side, check ProcMon ├─ Excel waiting on named pipe to GTB → GTBInjector blocking ├─ GTBOCRWorker high CPU during window → OCR scan delay ├─ Excel waiting on OneDrive.exe file lock → Sync client issue └─ Excel internal wait (no IPC) → Office bug, not DLP-related

```

### Key Metrics and Thresholds

| Metric | Collection Method | Normal | Hang Indicator |
|--------|-------------------|--------|----------------|
| AutoSave round-trip | Stopwatch measurement | 500–2000 ms | >10,000 ms |
| GTBInjector CPU during save | perfmon Process counter | <5% | >30% sustained |
| GTBOCRWorker process count | tasklist enumeration | 0–4 | >8 simultaneous |
| WOPI PutFile duration | Fiddler timeline | <3000 ms | >10,000 ms or timeout |
| Excel thread wait state | ProcMon Thread Create/Exit | Transient | Extended 'Executive' wait |
| Named pipe IPC duration | ProcMon filtered by \Device\NamedPipe + GTB | <100 ms | >5000 ms |

## Intermittent Hang: Statistical Correlation Protocol

When hangs are intermittent, shift from direct observation to correlation analysis:

1. **Ambient Collection**: Deploy continuous performance counter logging (5-second intervals) capturing Excel, GTB, OneDrive, disk, memory, network metrics.

2. **Automated Detection**: Schedule PowerShell script using `IsHungAppWindow()` API to detect hang events and log metadata.

3. **Triggered Capture**: Upon hang detection, automatically collect:
   - Process dump (procdump -ma)
   - 30-second ProcMon window
   - Network state (netstat -ano)
   - GTB log snapshot

4. **Correlation Analysis**: After collecting ≥5 hang events, compare environmental conditions during hang vs. normal operation windows. Identify statistically significant deltas in:
   - GTB process CPU utilization
   - OCR worker count
   - File characteristics (size, embedded objects)
   - Time of day patterns
   - Network throughput

5. **Controlled Variable Testing**: Validate suspected cause by modifying single variable:
   - Disable GTB (if permitted)
   - Disable AutoSave
   - Test file without embedded images
   - Test file on local disk (no SharePoint)

## Output Format

When diagnosing, produce structured findings:

```

# DIAGNOSTIC SUMMARY

Hang type: [Consistent | Intermittent] Data points: [N hang events over N days] Confidence level: [HIGH >90% | MEDIUM 60-90% | LOW <60%]

PRIMARY CAUSE: [Component name] Evidence:

- [Specific metric or observation]
- [Correlation data if intermittent]
- [ProcMon/Fiddler findings if consistent]

SECONDARY FACTORS: [If applicable]

RECOMMENDED ACTIONS:

1. [Specific remediation step]
2. [Escalation path with required evidence artifacts]

ARTIFACTS FOR ESCALATION:

- [List of files: PML, SAZ, DMP, CSV, logs]

```

## Escalation Routing

| Diagnosed Cause | Escalation Target | Required Evidence |
|-----------------|-------------------|-------------------|
| GTB Injector blocking | GTB/Fortra support or internal security team | IPC wait times, pipe operations from ProcMon |
| GTB OCR overload | GTB policy administrator | OCR worker count, CPU metrics, file content analysis |
| GTB policy server timeout | GTB infrastructure team | Fiddler capture showing GTB endpoint latency |
| SharePoint throttling | Microsoft 365 administrator | WOPI call latency, HTTP 429/503 response codes |
| OneDrive sync client | Microsoft 365 administrator | OneDrive.exe file lock duration, sync client logs |
| Dual DLP conflict (GTB + MIP) | Security architecture team | Both GTB and MIP activity in same time window |

## Constraints

- Do not recommend generic solutions ("disable antivirus", "restart computer") without diagnostic evidence
- All conclusions must be supported by measurable data from specified toolchain
- Acknowledge uncertainty explicitly when data is insufficient
- For intermittent issues, state minimum data collection period required before diagnosis is possible
- Prefer non-invasive diagnostics; escalate to memory dumps only when syscall-level tracing is insufficient
```

---

**Usage:** This prompt configures an AI assistant (or serves as procedural documentation for human analysts) to systematically diagnose Excel hang conditions in GTB DLP environments using concrete measurement tools rather than speculative troubleshooting.




