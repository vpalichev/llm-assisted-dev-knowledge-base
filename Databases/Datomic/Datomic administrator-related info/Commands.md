#### Create new datomic db

```clojure
(require '[datomic.api :as d])

;; Create database (returns true if created, false if already exists)
(d/create-database "datomic:sql://my-new-db?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic")
;=> true

;; Then connect
(def conn (d/connect "datomic:sql://my-new-db?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic"))
```

#### List all Datomic DBs:
```clojure
(require '[datomic.api :as d])

;; Storage URI (use * as wildcard for database name)
(def storage-uri "datomic:sql://*?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic")

(d/get-database-names storage-uri)
;=> ("my-app" "testing" "analytics")
```






------------------------
```clojure
(require '[datomic.api :as d])

;; Storage URI (use * as wildcard for database name)
(def storage-uri "datomic:sql://*?jdbc:postgresql://localhost:5432/datomic?user=datomic&password=datomic")

(d/get-database-names storage-uri)
;=> ("my-app" "testing" "analytics")
```

**Database stats (from a connection):**

```clojure
(def conn (d/connect uri))
(def db (d/db conn))

;; Current basis-t (latest transaction number)
(d/basis-t db)
;=> 1042

;; As-of a specific point in time
(d/as-of-t db)
;=> nil  ; (nil if current)
```

**Check if database exists:**

```clojure
(d/create-database uri)  
;=> true  if created
;=> false if already exists
```

**Delete a database:**

```clojure
(d/delete-database uri)
;=> true
```

**Key difference in URIs:**

|Purpose|URI format|
|---|---|
|`get-database-names`|`datomic:sql/?jdbc:...` (no db name, note the `?` right after `sql://`)|
|`connect` / `create-database`|`datomic:sql://my-app?jdbc:...` (with db name)|