This is a well-diagnosed problem and you've already identified the right solution space. Here's my analysis of why some approaches work and others don't, and what I'd actually recommend.

## Why more quoting rules won't work

Adding quoting guidance to CLAUDE.md is tempting but fundamentally doomed for your three-layer case. The problem isn't that Claude doesn't know the escaping rules — it's that **the correct escaping for three nested layers is context-dependent, fragile, and often ambiguous even for humans**. A single `$` or backtick in the Clojure code changes the escaping strategy for the entire command. An LLM generating these strings programmatically will get it wrong intermittently in ways that are hard to debug. This is an architecture problem, not a knowledge problem.

## The core principle: code must never pass through shell argument parsing

Every solution that passes code as a **command-line argument** is vulnerable, because every shell layer between Claude Code and the evaluator consumes a round of parsing. Even the file-based approach has a trap:

```bash
# STILL BROKEN — bash expands $code before clj-nrepl-eval sees it
code=$(cat /tmp/eval.clj)
clj-nrepl-eval -p 7888 "$code"
```

If the Clojure code contains `$`, backticks, `"`, or `\`, bash mangles it inside the `"$code"` expansion. You'd need `clj-nrepl-eval` to read from **stdin or a file directly**, bypassing the shell argument vector entirely.

## Recommended solutions, ranked

### 1. Direct nREPL client in Node.js (best — eliminates ALL quoting)

nREPL is just bencode over TCP. A ~40 line Node.js script can connect to the socket, send an eval op, and print the result. Code goes from a file straight into a TCP write buffer — zero shell layers involved.

```
Claude Code writes .clj file → Node.js reads file → TCP send to nREPL → result on stdout
```

Claude Code already has a file-creation tool that doesn't go through bash. So the flow is: create file (no shell), then run `node eval-nrepl.js --port 7888 --file /tmp/eval.clj` (the only shell argument is a filename — no special characters, no quoting). You could also use this for the Shadow CLJS case by having the script wrap the code in the `shadow.cljs.devtools.api/cljs-eval` call internally based on a `--cljs --build app` flag.

The `nrepl-client` npm package exists, or you can do raw bencode — it's a trivial format (nREPL messages are just `d4:code...e` dictionaries).

### 2. Stdin-based wrapper using `rep` or similar

[`rep`](https://github.com/eraserhd/rep) is a purpose-built CLI nREPL client that reads from stdin:

```bash
echo '(+ 1 2)' | rep -p 7888
```

For the file case:

```bash
rep -p 7888 < /tmp/eval.clj
```

stdin piping bypasses argument parsing entirely — the shell reads the bytes from the file/pipe and passes them to the process's stdin file descriptor without interpretation. The only risk is if `echo` is used instead of file redirection (echo can mangle backslashes depending on the shell). **Always use `< file` redirection, not `cat file |`**, to minimize shell involvement.

Check if `rep` has Windows binaries or builds with Go on Windows. If not, the Node.js approach from option 1 is more reliable.

### 3. Base64 transport (good fallback)

If you can't change the nREPL client, a wrapper that decodes base64 and pipes to stdin works:

```bash
# eval-nrepl.sh — installed on PATH
#!/bin/bash
# Usage: eval-nrepl.sh <port> <base64-encoded-code>
echo "$2" | base64 -d | clj-nrepl-eval -p "$1" --stdin
```

But this only works if `clj-nrepl-eval` supports `--stdin`. If it doesn't, you need a variant that writes to a temp file and passes `--file`:

```bash
#!/bin/bash
tmpfile=$(mktemp /tmp/nrepl-eval-XXXXXX.clj)
echo "$2" | base64 -d > "$tmpfile"
clj-nrepl-eval -p "$1" "$(cat "$tmpfile")"  # ← STILL has the expansion problem
rm -f "$tmpfile"
```

See the trap? Even with base64, if the final delivery mechanism is a shell argument, you're back to square one. Base64 only helps if the **decoding happens inside the receiving process**, not in bash. This makes base64 viable only in combination with stdin or file-based input on the receiving end.

### 4. Babashka as nREPL client (if you're already in the Clojure ecosystem)

`bb` (Babashka) can act as an nREPL client and reads from stdin naturally:

```bash
bb -e '(load-file "/tmp/eval.clj")' # local eval, not nREPL
```

For nREPL, you'd write a small Babashka script that connects to the port and evals. Babashka is a single static binary (~25MB), starts in ~10ms, and has nREPL client libraries available. It's arguably the most natural fit for a Clojure workflow, though it adds another tool to install.

## What to put in CLAUDE.md

Don't try to teach Claude the quoting rules for multi-layer escaping. Instead, tell it which tool to use and how:

```markdown
### Clojure/ClojureScript evaluation — MANDATORY WORKFLOW
- NEVER pass Clojure code as a shell argument to clj-nrepl-eval directly.
  Multi-layer quoting between Bash → clj-nrepl-eval → Clojure/CLJS is
  unsolvable via escaping. It WILL break on $, backticks, quotes, and
  backslashes.
- ALWAYS use the file-based workflow:
  1. Write the Clojure code to /tmp/nrepl-eval.clj (use file creation, not echo)
  2. Run: node ~/tools/eval-nrepl.js --port <port> --file /tmp/nrepl-eval.clj
- For ClojureScript (Shadow CLJS):
  1. Write the ClojureScript code to /tmp/nrepl-eval.cljs
  2. Run: node ~/tools/eval-nrepl.js --port <port> --file /tmp/nrepl-eval.cljs --cljs --build app
- NEVER attempt to escape nested quotes manually. If you find yourself adding
  backslashes inside Clojure strings inside bash strings, you are doing it
  wrong. Use the file workflow.
```

## Summary

|Approach|Quoting layers eliminated|Setup cost|Reliability|
|---|---|---|---|
|Node.js nREPL client + file|All of them|~1 hour|Bulletproof|
|`rep` or similar stdin client|All of them|Install binary|High (if Windows build exists)|
|Babashka nREPL script|All of them|Install bb|High|
|Base64 + stdin wrapper|Most|Small script|Good if receiver supports stdin|
|Better CLAUDE.md quoting rules|None|Free|Will fail intermittently|

The Node.js nREPL client is the move. You already have Node.js installed, it's one file, it runs natively on Windows, and it reduces the problem from "three-layer quoting nightmare" to "pass a filename as a CLI argument" — which is trivially correct in every shell.