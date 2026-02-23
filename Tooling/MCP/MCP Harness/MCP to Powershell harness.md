---

tags:

- babashka
- powershell
- harness
- architecture
- windows created: 2026-02-20

---

# The PowerShell Harness

## Overview

```
clj-nrepl-eval (bb.exe, patched UTF-8 *out*)
    → nREPL on port 8950 (Clojure JVM)
        → harness.http/get-json
            → clj-http → HTTP → Pode server (port 8900, PowerShell 7)
                → SharePoint PnP → Microsoft Graph / SharePoint REST
```

---

## Entry point: `harness.shell`

Three functions for invoking PowerShell:

`ps` — runs a PowerShell command, returns stdout as a string.

`ps!` — same but returns a map `{:exit :out :err}` (for when you need the exit code).

`ps-script` — for multi-line scripts.

> [!important] Encoding: Cp866, not UTF-8 The shell harness invokes `pwsh.exe` and decodes stdout with **Cp866** (OEM codepage for Russian Windows), because PowerShell on this machine outputs CP866 bytes to the pipe regardless of what `[Console]::OutputEncoding` says.

---

## HTTP layer: `harness.http`

`get-json`, `post-json` — call the Pode server over HTTP, parse JSON into Clojure maps.

`raw-get`, `raw-post` — same but return the raw string (useful for byte-level inspection).

> [!note] `base-url` is built from `:server-port` in `config/main-config.edn` at namespace load time — no hardcoded ports.

---

## Server lifecycle: `harness.server`

`start!` — invokes `pode/start.bat` which opens a new PowerShell window and returns immediately (non-blocking). Returns `:started`.

`stop!` — hits `/api/shutdown` on the Pode server (which calls `Close-PodeServer` internally). Returns `{:status "shutting-down", :pid ...}`.

`pid` — hits `/api/pid` to get the running server's PID.

---

## Path mangling workaround

> [!warning] MSYS2 path conversion `clj-nrepl-eval` on Windows (Git Bash) converts any string literal starting with `/` into a Windows path when the containing expression has a namespace-qualified function call. So `/api/health` becomes `C:/Program Files/Git/api/health`.
> 
> All HTTP calls via `clj-nrepl-eval` must build the leading slash at runtime to avoid the MSYS2 `arg_heuristic_with_exclusions()` heuristic:
> 
> ```clojure
> (str (char 47) "api/health")
> ```
> 
> See [[Fix - clj-nrepl-eval Cyrillic output on Windows]] and [[MSYS2 Path Conversion]] for full analysis.