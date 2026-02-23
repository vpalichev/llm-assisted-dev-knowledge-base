
  ---                                                                                                                                                                                                                                                                                Babashka on Windows: Cyrillic Output Corrupted to ? — Root Cause Analysis                                                                                                                                                                                                        
                                                                                                                                                                                                                                                                                   
  Environment
  
  - OS: Windows Server 2022 (Russian locale, ANSI codepage CP1251, OEM codepage CP866)
  - Shell: Git Bash (MSYS2/MinTTY)
  - Babashka: bb.exe — GraalVM native image
  - Context: clj-nrepl-eval (a Babashka script) connects to a Clojure nREPL server over TCP, evaluates expressions, and prints results to stdout

  ---
  Symptom

  Any Clojure expression returning a string with Cyrillic characters, when evaluated via clj-nrepl-eval and printed in the terminal, shows ? for every Cyrillic character:

  => "?????? ?????"       ; expected: "Активы сайта"

  Initial assumption (later proven wrong): this was a MinTTY rendering artifact and the bytes in the pipe were correct.

  ---
  Investigation — Probes in Causal Order

  PROBE 1 — Capture bb.exe stdout to a binary file

  clj-nrepl-eval -p 8950 '(:Title (nth result 5))' > /tmp/probe1.bin
  xxd /tmp/probe1.bin

  Result:
  3d3e 2022 3f3f 3f3f 3f3f 203f 3f3f 3f3f 22  => "?????? ?????"

  0x3F bytes in the file. Corruption is real — not a terminal rendering issue. bb.exe is writing literal ? bytes to its stdout before they ever reach MinTTY.

  PROBE 2 — JVM writes the same string directly to a file (bypassing bb)

  (java.nio.file.Files/write path (.getBytes s "UTF-8") ...)

  Result:
  d090 d0ba d182 d0b8 d0b2 d18b 20d1 81d0 b0d0 b9d1 82d0 b0

  Correct UTF-8 for Активы сайта. The JVM nREPL server has correct Unicode. Corruption is not upstream of bb.exe.

  PROBE 3 — bb.exe generates Cyrillic internally from raw bytes, writes via (print ...)

  bb -e '(print (String. (byte-array [0xD0 0x9F ...]) "UTF-8"))' > /tmp/probe3.bin
  xxd /tmp/probe3.bin

  Result: 3f3f3f3f3f3f — ? bytes. Not a CLI argument passing issue. The corruption happens even with a string constructed at runtime inside bb.exe itself.

  PROBE 4 — Write raw bytes directly to System.out, bypassing char encoding

  bb -e '(.write System/out (byte-array [0xD0 0x9F 0xD1 0x80 ...]))' > /tmp/probe4.bin
  xxd /tmp/probe4.bin

  Result: d09f d180 d0b8 d0b2 d0b5 d182 — correct UTF-8. The OS file handle is clean. Corruption is in the character encoding layer, not in the underlying stream.

  PROBE 5 — Call Java PrintStream.print(String) directly

  bb -e '(.print System/out (String. (byte-array [...]) "UTF-8"))' > /tmp/probe5.bin

  Result: Correct UTF-8 bytes. Java's PrintStream.print() works. Corruption is not in System.out itself.

  PROBE 6 — Check reported charsets

  (Charset/defaultCharset)         ; => UTF-8
  (.charset System/out)            ; => UTF-8
  (.getEncoding *out*)             ; => Cp1252   ← !!!

  *out* — Clojure's standard output writer — uses Cp1252, not UTF-8.

  ---
  Root Cause

  Clojure's *out* is a java.io.OutputStreamWriter wrapping System/out. It is initialized with the platform's default charset at the time clojure.core is loaded. In a GraalVM native image (such as Babashka's bb.exe), class initialization that occurs at image build time is    
  frozen into the image.

  When the Babashka native image was built, the build machine's Windows ANSI codepage was Cp1252 (Western European). OutputStreamWriter captured that charset into the image. At runtime on a Russian Windows machine — even though Charset/defaultCharset() and
  System.out.charset() correctly return UTF-8 — *out* is already a live object inside the native image with Cp1252 frozen in.

  CP1252 covers only U+0000–U+00FF (with some exceptions in 0x80–0x9F). Cyrillic characters occupy U+0400–U+04FF — entirely outside CP1252's range. Java's CharsetEncoder replaces every unmappable character with the charset's replacement byte: 0x3F (?).

  All Clojure output functions (print, println, pr, pr-str when used with *out*) go through this encoder. The corruption happens before the bytes reach the pipe, MinTTY, or any terminal.

  ---
  Causal Chain

  Babashka native image built on Windows with Cp1252 ANSI codepage
    └─► clojure.core/*out* = (OutputStreamWriter. System/out) captured at build time
          └─► *out*.encoding = "Cp1252" frozen into bb.exe binary

  At runtime on any machine:
    JVM nREPL server evaluates expression
      └─► result = "Активы сайта" (correct Unicode, U+0410 U+043A ...)
            └─► nREPL bencode-encodes as UTF-8 bytes over TCP socket
                  └─► bb.exe receives bytes, decodes as UTF-8 ✓
                        └─► bb.exe calls (println result)
                              └─► Clojure print → *out* OutputStreamWriter (Cp1252)
                                    └─► CharsetEncoder: U+0410 → no Cp1252 mapping
                                          └─► replacement byte: 0x3F ('?')
                                                └─► stdout receives 3F 3F 3F 3F 3F 3F
                                                      └─► terminal shows: ??????

  ---
  Verification of Fix

  Bypassing *out* with an explicit UTF-8 writer produces correct bytes:

  (let [w (java.io.PrintWriter. (java.io.OutputStreamWriter. System/out "UTF-8") true)
        s (String. (byte-array [0xD0 0x9F 0xD1 0x80 0xD0 0xB8 0xD0 0xB2 0xD0 0xB5 0xD1 0x82]) "UTF-8")]
    (.print w s)
    (.flush w))

  Output: d09f d180 d0b8 d0b2 d0b5 d182 — correct UTF-8 for "Привет".

  ---
  Key Observations for a Potential Fix

  1. Charset/defaultCharset() lying at runtime is a red herring — it reflects the runtime JVM property, not what *out* was constructed with.
  2. System.out.charset() also returns UTF-8 but this is the PrintStream's charset, which is separate from *out*'s OutputStreamWriter charset.
  3. Setting -Dfile.encoding=UTF-8 at bb.exe startup (e.g. via JAVA_TOOL_OPTIONS) would not help because *out* is already constructed and frozen in the image.
  4. A proper fix would require Babashka to re-initialize *out* at image runtime (not build time), or to initialize it lazily using the runtime's defaultCharset().
  5. A workaround for Babashka scripts that need to output non-ASCII: re-bind or reset *out* to a fresh OutputStreamWriter with an explicit "UTF-8" charset at script startup.

  ---
  Affected Versions / Scope

  - Reproduced on: bb.exe (Windows native image), Windows Server 2022, Russian locale
  - Likely affects: any Windows locale whose ANSI codepage is not UTF-8 (i.e., all pre-UTF-8 Windows locales — CP1252, CP1251, CP932, etc.)
  - Not affected: Babashka on Linux/macOS (default charset is typically UTF-8, so *out* is initialized correctly)
  - Not affected: standard Clojure JVM (not a native image; *out* is initialized at JVM startup, not baked in)