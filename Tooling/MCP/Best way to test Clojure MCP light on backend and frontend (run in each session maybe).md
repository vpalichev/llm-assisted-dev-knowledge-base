### Given that:
#### Local TCP Port Numbers:

- BACKEND_REPL = 7888 (be careful, can be different)
- FRONTEND_REPL = 9000 (be careful, can be different)
- SHADOW_BUILD = :app (be careful, can be different)

#### Task context:
I are using Claude Code configured with hooks from Clojure MCP Light (be careful, it's "Light" version, not regular version) tooling for Clojure development. I need to verify that that Clojure MCP Light hooks are working correctly.

#### My setup:

- clj-nrepl-eval CLI tool for REPL evaluation (ports defined in CLAUDE.md: <BACKEND_REPL>, <FRONTEND_REPL>)
- clj-paren-repair-claude-hook --cljfmt runs on PreToolUse, PostToolUse (for Write/Edit), and SessionEnd

**Important:** `clj-nrepl-eval` does NOT persist REPL session state between calls. For ClojureScript, use `shadow.cljs.devtools.api/cljs-eval` directly instead of `(shadow/repl :app)`.

In order to test browser connectivity, user has to open the app in a browser (http://localhost:80 - be careful, can be different) and keep that tab open for the CLJS REPL to connect, so first of all ask him user if he launched the app in browser 

#### Please help me verify:

1. **Backend REPL:** `clj-nrepl-eval -p <BACKEND_REPL> "(+ 1 2 3)"` → should return 6
2. **Frontend connectivity:** `clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/repl-runtimes <SHADOW_BUILD>)"` → should return list with browser info
3. **Frontend eval:** `clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(.-title js/document)\" {})"` → should return page title
4. **Paren repair:** Create Clojure file with correct indentation but missing paren -> should auto-fix (Paren repair uses indentation to infer where missing parens belong)
5. **Formatting hook:** Create separate Clojure file with bad indentation (balanced parens) -> should auto-format

  **Note:** Tests 4 and 5 must use files inside the project directory (e.g., `src/`), not temp directories.
  The `--cljfmt` option requires project context (deps.edn or project.clj in parent directories) to run formatting.
  This is a conservative design choice in the tool, not a technical requirement of cljfmt rules.

Let me know what works and what doesn't.
