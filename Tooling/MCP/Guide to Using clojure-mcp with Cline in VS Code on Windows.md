### Guide to Using clojure-mcp with Cline in VS Code on Windows

**clojure-mcp** is a powerful Model Context Protocol (MCP) server for Clojure development. It connects an AI agent (via Cline) to a Clojure nREPL, providing specialized tools for REPL interaction, syntax-aware editing (e.g., automatic parenthesis fixing, linting, formatting), file operations, and more. This enables AI-assisted Clojure coding with high edit acceptance rates and interactive evaluation.

**Cline** is a VS Code extension (originally called Claude Dev) that acts as an autonomous AI coding agent, supporting MCP servers for enhanced tooling.

These steps are tailored for **Windows**.

#### Prerequisites

1. Install **Java JDK 17 or later** (e.g., from Adoptium or Oracle).
2. Install **Clojure CLI** (follow official instructions: download the installer from [https://clojure.org/guides/install_clojure](https://clojure.org/guides/install_clojure?referrer=grok.com) and run it).
3. Install **VS Code**.
4. Install the **Cline** extension:
    - Open VS Code → Extensions → Search for "Cline" (publisher: saoudrizwan) → Install.
5. Configure an API key in Cline settings (e.g., Anthropic, OpenAI, or OpenRouter for Claude models).

#### Step 1: Set Up a Clojure Project with nREPL

Create a new project (if you don't have one):

text

```
clojure -Tclj-new app :name myproject/myapp
cd myapp
```

Add an nREPL alias to your deps.edn (in the project root):

edn

`{:aliases {:nrepl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.1"}} :main-opts ["-m" "nrepl.cmdline" "--port" "7888"]}}}`

Start the nREPL server in a terminal (from project directory):

text

```
clojure -M:nrepl
```

- It should listen on port 7888 (leave this terminal running).

#### Step 2: Install clojure-mcp

Add the alias to your global ~/.clojure/deps.edn (create if missing):

edn

`{:aliases {:mcp {:deps {org.slf4j/slf4j-nop {:mvn/version "2.0.16"} com.bhauman/clojure-mcp {:git/url "https://github.com/bhauman/clojure-mcp.git" :git/tag "v0.2.0" :git/sha "978d3f1"}} :exec-fn clojure-mcp.main/start}}}`

(Optional) Install **ripgrep** (rg.exe) for better file search performance: Download from [https://github.com/BurntSushi/ripgrep/releases](https://github.com/BurntSushi/ripgrep/releases?referrer=grok.com) and add to PATH.

#### Step 3: Configure clojure-mcp in Cline

Cline stores MCP settings in a JSON file on Windows: %APPDATA%\Code\User\globalStorage\saoudrizwan.claude-dev\settings\cline_mcp_settings.json

(Or for VS Code Insiders: replace Code with Code - Insiders.)

1. Create the folder/file if it doesn't exist.
2. Edit cline_mcp_settings.json (open in VS Code):

JSON

```
{
  "mcpServers": {
    "clojure-mcp": {
      "command": "cmd",
      "args": [
        "/c",
        "clojure -X:mcp :port 7888"
      ]
    }
  }
}
```

- This uses Windows cmd.exe to run the clojure-mcp command.
- Adjust if your project needs :not-cwd true (add :not-cwd true to args: ":not-cwd", "true").
- If clojure isn't in PATH globally, use full path (e.g., "C:\\Users\\YourUser\\AppData\\Local\\bin\\clojure.exe").

#### Step 4: Start Using in VS Code

1. Restart VS Code.
2. Open your Clojure project folder in VS Code.
3. Open the Cline sidebar (icon on left, or Ctrl+Shift+P → "Cline: Open").
4. Start a new chat/task.
5. Cline will auto-launch the MCP server when needed and connect to your running nREPL.

(Optional project config): Create .clojure-mcp/config.edn in project root for safety (e.g., restrict file writes):

edn

`{:allowed-directories ["." "src" "test" "resources"] :write-file-guard :full-read}`

#### Verification

1. In Cline chat, ask something like: "What namespaces are loaded?" or "Evaluate (+ 1 2) in the REPL."
    - Success: Cline uses clojure-mcp tools (e.g., eval_form, list_namespaces) and returns results.
2. Check Cline's system prompt (visible in chat or logs) for "Connected MCP Servers" mentioning clojure-mcp.
3. Test editing: Ask to "Add a function to src/myapp/core.clj" – it should apply syntax-aware patches without paren issues.
4. If errors:
    - Check terminal where nREPL runs for connection.
    - Check clojure-mcp output (Cline launches it – watch task logs).
    - Ensure port 7888 is free and no firewall blocks localhost.
    - In Cline MCP settings (sidebar → MCP icon), toggle/check status.

This setup turns Cline into a powerful Clojure-aware AI assistant with live REPL access. For advanced features (e.g., bash tools), ensure your environment is configured properly. Enjoy REPL-driven AI development!