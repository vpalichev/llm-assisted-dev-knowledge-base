## Fixing MSYS2 Path Conversion for clj-nrepl-eval

  

### Problem

  

On Windows with Git Bash (MINGW64), `clj-nrepl-eval` corrupts Clojure expressions containing Unix-style path strings when passed as command-line arguments. For example:

  

```bash

clj-nrepl-eval -p 8950 '(harness.http/get-json "/api/health")'

# Fails: NumberFormatException "8900C:"

# Because Git Bash converts "/api/health" → "C:/Program Files/Git/api/health"

# making clj-http build the URL: http://localhost:8900C:/Program Files/Git/api/health

```

  

**Root cause:** MINGW64's path conversion heuristic triggers when a namespace-qualified symbol (containing `/`) is immediately followed by a string starting with `/`. Git Bash mistakes it for a shell command taking a file path argument.

  

### Affected Tools

  

Any bbin-installed Babashka tool from clojure-mcp-light:

- `clj-nrepl-eval`

- `clj-paren-repair`

  

### Prerequisites

  

- Windows with Git Bash (Git for Windows, `MSYSTEM=MINGW64`)

- `clj-nrepl-eval` and/or `clj-paren-repair` installed via bbin

  

### Fix

  

Add the following to `~/.bashrc` (create the file if it doesn't exist — on Windows this is `C:\Users\<username>\.bashrc`):

  

```bash

# Wrap bb-based tools to disable MSYS2 argument path conversion.

# Prevents Git Bash from mangling Unix-style path strings (e.g. "/api/health")

# passed as arguments. Must wrap each tool individually — bb() wrapper doesn't

# help for shebang scripts since bash execs them directly without invoking bb().

clj-nrepl-eval()     { MSYS2_ARG_CONV_EXCL="(" command clj-nrepl-eval "$@"; }

clj-paren-repair()   { MSYS2_ARG_CONV_EXCL="(" command clj-paren-repair "$@"; }

```

  

Also create `~/.bash_profile` if it doesn't exist, so `.bashrc` is sourced in login shells (which is what Git Bash opens by default):

  

```bash

# Source .bashrc for login shells

[[ -f ~/.bashrc ]] && source ~/.bashrc

```

  

### How It Works

  

`MSYS2_ARG_CONV_EXCL="("` tells MINGW64 to skip path conversion for any argument starting with `(`. Since every Clojure expression starts with `(`, all code arguments are protected. Script paths and flags like `-p 8950` are not affected and continue to be converted normally.

  

### Verify the Fix

  

Open a new Git Bash terminal (or run `source ~/.bashrc`), then:

  

```bash

# Should print the expression unchanged — no C:/Program Files/Git/... corruption

bb -e '(print (first *command-line-args*))' -- '(harness.http/get-json "/api/health")'

# Expected: (harness.http/get-json "/api/health")

  

# End-to-end test (requires a running nREPL on port 8950)

clj-nrepl-eval -p 8950 '(harness.http/get-json "/api/health")'

# Expected: => {:status "ok"}

```

  

### Notes

  

- If you install additional bbin tools from clojure-mcp-light in the future, add them to `.bashrc` with the same pattern.

- This fix is not needed in PowerShell, CMD, or Calva (VS Code REPL) — only Git Bash is affected.

- `MSYS2_ARG_CONV_EXCL` is a MSYS2/Git-for-Windows variable. Setting it per-invocation (inside the function) means it only applies to that one call, not globally.