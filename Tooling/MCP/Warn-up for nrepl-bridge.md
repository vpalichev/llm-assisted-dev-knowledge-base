.?> Session warm-up. Read the project's CLAUDE.md and OVERVIEW.md. Then:                                                                                                                                                                             
  1. Check backend nREPL: nrepl_send {form: "(+ 1 2 3)"}                                                                                                                                                                                               
  2. Check browser count: nrepl_send {form: "(count (shadow.cljs.devtools.api/repl-runtimes <SHADOW_BUILD>))"}                                                                                                                                              
  - If 0: tell me to open the app in the browser
     - If >1: tell me to close extras                                                                                                                                                                                                                  
  3. Check frontend eval: nrepl_send {form: "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(.-title js/document)\" {})"}
  4. Verify paren repair works in the eval pipeline: nrepl_send {form: "(+ 1 2"}
     - Should show "[Paren repair applied]" and return 3
  5. List existing namespaces under src/ and summarize what's built
  6. Check shadow-cljs.edn for build ID and ports
  7. Check deps.edn for available aliases and dependencies
  8. Report results as a short checklist with pass/fail for each item

  Get <SHADOW_BUILD> and port numbers from shadow-cljs.edn and deps.edn,
  do not hardcode them. If any check fails, stop and report — do not
  proceed to the next check.