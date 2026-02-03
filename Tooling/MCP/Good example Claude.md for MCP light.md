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

# Get page HTML

clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval :app \"(.-innerHTML js/document.body)\" {})"

  

# Query DOM element

clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval :app \"(js/document.querySelector \\\"#app\\\")\" {})"

  

# Check app state

clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval :app \"@re-frame.db/app-db\" {})"

```

  

Note: ClojureScript code is passed as a string, so escape inner quotes with \\"

  

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