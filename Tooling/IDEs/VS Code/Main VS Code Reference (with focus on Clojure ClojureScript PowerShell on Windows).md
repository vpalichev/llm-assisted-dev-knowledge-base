# Core Terminology

**Workspace**: Project-scoped configuration container. Persists as `.code-workspace` file or `.vscode/` directory. Settings override user-level configuration.

**Extension**: Runtime plugin providing language servers, debuggers, or tooling integration.

**Command Palette**: Command interface accessing all registered actions. Primary navigation mechanism for non-memorized operations.

**Integrated Terminal**: Embedded terminal multiplexer. Supports PowerShell, CMD, WSL shells with configurable profiles.

**IntelliSense**: Language server protocol (LSP) driven completion, signature help, hover information, symbol navigation.

**Settings Sync**: Cross-machine synchronization of configuration state via OAuth-authenticated storage backend.

**Launch Configuration**: Debugger adapter protocol (DAP) configuration manifest in `launch.json`.

**Tasks**: Build automation primitives defined in `tasks.json`. Supports shell commands, problem matchers, dependency graphs.

## Configuration Files

### File Hierarchy
```
%APPDATA%\Code\User\
├── settings.json          # User-global
├── keybindings.json       # User-global
└── snippets/              # User-global snippets

<workspace>/
└── .vscode/
    ├── settings.json      # Workspace-scoped, overrides user
    ├── launch.json        # Debugger configurations
    ├── tasks.json         # Task definitions
    └── *.code-snippets    # Workspace snippets
```

### settings.json: Clojure/ClojureScript (Calva)
```json
{
  "calva.prettyPrintingOptions": {
    "enabled": true,
    "width": 80,
    "maxLength": 50
  },
  "calva.autoConnect": true,
  "calva.jackInEnv": {
    "JAVA_HOME": "C:\\Program Files\\Java\\jdk-17"
  },
  "calva.replConnectSequences": [
    {
      "name": "deps.edn",
      "projectType": "deps.edn",
      "cljsType": "shadow-cljs",
      "afterCLJReplJackInCode": "(require '[clojure.tools.namespace.repl :refer [refresh]])"
    }
  ],
  "calva.paredit.defaultKeyMap": "original",
  "calva.fmt.configPath": ".cljfmt.edn",
  "[clojure]": {
    "editor.defaultFormatter": "betterthantomorrow.calva",
    "editor.formatOnSave": true,
    "editor.parameterHints.enabled": true
  },
  "[clojurescript]": {
    "editor.defaultFormatter": "betterthantomorrow.calva",
    "editor.formatOnSave": true
  }
}
```

### settings.json: PowerShell
```json
{
  "powershell.codeFormatting.preset": "OTBS",
  "powershell.codeFormatting.whitespaceAroundPipe": true,
  "powershell.scriptAnalysis.enable": true,
  "powershell.scriptAnalysis.settingsPath": ".vscode/PSScriptAnalyzerSettings.psd1",
  "powershell.integratedConsole.focusConsoleOnExecute": false,
  "powershell.powerShellDefaultVersion": "PowerShell (x64)",
  "[powershell]": {
    "editor.defaultFormatter": "ms-vscode.powershell",
    "editor.formatOnSave": true,
    "editor.tabSize": 4,
    "editor.insertSpaces": true,
    "files.encoding": "utf8bom"
  }
}
```

### keybindings.json

Custom key mappings override defaults. Context-aware via `when` clauses.
```json
[
  {
    "key": "ctrl+shift+e",
    "command": "calva.evaluateSelection",
    "when": "editorTextFocus && editorLangId =~ /clojure/"
  },
  {
    "key": "ctrl+alt+r",
    "command": "calva.loadFile",
    "when": "editorLangId =~ /clojure/"
  }
]
```

### launch.json: PowerShell DAP Configuration
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "PowerShell",
      "request": "launch",
      "name": "PS: Launch Current File",
      "script": "${file}",
      "cwd": "${workspaceFolder}"
    },
    {
      "type": "PowerShell",
      "request": "attach",
      "name": "PS: Attach to Host Process",
      "processId": "${command:PickPSHostProcess}",
      "runspaceId": 1
    }
  ]
}
```

## Essential Extensions

### Clojure/ClojureScript Stack

**Calva** (`betterthantomorrow.calva`)  
nREPL client with paredit, REPL evaluation, inline results, debugger integration, LSP server.

**clj-kondo** (`borkdude.clj-kondo-lsp`)  
Static analyzer providing real-time linting, unused binding detection, arity validation.

**Rainbow Brackets** (`2gua.rainbow-brackets`)  
Hierarchical bracket colorization. Essential for navigating deeply nested s-expressions.

### PowerShell

**PowerShell** (`ms-vscode.powershell`)  
Official language server with PSScriptAnalyzer integration, integrated console, DAP debugger.

## Keyboard Shortcuts (Windows)

### Universal VS Code
```
Ctrl+Shift+P        Command Palette (all commands)
Ctrl+P              Quick Open (file/symbol navigation)
Ctrl+B              Toggle sidebar visibility
Ctrl+`              Toggle integrated terminal
Ctrl+Tab            Cycle open editors
Ctrl+K Ctrl+S       Keyboard shortcuts editor
Ctrl+,              Settings UI
F12                 Go to definition
Alt+F12             Peek definition
Shift+F12           Find all references
Ctrl+D              Multi-cursor: add next match
Ctrl+Shift+L        Multi-cursor: select all matches
Alt+Up/Down         Move line
Shift+Alt+Up/Down   Duplicate line
Ctrl+/              Toggle line comment
Ctrl+K Ctrl+C       Add line comment
Ctrl+K Ctrl+U       Remove line comment
```

### Calva (Clojure/ClojureScript)
```
Ctrl+Alt+C Ctrl+Alt+J       Jack-in (start nREPL)
Ctrl+Alt+C Ctrl+Alt+C       Connect to running nREPL
Ctrl+Enter                  Evaluate current form
Ctrl+Alt+C Space            Evaluate top-level form
Ctrl+Alt+C Ctrl+Alt+Space   Evaluate enclosing form
Ctrl+Alt+C E                Load current file
Ctrl+Alt+C N                Load namespace
Ctrl+Alt+C T                Run test at cursor
Ctrl+Alt+C Ctrl+Alt+T       Run all tests in namespace
Ctrl+Alt+C Shift+T          Run all tests in project
Ctrl+Alt+.                  Go to definition
Ctrl+Alt+C D                Show documentation
Ctrl+Alt+C L H              Toggle REPL history

Paredit (Structural Editing):
Ctrl+Right                  Slurp forward
Ctrl+Left                   Barf forward
Ctrl+Alt+Right              Slurp backward
Ctrl+Alt+Left               Barf backward
Ctrl+Shift+S                Splice sexp
Ctrl+Alt+S                  Split sexp
Ctrl+Alt+Up                 Raise sexp
Ctrl+Shift+Delete           Kill sexp forward
Ctrl+Shift+Backspace        Kill sexp backward
```

### PowerShell Extension
```
F5                  Run script/file
F8                  Run selection
Shift+F5            Stop execution
F9                  Toggle breakpoint
Ctrl+Shift+F9       Remove all breakpoints
F10                 Step over
F11                 Step into
Shift+F11           Step out
F2                  Rename symbol
Shift+Alt+F         Format document
Ctrl+K Ctrl+F       Format selection
Ctrl+Space          Trigger IntelliSense
Ctrl+Shift+Space    Trigger parameter hints
```

## Clojure/ClojureScript Environment Setup

### Prerequisites

**JDK 11+**: Required for Clojure runtime.
```powershell
# Verify installation
java -version
# Set JAVA_HOME if not configured
[System.Environment]::SetEnvironmentVariable('JAVA_HOME', 'C:\Program Files\Java\jdk-17', 'Machine')
```

**Clojure CLI**: Official deps.edn build tool.
```powershell
# Install via scoop
scoop install clojure

# Verify
clj --version
```

**Shadow-cljs**: ClojureScript build tool with npm integration.
```powershell
npm install -g shadow-cljs
```

### Project Initialization

**deps.edn Project**:
```powershell
clj -Tnew app :name com.domain/project-name
```

**shadow-cljs Project**:
```powershell
npx create-cljs-project project-name
```

### nREPL Connection

**Jack-in**: Calva spawns nREPL server automatically. Reads project type from `deps.edn`, `shadow-cljs.edn`, `project.clj`.

**Connect to Running REPL**: For external nREPL processes:
1. Start nREPL: `clj -Sdeps '{:deps {nrepl/nrepl {:mvn/version "1.0.0"}}}' -M -m nrepl.cmdline`
2. Calva → Connect to Running REPL Server
3. Specify `localhost:<port>` from nREPL output

**shadow-cljs REPL**: Jack-in automatically detects shadow-cljs configuration. Starts watch process and connects to build-specific nREPL.

## PowerShell Environment Setup

### Runtime Selection

**Windows PowerShell 5.1**: Pre-installed. Limited to Windows. Legacy compatibility.

**PowerShell 7+**: Cross-platform, modern syntax, performance improvements.
```powershell
# Install via winget
winget install --id Microsoft.PowerShell --source winget

# Verify
pwsh -Version
```

### Extension Configuration

**Default Version**: Command Palette → "PowerShell: Show Session Menu" → Select version.

**Execution Policy**:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### PSScriptAnalyzer Configuration

`.vscode/PSScriptAnalyzerSettings.psd1`:
```powershell
@{
    IncludeRules = @(
        'PSAvoidUsingCmdletAliases',
        'PSUseApprovedVerbs',
        'PSAvoidUsingWriteHost'
    )
    ExcludeRules = @(
        'PSAvoidUsingPositionalParameters'
    )
    Rules = @{
        PSProvideCommentHelp = @{
            Enable = $true
            ExportedOnly = $false
            Placement = 'before'
        }
        PSUseCompatibleSyntax = @{
            Enable = $true
            TargetVersions = @('7.0', '5.1')
        }
    }
}
```

## Advanced Development Configuration

### Linting

**clj-kondo Configuration**: `.clj-kondo/config.edn` in project root.
```clojure
{:linters {:unresolved-symbol {:level :warning
                                :exclude [(user)]}
           :unused-binding {:level :warning}
           :shadowed-var {:level :warning}}
 :lint-as {clojure.test.check.properties/for-all clojure.core/let
           clojure.core.async/go-loop clojure.core/loop}}
```

Auto-generated analysis cache: `.clj-kondo/.cache/`. Commit to version control for consistent linting across environments.

### Formatting

**Calva cljfmt**: `.cljfmt.edn` configuration:
```clojure
{:indents {defroutes [[:inner 0]]
           GET [[:inner 0]]
           POST [[:inner 0]]
           PUT [[:inner 0]]
           DELETE [[:inner 0]]}
 :remove-consecutive-blank-lines? true
 :insert-missing-whitespace? true
 :remove-surrounding-whitespace? true
 :remove-trailing-whitespace? true
 :indentation? true
 :align-associative? false}
```

**PowerShell Formatting**: Handled by PowerShell extension using PSScriptAnalyzer rules. Presets: `OTBS`, `Allman`, `Stroustrup`.

### Snippets

**Workspace Snippets**: `.vscode/*.code-snippets`

**Clojure Example**:
```json
{
  "defn with spec": {
    "prefix": "defns",
    "body": [
      "(s/fdef ${1:function-name}",
      "  :args (s/cat ${2::arg-spec})",
      "  :ret ${3:ret-spec})",
      "",
      "(defn ${1:function-name}",
      "  \"${4:docstring}\"",
      "  [${5:args}]",
      "  ${0:body})"
    ]
  }
}
```

**PowerShell Example**:
```json
{
  "Advanced Function": {
    "prefix": "funcadv",
    "body": [
      "function ${1:Verb-Noun} {",
      "    [CmdletBinding(SupportsShouldProcess=\\$true)]",
      "    [OutputType([${2:Type}])]",
      "    param (",
      "        [Parameter(Mandatory=\\$true, ValueFromPipeline=\\$true)]",
      "        [ValidateNotNullOrEmpty()]",
      "        [${3:string}]\\$${4:ParameterName}",
      "    )",
      "    begin { ${5:# Initialization} }",
      "    process { ${6:# Per-item processing} }",
      "    end { ${0:# Cleanup} }",
      "}"
    ]
  }
}
```

### Debugger Configuration

**Calva Debugging**: Uses `cider-nrepl` middleware. Insert `#break` in code or instrument via Command Palette.

Enable in `deps.edn`:
```clojure
{:aliases
 {:dev {:extra-deps {cider/cider-nrepl {:mvn/version "0.42.1"}}
        :main-opts ["-m" "nrepl.cmdline" "--middleware" "[cider.nrepl/cider-middleware]"]}}}
```

**Conditional Breakpoints**: Right-click gutter → Add Conditional Breakpoint → Enter expression.

PowerShell example: `$item.Status -eq 'Failed'`

### Terminal Profiles
```json
{
  "terminal.integrated.profiles.windows": {
    "PowerShell 7": {
      "path": "C:\\Program Files\\PowerShell\\7\\pwsh.exe",
      "args": ["-NoLogo"],
      "icon": "terminal-powershell"
    },
    "Windows PowerShell": {
      "path": "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
      "icon": "terminal-powershell"
    },
    "Clojure REPL": {
      "path": "C:\\Windows\\System32\\cmd.exe",
      "args": ["/k", "clj"],
      "icon": "terminal"
    }
  },
  "terminal.integrated.defaultProfile.windows": "PowerShell 7"
}
```

### Multi-root Workspaces

Monorepo or polyglot project support. File → Add Folder to Workspace. Save as `.code-workspace`:
```json
{
  "folders": [
    { "path": "backend" },
    { "path": "frontend" }
  ],
  "settings": {
    "calva.replConnectSequences": [
      {
        "name": "Backend",
        "projectType": "deps.edn",
        "projectRootPath": ["backend"]
      },
      {
        "name": "Frontend",
        "projectType": "shadow-cljs",
        "projectRootPath": ["frontend"]
      }
    ]
  }
}
```

### Settings Sync Exclusions

Exclude environment-specific settings:
```json
{
  "settingsSync.ignoredSettings": [
    "calva.jackInEnv",
    "powershell.powerShellDefaultVersion",
    "terminal.integrated.profiles.windows"
  ]
}
```

### Workspace Trust

Restricts extension execution in untrusted workspaces. Explicitly trust: Command Palette → "Manage Workspace Trust".

Configuration: `security.workspace.trust.enabled`

### REPL History Persistence

Calva stores REPL history in `.calva/repl-history/`. Per-session persistence with search. Toggle: `Ctrl+Alt+C L H`.

### LSP Performance Tuning

**clj-kondo LSP**:
```json
{
  "clj-kondo.lspServerPath": "C:\\path\\to\\clj-kondo.exe",
  "clj-kondo.lintOnChange": true
}
```

**PowerShell IntelliSense Cache**:
```json
{
  "powershell.integratedConsole.suppressStartupBanner": true,
  "powershell.integratedConsole.startInBackground": false,
  "powershell.developer.editorServicesLogLevel": "Error"
}
```

### Problem Matchers

Extract build errors for Problems panel. `tasks.json`:
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "shadow-cljs watch",
      "type": "shell",
      "command": "npx",
      "args": ["shadow-cljs", "watch", "app"],
      "problemMatcher": {
        "owner": "clojurescript",
        "fileLocation": "absolute",
        "pattern": {
          "regexp": "^--+\\s+(.*)\\s+--+$",
          "file": 1,
          "message": 0
        }
      },
      "isBackground": true
    }
  ]
}
```

### Extension Recommendations

`.vscode/extensions.json`:
```json
{
  "recommendations": [
    "betterthantomorrow.calva",
    "borkdude.clj-kondo-lsp",
    "2gua.rainbow-brackets",
    "ms-vscode.powershell"
  ]
}
```

Prompts collaborators to install required extensions on workspace open.