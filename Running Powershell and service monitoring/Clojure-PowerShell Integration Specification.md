

**Version 1.0 — Technical Specification Document**

---

## Executive Summary

This specification establishes patterns and conventions for integrating Clojure applications with PowerShell scripts on Windows platforms. It addresses the challenge of orchestrating PowerShell operations that produce diverse artifacts including files, database mutations, and system state changes, while ensuring the Clojure application maintains reliable knowledge of what was produced and whether operations succeeded.

---

## 1. Design Principles

The integration layer adheres to the following core principles:

1. **Explicit over implicit:** Scripts declare their artifacts and requirements via manifest files rather than relying on conventions alone.
2. **Structured communication:** All inter-process communication uses JSON with well-defined schemas.
3. **Idempotency where possible:** Scripts should be safely re-runnable; state mutations include idempotency keys.
4. **Fail-fast with recovery:** Operations fail quickly on errors but provide sufficient context for recovery.
5. **Defense in depth:** Multiple validation layers ensure data integrity across the boundary.

---

## 2. Artifact Contracts

### 2.1 Artifact Categories

Scripts produce artifacts in four categories, each with distinct handling requirements:

|Category|Examples|Discovery Method|
|---|---|---|
|File Output|CSV, XLSX, PDF, reports|Manifest file listing paths|
|Directory Structure|Downloaded files, extracted archives|Manifest with root path + file list|
|Database Mutation|INSERT/UPDATE operations|Batch ID in result, query by Clojure|
|System State|Registry, services, config files|Before/after state in result object|

### 2.2 Script Manifest Structure

Each PowerShell script must have a companion manifest file (`script-name.manifest.json`) declaring its contract:

```json
{
  "scriptId": "data-export-customers",
  "version": "1.2.0",
  "description": "Exports customer data to CSV",
  "parameters": {
    "startDate": { "type": "datetime", "required": true },
    "outputDir": { "type": "path", "required": false,
                   "default": "$env:TEMP\\exports" }
  },
  "artifacts": {
    "files": [
      { "pattern": "customers_*.csv", "description": "Customer data export" },
      { "pattern": "summary.json", "description": "Export statistics" }
    ],
    "database": { "tables": ["export_log"], "operation": "insert" }
  },
  "timeout": 300,
  "idempotent": false
}
```

### 2.3 Artifact Discovery

Clojure discovers artifacts through a three-phase process:

1. **Pre-execution:** Load manifest to understand expected artifacts
2. **Post-execution:** Parse result envelope (see Section 4) containing actual artifact locations
3. **Validation:** Verify artifacts exist and match expected patterns from manifest

---

## 3. Output Locations

### 3.1 Directory Hierarchy

All script operations use a standardized directory structure rooted at the application data folder:

```
%LOCALAPPDATA%\AppName\
├── scripts\          # PowerShell scripts and manifests
├── work\             # Ephemeral working directories
│   └── {run-id}\     # Per-execution isolation
├── output\           # Permanent artifact storage
│   └── {date}\       # Date-partitioned
├── logs\             # Execution logs
└── config\           # Configuration files
```

### 3.2 Path Communication

Paths are communicated bidirectionally as follows:

1. **Clojure to PowerShell:** Paths passed as script parameters are always absolute and use forward slashes for JSON compatibility, converted to Windows format within scripts.
2. **PowerShell to Clojure:** Result envelopes contain absolute paths using forward slashes.
3. **Environment variables:** Scripts receive `APP_ROOT`, `WORK_DIR`, and `OUTPUT_DIR` environment variables set by the Clojure invoker.

### 3.3 Temporary vs Permanent Storage

Scripts distinguish between work directories (automatically cleaned after configurable retention period) and output directories (permanent storage requiring explicit deletion). The Clojure application manages cleanup of work directories older than the retention threshold (default: 7 days).

---

## 4. Data Exchange Format

### 4.1 Result Envelope Schema

All PowerShell scripts output a JSON result envelope to stdout as their final action:

```json
{
  "status": "success | partial | error",
  "runId": "uuid",
  "scriptId": "script-identifier",
  "startTime": "ISO-8601",
  "endTime": "ISO-8601",
  "artifacts": {
    "files": [{ "path": "...", "size": 12345, "checksum": "sha256:..." }],
    "directories": [{ "path": "...", "fileCount": 42 }],
    "database": { "batchId": "...", "rowsAffected": 100 }
  },
  "errors": [{ "code": "ERR_XXX", "message": "...", "context": {} }],
  "warnings": ["..."],
  "metrics": { "recordsProcessed": 1000, "duration": 45.2 }
}
```

### 4.2 Clojure Parsing

The Clojure application parses results using Cheshire for JSON deserialization with keyword keys enabled. A dedicated namespace provides schema validation using clojure.spec:

```clojure
(ns app.powershell.result
  (:require [cheshire.core :as json]
            [clojure.spec.alpha :as s]))

(s/def ::status #{:success :partial :error})
(s/def ::run-id uuid?)
(s/def ::result-envelope (s/keys :req-un [::status ::run-id]))

(defn parse-result [json-str]
  (let [result (json/parse-string json-str true)]
    (when-not (s/valid? ::result-envelope result)
      (throw (ex-info "Invalid result envelope"
                      {:explain (s/explain-str ::result-envelope result)})))
    result))
```

### 4.3 Output Isolation

Scripts must write the result envelope as the final line of stdout, prefixed with a marker (`###RESULT###`) to distinguish it from diagnostic output. Any stdout content before the marker is captured as log output. Stderr is captured separately for error diagnostics.

---

## 5. Completion Signaling

### 5.1 Exit Codes

PowerShell scripts use standardized exit codes:

|Code|Status|Description|
|---|---|---|
|0|Success|All artifacts produced successfully|
|1|Partial Success|Some artifacts produced; see result envelope|
|2|Error|Operation failed; result envelope contains errors|
|3|Invalid Parameters|Script invoked with invalid arguments|
|4|Timeout|Operation exceeded time limit|
|5|Permission Denied|Insufficient privileges for operation|

### 5.2 Multi-Phase Signal Protocol

For long-running operations, scripts implement a heartbeat protocol:

1. **Progress file:** Scripts write progress to `{work-dir}/progress.json` updated every 10 seconds
2. **Clojure polling:** The invoker monitors the progress file to detect stalled operations
3. **Completion marker:** A `{work-dir}/.complete` file signals that `result.json` is ready for reading

---

## 6. Invocation Patterns

### 6.1 Process Invocation

The primary invocation method uses `clojure.java.process` (Clojure 1.12+):

```clojure
(ns app.powershell.invoker
  (:require [clojure.java.process :as proc]
            [app.powershell.result :as result]))

(defn invoke-script [{:keys [script-path params env timeout-ms]}]
  (let [args ["powershell.exe" "-NoProfile" "-ExecutionPolicy" "Bypass"
              "-File" script-path (encode-params params)]
        {:keys [out err exit]} (proc/exec {:env env :timeout timeout-ms} args)]
    (if exit
      {:exit-code exit
       :result (result/parse-result (extract-result out))
       :stderr err}
      {:error :timeout})))
```

### 6.2 Synchronous vs Asynchronous

1. **Synchronous:** Default for operations under 30 seconds; uses `proc/exec` which blocks until completion.
2. **Asynchronous:** For long-running operations; uses `proc/start` with streaming I/O. Uses the heartbeat protocol for progress monitoring.

```clojure
(defn invoke-script-async [{:keys [script-path params env]}]
  (let [args ["powershell.exe" "-NoProfile" "-ExecutionPolicy" "Bypass"
              "-File" script-path (encode-params params)]
        process (proc/start {:env env :err :stdout} args)]
    {:process process
     :stdout (proc/stdout process)
     :stdin (proc/stdin process)}))
```

### 6.3 Parameter Encoding

Parameters are passed as a single JSON-encoded argument to avoid shell escaping issues. Scripts parse this using `ConvertFrom-Json`. Complex types (dates, paths, nested structures) are encoded in the JSON and reconstructed by the PowerShell script.

---

## 7. Error Handling & Recovery

### 7.1 Error Classification

Errors are classified into categories that determine handling strategy:

|Category|Example Codes|Recovery Strategy|
|---|---|---|
|Transient|ERR_NETWORK, ERR_TIMEOUT|Automatic retry with backoff|
|Validation|ERR_INVALID_PARAM, ERR_SCHEMA|Report to user; do not retry|
|Resource|ERR_DISK_FULL, ERR_PERMISSION|Alert user; require intervention|
|Fatal|ERR_CORRUPT_DATA, ERR_INTERNAL|Log details; escalate to support|

### 7.2 Partial Failure Handling

When scripts produce some but not all expected artifacts:

1. Result envelope includes both successful artifacts and error details
2. Clojure decides whether to use partial results based on operation criticality
3. Incomplete artifacts are marked with `partial: true` flag in the result

### 7.3 Cleanup Protocol

On failure, scripts must clean up incomplete artifacts:

- **Work directory:** Left in place with error state for debugging; cleaned by retention policy
- **Output directory:** Incomplete files are deleted; complete files remain
- **Database:** Transactions are rolled back; script reports what was committed before failure

---

## 8. Type Mapping

### 8.1 JSON to Clojure Mappings

|PowerShell/JSON|Clojure|Notes|
|---|---|---|
|string|String|Direct mapping|
|number (int)|Long|Cheshire default|
|number (decimal)|Double or BigDecimal|Use `:bigdec true` for money|
|boolean|Boolean|Direct mapping|
|null|nil|Direct mapping|
|array|Vector|Default Cheshire behavior|
|object|Map (keyword keys)|Via `parse-string true` flag|
|ISO-8601 string|java.time.Instant|Custom decoder for dates|
|UUID string|java.util.UUID|Custom decoder|

### 8.2 File Artifact Mapping

File artifacts are represented as Clojure records with associated operations:

```clojure
(defrecord FileArtifact [path size checksum created-at content-type])

(defprotocol ArtifactOps
  (exists? [this])
  (read-content [this])
  (delete! [this])
  (move! [this dest-path]))
```

---

## 9. Script Organization

### 9.1 Directory Structure

```
resources/
└── powershell/
    ├── lib/                    # Shared modules
    │   ├── ResultEnvelope.psm1
    │   ├── Logging.psm1
    │   └── Database.psm1
    ├── data-export/
    │   ├── Export-Customers.ps1
    │   └── Export-Customers.manifest.json
    └── data-import/
        ├── Import-Products.ps1
        └── Import-Products.manifest.json
```

### 9.2 Naming Conventions

- **Scripts:** Verb-Noun.ps1 following PowerShell conventions (`Export-Customers.ps1`)
- **Manifests:** Same name with `.manifest.json` suffix
- **Modules:** PascalCase.psm1 for shared libraries
- **Script IDs:** kebab-case identifiers used in manifests and Clojure code

### 9.3 Script Template

All scripts follow this structure:

```powershell
#Requires -Version 5.1
param(
    [Parameter(Mandatory=$true)]
    [string]$ParamsJson
)

$ErrorActionPreference = 'Stop'
Import-Module "$PSScriptRoot/../lib/ResultEnvelope.psm1"

$params = $ParamsJson | ConvertFrom-Json
$result = New-ResultEnvelope -ScriptId 'script-name'

try {
    # Script logic here
    $result.Status = 'success'
} catch {
    $result | Add-Error -Code 'ERR_INTERNAL' -Message $_.Exception.Message
    $result.Status = 'error'
} finally {
    $result | Write-ResultEnvelope
    exit ($result.Status -eq 'success' ? 0 : 2)
}
```

---

## 10. Database Coordination

### 10.1 Transaction Boundaries

Database operations follow these principles:

1. **Single transaction per script:** Scripts wrap all database operations in a single transaction
2. **Batch identifier:** All rows include a `batch_id` column linking them to the script execution
3. **Clojure verification:** After script completion, Clojure queries the batch to verify row counts

### 10.2 Idempotency Patterns

Scripts implement idempotency using one of these strategies:

- **Natural key:** UPSERT using business keys (preferred when available)
- **Idempotency key:** Caller provides a unique key; script checks before insert
- **Batch replacement:** Delete previous batch records before inserting new ones

### 10.3 Result Querying

Clojure queries database results using the batch ID from the result envelope:

```clojure
(defn fetch-batch-results [db-spec batch-id]
  (jdbc/execute! db-spec
    ["SELECT * FROM import_results WHERE batch_id = ?" batch-id]))

(defn verify-batch-complete [db-spec batch-id expected-count]
  (let [actual (jdbc/execute-one! db-spec
                 ["SELECT COUNT(*) as cnt FROM import_results
                   WHERE batch_id = ?" batch-id])]
    (= expected-count (:cnt actual))))
```

---

## 11. Security Considerations

### 11.1 Execution Policy

Scripts are invoked with `-ExecutionPolicy Bypass` to avoid policy conflicts. Security is enforced through:

1. Scripts bundled within the application package (not downloaded at runtime)
2. Script paths validated against allowed directories before execution
3. All parameters passed as JSON to avoid injection attacks

### 11.2 Input Sanitization

The Clojure layer validates all inputs before script invocation:

```clojure
(defn validate-script-params [manifest params]
  (let [spec (manifest->spec manifest)]
    (when-not (s/valid? spec params)
      (throw (ex-info "Invalid parameters"
                      {:explain (s/explain-str spec params)})))
    params))
```

### 11.3 Credential Management

Database and service credentials follow these rules:

- **Never in parameters:** Credentials are not passed as script parameters
- **Environment variables:** Set by Clojure invoker from secure storage
- **Windows Credential Manager:** Scripts may retrieve credentials using `Get-StoredCredential`
- **Never logged:** Result envelopes must never contain credentials

### 11.4 Privilege Requirements

Scripts declare privilege requirements in their manifests. The Clojure application checks current user privileges before invoking elevated scripts and provides appropriate UI for UAC elevation when required.

---

## 12. Testing Strategy

### 12.1 Integration Test Architecture

Tests verify the complete integration path:

1. **Manifest validation:** Verify all scripts have valid manifests
2. **Invocation tests:** Execute scripts with known inputs, verify outputs
3. **Error scenario tests:** Verify proper handling of failure modes
4. **Database verification:** Query expected rows after database-writing scripts

### 12.2 Mock Artifact Generation

For unit testing Clojure code, mock artifacts are generated:

```clojure
(defn mock-script-result [overrides]
  (merge
    {:status :success
     :run-id (random-uuid)
     :script-id "test-script"
     :artifacts {:files []}}
    overrides))
```

### 12.3 Database Test Isolation

Database tests use transaction rollback or dedicated test schemas. Each test generates a unique batch ID and cleans up its records after verification. The test harness provides utilities for setting up test databases and verifying expected state.

---

## Appendix A: Error Codes Reference

|Error Code|Description|
|---|---|
|ERR_INVALID_PARAM|Parameter validation failed|
|ERR_NETWORK|Network connectivity issue (transient)|
|ERR_TIMEOUT|Operation exceeded time limit|
|ERR_DISK_FULL|Insufficient disk space for output|
|ERR_PERMISSION|Access denied to resource|
|ERR_DB_CONNECTION|Database connection failed|
|ERR_DB_CONSTRAINT|Database constraint violation|
|ERR_FILE_NOT_FOUND|Required input file missing|
|ERR_PARSE|Failed to parse input data|
|ERR_INTERNAL|Unexpected internal error|

---

## Appendix B: Version History

|Version|Date|Changes|
|---|---|---|
|1.0|2025-01|Initial specification release|