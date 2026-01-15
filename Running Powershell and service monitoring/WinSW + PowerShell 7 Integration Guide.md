Companion guide for running PowerShell scripts as Windows services using WinSW, integrated with the Clojure orchestration layer.

---

## 1. When to Use WinSW

Use WinSW instead of direct process invocation for:

- **Long-running tasks** — Operations exceeding typical timeout thresholds (minutes to hours)
- **Scheduled/continuous operations** — Scripts that run on intervals or monitor for events
- **Startup requirements** — Scripts that must run before user login
- **Recovery requirements** — Scripts that should auto-restart on failure

Direct `clojure.java.process` invocation remains appropriate for short, request-response style operations.

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Clojure Application                                            │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │ Service Manager │  │ File Watcher     │  │ Result Parser  │  │
│  │ (WinSW control) │  │ (artifact poll)  │  │ (JSON envelope)│  │
│  └────────┬────────┘  └────────┬─────────┘  └───────┬────────┘  │
└───────────┼────────────────────┼────────────────────┼───────────┘
            │                    │                    │
            ▼                    ▼                    ▼
┌───────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
│ WinSW Service     │  │ Work Directory  │  │ Result Files        │
│ (manages pwsh.exe)│  │ progress.json   │  │ result.json         │
└─────────┬─────────┘  └─────────────────┘  └─────────────────────┘
          │
          ▼
┌─────────────────────────────────────────┐
│ pwsh.exe (PowerShell 7)                 │
│ Running script as service worker        │
└─────────────────────────────────────────┘
```

Communication shifts from stdout to file-based:

|Concern|Direct Invocation|WinSW Service|
|---|---|---|
|Result envelope|stdout|`{work-dir}/result.json`|
|Progress|progress.json (optional)|progress.json (required)|
|Completion signal|Process exit|`.complete` marker file|
|Errors|stderr + exit code|`result.json` + Windows Event Log|

---

## 3. WinSW Configuration

### 3.1 Service Definition

Each long-running script gets a WinSW XML configuration:

```xml
<!-- Export-Customers-Service.xml -->
<service>
  <id>app-export-customers</id>
  <name>AppName - Customer Export Service</name>
  <description>Runs customer data export operations</description>
  
  <executable>pwsh.exe</executable>
  <arguments>-NoProfile -NonInteractive -ExecutionPolicy Bypass -File "%BASE%\Export-Customers.ps1"</arguments>
  
  <workingdirectory>%BASE%</workingdirectory>
  
  <log mode="roll-by-size">
    <sizeThreshold>10240</sizeThreshold>
    <keepFiles>5</keepFiles>
  </log>
  
  <onfailure action="restart" delay="10 sec"/>
  <onfailure action="restart" delay="30 sec"/>
  <onfailure action="none"/>
  
  <resetfailure>1 hour</resetfailure>
  
  <env name="APP_ROOT" value="%LOCALAPPDATA%\AppName"/>
  <env name="WORK_DIR" value="%LOCALAPPDATA%\AppName\work\%SERVICE_ID%"/>
  <env name="OUTPUT_DIR" value="%LOCALAPPDATA%\AppName\output"/>
  
  <serviceaccount>
    <username>.\ServiceAccount</username>
    <password>MANAGED_SERVICE_ACCOUNT</password>
    <allowservicelogon>true</allowservicelogon>
  </serviceaccount>
  
  <securitydescriptor>D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;RPWPCR;;;IU)</securitydescriptor>
</service>
```

### 3.2 Key Configuration Options

|Element|Purpose|
|---|---|
|`<onfailure>`|Restart policy on script crash|
|`<resetfailure>`|Time before failure counter resets|
|`<log>`|Captures stdout/stderr to rotating files|
|`<env>`|Pass environment variables matching spec conventions|
|`<serviceaccount>`|Run with specific credentials for DB/network access|

### 3.3 Directory Structure

```
%LOCALAPPDATA%\AppName\
├── services/
│   ├── Export-Customers-Service.xml
│   ├── Export-Customers-Service.exe      # WinSW renamed executable
│   ├── Export-Customers-Service.wrapper.log
│   └── Export-Customers-Service.out.log
├── scripts/
│   └── data-export/
│       ├── Export-Customers.ps1
│       └── Export-Customers.manifest.json
├── work/
│   └── app-export-customers/             # Service-specific work dir
│       ├── progress.json
│       ├── result.json
│       └── .complete
└── output/
```

---

## 4. PowerShell 7 Script Adaptations

### 4.1 Service-Aware Script Template

```powershell
#Requires -Version 7.0
param(
    [string]$ParamsJson = '{}'
)

$ErrorActionPreference = 'Stop'
Import-Module "$PSScriptRoot/../lib/ResultEnvelope.psm1"

# Service-mode paths from environment
$workDir = $env:WORK_DIR ?? (Join-Path $env:TEMP "ps-work")
$outputDir = $env:OUTPUT_DIR ?? (Join-Path $env:LOCALAPPDATA "AppName\output")

# Ensure directories exist
New-Item -ItemType Directory -Force -Path $workDir | Out-Null
New-Item -ItemType Directory -Force -Path $outputDir | Out-Null

$params = $ParamsJson | ConvertFrom-Json -AsHashtable
$result = New-ResultEnvelope -ScriptId 'export-customers' -WorkDir $workDir

# Progress reporting for long operations
function Update-Progress {
    param([int]$Percent, [string]$Status)
    @{
        percent = $Percent
        status = $Status
        timestamp = (Get-Date -Format 'o')
    } | ConvertTo-Json | Set-Content (Join-Path $workDir 'progress.json')
}

try {
    Update-Progress -Percent 0 -Status 'Starting'
    
    # Script logic here...
    Update-Progress -Percent 50 -Status 'Processing records'
    
    # More logic...
    Update-Progress -Percent 100 -Status 'Complete'
    
    $result.Status = 'success'
} catch {
    $result | Add-Error -Code 'ERR_INTERNAL' -Message $_.Exception.Message
    $result.Status = 'error'
    
    # Log to Windows Event Log for service visibility
    Write-EventLog -LogName Application -Source 'AppName' -EventId 1001 `
        -EntryType Error -Message $_.Exception.Message
} finally {
    # Write result to file (not stdout)
    $result | Write-ResultEnvelopeToFile -Path (Join-Path $workDir 'result.json')
    
    # Signal completion
    New-Item -ItemType File -Force -Path (Join-Path $workDir '.complete') | Out-Null
}
```

### 4.2 PowerShell 7 Advantages

Leverage PS7 features in scripts:

```powershell
# Parallel processing with ForEach-Object -Parallel
$customers | ForEach-Object -Parallel {
    Export-CustomerRecord -Customer $_ -OutputDir $using:outputDir
} -ThrottleLimit 4

# Ternary operator
$status = $success ? 'success' : 'error'

# Null-coalescing
$timeout = $params.timeout ?? 300

# Pipeline chain operators
Get-Data || Write-Error "Failed to get data" && exit 2

# JSON streaming for large datasets
Get-Content $inputFile | ConvertFrom-Json -AsHashtable | ForEach-Object { ... }
```

---

## 5. Clojure Service Management

### 5.1 WinSW Control Functions

```clojure
(ns app.service.winsw
  (:require [clojure.java.process :as proc]
            [clojure.java.io :as io]))

(def services-dir 
  (io/file (System/getenv "LOCALAPPDATA") "AppName" "services"))

(defn- winsw-exec [service-id command]
  (let [exe (io/file services-dir (str service-id ".exe"))]
    (proc/exec {} [(.getPath exe) command])))

(defn install-service [service-id]
  (winsw-exec service-id "install"))

(defn uninstall-service [service-id]
  (winsw-exec service-id "uninstall"))

(defn start-service [service-id]
  (winsw-exec service-id "start"))

(defn stop-service [service-id]
  (winsw-exec service-id "stop"))

(defn restart-service [service-id]
  (winsw-exec service-id "restart"))

(defn service-status [service-id]
  (let [{:keys [out exit]} (winsw-exec service-id "status")]
    (cond
      (= exit 0) :running
      (= exit 1) :stopped
      :else :unknown)))
```

### 5.2 Result Polling

```clojure
(ns app.service.results
  (:require [clojure.java.io :as io]
            [cheshire.core :as json]
            [app.powershell.result :as result]))

(def work-base 
  (io/file (System/getenv "LOCALAPPDATA") "AppName" "work"))

(defn work-dir [service-id]
  (io/file work-base service-id))

(defn complete? [service-id]
  (.exists (io/file (work-dir service-id) ".complete")))

(defn read-progress [service-id]
  (let [f (io/file (work-dir service-id) "progress.json")]
    (when (.exists f)
      (json/parse-string (slurp f) true))))

(defn read-result [service-id]
  (let [f (io/file (work-dir service-id) "result.json")]
    (when (.exists f)
      (result/parse-result (slurp f)))))

(defn poll-until-complete 
  [service-id {:keys [timeout-ms poll-interval-ms on-progress]
               :or {timeout-ms 300000 poll-interval-ms 1000}}]
  (let [deadline (+ (System/currentTimeMillis) timeout-ms)]
    (loop []
      (cond
        (complete? service-id)
        (read-result service-id)
        
        (> (System/currentTimeMillis) deadline)
        {:error :timeout}
        
        :else
        (do
          (when on-progress
            (when-let [p (read-progress service-id)]
              (on-progress p)))
          (Thread/sleep poll-interval-ms)
          (recur))))))
```

### 5.3 Trigger-Based Execution

For on-demand service tasks, use a trigger file pattern:

```clojure
(defn trigger-task [service-id params]
  (let [work (work-dir service-id)
        trigger-file (io/file work "trigger.json")]
    ;; Clean previous run
    (doseq [f [".complete" "result.json" "progress.json"]]
      (io/delete-file (io/file work f) true))
    ;; Write trigger with parameters
    (spit trigger-file (json/generate-string params))
    ;; Poll for completion
    (poll-until-complete service-id {})))
```

Corresponding PowerShell watches for triggers:

```powershell
# Service main loop
while ($true) {
    $triggerFile = Join-Path $env:WORK_DIR 'trigger.json'
    
    if (Test-Path $triggerFile) {
        $params = Get-Content $triggerFile | ConvertFrom-Json -AsHashtable
        Remove-Item $triggerFile
        
        # Execute task with params
        Invoke-ExportTask -Params $params
    }
    
    Start-Sleep -Seconds 1
}
```

---

## 6. Deployment

### 6.1 Service Installation Script

```powershell
# Install-Services.ps1
#Requires -RunAsAdministrator
#Requires -Version 7.0

param(
    [string]$AppRoot = "$env:LOCALAPPDATA\AppName"
)

$servicesDir = Join-Path $AppRoot 'services'
$winswUrl = 'https://github.com/winsw/winsw/releases/download/v2.12.0/WinSW-x64.exe'

# Download WinSW if needed
$winswMaster = Join-Path $servicesDir 'WinSW.exe'
if (-not (Test-Path $winswMaster)) {
    Invoke-WebRequest -Uri $winswUrl -OutFile $winswMaster
}

# Install each service
Get-ChildItem -Path $servicesDir -Filter '*.xml' | ForEach-Object {
    $serviceId = $_.BaseName
    $serviceExe = Join-Path $servicesDir "$serviceId.exe"
    
    # Copy WinSW as service executable
    if (-not (Test-Path $serviceExe)) {
        Copy-Item $winswMaster $serviceExe
    }
    
    # Install service
    & $serviceExe install
    
    # Create work directory
    $workDir = Join-Path $AppRoot "work\$serviceId"
    New-Item -ItemType Directory -Force -Path $workDir | Out-Null
}

# Register event log source
New-EventLog -LogName Application -Source 'AppName' -ErrorAction SilentlyContinue
```

### 6.2 Service Manifest Extension

Add WinSW metadata to script manifests:

```json
{
  "scriptId": "export-customers",
  "version": "1.2.0",
  "executionMode": "service",
  "service": {
    "id": "app-export-customers",
    "restartOnFailure": true,
    "restartDelay": 10,
    "maxRestarts": 3
  },
  "parameters": { },
  "artifacts": { }
}
```

---

## 7. Monitoring

### 7.1 Service Health Check

```clojure
(defn health-check [service-id]
  (let [status (service-status service-id)
        progress (read-progress service-id)]
    {:service-id service-id
     :status status
     :last-progress progress
     :stalled? (when progress
                 (> (- (System/currentTimeMillis)
                       (parse-timestamp (:timestamp progress)))
                    60000))}))

(defn all-services-health []
  (->> (file-seq services-dir)
       (filter #(.endsWith (.getName %) ".xml"))
       (map #(-> (.getName %) (subs 0 (- (count (.getName %)) 4))))
       (map health-check)))
```

### 7.2 Windows Event Log Integration

Query service errors from Clojure:

```clojure
(defn recent-service-errors [service-id hours]
  (let [{:keys [out]} 
        (proc/exec {} 
          ["pwsh.exe" "-NoProfile" "-Command"
           (format "Get-EventLog -LogName Application -Source 'AppName' -EntryType Error -After (Get-Date).AddHours(-%d) | ConvertTo-Json" 
                   hours)])]
    (json/parse-string out true)))
```

---

## 8. Comparison: Direct vs Service Execution

|Aspect|Direct (`proc/exec`)|WinSW Service|
|---|---|---|
|Startup time|Fast|Slower (service overhead)|
|Max duration|Minutes|Hours/days|
|User session|Required|Independent|
|Result delivery|Immediate (stdout)|Polled (file)|
|Failure recovery|Manual retry|Auto-restart|
|Credential scope|User context|Service account|
|Logging|Application captures|WinSW + Event Log|

Choose direct invocation for interactive, short operations. Choose WinSW for background, long-running, or scheduled tasks.