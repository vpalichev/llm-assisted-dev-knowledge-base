 Reproduction Test Summary                                                                                                                                                                                                                                    
     
  Date: 2026-02-04                                                                                                                                                                                                                                               
  Tool: clj-paren-repair-claude-hook --cljfmt
  Hook trigger: PostToolUse on Write                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          
  Test Setup                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  
  Identical Clojure code with bad indentation written to two locations:                                                                                                                                                                                                                                                                                                                                                                                                                                                       
  (defn foo [x]
  (+ x 1))

  (defn bar [a b]
  (let [sum (+ a b)]
  {:result sum}))

  Results
  ┌───────────────────┬─────────────────────────────────────────────────┬────────────┬───────────────────────┐
  │       File        │                    Location                     │ Formatted? │        Result         │
  ├───────────────────┼─────────────────────────────────────────────────┼────────────┼───────────────────────┤
  │ repro_temp.clj    │ C:\Users\...\AppData\Local\Temp\...\scratchpad\ │ ❌         │ Indentation unchanged │
  ├───────────────────┼─────────────────────────────────────────────────┼────────────┼───────────────────────┤
  │ repro_project.clj │ D:\projects\reporting-planner-cljc-v05\src\clj\ │ ✅         │ Properly indented     │
  └───────────────────┴─────────────────────────────────────────────────┴────────────┴───────────────────────┘
  Output Comparison

  Temp directory (NOT formatted):
  (defn foo [x]
  (+ x 1))

  (defn bar [a b]
  (let [sum (+ a b)]
  {:result sum}))

  Project directory (formatted):
  (defn foo [x]
    (+ x 1))

  (defn bar [a b]
    (let [sum (+ a b)]
      {:result sum}))

  Conclusion

  The --cljfmt flag silently skips formatting for files outside a Clojure project context. The only difference between the two files was their location - the project directory contains deps.edn at its root, the temp directory does not.

 No security implications for this behavior identified. Silent skipping appears to be an oversight or overly conservative design choice. The expected behavior would be to fall back to cljfmt defaults when no project config is found, not to skip formatting entirely.       



Here's a draft GitHub issue:                                                                                                                                                                                                                                   
  ---                                                                                                                                                                                                                                                            Title: --cljfmt flag silently skips files outside project directories
                                                                                                                                                                                                                                                                 Labels: bug, enhancement

  ---
  Summary

  The clj-paren-repair-claude-hook --cljfmt flag only formats files that are inside a Clojure project (where deps.edn or project.clj exists in a parent directory). Files in temp directories or outside project roots are silently skipped without formatting.
  Reproduction

  1. Create a Clojure file with bad indentation inside a project:
  echo "(defn foo [x]
  (+ x 1))" > ~/my-clojure-project/src/test.clj
  → File gets formatted correctly ✅

  2. Create the same file outside any project (e.g., temp directory):
  echo "(defn foo [x]
  (+ x 1))" > /tmp/test.clj
  → File is NOT formatted, silently skipped ❌

  Expected Behavior

  Files should be formatted regardless of location. The cljfmt core API works without project context:

  (require '[cljfmt.core :as fmt])
  (fmt/reformat-string "(defn foo [x]\n(+ x 1))")
  ;; => "(defn foo [x]\n  (+ x 1))"

  Standard Clojure indentation rules are universal and don't require project-specific configuration.

  Likely Cause

  The tool appears to use either:
  - cljfmt.config/load-config which searches for project markers
  - cljfmt.tool/* functions that expect project structure
  - Explicit project root detection before invoking cljfmt

  When no project context is found, formatting is skipped rather than falling back to defaults.

  Suggested Fix

  Call cljfmt.core/reformat-string directly with default options when no project config is found:

  (defn format-code-string [code-string]
    (let [cfg (or (try (cljfmt.config/load-config)
                       (catch Exception _ nil))
                  {})]  ; Empty map = use built-in defaults
      (fmt/reformat-string code-string cfg)))

  Or simply:
  (fmt/reformat-string code-string) ; defaults work fine

  Use Case

  When using Claude Code with hooks, Claude sometimes creates test files in temp/scratchpad directories. These should still be formatted. The project context requirement seems like an implementation artifact rather than an intentional design choice.      

  Environment

  - OS: Windows 10/11
  - Tool: clj-paren-repair-claude-hook --cljfmt
  - Context: Claude Code hooks (PreToolUse/PostToolUse on Write/Edit)