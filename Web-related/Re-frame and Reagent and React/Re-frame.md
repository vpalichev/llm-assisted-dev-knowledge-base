  ---
                                                                                                                                                                                                                                                                     Reagent renders your UI by calling functions.
                                                                                                                                                                                                                                                                      
  When you write:                                                                                                                                                                                                                                                     
  (defn my-component []
    [:div "hello"])
  Reagent calls my-component every time it needs to re-render it.

  ---
  Subscriptions are reactive — but only if Reagent is watching.

  When Reagent calls a function and sees a @(rf/subscribe ...) inside it, it says: "ok, I'm watching this value — when it changes, I'll call this function again."

  But if the deref happens outside a function Reagent is calling as a render, Reagent doesn't see it, doesn't track it, and never re-renders.

  ---
  Form-2 components have TWO functions.

  Sometimes you need local state (like the search box text). For that you write:

  (defn employees-panel []          ;; <- Reagent calls this ONCE at startup
    (let [filter-val (r/atom "")]   ;; <- creates the search box atom once
      (fn []                        ;; <- THIS inner fn is the actual render fn
        ...)))                      ;;    Reagent calls this one on every re-render

  Reagent only calls the outer function once to get the inner function back. After that, it only ever calls the inner function to render.

  ---
  The bug.

  I put the subscription in the outer function:

  (defn employees-panel []
    (let [{:keys [items loading?]} @(rf/subscribe [:employees])  ;; outer — runs ONCE
          filter-val (r/atom "")]
      (fn []  ;; inner render fn
        ...)))

  So the sequence was:
  1. Component mounts → outer fn runs once → fetch not done yet → loading? = true, items = []
  2. Fetch completes → app-db updates → but Reagent isn't watching, so inner fn never gets called again
  3. Screen stays frozen on "Loading…" forever

  ---
  The fix — move the subscription inside the inner fn:

  (defn employees-panel []
    (let [filter-val (r/atom "")]   ;; outer: setup only
      (fn []                        ;; inner: actual render
        (let [{:keys [items loading?]} @(rf/subscribe [:employees])]  ;; Reagent watches this
          ...))))

  Now Reagent tracks the subscription during every render call, sees the data change, and re-renders.

❯ dude, why do you have to make things so somplicated, to quote Avril                                                                                                                                                                                                                                                                       
● Fair point. The rule is simple:                                                                                                                                                                                                                                     
   
  @subscribe always goes inside the render function, never outside it.                                                                                                                                                                                                   
  That's it.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                
❯ subscribe is reframe?                                                                                                                                                                                                                                               
               
● Yes. @(rf/subscribe [...]) is re-frame. The @ is plain Clojure — it dereferences any atom/reactive thing. The rf/subscribe part is re-frame specifically.   