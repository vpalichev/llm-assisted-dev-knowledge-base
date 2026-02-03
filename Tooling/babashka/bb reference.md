```
For clojure MCP:
bb nrepl-server 1667
claude
/mcp


Update `.mcp.json` in current folder to use it:
{

  "mcpServers": {
    "clojure-mcp": {
      "command": "cmd",
      "args": ["/c", "clojure", "-X:mcp-bb"]
    }
  }
}



Add to `%USERPROFILE%\.clojure\deps.edn`:

{:aliases
 {
  :mcp-bb {:deps {org.slf4j/slf4j-nop {:mvn/version "2.0.16"}
                  com.bhauman/clojure-mcp {:git/url "https://github.com/bhauman/clojure-mcp.git"
                                           :git/tag "v0.1.12"
                                           :git/sha "79b9d5a"}}
           :exec-fn clojure-mcp.main/start-mcp-server
           :exec-args {:port 1667 :nrepl-env-type :bb}}}}
```



bb.edn is required in folder also with this for utils to load:
{:paths ["C:/Users/v.palichev/.babashka/src"]}

# EXECUTION MODES
```powershell
# EXECUTION MODES
bb script.clj                    # Run script file
bb -e "(+ 1 2)"                  # Evaluate expression
bb                               # Start REPL
bb -m my.namespace               # Run -main function from namespace

# CLI FLAGS
bb --version                     # Show version
bb --help                        # Show help
bb --classpath src:lib script.clj  # Add to classpath
bb --prn -e "(+ 1 2)"           # Print result as Clojure data
bb --stream -e "(+ 1 2)"        # Stream processing mode
bb --verbose script.clj          # Verbose output
```
# bb.edn
```clojure
;; bb.edn - PROJECT CONFIGURATION FILE
{:paths ["src" "resources"]           ; Classpath directories
 :deps {babashka/fs {:mvn/version "0.5.20"}  ; Maven dependencies
        clj-http/clj-http {:mvn/version "3.12.3"}}
 
 :tasks                               ; Task definitions (like make/npm scripts)
 {:requires ([clojure.string :as str])
  
  ;; Simple task
  hello {:task (println "Hello")}
  
  ;; Task with shell command
  build {:task (shell "pwsh -Command dotnet build")}
  
  ;; Task with dependencies
  test {:depends [build]
        :task (shell "pwsh -Command dotnet test")}
  
  ;; Parameterized task
  deploy {:task (let [env (first *command-line-args*)]
                  (println "Deploying to" env))}}}
```

```powershell
# TASK EXECUTION
bb hello                         # Run 'hello' task
bb test                          # Run 'test' task (runs 'build' first)
bb deploy staging                # Run 'deploy' task with argument
bb tasks                         # List all available tasks
```

```clojure
;; deps.edn - ALTERNATIVE DEPENDENCY FORMAT (tools.deps compatible)
{:deps {babashka/fs {:mvn/version "0.5.20"}
        org.clojure/data.json {:mvn/version "2.4.0"}}
 :aliases {:dev {:extra-paths ["dev"]}}}
```

```powershell
# CLASSPATH MANAGEMENT
bb --classpath src:lib:test script.clj     # Multiple paths (colon-separated even on Windows)
bb --config bb-dev.edn tasks               # Use alternative config file

# DEPENDENCY DOWNLOAD (requires bb.edn or deps.edn)
bb --download                    # Download declared dependencies
```

```clojure
;; ACCESSING DEPENDENCIES IN SCRIPT
(require '[babashka.fs :as fs])
(require '[clojure.data.json :as json])

(println (fs/exists? "file.txt"))
(println (json/write-str {:key "value"}))
```

```powershell
# REPL FEATURES
bb repl                          # Start socket REPL server
bb nrepl-server 1667            # Start nREPL server on port 1667

# CONNECT FROM EDITOR
# VS Code: Calva extension → "Connect to Running REPL" → localhost:1667
```

```clojure
;; UBERJAR/UBERSCRIPT CREATION
;; Create standalone executable script with dependencies bundled
```

```powershell
bb uberjar my-app.jar -m my.app.core  # Create uberjar
bb uberscript my-app.clj -m my.app.core  # Create standalone script

# Run generated artifacts
java -jar my-app.jar
bb my-app.clj
```

```powershell
# USEFUL BABASHKA LIBRARIES (bundled, no deps needed)
# babashka.process    - Shell command execution
# babashka.fs         - Filesystem operations
# babashka.curl       - HTTP client
# cheshire.core       - JSON parsing
# clojure.data.csv    - CSV parsing
# clojure.java.shell  - Shell execution
# clojure.tools.cli   - CLI argument parsing
```

```clojure
;; COMMON TOOLING PATTERNS

;; 1. BUILD AUTOMATION
(require '[babashka.process :refer [shell]])
(shell "pwsh -Command dotnet clean")
(shell "pwsh -Command dotnet build")
(shell "pwsh -Command dotnet test")

;; 2. FILE PROCESSING
(require '[babashka.fs :as fs])
(doseq [f (fs/glob "src" "**.clj")]
  (println "Processing" (str f)))

;; 3. HTTP REQUESTS
(require '[babashka.curl :as curl])
(def response (curl/get "https://api.github.com/repos/babashka/babashka"))
(println (:status response))

;; 4. CLI ARGUMENT PARSING
(require '[clojure.tools.cli :refer [parse-opts]])
(def cli-options [["-p" "--port PORT" "Port number" :default 8080 :parse-fn #(Integer/parseInt %)]])
(def {:keys [options]} (parse-opts *command-line-args* cli-options))
```

```powershell
# ENVIRONMENT DETECTION
bb -e "(System/getProperty \"os.name\")"    # "Windows 11"
bb -e "(System/getenv \"USERNAME\")"        # Current user
bb -e "(System/getProperty \"user.dir\")"   # Current directory
```

```clojure
;; CONDITIONAL WINDOWS LOGIC
(def windows? (re-find #"(?i)windows" (System/getProperty "os.name")))

(if windows?
  (shell "pwsh -Command Get-Process")
  (shell "ps aux"))
```

```powershell
# INTEGRATION WITH POWERSHELL
Get-ChildItem *.txt | ForEach-Object { bb process.clj $_.FullName }

# SCHEDULED TASKS (Windows Task Scheduler)
# Action: pwsh.exe
# Arguments: -File C:\path\to\wrapper.ps1
# wrapper.ps1 contains: bb C:\scripts\task.clj
```

```clojure
;; POD SYSTEM (native library plugins)
;; bb.edn
{:pods {org.babashka/go-sqlite3 {:version "0.1.0"}}}

;; script.clj
(require '[pod.babashka.go-sqlite3 :as sqlite])
(def db (sqlite/open "data.db"))
```

```powershell
# DEBUGGING
bb --verbose script.clj          # Show classpath and loading info
bb --time script.clj             # Show execution time

# ERROR HANDLING - exit codes
bb script.clj
echo $LASTEXITCODE              # Check exit code (0 = success)
```




