## For user preferences (not JSON settings):

**The file is:**

```
~/.claude/CLAUDE.md
```

On Windows, `~` translates to `%USERPROFILE%`, so:

```
%USERPROFILE%\.claude\CLAUDE.md
```

**NOT** `%USERPROFILE%\.claude-code\preferences.md` like I said (I hallucinated that path).

## For JSON settings:

The actual location is:

```
~/.claude/settings.json
```

On Windows:

```
%USERPROFILE%\.claude\settings.json
```

## Important distinction:

- **CLAUDE.md** = Markdown instructions/preferences (natural language instructions to Claude)
- **settings.json** = JSON configuration (tool permissions, MCP servers, etc.)

For your use case (telling Claude about Windows), you want the **CLAUDE.md** file with markdown instructions.

**So the correct answer is:**

```
%USERPROFILE%\.claude\CLAUDE.md
```

Thanks for catching that! This is exactly why it's good to verify claims rather than trusting them blindly.