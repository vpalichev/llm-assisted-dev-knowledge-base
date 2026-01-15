# clojure-mcp-light on Windows (Fixed)

## Run

powershell

````powershell
# Terminal 1: Backend REPL
clojure -M:nrepl

# Terminal 2: Frontend REPL + watch
npx shadow-cljs watch app

# Terminal 3: Claude Code
claude
```

## Verify
```
Evaluate (+ 1 2) using the backend REPL
```
```
Evaluate (js/console.log "test") using the frontend REPL
````


## Install Babashka

powershell

```powershell
scoop install babashka
```

Verify:

powershell

```powershell
bb --version
```

## Install bbin

powershell

```powershell
iex (bb -e '(slurp "https://raw.githubusercontent.com/babashka/bbin/main/install.clj")')
```

Add `%USERPROFILE%\.local\bin` to PATH.

Verify:

powershell

```powershell
bbin --version
```

## Install clojure-mcp-light Tools

**Hook tool (works directly):**

powershell

```powershell
bbin install https://github.com/bhauman/clojure-mcp-light.git --tag v0.2.1
```

**REPL eval tool (two-step due to Windows quoting):**

Step 1 — install without main-opts:

powershell

```powershell
bbin install https://github.com/bhauman/clojure-mcp-light.git --tag v0.2.1 --as clj-nrepl-eval
```

Step 2 — fix the generated script:

powershell

```powershell
$file = "$env:USERPROFILE\.local\bin\clj-nrepl-eval"
(Get-Content $file) -replace '\(def script-main-opts nil\)', '(def script-main-opts ["-m" "clojure-mcp-light.nrepl-eval"])' | Set-Content $file
```

**Verify tools installed:**

powershell

```powershell
clj-paren-repair-claude-hook --help
clj-nrepl-eval --help
```

Both should display help text without errors.

## Configure Hooks

Create `%USERPROFILE%\.claude\settings.json`:

json

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

Create `%USERPROFILE%\.claude\CLAUDE.md`:

markdown

```markdown
# Clojure REPL

Discover ports: clj-nrepl-eval --discover-ports
Evaluate: clj-nrepl-eval -p <port> "<code>"
Always use :reload when requiring namespaces.
```

## Project deps.edn

clojure

```clojure
{:aliases
 {:nrepl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.1"}}
          :main-opts ["-m" "nrepl.cmdline" "--port" "7888"]}}}
```

## Run

powershell

````powershell
# Terminal 1
clojure -M:nrepl

# Terminal 2
claude
```

## Verify Hooks Are Active

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
````

Open `src/broken.clj`. If hooks work, file contains:

clojure

```clojure
(defn foo [x]
  (+ x 1))
```

Missing paren was auto-repaired. If hooks failed, file has the original broken code.

**Method 3: Enable logging**

Change hook command in `settings.json`:

json

```json
"command": "clj-paren-repair-claude-hook --cljfmt --log-level debug"
```

Restart Claude Code. After any Clojure file edit, check `.clojure-mcp-light-hooks.log` in your project directory.

Seem not to work. I'll restart computer and then we

Jan 5

Anticipated restart and continuation of troubleshooting efforts.

The user is going to restart their computer to try to get clojure-mcp-light hooks working. They'll continue after the restart.

Understood. After restart:

1. Open fresh terminal
2. Verify tools still work:

powershell

```powershell
   clj-paren-repair-claude-hook --help
   clj-nrepl-eval --help
```

3. Start Claude Code in a project folder
4. Try the intentional error test

Report back what happens.

My project has deps clojure backend in src/clj and shadow-cljs clojurescript frontend in src/cljs.

What are my configuration steps for Clojure MCP light?

Jan 5

Synthesized dual-port REPL configuration for full-stack Clojure setup.

The user has a project with:

- Clojure backend in src/clj (deps.edn)
- ClojureScript frontend in src/cljs (shadow-cljs.edn)

They want to use clojure-mcp-light with this setup.

For clojure-mcp-light:

1. Hooks work on all .clj, .cljs, .cljc files automatically - no special config needed
2. REPL eval needs two ports - one for Clojure nREPL, one for shadow-cljs nREPL
3. They need to configure CLAUDE.md to tell the LLM about both ports

Let me give a concise configuration guide.

# clojure-mcp-light for Dual Clojure/ClojureScript Project

## Hooks

No changes needed. Hooks intercept all `.clj`, `.cljs`, `.cljc` edits automatically.

## Project nREPL Configuration

**deps.edn:**

clojure

```clojure
{:paths ["src/clj"]
 :aliases
 {:nrepl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.1"}}
          :main-opts ["-m" "nrepl.cmdline" "--port" "7888"]}}}
```

**shadow-cljs.edn:**

clojure

```clojure
{:source-paths ["src/cljs"]
 :nrepl {:port 9000}
 :builds {:app {...}}}
```

## Project CLAUDE.md

Create `CLAUDE.md` in project root:

markdown

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

## Run

powershell

````powershell
# Terminal 1: Backend REPL
clojure -M:nrepl

# Terminal 2: Frontend REPL + watch
npx shadow-cljs watch app

# Terminal 3: Claude Code
claude
```

## Verify
```
Evaluate (+ 1 2) using the backend REPL
```
```
Evaluate (js/console.log "test") using the frontend REPL
````

```
Evaluate (js/console.log "test") using the frontend REPL
```

gives me this:

● It appears the frontend REPL connection is stateless between invocations. To properly evaluate (js/console.log "test") in the frontend, you would need to maintain a persistent REPL session. The setup requires: 1. Connecting to the shadow-cljs REPL with (shadow/repl :app) to enter ClojureScript mode 2. Then evaluating the JavaScript interop code Each separate clj-nrepl-eval call resets the connection state. For interactive ClojureScript development, you may want to use a persistent REPL client (like connecting directly with your editor or using a REPL process that stays running) rather than individual command invocations.

Show more

Jan 5

Questioned the accuracy of the reported limitation.

The user is reporting that clj-nrepl-eval doesn't maintain state between invocations. This is actually incorrect based on the clojure-mcp-light README which says:

"The REPL session persists between evaluations - namespaces and state are maintained."

So either:

1. Claude is confused about how clj-nrepl-eval works
2. There's something specific about shadow-cljs that requires a different approach

Looking at the README more carefully:

"clj-nrepl-eval - nREPL evaluation from command line... Maintains persistent sessions per target"

So it should maintain sessions. The issue might be that shadow/repl :app needs to be called first to switch to ClojureScript mode, and then subsequent calls should work.

Let me think about this. The user needs to:

1. First call `clj-nrepl-eval -p 9000 "(shadow/repl :app)"` to switch to ClojureScript mode
2. Then call `clj-nrepl-eval -p 9000 "(js/console.log \"test\")"`

If sessions persist, this should work. Claude might be wrong here, or there might be something about how shadow-cljs nREPL works that's different.

I should tell the user to try the two-step approach and see if it actually works.

Claude is mistaken. clj-nrepl-eval does maintain session state between calls.

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

powershell

```powershell
clj-nrepl-eval -p 9000 "(shadow/repl :app)"
clj-nrepl-eval -p 9000 "(+ 1 2)"
```

Second call should return `3` from ClojureScript runtime, confirming session persists.