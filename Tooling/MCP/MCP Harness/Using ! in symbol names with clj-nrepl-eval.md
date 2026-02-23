---

tags:

- babashka
- bash
- clj-nrepl-eval
- windows
- quoting created: 2026-02-20

---

# Using `!` in symbol names with `clj-nrepl-eval`

Symbols ending in `!` (e.g. `start!`, `stop!`, `ps!`) require special handling in bash because `!` triggers history expansion and gets dropped somewhere in the `clj-nrepl-eval` toolchain.

---

## The rule

Never put `!` inside single quotes. Splice it between two single-quoted segments instead:

```bash
'(namespace/fn'\!' args)'
```

This is three bash tokens joined into one argument:

|Token|What it is|
|---|---|
|`'(namespace/fn'`|Single-quoted prefix|
|`\!`|Backslash-escaped literal `!`|
|`' args)'`|Single-quoted suffix|

---

## Examples

```bash
# Server lifecycle
clj-nrepl-eval -p 8950 '(harness.server/start'\!')'
clj-nrepl-eval -p 8950 '(harness.server/stop'\!')'

# Shell helpers
clj-nrepl-eval -p 8950 '(harness.shell/ps'\!' "Get-Date")'

# Checking a var
clj-nrepl-eval -p 8950 '(str (var harness.shell/ps'\!'))'
```

---

## What breaks without it

```bash
# WRONG — ! is dropped, calls ps instead of ps!
clj-nrepl-eval -p 8950 '(harness.shell/ps! "Get-Date")'
```

> [!danger] Silent wrong-function call The call succeeds but invokes the wrong function — `ps` returns a string, `ps!` returns a map. No error is thrown.

---

## Does not apply to Calva REPL

> [!tip] This is a `clj-nrepl-eval` / bash quirk only. In Calva, type `ps!` normally.