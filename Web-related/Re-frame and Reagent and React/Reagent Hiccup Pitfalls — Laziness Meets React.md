# Reagent/Hiccup Pitfalls — Laziness Meets React

Common bugs when ClojureScript's lazy sequences meet Reagent's Hiccup-to-React rendering. These come up repeatedly in real projects — especially as component files grow large.

See also: project-level version with full code context in `docs/reagent-hiccup-pitfalls.md` of any re-frame project.

---

## 1. `for` inside a Hiccup vector doesn't splice

`(for ...)` returns a lazy sequence. Inside `[:tr ... (for ...) ...]`, it becomes **one child** (a nested group), not N individual siblings. Columns misalign.

**Fix:** Use `into` to build a flat vector, or put `for` as the last/only child where Reagent handles it naturally.

```clojure
;; BAD
[:tr [:td "a"] (for [i (range 3)] ^{:key i} [:td i]) [:td "z"]]

;; GOOD
(into [:tr [:td "a"]] (conj (mapv (fn [i] [:td i]) (range 3)) [:td "z"]))
```

## 2. Missing `^{:key}` on list items

React can't track items without keys. Reordering, inserting, or deleting causes wrong items to update. Never use `(random-uuid)` as a key — it forces full re-mount every render.

```clojure
(for [item items]
  ^{:key (:id item)}   ;; stable, unique
  [:li (:name item)])
```

## 3. Subscriptions inside `for` need their own component

A subscription inside a `for` loop (in the parent's render body) won't trigger independent re-renders. Extract each item into its own `[component]` so it owns its subscription lifecycle.

```clojure
;; BAD — sub inside for in parent body
(for [id ids]
  ^{:key id} [:div (count @(rf/subscribe [:data-for id]))])

;; GOOD — each item is a component
(defn- item-row [id]
  (let [data @(rf/subscribe [:data-for id])]
    [:div (count data)]))

(for [id ids]
  ^{:key id} [item-row id])   ;; square brackets = component
```

## 4. Conditional buttons shift layout

`(when condition [:button "X"])` makes buttons appear/disappear, pushing siblings to different positions across rows.

**Fix:** Fixed-width placeholder slots:

```clojure
(defn- btn-slot [width-px child]
  [:span {:style {:display "inline-block" :width (str width-px "px")
                  :text-align "center"}}
   child])

[btn-slot 50 (when show? [:button "Open"])]   ;; empty space when hidden
```

## 5. Form-2: forgetting the inner function

If a component creates local state (e.g., `r/atom`), it must return an inner `fn` for rendering. Without it, every parent re-render creates a new atom — state resets.

```clojure
;; BAD — atom recreated every render
(defn my-input []
  (let [val (r/atom "")]
    [:input {:value @val :on-change #(reset! val (-> % .-target .-value))}]))

;; GOOD — outer runs once, inner renders
(defn my-input []
  (let [val (r/atom "")]
    (fn []
      [:input {:value @val :on-change #(reset! val (-> % .-target .-value))}])))
```

If the component takes props, the inner `fn` must repeat the same args.

## 6. `(component arg)` vs `[component arg]`

- `(my-comp data)` — plain function call. Inlines hiccup. No lifecycle, no independent re-render, subscriptions don't work properly.
- `[my-comp data]` — Reagent component mount. Gets its own React lifecycle, subscription tracking, and reconciliation.

**Rule:** Always use square brackets for components that have subscriptions or local state.

## 7. Forward references — `defn` order matters

ClojureScript compiles top-to-bottom. Using a private helper before it's defined causes clj-kondo warnings or runtime `nil`.

```clojure
(declare btn-slot)   ;; forward declaration at top

;; ... use btn-slot in components above its definition ...

(defn- btn-slot [w child] ...)   ;; defined later
```

---

## Quick Reference

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| `for` between siblings | Column misalignment | `into` for flat vector |
| Missing `^{:key}` | Wrong items on list change | Stable unique key |
| Subs in `for` | Stale data, no re-render | Extract into own `[component]` |
| Conditional `when` | Layout shifts | Fixed-width `btn-slot` |
| Form-2 missing inner fn | State resets | Return `(fn [] ...)` |
| `(comp)` vs `[comp]` | No lifecycle | Square brackets |
| Forward refs | Unresolved symbol | `(declare ...)` |
