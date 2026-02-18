## Architecture

Every interaction in re-frame follows this one-directional cycle:

```
User action → dispatch → event handler → app-db update → subscriptions → views re-render
```

All state lives in a single atom (`app-db`) — a big Clojure map that holds everything your app needs to render. You never modify it directly. Instead, you dispatch events ("something happened"), and event handlers produce a new version of the state.

Views don't read `app-db` directly either. They subscribe to specific slices of it, and only re-render when their slice changes.

### The purity problem

Event handlers are pure functions — data in, data out, no side effects. But your app obviously needs to interact with the outside world: read from localStorage, make HTTP calls, check the clock.

The solution is that the handler never touches the real world directly. Two layers sit between it and the outside:

**Coeffects (input layer):** Before your handler runs, re-frame reads from the outside world _for you_ and hands the results to your handler as plain data. Your handler never calls `(js/Date.)` — it just receives `{:now #inst "2025-..."}` in its arguments.

**Effects (output layer):** Your handler never calls `fetch()` or writes to localStorage. It returns a map saying "I want these things to happen": `{:http {:url "/api"} :local-storage {:key :prefs}}`. After the handler returns, re-frame reads that map and performs the actual I/O.

```
Outside world → coeffects (read external data, hand it to handler as plain data)
                    ↓
              cofx map {:db ... :now ... :local-storage ...}
                    ↓
              event handler (pure function — never touches the real world)
                    ↓
              effects map {:db ... :http ... :dispatch ...}
                    ↓
Outside world ← effects (read the map, perform the actual I/O)
```

The handler lives in a pure bubble. That's why testing is easy — you skip the boundary layers entirely, pass fake data in, and check what comes out. No mocks, no HTTP servers, no async complexity.

---

## Dispatching Events

### `rf/dispatch`

This is how you tell re-frame "something happened." You call it with an event vector — a Clojure vector where the first element is a keyword naming the event, and the rest is data the handler needs.

```clojure
(rf/dispatch [:add-todo "Buy coffee"])
```

`dispatch` is **asynchronous**. It puts the event on a queue and returns immediately. Your next line of code runs before the handler has executed. This means you can't dispatch an event and then read the updated state right away — it hasn't changed yet.

```clojure
(rf/dispatch [:set-count 5])
;; at this point, :set-count has NOT been processed yet
;; app-db still has the old value
```

Events are processed **in order, one at a time**. If you dispatch two events in a row, the second handler is guaranteed to see the state produced by the first. No race conditions.

Call it from anywhere — view components, callbacks, timers, WebSocket listeners:

```clojure
;; from a button click
[:button {:on-click #(rf/dispatch [:login])} "Log in"]

;; from a timer
(js/setInterval #(rf/dispatch [:tick]) 1000)
```

The event vector can carry any data, as long as the first element is a keyword:

```clojure
(rf/dispatch [:simple-event])
(rf/dispatch [:set-name "Alice"])
(rf/dispatch [:move-player {:x 10 :y 20} :north])
```

### `rf/dispatch-sync`

Same as `dispatch`, but processes the event **immediately, before returning**. The next line of code sees the fully updated `app-db`.

```clojure
(rf/dispatch-sync [:initialize-db])
;; app-db is fully updated here — safe to render
(reagent.dom/render [app] (.getElementById js/document "app"))
```

This exists to solve one specific problem: **app startup**. If you use regular `dispatch` to set up your initial state, the render call happens before the state is ready, and your components either crash or flash a broken state.

**Use it only for initialization.** It bypasses the event queue, which breaks the orderly one-at-a-time processing that makes re-frame predictable. In practice you'll see it exactly once per project — in the entry point — and nowhere else.

---

## Event Handlers

### `rf/reg-event-db`

Registers a handler that **only updates state**. This is the simpler of the two handler types. It receives the current `app-db` and the event vector, and returns a new `app-db`.

```clojure
(rf/reg-event-db
  :add-todo                          ;; event id to match
  (fn [db [_ text]]                  ;; db = current state, text = payload
    (update db :todos conj {:text text :done false})))
```

The handler is a pure function — same input always gives same output. It must return the **full** `app-db`, not just the changed piece. If you return `{:loading true}` instead of `(assoc db :loading true)`, you wipe out everything else in the state.

The functions `assoc`, `update`, `assoc-in`, `update-in`, and `dissoc` are your main tools here — they all return a complete new map with just the targeted part changed.

Testing is trivial because it's just a function call:

```clojure
(let [old-db {:todos []}
      new-db (handler old-db [:add-todo "Test"])]
  (assert (= 1 (count (:todos new-db)))))
```

No framework setup, no mocking, no async.

Use `reg-event-db` when your handler only needs to transform state. The moment you need side effects (HTTP calls, timers, localStorage writes), switch to `reg-event-fx`.

### `rf/reg-event-fx`

Registers a handler that can **update state and trigger side effects**. This is the more powerful handler type.

Two differences from `reg-event-db`:

**Input:** Instead of receiving raw `db`, it receives a **coeffects map** — a map that _contains_ `db` along with any injected external data. You destructure `:db` out of it:

```clojure
(fn [{:keys [db]} event]   ;; coeffects map, not bare db
  ...)
```

**Output:** Instead of returning a new `db`, it returns a **map of effects** — instructions for things that should happen:

```clojure
(rf/reg-event-fx
  :fetch-todos
  (fn [{:keys [db]} _]
    {:db         (assoc db :loading true)       ;; update state
     :http-xhrio {:method :get                  ;; make an HTTP request
                   :uri    "/api/todos"
                   :on-success [:todos-loaded]
                   :on-failure [:todos-failed]}}))
```

Each key in the returned map is an effect id. re-frame finds the matching `reg-fx` handler and executes it. Built-in effects:

|Effect|Description|
|---|---|
|`:db`|Replace `app-db` with this value|
|`:dispatch`|Fire a single event after this handler completes|
|`:dispatch-n`|Fire multiple events|
|`:dispatch-later`|Fire an event after a delay (takes `{:ms ... :dispatch ...}`)|

All keys are optional. You can return just side effects with no `:db`, or an empty map if nothing should happen.

The handler is still **pure** — it _describes_ what should happen but doesn't _perform_ it. The actual HTTP call, localStorage write, etc. happens later, handled by re-frame's effect system. That's why testing still works — you just check the returned map:

```clojure
(let [result (handler {:db {:todos []}} [:fetch-todos])]
  (assert (= true (get-in result [:db :loading])))
  (assert (= :get (get-in result [:http-xhrio :method]))))
```

Since the returned map is just data, you can build it with normal Clojure — conditionals, `merge`, `cond->`, whatever:

```clojure
(rf/reg-event-fx
  :submit-form
  (fn [{:keys [db]} _]
    (if (valid? (:form db))
      {:db         (assoc db :submitting true)
       :http-xhrio {:method :post :uri "/api/submit" :params (:form db)
                     :on-success [:submit-success]}}
      {:db (assoc db :errors (get-errors (:form db)))})))
```

---

## Subscriptions

### `rf/reg-sub`

Registers a **subscription** — a named query that derives data from `app-db`. Components don't read `app-db` directly because that would mean every component re-renders on _any_ state change. Subscriptions solve this — each component only re-renders when the specific data it subscribed to actually changed.

**Layer 2 subscriptions** extract raw data from `app-db`:

```clojure
(rf/reg-sub
  :todos
  (fn [db _]
    (:todos db)))
```

**Layer 3 subscriptions** derive data from _other subscriptions_. They only recompute when their inputs change:

```clojure
(rf/reg-sub
  :visible-todos
  :<- [:todos]                            ;; depends on :todos subscription
  :<- [:visibility-filter]                ;; depends on :visibility-filter subscription
  (fn [[todos filter-type] _]             ;; receives their current values
    (case filter-type
      :all    todos
      :done   (filter :done todos)
      :active (remove :done todos))))
```

Each `:<-` declares an input subscription. The values arrive in the same order in the function arguments. This creates a dependency graph where intermediate results are cached — if you change `:visibility-filter`, only `:visible-todos` recomputes, and anything that depends solely on unrelated state doesn't recompute at all.

Subscriptions can be **parameterized** — the query vector can carry arguments beyond the subscription id:

```clojure
(rf/reg-sub
  :todo-by-id
  (fn [db [_ id]]
    (first (filter #(= id (:id %)) (:todos db)))))

;; usage — each unique id creates its own cached subscription instance
@(rf/subscribe [:todo-by-id 42])
```

### `rf/subscribe`

Returns a **Reagent Reaction** — a reactive reference that tracks changes and triggers re-renders. `rf/subscribe` doesn't return the data itself; it returns a reactive container. You unwrap it with `@` (shorthand for `deref`) to get the actual value.

```clojure
(defn todo-list []
  (let [todos @(rf/subscribe [:todos])]     ;; @ unwraps the Reaction
    [:ul (for [t todos]
           ^{:key (:id t)}
           [:li (:text t)])]))
```

**Don't forget `@`.** Without it, you get a Reaction object instead of your data, which usually causes a confusing rendering error.

**Caching:** If two components both call `(rf/subscribe [:todos])`, they share the same subscription instance. The computation runs once, both get the cached result.

**Must be called inside a Reagent component** for reactivity to work. If called at the top level of a namespace or inside a non-reactive function, the value is frozen at evaluation time and won't trigger re-renders.

---

## Effects and Coeffects

This is where re-frame keeps your event handlers pure. Your handler never directly reads from or writes to the outside world. Instead, coeffects bring external data _in_ and effects carry instructions _out_.

### `rf/reg-fx`

Registers an **effect handler** — the function that actually performs a side effect. When your `reg-event-fx` handler returns a map like `{:console-log "hello"}`, re-frame finds the `:console-log` effect handler registered with `reg-fx` and calls it with `"hello"`.

```clojure
(rf/reg-fx
  :console-log
  (fn [message]
    (js/console.log message)))
```

Now any event handler can include `:console-log` in its returned effects map, and re-frame knows what to do with it.

More practical examples:

```clojure
(rf/reg-fx
  :local-storage
  (fn [{:keys [key value]}]
    (.setItem js/localStorage (name key) (js/JSON.stringify (clj->js value)))))

(rf/reg-fx
  :set-title
  (fn [title]
    (set! (.-title js/document) title)))
```

The handler receives whatever value was associated with its key in the effects map — a string, map, vector, nil, whatever makes sense for that effect.

**Keep effect handlers thin.** They should just do I/O. All decision-making belongs in the pure `reg-event-fx` handler. This way, you test the logic (pure, easy) separately from the I/O (impure, harder).

Third-party libraries like `re-frame-http-fx` are essentially just `reg-fx` registrations. When you include the library, it registers `:http-xhrio` behind the scenes using this same mechanism. If you ever want to swap HTTP libraries, you just register a different handler — none of your event handlers change.

### `rf/reg-cofx`

Registers a **coeffect handler** — the function that reads external data and hands it to your event handler as plain data. This is the input side of the boundary.

The problem it solves: if your handler needs the current time, you could call `(js/Date.)` inside it, but then it's no longer pure — same input gives different output every time, making it hard to test.

Instead, you register a coeffect that reads the time:

```clojure
(rf/reg-cofx
  :now
  (fn [cofx _]
    (assoc cofx :now (js/Date.))))
```

The handler receives the **current coeffects map** (which already contains `:db`) and must return it with your data added. Always `assoc` onto the existing map — returning a new map wipes out `:db` and everything else.

Your event handler then receives the time as plain data, keeping it pure:

```clojure
(rf/reg-event-fx
  :log-action
  [(rf/inject-cofx :now)]                     ;; "I need the current time"
  (fn [{:keys [db now]} [_ action]]           ;; :now is just data now
    {:db (update db :log conj {:action action :time now})}))
```

In tests, you skip the coeffect entirely and pass whatever value you want:

```clojure
(let [result (handler {:db {:log []} :now fake-time} [:log-action "click"])]
  ;; deterministic, no mocking needed
  (assert (= fake-time (-> result :db :log first :time))))
```

### `rf/inject-cofx`

An **interceptor** that connects registered coeffects to specific event handlers. Registering a coeffect with `reg-cofx` doesn't mean it runs everywhere — each handler must explicitly request it using `inject-cofx`.

Interceptors sit in a vector between the event id and the handler function:

```clojure
(rf/reg-event-fx
  :add-todo
  [(rf/inject-cofx :random-uuid)          ;; interceptor 1: inject a UUID
   (rf/inject-cofx :now)]                 ;; interceptor 2: inject current time
  (fn [{:keys [db random-uuid now]} [_ text]]
    {:db (update db :todos conj {:id random-uuid :text text :created-at now :done false})}))
```

They run in order before the handler executes. First `:random-uuid` is fetched and added to the cofx map, then `:now` is fetched and added, then the handler runs with everything available.

Accepts an optional second argument that gets forwarded to the `reg-cofx` handler, making it reusable:

```clojure
;; same :local-storage coeffect, different keys
[(rf/inject-cofx :local-storage :auth-token)]
[(rf/inject-cofx :local-storage :user-prefs)]
```

**Only useful with `reg-event-fx`**, not `reg-event-db`. `reg-event-db` handlers only receive `db` as their first argument, so any injected coeffects are invisible to them.

The `:db` coeffect is injected automatically for every handler — you never need `(rf/inject-cofx :db)`.

---

## Utility

### `rf/db`

The `app-db` atom itself. Rarely accessed directly — use subscriptions instead. Occasionally useful for debugging:

```clojure
@re-frame.db/app-db   ;; snapshot of entire app state
```

### `rf/clear-subscription-cache!`

Clears all cached subscription instances. During development with hot reloading, subscriptions can get stale or hold references to old handler functions. Clearing the cache forces fresh ones:

```clojure
(defn ^:dev/after-load on-reload []
  (rf/clear-subscription-cache!)
  (mount-root))
```

---

## Quick Decision Guide

|Situation|Use|
|---|---|
|Fire an event from a button click|`rf/dispatch`|
|Set up initial app state at startup|`rf/dispatch-sync`|
|Handler only updates `app-db`|`rf/reg-event-db`|
|Handler needs HTTP calls, timers, other side effects|`rf/reg-event-fx`|
|Extract or derive data from `app-db` for views|`rf/reg-sub`|
|Access subscription data in a component|`@(rf/subscribe [...])`|
|Define how a side effect is actually performed|`rf/reg-fx`|
|Inject external data (time, localStorage) into a handler|`rf/reg-cofx` + `rf/inject-cofx`|
