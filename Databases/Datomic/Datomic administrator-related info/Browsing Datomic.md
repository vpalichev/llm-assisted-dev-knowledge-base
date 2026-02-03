# Exploring Datomic databases: tools and workflows for 2025

**The Datomic tooling landscape has shifted significantly** in 2024-2025, with Cognitect's REBL deprecated and replaced by the open-source Morse, while community tools like Portal and Hyperfiddle's Datomic Browser have emerged as the dominant exploration options. Unlike SQL databases with mature GUI ecosystems, Datomic exploration centers on **REPL-driven development** augmented by specialized data browsers—a workflow that aligns with Clojure's interactive programming philosophy but requires different mental models than traditional database administration.

The most important tools available today are: **Portal** (the most popular open-source data browser at ~1,000 GitHub stars), **Morse** (official Cognitect successor to REBL), **Datomic Console** (web UI bundled with Datomic Pro), and **Hyperfiddle Datomic Browser** (purpose-built production diagnostics tool). For schema visualization, **Schema Cartographer** and **Schema Voyager** lead the ecosystem.

## Official tools from Cognitect and Nubank

Cognitect (now under Nubank ownership since 2020) provides several official tools, though the ecosystem has undergone major changes. The trend has been toward **open-sourcing tools under Apache 2.0**, making Datomic development more accessible.

**Datomic Console** remains the primary official GUI for exploring Datomic Pro databases. This web-based interface provides schema exploration with hierarchical tree views, a visual query builder supporting `:find`, `:with`, `:where`, and `:in` clauses, entity examination with bidirectional navigation, and transaction history browsing at multiple time scales. To launch it, run `bin/console -p 8080 dev datomic:dev://localhost:4334/` from the Datomic distribution and access it at `http://localhost:8080/browse/`. Note that Console is **only available for Datomic Pro** (not Datomic Cloud) and has had version compatibility issues requiring standalone updates—version 0.1.225 fixed a regression affecting Datomic 1.0.6165+.

**Morse** is the new official data browser, released April 2023 as the **open-source successor to REBL**. REBL's repository was archived on July 16, 2024, with all users directed to migrate to Morse. Key improvements include Apache 2.0 licensing (no commercial restrictions), remote inspection via Replicant libraries, and on-demand inspection via an `inspect` API. Setup is straightforward:

```clojure
;; Install as CLI tool
clj -Ttools install-latest :lib io.github.nubank/morse :as morse

;; Or add to deps.edn
io.github.nubank/morse {:git/tag "v2023.04.30.01" :git/sha "d99b09c"}
```

**Datomic Local** (formerly dev-local) provides a full Client API implementation for local development without server connectivity. Now Apache 2.0 licensed, it supports memory-only databases (`:storage-dir :mem`) and can import data from Datomic Cloud. This makes it invaluable for testing and small single-process applications.

## Third-party GUI tools and web interfaces

Unlike PostgreSQL with pgAdmin or MySQL with Workbench, **Datomic lacks a mature standalone GUI comparable to traditional database tools**. DBeaver, DataGrip, TablePlus, and other universal database tools do not support Datomic due to its unique Datalog query language and immutable data model. However, several specialized alternatives exist.

**Hyperfiddle Datomic Browser** is the most notable recent addition, featured at Clojure/conj 2025. This web-based tool is designed for production support and diagnostics, offering fluent virtual scroll for **50,000+ records**, searchable EAVT index access, entity navigation with reverse refs, query performance diagnostics (io-stats, query-stats), and PII protection support. At only ~300 lines of code, it's designed to be forked and customized. It's free for individual local development but requires a license for production deployment.

**datomic-graph-viz** (released v1.0.0 in May 2025) visualizes Datomic data as an interactive explorable graph, with nodes color-coded by attribute sets. Run it with Babashka using `bb start <datomic-connection-string>`.

**Dataspex** is a newer browser extension (Chrome/Firefox) with explicit Datomic and Datascript support. It enables entity relationship navigation, including reverse refs, tracks changes over time with audit trails, and integrates with Calva Power Tools. Install via `{:deps {no.cjohansen/dataspex {:mvn/version "2025.06.7"}}}`.

Older projects like **Datomicism** (81 stars) and **Lewis** are largely abandoned but historically notable—Datomicism provided a Smalltalk-like workspace environment with drag-and-drop widgets and real-time schema validation.

## REPL-based data visualization with Portal, Reveal, and Clerk

The dominant pattern for Datomic exploration is **REPL-driven development** augmented by data visualization tools. Portal, Reveal, and Clerk represent three different approaches to this workflow.

**Portal** (github.com/djblue/portal) is the most actively maintained option with ~1,000 GitHub stars, 113 releases, and dedicated VS Code and IntelliJ plugins. It integrates with `tap>` to automatically capture REPL evaluations and supports Datomic via the `com.datomic/dev.datafy` library, which extends Clojure's datafy/nav protocols for entity navigation:

```clojure
{:deps {djblue/portal {:mvn/version "0.62.0"}
        com.datomic/dev.datafy {:git/sha "4a9dffb" :git/tag "v0.1"
                                :git/url "https://github.com/Datomic/dev.datafy"}}}

;; Start Portal and tap Datomic query results
(require '[portal.api :as p])
(def portal (p/open {:launcher :vs-code}))
(add-tap #'p/submit)
(tap> (d/q '[:find ?e ?name :where [?e :user/name ?name]] (d/db conn)))
```

**Reveal** (vlaaad/reveal) takes a different approach as an in-process REPL output pane using JavaFX, maintaining references to actual objects rather than just their printed representations. It excels at Vega/Vega-Lite visualizations of query results. A free core version exists alongside a Pro version ($9.99/month) with additional features.

**Clerk** (nextjournal/clerk) enables notebook-style exploration where Clojure namespaces become literate documents with automatic incremental recomputation. It's particularly valuable for documenting Datomic exploration workflows and sharing analysis with team members via static HTML output.

## Datomic Console: current status and usage

Datomic Console **remains available** as the official web-based GUI bundled with Datomic Pro distributions. It is not available for Datomic Cloud deployments.

The Console provides five main capabilities: a **Schema tab** for browsing attribute definitions organized by namespace; a **Query tab** with both visual builder and direct text editing; an **Entity tab** for examining entities and navigating relationships bidirectionally; a **Transactions tab** with a timeline browser supporting day/hour/minute/second scales; and an **Index tab** for direct EAVT, AVET, and VAET index lookups.

**Version compatibility** requires attention. In 2020, a regression caused Console to fail on Datomic versions 1.0.6165+ with the error "This version of Console requires Datomic 0.8.4096.0 to run." This was fixed in Console 0.1.225 (May 2020), available as a standalone download at https://my.datomic.com/downloads/console. Users running older Datomic versions bundled with pre-0.1.225 Console should download this update.

## Schema visualization and community tools

For understanding Datomic's data model, several schema-focused tools have emerged from the community.

**Schema Cartographer** (159 GitHub stars) is the most established schema visualization tool, offering interactive relationship diagrams with the ability to create, edit, and share schemas visually. It has split into separate repositories for cloud (`schema-cartographer-cloud`) and on-prem (`schema-cartographer-on-prem`) deployments.

**Schema Voyager** (45 stars) generates interactive HTML documentation pages showing collections, attributes, and reference diagrams. It supports "supplemental properties" for tracking what refs point to, deprecated attributes, and superseded fields. A live example for the mbrainz schema is available at mainej.github.io/schema-voyager/mbrainz-schema.html.

**Alzabo** takes a different approach, generating HTML documentation and Datomic schemas from EDN definition files, with experimental support for using LLMs to generate domain ontologies.

## Workflows and best practices for Datomic exploration

The **REPL-driven workflow** is central to effective Datomic exploration. Key patterns include:

**Always capture a database value** for consistent exploration within a unit of work. Since Datomic databases are immutable values, holding a reference ensures queries see a consistent point-in-time view:

```clojure
(def db (d/db conn))  ;; Snapshot for exploration
```

**Use pull patterns liberally** for hierarchical data retrieval. The wildcard `[*]` is valuable for initial exploration, while specific patterns with nested maps handle reference navigation efficiently. Reverse references use underscore prefixes: `[:release/_artists]` finds all releases referencing an artist.

**Explore history** using the history database filter for audit trails:

```clojure
(d/q '[:find ?attr ?val ?tx ?added
       :in $ ?e
       :where [?e ?a ?val ?tx ?added]
              [?a :db/ident ?attr]]
     (d/history db) entity-id)
```

**Query debugging** should proceed incrementally—test `:where` clauses one at a time, starting with the most selective clause first. Use query timeouts for protection: `(d/query {:query '[:find ...] :timeout 1000 :args [db]})`.

**Schema best practices** include grouping related attributes in namespaces (`:artist/name`, `:artist/country`), using idents for enumerated types, never removing or reusing attribute names (grow schema, don't break it), and annotating schema with `:db/doc` documentation strings.

## Recommended tool combinations for different use cases

For **general development**, the community consensus workflow (per Sean Corfield and others) combines Portal as the primary data inspector with `com.datomic/dev.datafy` for enhanced Datomic entity navigation, integrated via VS Code's Portal extension or IntelliJ's plugin.

For **production support and diagnostics**, Hyperfiddle Datomic Browser provides specialized capabilities including PII protection, query supervision to guard against runaway queries, and efficient virtual scrolling for large result sets.

For **schema documentation and team knowledge sharing**, Schema Voyager's HTML output or Schema Cartographer's interactive diagrams serve well. Clerk notebooks can document exploration workflows as shareable, reproducible documents.

For **ClojureScript applications using Datascript**, Dataspex offers native support with browser devtools integration, enabling the same exploration patterns for in-browser databases.

## Conclusion

Datomic's exploration tooling differs fundamentally from traditional SQL databases—rather than standalone GUI applications, the ecosystem emphasizes **REPL integration and data browsers** that fit Clojure's interactive development model. The recent shift to open-source licensing for Morse and Datomic Local, combined with active community tools like Portal and Hyperfiddle Datomic Browser, has made the ecosystem more accessible than ever.

The most practical setup for 2025 combines Portal (for general data inspection), Morse (for dedicated browsing sessions), and Schema Voyager or Schema Cartographer (for schema understanding)—all working alongside direct REPL interaction using pull patterns, entity navigation, and history queries. For those coming from SQL tools expecting a pgAdmin equivalent, the adjustment requires embracing REPL-driven exploration as the primary workflow, with visualization tools as powerful complements rather than replacements.