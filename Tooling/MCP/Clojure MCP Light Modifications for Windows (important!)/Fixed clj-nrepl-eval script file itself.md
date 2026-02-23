```

#!/usr/bin/env bb

; :bbin/start
;
; {:coords
;  {:git/url "https://github.com/bhauman/clojure-mcp-light.git",
;   :git/sha "401e2746d1dea1d29cd9e69c35f22e844e104c2d"},
;  :lib
;  org.babashka.bbin/script--693757147-https-github-com-bhauman-clojure-mcp-light-git}
;
; :bbin/end

(require '[babashka.process :as process]
         '[babashka.fs :as fs]
         '[clojure.string :as str])

(def script-root "C:\\Users\\v.palichev\\.gitlibs\\libs\\org.babashka.bbin\\script--693757147-https-github-com-bhauman-clojure-mcp-light-git\\401e2746d1dea1d29cd9e69c35f22e844e104c2d")
(def script-lib 'org.babashka.bbin/script--693757147-https-github-com-bhauman-clojure-mcp-light-git)
(def script-coords {:git/url "https://github.com/bhauman/clojure-mcp-light.git", :git/sha "401e2746d1dea1d29cd9e69c35f22e844e104c2d"})
(def script-main-opts ["-m" "clojure-mcp-light.nrepl-eval"])

(def tmp-edn
  (doto (fs/file (fs/temp-dir) (str (gensym "bbin")))
    (spit (str "{:deps {" script-lib script-coords "}}"))
    (fs/delete-on-exit)))

(def base-command
  (vec (concat ["bb"
                "--deps-root" script-root
                "--config" (str tmp-edn)
                "-cp" (str script-root "\\src")
                "-e" (str "(do"
                          " (alter-var-root #'*out* (constantly (java.io.PrintWriter. (java.io.OutputStreamWriter. System/out java.nio.charset.StandardCharsets/UTF_8) true)))"
                          " (alter-var-root #'*err* (constantly (java.io.PrintWriter. (java.io.OutputStreamWriter. System/err java.nio.charset.StandardCharsets/UTF_8) true)))"
                          " (require 'clojure-mcp-light.nrepl-eval)"
                          " (apply clojure-mcp-light.nrepl-eval/-main *command-line-args*))")]
               ["--"])))

(process/exec (into base-command *command-line-args*))

```


This was changed in three places