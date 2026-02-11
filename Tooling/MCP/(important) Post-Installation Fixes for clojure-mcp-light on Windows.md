

After installing via bbin, Windows users must manually fix the generated scripts in `C:\Users\<username>\.local\bin\`.

---

### Fix 1: Quote the `-m` Flag

Locate the `script-main-opts` line in each script.

**Before:**

```clojure
(def script-main-opts [-m clojure-mcp-light.nrepl-eval])
```

**After:**

```clojure
(def script-main-opts ["-m" "clojure-mcp-light.nrepl-eval"])
```

---

### Fix 2: Add Source Path to Classpath

Locate the `base-command` definition in each script.

**Before:**

```clojure
(def base-command
  (vec (concat ["bb" "--deps-root" script-root "--config" (str tmp-edn)]
               script-main-opts
               ["--"])))
```

**After:**

```clojure
(def base-command
  (vec (concat ["bb" 
                "--deps-root" script-root 
                "--config" (str tmp-edn)
                "-cp" (str script-root "\\src")]
               script-main-opts
               ["--"])))
```

---

### Scripts Requiring Fixes

|Script|Namespace|
|---|---|
|`clj-nrepl-eval`|`clojure-mcp-light.nrepl-eval`|
|`clj-paren-repair`|`clojure-mcp-light.paren-repair`|
|`clj-paren-repair-claude-hook`|`clojure-mcp-light.hook`|

---

### Verify

```powershell
clj-nrepl-eval --help
clj-paren-repair --help
clj-paren-repair-claude-hook --help
```