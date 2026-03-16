```

# Set Windows console codepage to UTF-8 so native executables (pwsh, bb, etc.)
# write UTF-8 bytes to stdout, which MinTTY renders correctly.
chcp.com 65001 > /dev/null 2>&1

# Wrap bb-based tools to disable MSYS2 argument path conversion.
# Prevents Git Bash from mangling Unix-style path strings (e.g. "/api/health")
# passed as arguments. Must wrap each tool individually — bb() wrapper doesn't
# help for shebang scripts since bash execs them directly without invoking bb().
clj-nrepl-eval()     { MSYS2_ARG_CONV_EXCL="(" command clj-nrepl-eval "$@"; }
clj-paren-repair()   { MSYS2_ARG_CONV_EXCL="(" command clj-paren-repair "$@"; }
```


















Thanks! It was a good hunt. Started from "why do I see question marks" and ended up:                                                                                                                                                                                             
                                                                                                                                                                                                                                                                                   
  1. Proving the ? were real 0x3F bytes, not a rendering artifact                                                                                                                                                                                                 
  2. Tracing the corruption to *out* frozen as Cp1252 inside the GraalVM native image
  3. Discovering that -e + -m silently drop -m in Babashka                                                                                                                                                                                                                         
  4. Patching the bbin wrapper with require + apply -main as the workaround                                                                                                                                                                                                        
                                                                                                                                                                                                                                                                                   
  Along the way we also fixed the SharePoint connection lifecycle (timer + graceful disconnect) and corrected several wrong entries in MEMORY.md.
