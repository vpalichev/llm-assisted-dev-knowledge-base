# Table of Contents

- [[#Run]]
- [[#Verify]]
- [[#Install Babashka]]
- [[#Install bbin]]
- [[#Install clojure-mcp-light Tools]]
- [[#Configure Hooks]]
- [[#Configure REPL Instructions]]
- [[#Project deps.edn]]
- [[#Run (Single REPL)]]
- [[#Verify Hooks Are Active]]
- [[#Dual Clojure/ClojureScript Project]]
- [[#Project nREPL Configuration]]
- [[#Project CLAUDE.md]]
- [[#Run (Dual REPL)]]
- [[#Frontend REPL Troubleshooting]]

---

# clojure-mcp-light on Windows (Fixed)

## Run

[[#Table of Contents|Back to TOC]]

```powershell
# Terminal 1: Backend REPL
clojure -M:nrepl

# Terminal 2: Frontend REPL + watch
npx shadow-cljs watch app

# Terminal 3: Claude Code
claude
```

## Verify

[[#Table of Contents|Back to TOC]]

```
Evaluate (+ 1 2) using the backend REPL
```

```
Evaluate (js/console.log "test") using the frontend REPL
```

## Install Babashka

[[#Table of Contents|Back to TOC]]

```powershell
scoop install babashka
```

Verify:

```powershell
bb --version
```

## Install bbin

[[#Table of Contents|Back to TOC]]

```powershell
iex (bb -e '(slurp "https://raw.githubusercontent.com/babashka/bbin/main/install.clj")')
```

Add `%USERPROFILE%\.local\bin` to PATH.

Verify:

```powershell
bbin --version
```

## Install clojure-mcp-light Tools

[[#Table of Contents|Back to TOC]]

**Hook tool (works directly):**

```powershell
bbin install https://github.com/bhauman/clojure-mcp-light.git --tag v0.2.1
```

**REPL eval tool (two-step due to Windows quoting):**

Step 1 — install without main-opts:

```powershell
bbin install https://github.com/bhauman/clojure-mcp-light.git --tag v0.2.1 --as clj-nrepl-eval
```

Step 2 — fix the generated script:

```powershell
$file = "$env:USERPROFILE\.local\bin\clj-nrepl-eval"
(Get-Content $file) -replace '\(def script-main-opts nil\)', '(def script-main-opts ["-m" "clojure-mcp-light.nrepl-eval"])' | Set-Content $file
```

**Verify tools installed:**

```powershell
clj-paren-repair-claude-hook --help
clj-nrepl-eval --help
```

Both should display help text without errors.

## Configure Hooks

[[#Table of Contents|Back to TOC]]

Create `%USERPROFILE%\.claude\settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "clj-paren-repair-claude-hook --cljfmt"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "clj-paren-repair-claude-hook --cljfmt"
          }
        ]
      }
    ]
  }
}
```

## Configure REPL Instructions

[[#Table of Contents|Back to TOC]]

Create `%USERPROFILE%\.claude\CLAUDE.md`:

```markdown
# Clojure REPL

Discover ports: clj-nrepl-eval --discover-ports
Evaluate: clj-nrepl-eval -p <port> "<code>"
Always use :reload when requiring namespaces.
```

## Project deps.edn

[[#Table of Contents|Back to TOC]]

```clojure
{:aliases
 {:nrepl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.1"}}
          :main-opts ["-m" "nrepl.cmdline" "--port" "7888"]}}}
```

## Run (Single REPL)

[[#Table of Contents|Back to TOC]]

```powershell
# Terminal 1
clojure -M:nrepl

# Terminal 2
claude
```

## Verify Hooks Are Active

[[#Table of Contents|Back to TOC]]

**Method 1: Check tool output**

Ask Claude to create a Clojure file:

```
Create src/test.clj with a simple hello function
```

After Claude writes the file, press `Ctrl+R` or click the edit in the output. Expand the tool details. Look for:

```
PreToolUse:Write hook succeeded:
```

**Method 2: Intentional error test**

Prompt Claude:

```
Create src/broken.clj with this exact content, do not fix it:
(defn foo [x]
  (+ x 1)
```

Open `src/broken.clj`. If hooks work, file contains:

```clojure
(defn foo [x]
  (+ x 1))
```

Missing paren was auto-repaired. If hooks failed, file has the original broken code.

**Method 3: Enable logging**

Change hook command in `settings.json`:

```json
"command": "clj-paren-repair-claude-hook --cljfmt --log-level debug"
```

Restart Claude Code. After any Clojure file edit, check `.clojure-mcp-light-hooks.log` in your project directory.

---

## Dual Clojure/ClojureScript Project

[[#Table of Contents|Back to TOC]]

For projects with:
- Clojure backend in `src/clj` (deps.edn)
- ClojureScript frontend in `src/cljs` (shadow-cljs.edn)

**Hooks:** No changes needed. Hooks intercept all `.clj`, `.cljs`, `.cljc` edits automatically.

## Project nREPL Configuration

[[#Table of Contents|Back to TOC]]

**deps.edn:**

```clojure
{:paths ["src/clj"]
 :aliases
 {:nrepl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.1"}}
          :main-opts ["-m" "nrepl.cmdline" "--port" "7888"]}}}
```

**shadow-cljs.edn:**

```clojure
{:source-paths ["src/cljs"]
 :nrepl {:port 9000}
 :builds {:app {...}}}
```

## Project CLAUDE.md

[[#Table of Contents|Back to TOC]]

Create `CLAUDE.md` in project root:

```markdown
# Project Structure

- Backend (Clojure): src/clj
- Frontend (ClojureScript): src/cljs

# REPL

Backend: clj-nrepl-eval -p 7888 "<code>"
Frontend: clj-nrepl-eval -p 9000 "<code>"

For ClojureScript, first connect to build:
clj-nrepl-eval -p 9000 "(shadow/repl :app)"

Always use :reload when requiring namespaces.
```

## Run (Dual REPL)

[[#Table of Contents|Back to TOC]]

```powershell
# Terminal 1: Backend REPL
clojure -M:nrepl

# Terminal 2: Frontend REPL + watch
npx shadow-cljs watch app

# Terminal 3: Claude Code
claude
```

## Frontend REPL Troubleshooting

[[#Table of Contents|Back to TOC]]

If frontend REPL evaluation fails with session state issues:

**Note:** `clj-nrepl-eval` maintains session state between calls.

Try explicitly:

```
First run: clj-nrepl-eval -p 9000 "(shadow/repl :app)"
Then run: clj-nrepl-eval -p 9000 "(js/console.log \"test\")"
```

Or tell Claude:

```
Run these two commands in sequence:
1. clj-nrepl-eval -p 9000 "(shadow/repl :app)"
2. clj-nrepl-eval -p 9000 "(js/console.log \"test\")"
```

If it still fails, test manually in PowerShell:

```powershell
clj-nrepl-eval -p 9000 "(shadow/repl :app)"
clj-nrepl-eval -p 9000 "(+ 1 2)"
```

Second call should return `3` from ClojureScript runtime, confirming session persists.