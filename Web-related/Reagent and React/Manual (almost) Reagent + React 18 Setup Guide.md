

## 1. Initialize project
```bash
mkdir my-reagent-app && cd my-reagent-app
npm init -y
```

## 2. Install npm dependencies
```bash
npm install --save-dev shadow-cljs
npm install react@18 react-dom@18
```

## 3. Create folder structure
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
```clojure
{:source-paths ["src"]
 :dependencies [[reagent "1.2.0"]]
 :builds {:app {:target :browser
                :output-dir "public/js"
                :asset-path "/js"
                :modules {:main {:init-fn my-app.core/init}}}}}
```

## 5. Create `public/index.html`
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
```clojure
(ns my-app.core
  (:require [reagent.dom :as rdom]))

(defn app []
  [:div
   [:h1 "Hello Reagent!"]])

(defn init []
  (rdom/render [app]
               (.getElementById js/document "app")))
```

## 7. Development
```bash
npx shadow-cljs watch app
```

Open http://localhost:8020

## 8. Production build
```bash
npx shadow-cljs release app
```