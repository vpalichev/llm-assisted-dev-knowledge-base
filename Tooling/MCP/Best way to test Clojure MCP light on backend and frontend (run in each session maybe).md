### Given that:
#### Local TCP Port Numbers:

- BACKEND_REPL = NNNN (recheck in deps.edn)
- FRONTEND_REPL = NNNN (recheck in shadow-cljs)
- SHADOW_BUILD = :app (be careful, could be different,  recheck in shadow-cljs)

#### Task context:
I are using Claude Code configured with hooks from Clojure MCP Light (be careful, it's "Light" version, not regular version) tooling for Clojure development. I need to verify that that Clojure MCP Light hooks are working correctly.

#### My setup:

- clj-nrepl-eval CLI tool for REPL evaluation (ports defined in CLAUDE.md: <BACKEND_REPL>, <FRONTEND_REPL>)
- clj-paren-repair-claude-hook --cljfmt runs on PreToolUse, PostToolUse (for Write/Edit), and SessionEnd

**Important:** `clj-nrepl-eval` does NOT persist REPL session state between calls. For ClojureScript, use `shadow.cljs.devtools.api/cljs-eval` directly instead of `(shadow/repl <SHADOW_BUILD>)`.

In order to test browser connectivity, user has to open the app in a browser (http://localhost:80 - be careful, can be different) and keep that tab open for the CLJS REPL to connect, so first of all ask him user if he launched the app in browser 

If frontend is not available, check backend only
#### Please help me verify:

1. **Backend REPL:** `clj-nrepl-eval -p <BACKEND_REPL> "(+ 1 2 3)"` → should return 6
2. **Frontend connectivity:** `clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/repl-runtimes <SHADOW_BUILD>)"` → should return list with browser info
3. **Frontend eval:** `clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(.-title js/document)\" {})"` → should return page title
4. **Paren repair:** Create Clojure file with correct indentation but missing paren -> should auto-fix (Paren repair uses indentation to infer where missing parens belong)
5. **Formatting hook:** Create separate Clojure file with bad indentation (balanced parens) -> should auto-format

  **Note:** Tests 4 and 5 must use files inside the project directory (e.g., `src/`), not temp directories.
  The `--cljfmt` option requires project context (deps.edn or project.clj in parent directories) to run formatting.
  This is a conservative design choice in the tool, not a technical requirement of cljfmt rules.

Let me know what works and what doesn't. Use green / red emojis in final result  for visibility.


### LLM Pre-flight Check

**Context:** On Windows with Git Bash (MINGW64), Clojure expressions containing Unix-style path strings get silently corrupted when passed as command-line arguments to `clj-nrepl-eval`. For example, `"/api/health"` becomes `"C:/Program Files/Git/api/health"`, causing cryptic Java exceptions at runtime. The fix is a pair of bash wrapper functions in `~/.bashrc` that suppress this conversion. Without the fix active, `clj-nrepl-eval` calls with path strings will fail.

Before calling `clj-nrepl-eval` from Git Bash, please make sure that:

- `~/.bashrc` exists and contains the wrapper functions for `clj-nrepl-eval` and `clj-paren-repair` (see Fix section above)

- `~/.bash_profile` exists and sources `~/.bashrc`

- The current shell has the wrappers active — verify with `type clj-nrepl-eval`, which should say `clj-nrepl-eval is a function`, not `clj-nrepl-eval is /c/Users/...`

- If the wrappers are not active, run `source ~/.bashrc` before proceeding

- If the error `NumberFormatException "8900C:"` (or similar with a drive letter) appears, the wrapper is not active

  Sanity check — confirm the wrapper is active and protects clj-nrepl-eval:

  #### Step 1: wrapper must be a shell function, not a binary path
  type clj-nrepl-eval
  #### Expected: clj-nrepl-eval is a function
  #### Bad:      clj-nrepl-eval is /c/Users/...

  If it shows a binary path, run source ~/.bashrc and re-check before proceeding.

  #### Step 2: clj-nrepl-eval must not mangle Unix-style path strings
  #### (requires nREPL running on <BACKEND_REPL>)      
  clj-nrepl-eval -p <BACKEND_REPL> '(str "/foo/bar")'
  #### Expected: => "/foo/bar"
  #### Bad:      => "C:/Program Files/Git/foo/bar"  (wrapper not active)

  Note: running bb -e '...' -- '(ns/fn "/foo/bar")' will show path mangling even on a correctly configured machine — that's expected. Only clj-nrepl-eval itself needs to be protected.