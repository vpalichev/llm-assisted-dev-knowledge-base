**Code Folding Keyboard Shortcuts in Visual Studio Code (Windows):**

|Operation|Shortcut|
|---|---|
|Fold current code block|`Ctrl+Shift+[`|
|Unfold current code block|`Ctrl+Shift+]`|
|Fold all code blocks|`Ctrl+K` then `Ctrl+0` (zero)|
|Unfold all code blocks|`Ctrl+K` then `Ctrl+J`|
|Fold all regions|`Ctrl+K` then `Ctrl+8`|
|Unfold all regions|`Ctrl+K` then `Ctrl+9`|

#### Quick filter in Go to Symbol (most people do this)

- Press **Ctrl+Shift+O**
- Immediately type a colon **:** (or sometimes people write @: in docs — just type :)


**Custom Keybinding Configuration:**

Modify keybindings through File → Preferences → Keyboard Shortcuts or edit `keybindings.json` directly:

```json
{
  "key": "ctrl+alt+[",
  "command": "editor.fold",
  "when": "editorTextFocus"
},
{
  "key": "ctrl+alt+]",
  "command": "editor.unfold",
  "when": "editorTextFocus"
}
```

This configuration remaps fold/unfold to `Ctrl+Alt+[` and `Ctrl+Alt+]` if preferred.



**Move Editor Group:**

- `Ctrl+K ←` — left
- `Ctrl+K →` — right