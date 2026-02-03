
Add to your VS Code `keybindings.json` (`Ctrl+Shift+P` → "Open Keyboard Shortcuts (JSON)"):
{
  "key": "ctrl+shift+v",
  "command": "editor.action.insertSnippet",
  "args": {
    "snippet": "${CLIPBOARD/\\\\/\\//g}"
  },
  "when": "editorTextFocus"
}
## VS Code Snippet Transformation: Technical Breakdown

### Snippet Variable Transformation Syntax

VS Code snippets support a transformation syntax applied to variables:

```
${VARIABLE/PATTERN/REPLACEMENT/FLAGS}
```

|Component|Description|
|---|---|
|`VARIABLE`|Built-in variable (e.g., `CLIPBOARD`, `TM_SELECTED_TEXT`, `TM_FILENAME`)|
|`PATTERN`|JavaScript regular expression|
|`REPLACEMENT`|Replacement string (supports backreferences `$1`, `$2`, etc.)|
|`FLAGS`|Regex flags: `g` (global), `i` (case-insensitive), etc.|

The delimiter is `/`. This is significant for escaping.

---

### Escaping Layers in This Snippet

The value passes through **two parsing stages**:

```
JSON Parser → VS Code Snippet Parser
```

#### Layer 1: JSON String Parsing

JSON uses `\` as an escape character. The string:

```json
"${CLIPBOARD/\\\\/\\//g}"
```

After JSON parsing becomes:

```
${CLIPBOARD/\\/\//g}
```

|JSON Sequence|Parsed Result|Reason|
|---|---|---|
|`\\\\`|`\\`|Each `\\` → single `\`|
|`\\/`|`\/`|`\\` → `\`, then literal `/`|
|`/g`|`/g`|No escaping needed|

#### Layer 2: VS Code Snippet Transformation Parsing

The snippet parser receives:

```
${CLIPBOARD/\\/\//g}
```

Now parsed as transformation syntax `${VAR/pattern/replacement/flags}`:

|Component|Raw Value|Interpretation|
|---|---|---|
|Variable|`CLIPBOARD`|Contents of system clipboard|
|Pattern|`\\`|Regex: literal backslash (`\` escaped as `\\`)|
|Replacement|`\/`|Literal `/` (escaped because `/` is the delimiter)|
|Flags|`g`|Global: replace all occurrences|

---

### Why Each Escape Is Necessary

**Pattern (`\\\\` in JSON → `\\` in regex):**

```
Backslash in regex requires escaping: \  →  \\
Backslash in JSON requires escaping:   \\  →  \\\\
```

**Replacement (`\\/` in JSON → `\/` in snippet → `/` literal):**

```
Forward slash is the delimiter in snippet syntax
To insert literal /, escape it: \/
In JSON: \/ becomes \\/ 
```

---

### Execution Flow

```
1. User copies: C:\Users\data\file.txt
2. User presses: Ctrl+Shift+V
3. VS Code reads CLIPBOARD: "C:\Users\data\file.txt"
4. Regex /\\/g matches each \
5. Each \ replaced with /
6. Result inserted: "C:/Users/data/file.txt"
```

---

### Escaping Summary Table

|Goal|Regex|Snippet Syntax|JSON String|
|---|---|---|---|
|Match `\`|`\\`|`\\`|`\\\\`|
|Replace with `/`|`/`|`\/`|`\\/`|

Total: `${CLIPBOARD/\\\\/\\//g}`






## VS Code Keybinding Configuration: Line-by-Line Reference

```json
{
  "key": "ctrl+shift+v",
  "command": "editor.action.insertSnippet",
  "args": {
    "snippet": "${CLIPBOARD/\\\\/\\//g}"
  },
  "when": "editorTextFocus"
}
```

---

### `"key": "ctrl+shift+v"`

**Property:** `key`

**Purpose:** Defines the keyboard shortcut that triggers this binding.

**Value:** `ctrl+shift+v` — simultaneous press of Control, Shift, and V keys.

**Note:** This overrides the default `Ctrl+Shift+V` behavior in VS Code (which is "Paste without formatting" in some contexts). Choose a different combination if preserving that functionality is desired.

---

### `"command": "editor.action.insertSnippet"`

**Property:** `command`

**Purpose:** Specifies which VS Code command executes when the keybinding activates.

**Value:** `editor.action.insertSnippet` — a built-in command that inserts a snippet at the current cursor position. Unlike static text insertion, this command supports snippet syntax including variables, placeholders, and transformations.

---

### `"args": { ... }`

**Property:** `args`

**Purpose:** Passes arguments to the command. Structure and accepted properties vary per command.

**For `insertSnippet`:** Accepts a `snippet` property containing the snippet body as a string.

---

### `"snippet": "${CLIPBOARD/\\\\/\\//g}"`

**Property:** `snippet` (nested under `args`)

**Purpose:** Defines the snippet content to insert.

**Value breakdown:**

|Component|Description|
|---|---|
|`${CLIPBOARD}`|Built-in snippet variable. Resolves to current system clipboard contents at execution time.|
|`/.../.../.../`|Transformation syntax. Applies a regex find-replace operation to the variable's value.|
|`\\\\/`|JSON-escaped regex pattern. After JSON parsing: `\\`. Matches a single backslash character.|
|`\\//`|JSON-escaped replacement string. After JSON parsing: `\/`. Inserts a literal forward slash (escaped because `/` delimits transformation components).|
|`g`|Regex flag. Global matching — replaces all occurrences, not just the first.|

**Transformation syntax template:**

```
${VARIABLE/PATTERN/REPLACEMENT/FLAGS}
```

---

### `"when": "editorTextFocus"`

**Property:** `when`

**Purpose:** Conditional expression that restricts when this keybinding is active. VS Code evaluates this context expression and only triggers the binding if it returns true.

**Value:** `editorTextFocus` — a built-in context key that evaluates to `true` when:

- A text editor is open
- The editor has keyboard focus (cursor is active in the editing area)

**Effect:** Prevents the keybinding from triggering when focus is on the sidebar, terminal, command palette, or other non-editor UI elements.

---

### Execution Summary

|Step|Action|
|---|---|
|1|User presses `Ctrl+Shift+V`|
|2|VS Code checks `when` clause: is a text editor focused?|
|3|If true, executes `editor.action.insertSnippet`|
|4|Command reads `snippet` argument|
|5|`${CLIPBOARD}` resolves to clipboard contents|
|6|Transformation replaces all `\` with `/`|
|7|Result inserts at cursor position|