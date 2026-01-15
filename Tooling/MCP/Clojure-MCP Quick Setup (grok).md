# Clojure-MCP Quick Setup

## 1. Add nREPL to Your Project

**deps.edn:**

clojure

```clojure
{:aliases
 {:nrepl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.1"}}
          :main-opts ["-m" "nrepl.cmdline" "--port" "7888"]}}}
```

**Leiningen:** `lein repl :headless :port 7888`

## 2. Install Clojure-MCP

Add to `~/.clojure/deps.edn`:

clojure

```clojure
{:aliases
 {:mcp
  {:deps {org.slf4j/slf4j-nop {:mvn/version "2.0.16"}
          com.bhauman/clojure-mcp {:git/url "https://github.com/bhauman/clojure-mcp.git"
                                   :git/tag "v0.1.12"
                                   :git/sha "79b9d5a"}}
   :exec-fn clojure-mcp.main/start-mcp-server
   :exec-args {:port 7888}}}}
```

## 3. Configure Your MCP Client

**For Cline** (`.vscode/mcp.json` or Cline MCP settings):

json

```json
{
  "mcpServers": {
    "clojure-mcp": {
      "command": "clojure",
      "args": ["-X:mcp", ":port", "7888"]
    }
  }
}
```

**For Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):

json

```json
{
  "mcpServers": {
    "clojure-mcp": {
      "command": "/bin/bash",
      "args": ["-c", "clojure -X:mcp :port 7888"]
    }
  }
}
```

## 4. Run

bash

```bash
# Terminal 1: Start nREPL in your project
cd /path/to/project && clojure -M:nrepl

# Terminal 2 (optional, client usually starts this):
clojure -X:mcp :port 7888
```

## Key Tools

|Tool|Use|
|---|---|
|`clojure_eval`|Evaluate code in REPL|
|`clojure_edit`|Structure-aware file edits|
|`read_file`|View files with collapsed forms|

## Example Prompt

> "Use clojure_eval to test `(+ 1 2 3)`, then create a function that doubles a number and validate it works."

The LLM will evaluate code directly in your REPL, auto-fix bracket issues via parinfer, and lint with clj-kondo—no more bracket-guessing loops.