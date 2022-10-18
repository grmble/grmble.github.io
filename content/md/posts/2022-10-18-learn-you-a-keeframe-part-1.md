{:title "Learn You A Kee-Frame, Part 1"
 :layout :post
 :tags  ["clojure"]}


## A minimal shadow-cljs project

```bash
yarn add -D shadow-cljs
```

In `shadow-cljs.edn`:
```clojure
{:source-paths ["src" "test"]
 :dev-http {8080 "public"}
 :builds
 {:app
  {:target :browser
   :modules {:main {:init-fn grmble.lyakf.frontend.app/init}}}

  :test
  {:target :node-test
   :output-to "public/js/test.js"
   :autorun true}}}
```

In `grmble.lyakf.frontend.app` there is a hello-world function `init`.

In `public/index.html` there is a minimal html file that loads `js/main.js`.

And there is a very basic test:

```clojure
(ns grmble.lyakf.frontend.app-test
  (:require
   [cljs.test :refer [deftest is]]
   [grmble.lyakf.frontend.app :as app]))

(deftest init-test
  (is (nil? (app/init)))
  (is (= 1 1)))
```

Using `yarn shadow-cljs watch app test` the application compiled (with hot reload)
and the tests are executed.
