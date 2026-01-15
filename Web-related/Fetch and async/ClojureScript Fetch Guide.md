# Table of Contents

- [[#The Basic Function]]
- [[#What Happens Step-by-Step]]
- [[#Why Two .then Calls?]]
- [[#Using It]]
- [[#Why Headers Come Separately]]
- [[#How Promises Work]]
- [[#HTTP Errors]]
- [[#Alternative: core.async]]
- [[#Why Everything Is Async in Browsers]]
- [[#Promise Chaining]]

---

## The Basic Function

[[#Table of Contents|Back to TOC]]

```clojure
(defn fetch-json [url]
  (-> (js/fetch url)
      (.then #(.json %))
      (.then #(js->clj % :keywordize-keys true))))
```

## What Happens Step-by-Step

[[#Table of Contents|Back to TOC]]

### 1. Start the Request

```clojure
(js/fetch url)
```

- Opens network connection to server
- Sends HTTP GET request
- Returns a Promise immediately
- Doesn't wait—your code keeps running

### 2. Headers Arrive

```clojure
(.then #(.json %))
```

- Server sends back status code and headers first
- Promise resolves with Response object
- Response has `.status`, `.headers`, but body isn't read yet
- `.json()` tells browser to read and parse the body
- Returns another Promise

### 3. Body Arrives

```clojure
(.then #(js->clj % :keywordize-keys true))
```

- Browser finishes downloading body data
- JSON parser converts it to JavaScript object
- Promise resolves with parsed data
- `js->clj` converts to ClojureScript map
- String keys become keywords: `"name"` → `:name`

## Why Two .then Calls?

[[#Table of Contents|Back to TOC]]

Because two separate async operations happen:

1. **Wait for headers** (fast, ~50ms) → get Response object
2. **Wait for body** (slower, depends on size) → get parsed data

Each `.then` waits for one operation to complete.

## Using It

[[#Table of Contents|Back to TOC]]

```clojure
(-> (fetch-json "https://api.example.com/users")
    (.then (fn [users] (prn users)))
    (.catch (fn [error] (prn "Error:" error))))
```

Your callback in `.then` runs when all data arrives. Could be instant or take seconds.

## Why Headers Come Separately

[[#Table of Contents|Back to TOC]]

HTTP sends response in two parts:

```
HTTP/1.1 200 OK              ← Headers (arrive first, small)
Content-Type: application/json
Content-Length: 5000

{"users": [...]}             ← Body (arrives second, can be large)
```

Fetch API lets you check the status code before downloading a potentially huge body. Older APIs (XMLHttpRequest) forced you to wait for everything.

## How Promises Work

[[#Table of Contents|Back to TOC]]

A Promise is a placeholder for data that doesn't exist yet:

- **Pending**: waiting for operation to finish
- **Fulfilled**: operation succeeded, data available
- **Rejected**: operation failed, error available

`.then(callback)` says "run this callback when fulfilled"
`.catch(callback)` says "run this callback when rejected"

Your code doesn't stop and wait—the browser runs your callbacks later when ready.

## HTTP Errors

[[#Table of Contents|Back to TOC]]

Important: HTTP status 404 or 500 don't trigger `.catch` automatically. Check manually:

```clojure
(defn fetch-json [url]
  (-> (js/fetch url)
      (.then (fn [response]
               (if (.-ok response)  ; true for 200-299 status
                 (.json response)
                 (js/Promise.reject (js/Error. "HTTP error")))))
      (.then #(js->clj % :keywordize-keys true))))
```

`.catch` only triggers on network failures (no connection, timeout, CORS block).

## Alternative: core.async

[[#Table of Contents|Back to TOC]]

Makes async code look synchronous:

```clojure
(require '[cljs.core.async :refer [go]]
         '[cljs.core.async.interop :refer [<p!]])

(defn fetch-json [url]
  (go
    (let [response (<p! (js/fetch url))
          data (<p! (.json response))]
      (js->clj data :keywordize-keys true))))

(go
  (let [users (<! (fetch-json "/api/users"))]
    (prn users)))
```

`<p!` unwraps promises like `await` in JavaScript. Still async underneath, just easier to read.

## Why Everything Is Async in Browsers

[[#Table of Contents|Back to TOC]]

JavaScript runs in a single thread. If network requests blocked (stopped and waited), the entire browser tab would freeze—no clicking, scrolling, or typing. Async operations let the browser stay responsive while waiting for network data.

Server-side Clojure (JVM) uses threads, so blocking is fine:

```clojure
(require '[clj-http.client :as http])

(def users (:body (http/get "/api/users" {:as :json})))
;; Blocks until complete—fine on servers, not in browsers
```

## Promise Chaining

[[#Table of Contents|Back to TOC]]

Each `.then` returns a new Promise. Values flow through the chain:

```clojure
(-> promise-1
    (.then fn-a)  ; Returns promise-2
    (.then fn-b)  ; Returns promise-3
    (.catch fn-c))
```

If any Promise rejects, execution jumps to `.catch`, skipping remaining `.then` calls. Like try-catch but for async operations.