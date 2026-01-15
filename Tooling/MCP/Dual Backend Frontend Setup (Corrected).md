

## Global Config

**`~/.clojure/deps.edn`:**

clojure

```clojure
{:aliases
 {:mcp-dual {:deps {org.slf4j/slf4j-nop {:mvn/version "2.0.16"}
                    com.bhauman/clojure-mcp {:git/url "https://github.com/bhauman/clojure-mcp.git"
                                             :git/tag "v0.1.12"
                                             :git/sha "79b9d5a"}}
             :exec-fn clojure-mcp.main-examples.shadow-main/start-mcp-server
             :exec-args {:port 7888
                         :shadow-port 9000
                         :shadow-build "app"}}}}
```

## Project Config

**`deps.edn`:**

clojure

```clojure
{:aliases
 {:nrepl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.1"}}
          :main-opts ["-m" "nrepl.cmdline" "--port" "7888"]}}}
```

**`shadow-cljs.edn`:**

clojure

```clojure
{:nrepl {:port 9000}
 :builds {:app {...}}}
```

**`.mcp.json`:**

json

```json
{
  "mcpServers": {
    "clojure-mcp": {
      "command": "cmd",
      "args": ["/c", "clojure", "-X:mcp-dual"]
    }
  }
}
```

## Run

bash

```bash
# Terminal 1
clojure -M:nrepl

# Terminal 2
npx shadow-cljs watch app

# Terminal 3
claude
```

## Usage

- "Evaluate in clojure: `(+ 1 2)`"
- "Evaluate in clojurescript: `(js/alert \"hi\")`"