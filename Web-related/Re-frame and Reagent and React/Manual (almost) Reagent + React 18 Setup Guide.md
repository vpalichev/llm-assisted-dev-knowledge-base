# Table of Contents

- [[#1. Initialize project]]
- [[#2. Install npm dependencies]]
- [[#3. Create folder structure]]
- [[#4. Create `shadow-cljs.edn`]]
- [[#5. Create `public/index.html`]]
- [[#6. Create `src/my_app/core.cljs`]]
- [[#7. Development]]
- [[#8. Production build]]

---

## 1. Initialize project

[[#Table of Contents|Back to TOC]]

```bash
mkdir my-reagent-app && cd my-reagent-app
npm init -y
```

## 2. Install npm dependencies

[[#Table of Contents|Back to TOC]]

```bash
npm install --save-dev shadow-cljs
npm install react@18 react-dom@18
```

## 3. Create folder structure

[[#Table of Contents|Back to TOC]]

```
my-reagent-app/
├── package.json
├── shadow-cljs.edn
├── public/
│   └── index.html
└── src/
    └── my_app/
        └── core.cljs
```

## 4. Create `shadow-cljs.edn`

[[#Table of Contents|Back to TOC]]

```clojure
{:source-paths ["src"]
 :dependencies [[reagent "1.2.0"]]
 :builds {:app {:target :browser
                :output-dir "public/js"
                :asset-path "/js"
                :modules {:main {:init-fn my-app.core/init}}}}}
```

## 5. Create `public/index.html`

[[#Table of Contents|Back to TOC]]

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>My Reagent App</title>
</head>
<body>
  <div id="app"></div>
  <script src="/js/main.js"></script>
</body>
</html>
```

## 6. Create `src/my_app/core.cljs`

[[#Table of Contents|Back to TOC]]

```clojure
(ns my-app.core
  (:require [reagent.dom.client :as rdom]))

(defonce root (atom nil))

(defn app []
  [:div
   [:h1 "Hello Reagent!"]])

(defn init []
  (let [container (.getElementById js/document "app")]
    (if-let [r @root]
      (rdom/render r [app])
      (let [r (rdom/create-root container)]
        (reset! root r)
        (rdom/render r [app])))))
```

## 7. Development

[[#Table of Contents|Back to TOC]]

```bash
npx shadow-cljs watch app
```

Open http://localhost:8020

## 8. Production build

[[#Table of Contents|Back to TOC]]

```bash
npx shadow-cljs release app
```