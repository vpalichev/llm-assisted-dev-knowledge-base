Problem: Multi-layer quote escaping in CLI agent environments on Windows                                                                                                                                                                                              
                                                                                                                                                                                                                                                                        
  I'm using Claude Code (an AI CLI agent) on Windows. The agent's shell is Git Bash. From Git Bash, it sometimes needs to invoke PowerShell (powershell -c "..."), and separately, it needs to evaluate Clojure/ClojureScript code via a command-line REPL tool called    clj-nrepl-eval.                                                                                                                                                                                                                                                       

  The quoting problem has multiple layers:

  1. Git Bash → PowerShell: Any PowerShell command must be wrapped in powershell -c "...", so the command itself can't use unescaped double quotes. Single quotes behave differently in PowerShell vs Bash.
  2. Git Bash → clj-nrepl-eval → Clojure: The REPL tool takes Clojure code as a string argument: clj-nrepl-eval -p 7888 "(some-clojure-code)". If the Clojure code contains strings, those inner quotes must be escaped for Bash.
  3. Git Bash → clj-nrepl-eval → Shadow CLJS API → ClojureScript (worst case): To evaluate ClojureScript in a browser REPL, you call a Clojure function that itself takes a ClojureScript code string as an argument:
  clj-nrepl-eval -p 9000 "(shadow.cljs.devtools.api/cljs-eval :app \"(js/alert \\\"hello\\\")\" {})"
  4. That's three nesting levels: Bash parses the outer quotes, Clojure parses the escaped quotes, and then ClojureScript parses the double-escaped quotes. If the ClojureScript code itself contains strings with special characters, it becomes essentially
  unwritable.
  5. The AI agent generates these commands programmatically as plain strings passed to a Bash tool. It has no access to alternative shells, no ability to write temp files silently, and no interactive session persistence (each command runs in a fresh shell
  invocation — no state carries over between calls).

  What I'm looking for:

  Suggest practical tooling solutions to reduce or eliminate this quoting problem. Consider approaches like:
  - Wrapper scripts or shell functions that accept code via stdin or temp files instead of arguments
  - A small proxy tool that reads code from a file rather than command-line args
  - Encoding schemes (base64?) to bypass quoting entirely
  - Modifications to clj-nrepl-eval or a replacement tool
  - Any other creative solutions

  Constraints: Windows 10/11, Git Bash available, PowerShell available, Node.js installed, Clojure CLI installed. The solution should be simple to set up and robust. The primary user of these commands is an AI agent, not a human, so ergonomics matter less than    
  reliability and correctness.