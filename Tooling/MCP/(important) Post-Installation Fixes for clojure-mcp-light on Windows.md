

Also! Bash MSYS heuristics MUST be added


             
● ┌────────────────────────────────────────────────────────────┬───────────────┬────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────┐                                                                        │                          Pattern                           │  Converted?   │                         Into what                          │        Potential reason for heuristic triggering         │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤                                                                      
  │ "/" (bare)                                                 │ ✅            │ "C:/Program Files/Git/"                                    │ bare root / → Git install dir                            │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤                                                                      
  │ "/a" (bare)                                                │ ✅            │ "A:/"                                                      │ single-letter first component → drive letter             │                                                                      
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ "/a/b/c" (bare)                                            │ ✅            │ "A:/b/c"                                                   │ single-letter first component → drive X:                 │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ "/api/health" (bare)                                       │ ✅            │ "C:/Program Files/Git/api/health"                          │ bare string starting with / — trivially a path           │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (a/c "/api/health")                                        │ ✅            │ (a/c "C:/Program Files/Git/api/health")                    │ even minimal a/c has /                                   │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (clojure.core/str "/api/health")                           │ ✅            │ (clojure.core/str "C:/Program Files/Git/api/health")       │ clojure.core/str has / — triggers regardless of fn fame  │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (get-json "/api/health")                                   │ ❌            │ —                                                          │ no / in fn name — looks like a plain word                │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (harness.http/get-json "/api/health")                      │ ✅            │ (harness.http/get-json "C:/Program Files/Git/api/health")  │ ns-qualified fn with /, path as first arg — full trigger │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (harness.http/get-json "/api/info")                        │ ✅            │ (harness.http/get-json "C:/Program Files/Git/api/info")    │ same                                                     │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (harness.http/get-json "/api/pid")                         │ ✅            │ (harness.http/get-json "C:/Program Files/Git/api/pid")     │ same                                                     │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (harness.http/post-json "/api/echo" {:msg "hello"})        │ ❌            │ —                                                          │ map as second arg — { breaks the heuristic               │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (harness.http/raw-get "/api/health")                       │ ✅            │ (harness.http/raw-get "C:/Program Files/Git/api/health")   │ ns-qualified fn, path as first arg — full trigger        │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (identity "/api/health")                                   │ ❌            │ —                                                          │ no / in fn name — plain word                             │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (shadow.cljs.devtools.api/cljs-eval :app "/api/health" {}) │ ❌            │ —                                                          │ :app before path — path must be immediate first arg      │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "/")                                                  │ ✅            │ (x/y "C:/Program Files/Git/")                              │ bare root / → Git install dir                            │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "/api/health")                                        │ ✅            │ (x/y "C:/Program Files/Git/api/health")                    │ ns-qualified fn + path as first arg                      │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "/api/health" "/other")                               │ ✅ first only │ (x/y "C:/Program Files/Git/api/health" "/other")           │ only first arg matches the pattern                       │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "/api/health" {:body "x"})                            │ ❌            │ —                                                          │ { in second arg breaks the heuristic                     │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "/c/Users/v.palichev")                                │ ✅            │ (x/y "C:/Users/v.palichev")                                │ /c/ matches drive-letter convention — works as intended  │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "/tmp/file.txt")                                      │ ✅            │ (x/y "C:/Users/V8D63~1.PAL/AppData/Local/Temp/1/file.txt") │ /tmp → %TEMP%, username 8.3-truncated                    │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "/usr/local/bin")                                     │ ✅            │ (x/y "C:/Program Files/Git/usr/local/bin")                 │ /usr is a known Unix root dir — strong trigger           │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y :key "/api/health")                                   │ ❌            │ —                                                          │ :key before path — path not immediate first arg          │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y {:a 1} "/api/health")                                 │ ❌            │ —                                                          │ map before path — { breaks the pattern                   │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "api/health")                                         │ ❌            │ —                                                          │ no leading / — not an absolute path                      │
  ├────────────────────────────────────────────────────────────┼───────────────┼────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────┤
  │ (x/y "http://localhost:8900/api/health")                   │ ❌            │ —                                                          │ URL scheme http:// explicitly whitelisted                │
  └────────────────────────────────────────────────────────────┴───────────────┴────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────┘


After installing via bbin, Windows users must manually fix the generated scripts in `C:\Users\<username>\.local\bin\`.


### Heuristics rules:
  ---                                                                                                                                                                                                                                                                         
  Layer 1 — Trigger conditions (ALL must be true to convert)
                                                                                                                                                                                                                                                                              
  1. Function symbol contains / — x/y, clojure.core/str, harness.http/get-json all trigger; identity, get-json, x don't                                                                                                                                                       
  2. Path string starts with / — relative paths like "api/health" are ignored
  3. Path is the immediate first argument — anything before it (:key, {:a 1}) blocks conversion
  4. No { anywhere in the whole argument string — presence of a map literal anywhere kills conversion for the entire form (explains why post-json was always safe)

  ---
  Layer 2 — Escape hatches (prevent conversion even when triggered)

  5. URL scheme prefix — http://, https:// etc. are whitelisted, string left untouched
  6. Only first path is converted — subsequent /path strings in the same form are ignored

  ---
  Layer 3 — Conversion rules (what it converts TO)

  7. Single-letter first component → drive letter: /a/b/c → A:/b/c
  8. Multi-letter first component → MSYS root: /api/health → C:/Program Files/Git/api/health
  9. Special mappings: /tmp → %TEMP%, /c/rest → C:/rest (intentional Git Bash drive convention)

  ---
  So MINGW64's heuristic is essentially: "if the text looks like a shell command (ns/cmd) followed by a file path argument, convert the path" — and it uses { as a signal that the context is a data structure, not a shell invocation.













Reproducing the MSYS2 path conversion fix for clj-nrepl-eval

  Problem

  On Windows with Git Bash (MINGW64), clj-nrepl-eval corrupts Clojure expressions containing Unix-style path strings when passed as command-line arguments. For example:

  clj-nrepl-eval -p 8950 '(harness.http/get-json "/api/health")'
  # Fails: NumberFormatException "8900C:"
  # Because Git Bash converts "/api/health" → "C:/Program Files/Git/api/health"
  # making clj-http build the URL: http://localhost:8900C:/Program Files/Git/api/health

  Root cause: MINGW64's path conversion heuristic triggers when a namespace-qualified symbol (containing /) is immediately followed by a string starting with /. Git Bash mistakes it for a shell command taking a file path argument.

  Affected tools

  Any bbin-installed Babashka tool from clojure-mcp-light:
  - clj-nrepl-eval
  - clj-paren-repair

  Prerequisites

  - Windows with Git Bash (Git for Windows, MSYSTEM=MINGW64)
  - clj-nrepl-eval and/or clj-paren-repair installed via bbin

  Fix

  Add the following to ~/.bashrc (create the file if it doesn't exist — on Windows this is C:\Users\<username>\.bashrc):

  # Wrap bb-based tools to disable MSYS2 argument path conversion.
  # Prevents Git Bash from mangling Unix-style path strings (e.g. "/api/health")
  # passed as arguments. Must wrap each tool individually — bb() wrapper doesn't
  # help for shebang scripts since bash execs them directly without invoking bb().
  clj-nrepl-eval()     { MSYS2_ARG_CONV_EXCL="(" command clj-nrepl-eval "$@"; }
  clj-paren-repair()   { MSYS2_ARG_CONV_EXCL="(" command clj-paren-repair "$@"; }

  Also create ~/.bash_profile if it doesn't exist, so .bashrc is sourced in login shells (which is what Git Bash opens by default):

  # Source .bashrc for login shells
  [[ -f ~/.bashrc ]] && source ~/.bashrc

  How it works

  MSYS2_ARG_CONV_EXCL="(" tells MINGW64 to skip path conversion for any argument starting with (. Since every Clojure expression starts with (, all code arguments are protected. Script paths and flags like -p 8950 are not affected and continue to be converted normally.      

  Verify the fix

  Open a new Git Bash terminal (or run source ~/.bashrc), then:

  # Should print the expression unchanged — no C:/Program Files/Git/... corruption
  bb -e '(print (first *command-line-args*))' -- '(harness.http/get-json "/api/health")'
  # Expected: (harness.http/get-json "/api/health")

  # End-to-end test (requires a running nREPL on port 8950)
  clj-nrepl-eval -p 8950 '(harness.http/get-json "/api/health")'
  # Expected: => {:status "ok"}

  Notes

  - If you install additional bbin tools from clojure-mcp-light in the future, add them to .bashrc with the same pattern.
  - This fix is not needed in PowerShell, CMD, or Calva (VS Code REPL) — only Git Bash is affected.
  - MSYS2_ARG_CONV_EXCL is a MSYS2/Git-for-Windows variable. Setting it per-invocation (inside the function) means it only applies to that one call, not globally.









---

### Fix 1: Quote the `-m` Flag

Locate the `script-main-opts` line in each script.

**Before:**

```clojure
(def script-main-opts [-m clojure-mcp-light.nrepl-eval])
```

**After:**

```clojure
(def script-main-opts ["-m" "clojure-mcp-light.nrepl-eval"])
```

---

### Fix 2: Add Source Path to Classpath

Locate the `base-command` definition in each script.

**Before:**

```clojure
(def base-command
  (vec (concat ["bb" "--deps-root" script-root "--config" (str tmp-edn)]
               script-main-opts
               ["--"])))
```

**After:**

```clojure
(def base-command
  (vec (concat ["bb" 
                "--deps-root" script-root 
                "--config" (str tmp-edn)
                "-cp" (str script-root "\\src")]
               script-main-opts
               ["--"])))
```

---

### Scripts Requiring Fixes

|Script|Namespace|
|---|---|
|`clj-nrepl-eval`|`clojure-mcp-light.nrepl-eval`|
|`clj-paren-repair`|`clojure-mcp-light.paren-repair`|
|`clj-paren-repair-claude-hook`|`clojure-mcp-light.hook`|

---

### Verify

```powershell
clj-nrepl-eval --help
clj-paren-repair --help
clj-paren-repair-claude-hook --help
```