---

tags:

- babashka
- windows
- graalvm
- encoding
- bugfix created: 2026-02-20

---

# Fix: `clj-nrepl-eval` outputs `?` for Cyrillic on Windows

## Problem

`clj-nrepl-eval` is a Babashka (`bb.exe`) script — a GraalVM native image. Babashka's `*out*` is an `OutputStreamWriter` frozen with **Cp1252** at native-image build time. Cp1252 has no Cyrillic (or CJK, or any non-Latin) mappings, so every such character is replaced with `?` (byte `0x3F`) before it reaches the terminal.

> [!important] This is **not** a terminal rendering issue. The `?` bytes are written by `bb.exe` itself and appear in redirected files too.

```bash
# Confirm the bug is present
bb -e '(println (.getEncoding *out*))'
# Cp1252  ← root cause

bb -e '(print (String. (byte-array [0xD0 0x9F 0xD1 0x80]) "UTF-8"))' > /tmp/t.bin
xxd /tmp/t.bin
# 3f 3f  ← corrupted (should be d0 9f d1 80)
```

---

## Prerequisites

- `bb.exe` (Babashka) installed and on `PATH`
- `clj-nrepl-eval` installed via bbin
- Git Bash (MSYS2) as the shell

---

## Step 1 — Find the wrapper

```bash
which clj-nrepl-eval
```

This prints the full path, typically:

```
/c/Users/<username>/.local/bin/clj-nrepl-eval
```

Open it and find:

1. The value of `script-main-opts` — contains the main namespace name (e.g. `["-m" "clojure-mcp-light.nrepl-eval"]`)
2. The definition of `base-command` — the `concat` that builds the child `bb` invocation

---

## Step 2 — Understand the wrapper structure

The wrapper is itself a `bb` script. It builds a command and uses `process/exec` to replace itself with a child `bb` process that runs the real tool. The fix must go into the **child** `bb` invocation — changing `*out*` in the wrapper process has no effect.

The original `base-command` looks like this:

```clojure
(def base-command
  (vec (concat ["bb"
                "--deps-root" script-root
                "--config" (str tmp-edn)
                "-cp" (str script-root "\\src")]
               script-main-opts   ; ["-m" "clojure-mcp-light.nrepl-eval"]
               ["--"])))
```

---

## Step 3 — Apply the fix

> [!warning] `-e` + `-m` do not compose Babashka does **not** compose `-e` and `-m`. When both are present, `-m` is silently ignored. The fix therefore replaces `-m <namespace>` entirely with a single `-e` expression that:
> 
> 1. Rebinds `*out*` and `*err*` to UTF-8 writers
> 2. Requires the main namespace
> 3. Calls its `-main` with `*command-line-args*`

Replace the `base-command` definition (keep everything else unchanged):

```clojure
(def base-command
  (vec (concat ["bb"
                "--deps-root" script-root
                "--config" (str tmp-edn)
                "-cp" (str script-root "\\src")
                "-e" (str "(do"
                          " (alter-var-root #'*out* (constantly (java.io.PrintWriter. (java.io.OutputStreamWriter. System/out java.nio.charset.StandardCharsets/UTF_8) true)))"
                          " (alter-var-root #'*err* (constantly (java.io.PrintWriter. (java.io.OutputStreamWriter. System/err java.nio.charset.StandardCharsets/UTF_8) true)))"
                          " (require 'clojure-mcp-light.nrepl-eval)"
                          " (apply clojure-mcp-light.nrepl-eval/-main *command-line-args*))")]
               ["--"])))
```

> [!tip] Different main namespace? If your tool has a different main namespace (not `clojure-mcp-light.nrepl-eval`), replace both occurrences with whatever was in `script-main-opts`.
> 
> For example, if `script-main-opts` was `["-m" "my.tool.main"]`:
> 
> - `(require 'my.tool.main)`
> - `(apply my.tool.main/-main *command-line-args*)`

---

## Step 4 — Verify

```bash
# 1. Basic smoke test — should return => nil, not crash
clj-nrepl-eval -p <nrepl-port> '(+ 1 2)'

# 2. Binary capture — must show UTF-8 bytes, not 3F
clj-nrepl-eval -p <nrepl-port> \
  '(String. (byte-array [0xD0 0x9F 0xD1 0x80 0xD0 0xB8 0xD0 0xB2 0xD0 0xB5 0xD1 0x82]) "UTF-8")' \
  > /tmp/verify.bin
xxd /tmp/verify.bin
# Must show: d0 9f d1 80 d0 b8 d0 b2 d0 b5 d1 82
# NOT:       3f 3f 3f 3f 3f 3f 3f 3f 3f 3f 3f 3f

# 3. Terminal display — should show Привет, not ??????
clj-nrepl-eval -p <nrepl-port> \
  '(String. (byte-array [0xD0 0x9F 0xD1 0x80 0xD0 0xB8 0xD0 0xB2 0xD0 0xB5 0xD1 0x82]) "UTF-8")'
# => "Привет"
```

---

## What NOT to do

> [!danger] Known dead ends

|Attempt|Why it doesn't work|
|---|---|
|`JAVA_TOOL_OPTIONS=-Dfile.encoding=UTF-8`|`*out*` is frozen in the image heap — system properties have no effect on it|
|`chcp 65001` in cmd.exe|Changes Windows console codepage, not Java's internal writer encoding|
|`alter-var-root` in user nREPL code|Runs in the nREPL server (separate JVM process), not in `bb.exe`|
|`-e <fix> -m <namespace>`|`-m` is silently ignored when `-e` is present in Babashka|
|`(java.io.OutputStreamWriter. stream "Cp1252")`|Throws `UnsupportedEncodingException` — Cp1252 is not registered as a named charset in the native image at runtime|

---

## If bbin reinstalls or updates the tool

> [!caution] `bbin install` regenerates the wrapper and **overwrites your fix**. Reapply [[#Step 3 — Apply the fix|Step 3]] after any reinstall or update of `clj-nrepl-eval`.

To detect whether the fix is present:

```bash
grep -c 'StandardCharsets' "$(which clj-nrepl-eval)"
# 2  ← fix is present
# 0  ← fix is missing, reapply Step 3
```