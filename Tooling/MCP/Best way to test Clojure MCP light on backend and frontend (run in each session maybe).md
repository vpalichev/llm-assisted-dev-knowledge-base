#### Ports

- BACKEND_REPL = 7888
- FRONTEND_REPL = 9000
- SHADOW_BUILD = :app

I have Claude Code configured with hooks for Clojure development. I need to verify that my hooks are working correctly.

My setup:

- clj-nrepl-eval CLI tool for REPL evaluation (ports defined in CLAUDE.md: <BACKEND_REPL>, <FRONTEND_REPL>)
- clj-paren-repair-claude-hook --cljfmt runs on PreToolUse, PostToolUse (for Write/Edit), and SessionEnd

**Important:** `clj-nrepl-eval` does NOT persist REPL session state between calls. For ClojureScript, use `shadow.cljs.devtools.api/cljs-eval` directly instead of `(shadow/repl :app)`.

Please help me verify:

1. **Backend REPL:** `clj-nrepl-eval -p <BACKEND_REPL> "(+ 1 2 3)"` → should return 6
2. **Frontend connectivity:** `clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/repl-runtimes <SHADOW_BUILD>)"` → should return list with browser info
3. **Frontend eval:** `clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(.-title js/document)\" {})"` → should return page title
4. **Formatting hook:** Create Clojure file with bad indentation → should auto-format
5. **Paren repair:** Write code with missing paren → should auto-fix

Let me know what works and what doesn't.