---
tags:
  - babashka
  - clj-nrepl-eval
  - windows
  - preflight
  - harness
  - sharepoint
created: 2026-02-20
updated: 2026-03-12
---

# Harness Preflight Check

> [!abstract] You are starting a session with the `sharepoint-pnp-pode-v02` project. Read this in full before doing anything.

## What this project is

A Clojure test harness (`src/harness/`) for a PowerShell Pode HTTP server (`pode/`) with SharePoint PnP integration. The harness connects to the server via nREPL. The port is defined in `config/main-config.edn` under `:backend-nrepl-port`. All interactive testing goes through `clj-nrepl-eval`.

---

## Mandatory reading before any REPL call

### Problem 1 — `!` in symbol names is silently dropped

`clj-nrepl-eval` passes arguments through a toolchain where `!` inside bash single quotes gets stripped. The call silently dispatches to the wrong function with no error.

**Rule**: Never put `!` inside single quotes. Always splice it between quote segments:

```bash
# WRONG — calls ps, not ps!
clj-nrepl-eval -p $PORT '(harness.shell/ps! "Get-Date")'

# CORRECT
clj-nrepl-eval -p $PORT '(harness.shell/ps'\!' "Get-Date")'
```

Applies to every symbol ending in `!`: `start!`, `stop!`, `ps!`, etc.

> [!tip] How to tell if you called the wrong one
> `ps` returns a string, `ps!` returns a map `{:exit :out :err}`. If you expected a map and got a string (or `""`), the `!` was dropped.

### Problem 2 — MSYS2 converts string literals starting with `/` to Windows paths

When `clj-nrepl-eval` (a native Windows binary) is launched from Git Bash, the MSYS2 runtime (`msys-2.0.dll`) scans every argument for Unix-like paths and converts them. A Clojure string `"/api/health"` becomes `"C:/Program Files/Git/api/health"`, causing a `NumberFormatException` when the port is parsed.

The trigger: a namespace-qualified symbol like `harness.http/get-json` contains a `/`, which puts the MSYS2 heuristic in "path-scanning mode" — then the subsequent `"/api/health"` gets swept up as a path continuation.

**Rule**: Never use string literals starting with `/` in `clj-nrepl-eval` calls. Build the leading slash at runtime:

```bash
# WRONG — /api/system/health gets converted to C:/Program Files/Git/api/system/health
clj-nrepl-eval -p $PORT '(harness.http/get-json "/api/system/health")'

# CORRECT — slash produced at runtime, invisible to MSYS2
clj-nrepl-eval -p $PORT '(harness.http/get-json (str (char 47) "api/system/health"))'
```

> [!note] This does not affect Calva. Only `clj-nrepl-eval` from Git Bash. See [[MSYS2 Path Conversion]] for full analysis.

### Problem 3 — Unicode (full, not just Cyrillic)

The shell harness (`harness.shell/ps`, `ps!`) forces `[Console]::OutputEncoding = UTF-8` before every command. This is required because PowerShell on Russian Windows defaults to OEM CP866 (Cyrillic-only) for pipe output. Without this, any non-ASCII character outside Cyrillic (Japanese, Arabic, emoji, etc.) is corrupted to `?` bytes.

The HTTP harness (`harness.http/get-json`) handles Unicode via `\uXXXX` JSON escaping in Pode routes — safe for any Unicode regardless of encoding.

### Problem 4 — `\"` inside `clj-nrepl-eval` commands breaks via JSON-based tools

When a bash command is issued through a JSON transport (such as an AI coding tool), `\"` in the command string is decoded by JSON to a bare `"` before bash sees it. If that `\"` was inside a single-quoted bash argument intended as a Clojure string escape, the quoting is silently corrupted.

**Symptom**: `EOF while reading string` or `Unsupported escape character` from nREPL.

**Rule**: Never embed `\"` in `clj-nrepl-eval` commands. Use `(char 34)` for a double-quote character and `(char 39)` for a single-quote character when you need them inside a Clojure string at runtime:

```bash
# WRONG — \" becomes bare " after JSON decode, breaks bash quoting
clj-nrepl-eval -p $PORT '(harness.shell/ps'\!' "Write-Output \"hello\"")'

# CORRECT — quote character produced at runtime, invisible to JSON
clj-nrepl-eval -p $PORT '(harness.shell/ps'\!' (str "Write-Output " (char 39) "hello" (char 39)))'
```

> [!note] This only affects commands issued via JSON-based tools (e.g. AI assistants). Commands typed directly in a terminal are unaffected — `\"` works fine there.

---

## Preflight sequence

Run these checks in order. Each must pass before proceeding.

First, read the nREPL port from `config/main-config.edn` (key `:backend-nrepl-port`) and substitute it for `$PORT` below.

### Step 0 — Wrapper active

```bash
type clj-nrepl-eval
# Expected: clj-nrepl-eval is a function
# Bad:      clj-nrepl-eval is /c/Users/...
```

> [!warning] If it shows a binary path, run `source ~/.bashrc` and re-check before proceeding.

### Step 1 — nREPL reachable

```bash
clj-nrepl-eval -p $PORT '(+ 1 2)'
# Expected: => 3
```

### Step 2 — `!` splice works

```bash
clj-nrepl-eval -p $PORT '(require (quote harness.shell) :reload)'
clj-nrepl-eval -p $PORT '(str (var harness.shell/ps'\!'))'
# Expected: => "#'harness.shell/ps!"
```

> [!warning] If you see `"#'harness.shell/ps"` (no `!`), the splice is broken.

### Step 3 — Path mangling is avoided

```bash
clj-nrepl-eval -p $PORT '(require (quote harness.http) :reload)'
clj-nrepl-eval -p $PORT '(harness.http/get-json (str (char 47) "api/system/health"))'
# Expected: => {:status "ok"}
```

> [!warning] If you see `NumberFormatException` or `"8900C:"` — either the wrapper is not active (check Step 0) or you used a literal `/` string.

> [!note] If the Pode server isn't running yet, this will return `Connection refused`. Start it first:
>
> ```bash
> clj-nrepl-eval -p $PORT '(require (quote harness.server) :reload)'
> clj-nrepl-eval -p $PORT '(harness.server/start'\!')'
> # wait ~5s for server to bind, then retry
> ```

### Step 4 — Unicode through shell (full Unicode)

```bash
clj-nrepl-eval -p $PORT '(require (quote harness.shell) :reload)'
clj-nrepl-eval -p $PORT '(harness.shell/ps'\!' (str "Write-Output " (char 39) "\u3053\u3093\u306b\u3061\u306f \ud83d\ude0a" (char 39)))'
# Expected: => {:exit 0, :out "こんにちは 😊\r\n", :err ""}
```

> [!danger] If you see question marks — the UTF-8 prefix was not applied.

### Step 5 — SharePoint Cyrillic via shell

```bash
clj-nrepl-eval -p $PORT '(let [r (harness.shell/ps'\!' "Import-Module PnP.CustomUtilities -ErrorAction Stop; $conn = Connect-SharePointWithConfig -ConfigPath D:/projects/sharepoint-pnp-pode-v02/config/sharepoint-config.json -SecretsPath D:/projects/sharepoint-pnp-pode-v02/config/sharepoint-secrets.json -CertificatePath D:/projects/sharepoint-pnp-pode-v02/config/certs/SillenoProjectControl.pfx -ReturnConnection; Get-PnPList -Connection $conn | Select-Object -ExpandProperty Title") titles (clojure.string/split-lines (:out r)) cyrillic (filter #(some (fn [c] (< 0x0400 (int c) 0x04FF)) %) titles)] (take 3 cyrillic))'
# Expected: => ("Активы сайта" "Библиотека ..." "...")
```

> [!danger] If Cyrillic appears as `???` — the UTF-8 OutputEncoding prefix is missing from `harness.shell`. See [[Fix - clj-nrepl-eval Cyrillic output on Windows]].

---

> [!success] All five pass. Harness is warm and all known edge cases are covered.