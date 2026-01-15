# Table of Contents

- [[#Quick Open (Primary Method)]]
- [[#Go to Symbol in Workspace]]
- [[#File Explorer Navigation]]
- [[#Search Across Files]]
- [[#Breadcrumb Navigation]]
- [[#Go to Definition]]
- [[#Recent Files]]
- [[#Terminal Integration]]
- [[#Calva Namespace Navigation]]
- [[#Advanced: Custom Keybinding for Project File Search]]
- [[#Performance Optimization]]

---

## Quick Open (Primary Method)

[[#Table of Contents|Back to TOC]]

`Ctrl+P` - Fuzzy file search across workspace. Type partial filename. Supports:

- Path segments: `src/core` matches `src/myapp/core.clj`
- Camel case: `uH` matches `userHandler.cljs`
- Wildcards: `*.edn` filters by extension

## Go to Symbol in Workspace

[[#Table of Contents|Back to TOC]]

`Ctrl+T` - Search functions, vars, defns across all files. Queries clj-kondo's symbol index.

Prefix with `@` after `Ctrl+P` for same functionality: `Ctrl+P` then `@symbolName`

## File Explorer Navigation

[[#Table of Contents|Back to TOC]]

`Ctrl+Shift+E` - Focus sidebar Explorer. Arrow keys + `Enter` to open.

Right-click → "Reveal in File Explorer" opens Windows Explorer at file location.

## Search Across Files

[[#Table of Contents|Back to TOC]]

`Ctrl+Shift+F` - Full-text search with regex support.

Options:

- `Alt+C` - Toggle case sensitivity
- `Alt+W` - Match whole word
- `Alt+R` - Enable regex

Filters: Click "files to include/exclude" for glob patterns:

```
Include: **/*.{clj,cljs,cljc}
Exclude: **/target/**, **/node_modules/**
```

## Breadcrumb Navigation

[[#Table of Contents|Back to TOC]]

File path breadcrumbs at editor top. Click segments for dropdown of siblings. Navigate hierarchy without leaving editor.

Toggle: `Ctrl+Shift+.`

## Go to Definition

[[#Table of Contents|Back to TOC]]

`F12` - Jump to definition of symbol under cursor. Works for:

- Clojure/ClojureScript: `defn`, `def`, namespaced symbols
- Cross-file navigation via Calva LSP

`Alt+F12` - Peek definition inline without navigation

`Shift+F12` - Find all references to symbol

## Recent Files

[[#Table of Contents|Back to TOC]]

`Ctrl+R` - Recent files dropdown (workspace context)

`Ctrl+Tab` - Cycle through editor tab history (MRU order)

## Terminal Integration

[[#Table of Contents|Back to TOC]]

```bash
# Open file from integrated terminal (Ctrl+`)
code filename.clj
```

## Calva Namespace Navigation

[[#Table of Contents|Back to TOC]]

`Ctrl+Alt+C N` followed by namespace selection - Load and navigate to namespace file.

## Advanced: Custom Keybinding for Project File Search

[[#Table of Contents|Back to TOC]]

`keybindings.json`:

```json
{
  "key": "ctrl+shift+t",
  "command": "workbench.action.quickOpen",
  "args": "**/*.{clj,cljs,cljc,edn}"
}
```

## Performance Optimization

[[#Table of Contents|Back to TOC]]

Exclude build artifacts from indexing in `settings.json`:

```json
{
  "search.exclude": {
    "**/target/**": true,
    "**/node_modules/**": true,
    "**/.shadow-cljs/**": true,
    "**/.cpcache/**": true,
    "**/.calva/**": true
  },
  "files.watcherExclude": {
    "**/target/**": true,
    "**/.shadow-cljs/**": true
  }
}
```

Reduces filesystem watchers, improves Quick Open indexing latency.