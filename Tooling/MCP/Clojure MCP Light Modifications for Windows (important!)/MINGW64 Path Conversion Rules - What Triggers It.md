

## Research Summary

This document investigates the MSYS2/MINGW64 automatic path conversion behavior that corrupts Clojure expression arguments when passed to `bb.exe` (Babashka) from Git Bash, specifically affecting `clj-nrepl-eval` from `clojure-mcp-light`.

---

## 1. MINGW64 Path Conversion Rules — What Triggers It

### Where It Happens

The path conversion is **not** performed by bash itself. It happens inside **`msys-2.0.dll`** (the MSYS2 fork of Cygwin's `cygwin1.dll`), specifically in the `spawn.cc` module during process creation. When an MSYS2/Cygwin process (like `bash.exe`) launches a **non-Cygwin** (i.e., native Windows) executable, the runtime intercepts every argument and applies heuristic path conversion via a function called `arg_heuristic_with_exclusions()`. The relevant source code lives at `winsup/cygwin/msys2_path_conv.cc` in the `msys2-runtime` repository (mirrored in `git-for-windows/msys2-runtime`).

The key decision point in `spawn.cc` is:

```c
if (real_path.iscygexec()) {
    // Pass arguments as-is to other MSYS2/Cygwin programs
} else {
    // Convert each argument via arg_heuristic_with_exclusions()
    // This is the path that fires for bb.exe, java.exe, python.exe, etc.
}
```

So: `bash.exe` (an MSYS2 binary linked against `msys-2.0.dll`) calls `bb.exe` (a native Windows GraalVM binary, **not** linked against `msys-2.0.dll`). The runtime detects `bb.exe` is a native program, and applies argument conversion to every argv element.

### The Heuristic — Why `ns/fn "/path"` Triggers but `identity "/path"` Does Not

The conversion operates on each **complete argument string** as a single unit. When you invoke:

```bash
bb -e '(harness.http/get-json "/api/health")'
```

Bash strips the outer single quotes and passes the literal string `(harness.http/get-json "/api/health")` as a single argv element to the spawn logic. The `msys2_path_conv.cc` heuristic then scans this string looking for **subsequences that resemble Unix paths**. The algorithm in `find_path_start_and_type()` and `sub_convert()` works roughly as follows:

1. It scans the argument character by character looking for a `/` that could start a path.
2. When it encounters something like `text/more-text`, it may treat the `/` as the beginning of a relative or absolute path component.
3. Critically, the heuristic considers context: the characters before and after the slash, whether the slash is preceded by alphanumeric characters (suggesting `word/word` which looks like a relative path), and whether subsequent content resembles path segments.

In your case, `harness.http/get-json "/api/health"` contains two path-triggering elements:

- `harness.http/get-json` — the `/` here is between two identifier-like tokens, so the heuristic may see `get-json` as a path-like continuation but ultimately this resolves to something the heuristic doesn't convert in isolation.
- `"/api/health"` — this is a classic absolute Unix path starting with `/`. The `/api/health` substring clearly matches the pattern of an absolute POSIX path.

The reason `(identity "/api/health")` doesn't trigger conversion appears to be about how the heuristic tokenizes. The `sub_convert()` function processes substrings between delimiters. When it encounters `identity "/api/health"`, the word `identity` contains no `/`, so the heuristic processes the quoted path segment differently. But when `harness.http/get-json` precedes it, the `/` in the namespace-qualified symbol causes the heuristic to start treating the surrounding content as path-like, and the subsequent `/api/health` gets swept up into the conversion as what appears to be a continuation of path-like content. The heuristic's behavior around `=`, `:`, and `/` delimiters is complex and contextual — it tracks state including whether it's inside quotes, what preceded the current position, and whether it has already seen a path-type indicator.

This is fundamentally a **false-positive in a heuristic system**. The MSYS2 project openly acknowledges this: the documentation says the conversion "in corner cases converts arguments that look like Unix paths while they are not, or detects lists of Unix paths where there are none."

### The Conversion Result

`/api/health` gets converted to the MSYS2 root + the path, which for Git for Windows is typically `C:/Program Files/Git/api/health`. This is the standard behavior for any absolute POSIX path that gets detected.

---

## 2. Environment Variables to Suppress Conversion

Three environment variables are relevant. They differ in scope and origin.

### `MSYS_NO_PATHCONV`

- **Origin**: Git for Windows extension (not in upstream MSYS2 until later). Introduced via `git-for-windows/msys2-runtime` PR #11.
- **Effect**: When **defined** (the value doesn't matter — even `MSYS_NO_PATHCONV=0` or `MSYS_NO_PATHCONV=""` suppresses conversion), it disables all argument path conversion for the current process and any native child processes spawned from the current MSYS2 shell.
- **Scope**: All arguments to all native executables launched while the variable is set.
- **Pitfall**: It's a blunt instrument. Setting it globally in `.bashrc` can break tools that _depend_ on path conversion (like `gcloud`, which needs MSYS2 to convert its Python script path). One user reported that `gcloud` stopped working entirely with `MSYS_NO_PATHCONV=1` globally set.
- **Usage**: Best used as a per-command prefix: `MSYS_NO_PATHCONV=1 bb -e '...'`

### `MSYS2_ARG_CONV_EXCL`

- **Origin**: Upstream MSYS2 feature.
- **Effect**: A `;`-delimited list of argument **prefixes**. Each argument is checked — if its beginning matches any listed prefix, conversion is skipped for that argument. Setting `MSYS2_ARG_CONV_EXCL="*"` disables all conversion (equivalent to `MSYS_NO_PATHCONV`).
- **Scope**: Read by `msys-2.0.dll` at process creation time. Applies to native child processes spawned by MSYS2 shells.
- **Pitfall for this case**: The prefix matching is against the **whole argument string**, not individual tokens within it. So your argument `(harness.http/get-json "/api/health")` would need a prefix match like `(` — which is valid but would exclude ALL arguments starting with `(`.

### `MSYS2_ENV_CONV_EXCL`

- **Effect**: Controls conversion of **environment variables**, not arguments. Irrelevant to the command-line argument problem, but useful if you also need to pass POSIX paths in env vars to native processes without conversion.

### Can These Be Set from Inside a Babashka Script?

**No, not effectively for the current invocation.** The conversion happens at the boundary between `bash.exe` (the MSYS2 process) and `bb.exe` (the native process). By the time `bb.exe` is running and your Babashka script has control, the arguments have **already been converted**. `*command-line-args*` in bb already contains the corrupted strings.

If your bb script then uses `ProcessBuilder` to launch a _child_ native process, setting `MSYS2_ARG_CONV_EXCL` in the child's environment **will not help** either, because `ProcessBuilder` in Java creates processes via the Windows API directly (`CreateProcessW`), not through `msys-2.0.dll`. The conversion only applies when an MSYS2/Cygwin binary spawns a native process — Java's `ProcessBuilder` is a native Windows API call and doesn't go through the MSYS2 runtime at all.

---

## 3. Is Conversion a Bash-Layer or DLL-Layer Concern?

**It is a DLL-layer concern**, specifically inside `msys-2.0.dll` / `cygwin1.dll`.

The conversion is applied inside the Cygwin/MSYS2 runtime's `spawn()` implementation, which wraps the Windows `CreateProcess` call. When `bash.exe` (linked against `msys-2.0.dll`) decides to execute `bb.exe`, the MSYS2 runtime:

1. Checks if the target is a Cygwin/MSYS2 executable (by looking for the Cygwin DLL dependency).
2. If the target is **not** a Cygwin executable, iterates over all arguments and calls `arg_heuristic_with_exclusions()` on each one.
3. Constructs a Windows command line from the (now-converted) arguments.
4. Calls `CreateProcess` with the modified command line.

This means:

- **Bash doesn't know about the conversion** — it just passes argv arrays to the runtime.
- **The native .exe doesn't know it happened** — it receives the already-converted arguments via `GetCommandLineW()`.
- **There is no way for the child process to undo the conversion** — the original argument values are lost.
- **`ProcessBuilder` in Java/bb bypasses this** — if bb itself spawns another native process, `msys-2.0.dll` is not in the call path (bb is native, not an MSYS2 binary).

---

## 4. Known Workarounds for Babashka/bb.exe on MINGW64

### What Other Tools Do

There is no established community pattern specifically for Babashka CLI tools that handle path-like strings. The Babashka ecosystem (including `neil`, `jet`, etc.) largely operates on macOS and Linux where this problem doesn't exist. Windows usage with Git Bash is a secondary use case.

The broader Windows-on-MSYS2 community uses several patterns:

- **Docker on Git Bash**: The most well-documented case. The community uses `MSYS_NO_PATHCONV=1` as a per-command prefix, or wrapper shell functions that export the variable before calling `docker.exe`.
- **Python/Node/PHP on Git Bash**: Users set `MSYS_NO_PATHCONV=1` or `MSYS2_ARG_CONV_EXCL="*"` per-command.
- **Symfony/PHP (CVE-2026-24739)**: A recent CVE was issued for Symfony's Process component because MSYS2 conversion could corrupt arguments containing `=`, leading to destructive file operations. The recommendation was to avoid MSYS2 shells for native Windows process launching, or to set `MSYS2_ARG_CONV_EXCL`.

### bbin and Babashka on Windows

- **bbin issue #57** documents Windows problems with bbin-installed scripts, though that specific issue was about path resolution in `deps/add-deps` (Java's `Path.relativize()` failing across drive roots), not about MSYS2 argument conversion.
- **No documented bbin handling** for MSYS2 path conversion exists. bbin generates shell wrapper scripts that simply call `bb` with the appropriate classpath and main options. These wrappers do not set any `MSYS_NO_PATHCONV` or `MSYS2_ARG_CONV_EXCL` variables.
- **clojure-mcp-light's CHANGELOG** mentions "Stdin support for clj-nrepl-eval — Evaluate code directly from stdin for easier piping and scripting workflows" — this suggests the stdin approach is already recognized as valuable, though it's framed as a convenience feature rather than a Windows workaround.

---

## 5. bbin and Windows

bbin does **not** have any documented handling for MSYS2 path conversion. The bbin-generated wrapper scripts are plain bash scripts that invoke `bb` with the necessary classpath arguments. No environment variable guards are set.

The one relevant open issue is **bbin #57** ("Issues running installed script on Windows 10"), which documents a different Windows path problem: Java's `Path.relativize()` failing when the bbin libs directory and the current working directory are on different drive roots. That issue is about Java path APIs, not MSYS2 argument mangling.

There are no open issues or PRs in the bbin repository about MSYS2/MINGW64 path conversion for arguments.

---

## 6. clojure-mcp-light Specifically

Searching the `bhauman/clojure-mcp-light` GitHub repository, there are **no open issues about Windows/MSYS2/Git Bash path conversion** for `clj-nrepl-eval`. The existing issues are about other topics (e.g., #6 about a `timbre/set-config!` symbol resolution error on install).

The project's CHANGELOG does mention stdin support for `clj-nrepl-eval` as a feature, and the CLAUDE.md file shows examples using pipe-based evaluation:

```bash
echo '(+ 1 2)' | bb -m clojure-mcp-light.hook
```

This suggests the project author (Bruce Hauman) may be aware of situations where piping is more reliable than command-line argument passing, even if the MSYS2 path conversion issue isn't explicitly documented.

---

## 7. Best Fix Approach — Evaluation of Options

### Option A: Bash Wrapper Function in `.bashrc` (RECOMMENDED — Simplest)

Create a wrapper function that pipes code via stdin instead of passing it as an argument:

```bash
# In ~/.bashrc
clj-nrepl-eval() {
    if [[ "$1" == "-p" && -n "$2" ]]; then
        local port="$2"
        shift 2
        # Remaining args are the code expression
        echo "$*" | command clj-nrepl-eval -p "$port"
    else
        # Fall through to the real command for other usage patterns
        command clj-nrepl-eval "$@"
    fi
}
```

Or simpler — just always use the stdin approach:

```bash
# In ~/.bashrc or as a wrapper script
nrepl-eval() {
    local port="$1"
    shift
    echo "$*" | clj-nrepl-eval -p "$port"
}
```

**Pros**: Zero dependencies, works immediately, no changes to clojure-mcp-light needed, completely avoids the MSYS2 argument conversion boundary.

**Cons**: Slight change in usage pattern (function call vs direct command with args).

### Option B: `MSYS_NO_PATHCONV` Prefix Wrapper (RECOMMENDED — Targeted)

Create a wrapper that sets the conversion suppression variable:

```bash
# In ~/.bashrc
clj-nrepl-eval() {
    MSYS_NO_PATHCONV=1 command clj-nrepl-eval "$@"
}
```

Or modify the bbin-generated wrapper script directly to include the variable.

**Pros**: Preserves the exact same CLI interface (arguments work as-is), targeted fix.

**Cons**: `MSYS_NO_PATHCONV=1` disables ALL path conversion for all arguments to `bb.exe` in that invocation. If any argument legitimately needs path conversion (unlikely for Clojure code, but possible for file path arguments), it would be affected. Also, the bbin wrapper gets regenerated on reinstall, so you'd need to patch it each time (or use the `.bashrc` approach instead).

### Option C: `.bat` or `.ps1` Shim (VIABLE — Robust)

Create a native Windows batch or PowerShell script that bypasses MSYS2 entirely:

```batch
@echo off
REM clj-nrepl-eval.bat
"C:\path\to\bb.exe" -cp "%BBIN_CLASSPATH%" -m clojure-mcp-light.nrepl-eval %*
```

Or in PowerShell:

```powershell
# clj-nrepl-eval.ps1
& "C:\path\to\bb.exe" -cp $env:BBIN_CLASSPATH -m clojure-mcp-light.nrepl-eval @args
```

**Pros**: Completely eliminates MSYS2 from the call path. When Git Bash calls a `.bat` file, it uses `cmd.exe` to execute it, which goes directly to `CreateProcess` without MSYS2 argument mangling. PowerShell similarly bypasses the MSYS2 runtime.

**Cons**: Requires maintaining a separate wrapper file, needs the classpath and bb path hardcoded or resolved. More setup complexity. Also, calling `.bat` from Git Bash has its own quoting quirks (different quote escaping rules between bash and cmd.exe).

### Option D: `MSYS2_ARG_CONV_EXCL` in the bbin Wrapper

You could modify the bbin-generated wrapper to set `MSYS2_ARG_CONV_EXCL` before calling `bb`:

```bash
#!/usr/bin/env bash
export MSYS2_ARG_CONV_EXCL="*"
exec bb -cp "..." -m clojure-mcp-light.nrepl-eval "$@"
```

**Pros**: Same as Option B — preserves CLI interface.

**Cons**: Same regeneration-on-reinstall issue. Also, `MSYS2_ARG_CONV_EXCL="*"` is checked by `msys-2.0.dll` when `bash.exe` spawns `bb.exe`, so it does work when set in the wrapper (bash reads its own environment, and the MSYS2 runtime reads it from the spawning process's environment at `CreateProcess` time). This is actually a good option if you can keep the wrapper stable.

**Important subtlety**: Since the bbin wrapper is itself a bash script (an MSYS2 program), and it calls `bb.exe` (a native program), setting `MSYS2_ARG_CONV_EXCL="*"` in the wrapper **before** the `exec bb` line will be in effect when `msys-2.0.dll` processes the `bb.exe` spawn. This works correctly.

### Option E: Reading Raw Windows Command Line via Java APIs

In theory, on Windows you can call `GetCommandLineW()` via JNI/JNA to get the raw command line string before Java's argv parsing. In Babashka, you'd need to use `java.lang.ProcessHandle` or native interop. However:

- `ProcessHandle.current().info().commandLine()` returns an `Optional<String>` and may not be available on all JVMs.
- Even if available, the command line at this point has **already been mangled** by MSYS2 — the raw Windows command line that `bb.exe` receives is the post-conversion version.

**This approach does not work** because the damage happens before the process starts.

### Recommended Strategy

For **immediate relief**: Use **Option A** (stdin piping) or **Option B** (`MSYS_NO_PATHCONV=1` wrapper function in `.bashrc`). Both are single-line changes that work today.

For **permanent robustness**: Use **Option D** — add `MSYS2_ARG_CONV_EXCL="*"` to the bbin wrapper, and consider filing an issue on `bbin` requesting Windows-aware wrapper generation that includes this variable. Alternatively, file an issue on `clojure-mcp-light` suggesting that the tool detect the MSYS2 environment and print a warning or instructions.

The **stdin approach** (Option A) is arguably the most robust because it completely sidesteps the problem at an architectural level — stdin content never passes through argument conversion. If `clj-nrepl-eval` already supports stdin (per the CHANGELOG), this may be the path of least resistance.

---

## Summary Table

|Approach|Effort|Robustness|Preserves CLI|Survives Reinstall|
|---|---|---|---|---|
|A. Stdin pipe wrapper|Low|Excellent|No (slightly different)|Yes|
|B. `MSYS_NO_PATHCONV=1` in `.bashrc`|Low|Good|Yes|Yes|
|C. `.bat`/`.ps1` shim|Medium|Excellent|Yes|Yes|
|D. `MSYS2_ARG_CONV_EXCL` in bbin wrapper|Low|Good|Yes|No|
|E. Raw command line via Java|High|Doesn't work|N/A|N/A|