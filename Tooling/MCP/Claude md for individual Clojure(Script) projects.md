# Codebase Documentation

  

Before exploring the codebase, read ARCHITECTURE.md - it contains:

- Complete directory structure with file purposes

- Key namespaces and their responsibilities

- Patterns (state management, routing, authentication, API)

- Data flow between frontend/backend

- Entry points and configuration

  

Only use exploration agents if the specific information isn't in ARCHITECTURE.md.

  

# Ports

  

- BACKEND_REPL = 7888

- FRONTEND_REPL = 9000

- BACKEND_API = 3000

- FRONTEND_DEV = 80

- SHADOW_BUILD = :app

  

# Project Structure

  

Full-stack Clojure/ClojureScript project:

- Backend (Clojure JVM): src/clj

- Frontend (ClojureScript browser): src/cljs

- Shared (CLJC): src/cljc

  

# REPL

  

Two separate REPLs - use the correct one based on what you're working on.

  

## Backend (Clojure JVM)

  

For code in src/clj:

```

clj-nrepl-eval -p <BACKEND_REPL> "<clojure-code>"

```

  

Example:

```

clj-nrepl-eval -p <BACKEND_REPL> "(+ 1 2 3)"

```

  

## Frontend (ClojureScript Browser)

  

IMPORTANT: `clj-nrepl-eval` does NOT persist REPL session state between calls.

This means `(shadow/repl :app)` won't help - the next call loses CLJS mode.

  

Use the shadow API to evaluate ClojureScript directly:

```

clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"<cljs-code>\" {})"

```

  

Examples:

```

# Check browser is connected (should return list with browser info)

clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/repl-runtimes <SHADOW_BUILD>)"

  

# Get page HTML

clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(.-innerHTML js/document.body)\" {})"

  

# Query DOM element

clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(js/document.querySelector \\\"#app\\\")\" {})"

  

# Check app state (explore src/cljs to find the state atom location first)

```

  

Note: ClojureScript code is passed as a string, so escape inner quotes with \\"

  
  
  
  

### shadow-cljs nREPL via clj-nrepl-eval

  

#### Banner — NOT an error

The shadow-cljs nREPL middleware prints the following on every eval

from a CLJ-mode session (which is every session created by clj-nrepl-eval):

```

;; shadow-cljs repl is NOT in CLJS mode

;; use (shadow/active-builds) to list builds available

;; use (shadow/repl <build-id>) to jack into a CLJS repl

```

This is informational. It means the session is in CLJ mode, which is correct.

Check for `=>` in the output to determine success. Do NOT retry based on this banner.

  

#### Evaluating ClojureScript from CLJ mode

Use `shadow.cljs.devtools.api/cljs-eval` — a CLJ-side function that dispatches

to the connected JS runtime. No need to switch to CLJS mode via `(shadow/repl :app)`.

```bash

# Correct — bash double-quote outer, escaped inner

clj-nrepl-eval -p 9000 "(shadow.cljs.devtools.api/cljs-eval :app \"(+ 1 2)\" {})"

# => {:results ["3"], :out "", :err "", :ns cljs.user}

```

  

#### Successful response format

A working cljs-eval returns: `=> {:results [...], :out "...", :err "...", :ns cljs.user}`

- `:results` — vector of stringified return values

- `:out` / `:err` — captured stdout/stderr from the JS runtime

- `=> nil` or empty results — JS runtime likely disconnected (browser tab closed or reloading)

  

#### String escaping in nested ClojureScript expressions

Each nesting level requires one additional layer of backslash escaping:

```bash

# Simple expression — one level of escaping

clj-nrepl-eval -p 9000 "(shadow.cljs.devtools.api/cljs-eval :app \"(+ 1 2)\" {})"

  

# String inside CLJS — two levels of escaping

clj-nrepl-eval -p 9000 "(shadow.cljs.devtools.api/cljs-eval :app \"(js/console.log \\\"hello\\\")\" {})"

```

  

3+ levels of nesting — DO NOT inline.

Define a helper function in a CLJS namespace and call it with simple arguments.

  

#### No result / nil result troubleshooting

If `cljs-eval` returns `nil` or no `=>` line:

1. Check runtime connectivity:

   `clj-nrepl-eval -p 9000 "(shadow.cljs.devtools.api/repl-runtimes :app)"`

   - Non-empty vector = browser connected (working)

   - Empty vector `[]` = no browser connected — open/refresh the app

2. Check build worker:

   `clj-nrepl-eval -p 9000 "(shadow.cljs.devtools.api/worker-running? :app)"`

   - `false` = watch not running, restart with `shadow-cljs watch app`

3. During hot-reload cycles the runtime briefly disconnects — retry after 1-2 seconds.

  
  
  

## General

  

Always use :reload when requiring namespaces.

  

# Configuration

  

- Backend config: config/backend-config.edn

- Server runs on port <BACKEND_API>

- Frontend dev server runs on port <FRONTEND_DEV> (shadow-cljs)

  

# Running the Project

  

Start both servers:

```

start.bat

```

  

Or separately:

- Backend: `clj -M:nrepl`

- Frontend: `npx shadow-cljs watch <SHADOW_BUILD>`

  

# Shell

  

This is Windows. The Bash tool runs Git Bash, so use Unix-style commands (`rm`, `ls`, `cp`), not cmd commands (`del`, `dir`, `copy`). For Windows-specific tasks, use `powershell -c "..."`.