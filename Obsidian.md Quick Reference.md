#### Notes support standard Markdown syntax plus Obsidian-specific extensions for internal linking and embedding.


How to add link to another file? 

**Type `[[` and start typing the note name** - autocomplete appears, press `Enter` to select.

**Or:**

1. Type or select text
2. Press `Ctrl+K`
3. Start typing note name, select from list

**Result**: `[[Note Name]]` or `[[Note Name|selected text]]`

**Type `[[Note Name#` then start typing the heading name** - autocomplete shows available headings.

**Syntax**: `[[Note Name#Heading Title]]`

**Example**: `[[Project Plan#Timeline]]`

Works with aliases too: `[[Note Name#Heading|display text]]`
# Obsidian.md Technical Reference Guide for Windows

## Core Architecture

**Obsidian** is a local-first, Markdown-based knowledge management application that operates on plain text files stored in the file system. The application implements a vault-based architecture where all notes reside within a designated directory structure.

**Vault**: The root directory containing all Markdown files (.md), configuration data (.obsidian subdirectory), and associated assets. Each vault maintains independent settings, plugins, and graph databases. The vault operates as a self-contained knowledge base with no mandatory cloud synchronization.

**Note**: A discrete Markdown file (.md extension) representing a single document within the vault. Notes support standard Markdown syntax plus Obsidian-specific extensions for internal linking and embedding.

**Workspace**: The application's UI state, including open panes, sidebar visibility, and active view configurations. Workspaces can be saved and restored through Core Plugins.

## Fundamental Linking Architecture

**Wikilink**: Obsidian's primary internal linking syntax, denoted by double brackets `[[Note Name]]`. The application resolves these references by searching for files matching the enclosed string within the vault, independent of directory location. Wikilinks function bidirectionally—linked notes automatically register backlinks.

**Backlink**: An automatically generated reverse reference. When Note A contains `[[Note B]]`, Note B's backlink pane displays Note A as a source reference. The backlink pane aggregates all notes referencing the current document.

**Alias**: An alternative display name for a wikilink, specified using pipe notation: `[[Actual Note Name|Display Text]]`. The link resolves to the actual note while rendering the display text in reading mode.

**Heading Link**: A wikilink targeting a specific section within a note, using hash notation: `[[Note Name#Heading]]`. Obsidian supports linking to any Markdown heading level (H1-H6).

**Block Reference**: A unique identifier assigned to individual paragraphs, list items, or code blocks using the caret notation: `^block-id`. These enable precise linking to specific content segments via `[[Note Name#^block-id]]`.

## Linking Implementation Strategies

**Hub-and-Spoke Topology**: Centralized notes (hubs) serve as index points linking to related content (spokes). Maps of Content (MOCs) implement this pattern by maintaining curated link collections organized by topic domain.

**Zettelkasten Method**: Atomic note architecture where each note contains a single, self-contained concept. Notes link to related concepts bidirectionally, forming an emergent knowledge network. The method emphasizes creating permanent notes with unique identifiers and explicit connection reasoning.

**Progressive Summarization**: Hierarchical note refinement through sequential highlighting and summarization layers. Original content remains intact while successive abstraction levels are added through formatting and internal linking to derivative notes.

**Tag Taxonomy vs. Link Structure**: Tags (`#tag-name`) provide flat categorical classification, while links create relational structure. Tags support hierarchical nesting using forward slashes: `#category/subcategory/topic`. The distinction: tags classify content type, links establish semantic relationships.

## Windows Keyboard Shortcuts

### Navigation and Search

- `Ctrl+O`: Quick Switcher (file navigation by name)
- `Ctrl+Shift+F`: Global search across all vault files
- `Ctrl+P`: Command Palette (function and command execution)
- `Ctrl+G`: Open graph view visualization
- `Alt+Left Arrow`: Navigate backward in history
- `Alt+Right Arrow`: Navigate forward in history
- `Ctrl+Tab`: Cycle through open tabs forward
- `Ctrl+Shift+Tab`: Cycle through open tabs backward

### Editing Operations

- `Ctrl+E`: Toggle between edit and reading mode
- `Ctrl+B`: Apply bold formatting (`**text**`)
- `Ctrl+I`: Apply italic formatting (`*text*`)
- `Ctrl+K`: Insert Markdown link syntax
- `Ctrl+]`: Indent line or selection
- `Ctrl+[`: Unindent line or selection
- `Ctrl+D`: Delete current line
- `Ctrl+Shift+D`: Toggle developer tools console

### Window and Pane Management

- `Ctrl+N`: Create new note in vault
- `Ctrl+W`: Close active pane
- `Ctrl+Shift+N`: Create new vault
- `Ctrl+\`: Toggle left sidebar visibility
- `Ctrl+Alt+Left Arrow`: Split current pane leftward
- `Ctrl+Alt+Right Arrow`: Split current pane rightward

### Advanced Functions

- `Ctrl+F`: In-note search (find within current document)
- `Ctrl+H`: In-note search and replace
- `Ctrl+Shift+I`: Open developer console (Electron framework access)
- `Ctrl+,`: Open Settings panel

## Critical Concepts

**Graph View**: A force-directed visualization rendering notes as nodes and wikilinks as edges. The local graph displays connections for the active note within a configurable depth radius. The global graph renders the entire vault's link structure. Color groups and filters can be applied via CSS selectors or search queries.

**Dataview**: A community plugin enabling SQL-like queries against note metadata. Inline fields (key-value pairs in note content) and YAML frontmatter serve as queryable data. Example syntax: `TABLE field1, field2 FROM #tag WHERE condition`.

**Frontmatter**: YAML-formatted metadata block positioned at note beginning, delimited by triple hyphens (`---`). Supports key-value pairs for tags, aliases, creation dates, and custom properties. Parsed as structured data accessible to plugins and templates.

**Template**: A reusable note structure with placeholder variables. Core Templates plugin supports variable substitution (date, time, title). Community plugin Templater extends functionality with JavaScript evaluation and dynamic content generation.

**Embeds**: Inline content inclusion using exclamation prefix: `![[Note Name]]` embeds entire note content, `![[Note#Heading]]` embeds specific section. Block embeds use `![[Note#^block-id]]`. Image embeds: `![[image.png]]`.

## File System Interaction

Obsidian maintains direct file system access—notes remain human-readable Markdown files editable by external applications. The `.obsidian` directory stores configuration JSON files, plugin data, and workspace states. This directory can be excluded from version control while preserving vault portability.

**Attachment Handling**: Binary files (images, PDFs, audio) can be stored within vault directory structure. Settings allow configuration of default attachment location. Links use Markdown image syntax or wikilink format with file extension: `![[diagram.png]]` or `![](attachments/diagram.png)`.

**Vault Portability**: Vaults transfer between systems by copying directory contents. Obsidian resolves relative paths, ensuring cross-platform compatibility. Windows-specific path considerations: backslash separators in file system are abstracted by the application's path resolver.

## Configuration Architecture

**Settings Hierarchy**: Application-level settings (appearance, hotkeys) apply globally. Vault-specific settings (.obsidian/config) persist per-vault. Core plugins can be toggled independently per vault. Community plugins require manual installation into `.obsidian/plugins/` directory structure.

**CSS Snippets**: Custom styling rules stored in `.obsidian/snippets/` as CSS files. Enable granular UI modifications without theme replacement. Snippets activate/deactivate via Appearance settings without requiring application restart.

**Hotkey Rebinding**: All functions support custom key mapping through Settings > Hotkeys. Conflicts display warnings. Search function filters available commands. Some system-level Windows shortcuts (e.g., `Alt+F4`) cannot be overridden.

## Windows-Specific Considerations

**Command Line Launch**: The installed executable typically resides in `%LocalAppData%\Obsidian\Obsidian.exe`. Vault opening via command line: `obsidian.exe obsidian://open?vault=VaultName`. URI protocol handlers enable external application integration.

**File Associations**: Windows file associations can be configured to open .md files with Obsidian. However, this may conflict with other Markdown editors. The application does not claim exclusive .md file association by default.

**Path Length Limitations**: Windows historically enforces 260-character path limits (MAX_PATH). While Windows 10+ supports extended paths, some operations may encounter issues with deeply nested vault structures. Recommendation: maintain vault at reasonable directory depth.

This reference provides foundational understanding of Obsidian's technical architecture and operational patterns on Windows systems. Advanced usage involves plugin API interaction, custom CSS development, and programmatic vault manipulation through external scripting.


#  Quick Reference

## Linking Syntax

**Basic link**: `[[Note Name]]`  
**Link with alias**: `[[Note Name|Display Text]]`  
**Link to heading**: `[[Note#Heading]]`  
**Link to block**: `[[Note#^block-id]]` (create block ID by adding `^block-id` after paragraph)  
**Embed note**: `![[Note Name]]`  
**Embed section**: `![[Note#Heading]]`  
**Embed block**: `![[Note#^block-id]]`

**Tags**: `#tag` or `#category/subcategory`

## Essential Shortcuts (Windows)

**Navigation**  
`Ctrl+O` - Open file by name  
`Ctrl+Shift+F` - Search all files  
`Ctrl+P` - Command palette  
`Ctrl+G` - Graph view  
`Alt+←/→` - Back/forward

**Editing**  
`Ctrl+E` - Toggle edit/preview  
`Ctrl+K` - Insert link  
`Ctrl+B` - Bold  
`Ctrl+I` - Italic  
`Ctrl+D` - Delete line  
`Ctrl+]/[` - Indent/unindent

**Files & Panes**  
`Ctrl+N` - New note  
`Ctrl+W` - Close pane  
`Ctrl+\` - Toggle sidebar  
`Ctrl+Alt+←/→` - Split pane

**Search**  
`Ctrl+F` - Find in current note  
`Ctrl+H` - Find and replace

## Linking Strategies

**MOC (Map of Content)**: Index note with organized links to topic cluster  
**Zettelkasten**: One idea per note, link related concepts bidirectionally  
**Tags vs Links**: Tags classify, links create relationships

**Backlinks panel** shows all notes linking to current note (automatic).


Gotchas:
1 Windows paths -- backslashes should be doubled
2 
