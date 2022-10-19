{:title "Learn You A Kee-Frame, Part 1"
 :layout :post
 :tags  ["clojure" "lyakf"]}

I am taking a look at [kee-frame][keeframe].  This also implies 
[re-frame][reframe] - I've read about it, there seemed to be a lot of boilerplate, 
but I've never used it.

But I continue to hear good things about it, so here we are.


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

`yarn shadow-cljs watch app test` starts he application (with hot reload)
and the tests are executed.


## Kee-Frame Dependencies

I tried adding the kee-frame dependency to `shadow-cljs.edn`,
but there seems little additional tooling.  E.g. how do
I get a dependency tree?  Because kee-frame has not updated
in some months, and it is pulling in some outdated dependencies.

So I switched to using `deps.edn` for dependencies.

This is the new `shadow-cljs.edn`:
```clojure
{:deps {:aliases [:shadow]}

 :dev-http {8080 "public"}

 :builds
 {:app
  {:target :browser
   :modules {:main {:init-fn grmble.lyakf.frontend.app/init!
                    :preloads [re-frisk.preload]}}}


  :test
  {:target :node-test
   :output-to "public/js/test.js"
   :autorun true}}}
```

And this is `deps.edn`:
```clojure
{:deps {kee-frame/kee-frame {:mvn/version "1.3.2"}

        reagent/reagent                       {:mvn/version "1.1.1"}
        ;; these are all kee-frame deps - then ran neil upgrade
        re-frame/re-frame                     {:mvn/version "1.3.0"}
        re-chain/re-chain                     {:mvn/version "1.3"}
        metosin/reitit-core                   {:mvn/version "0.5.18"}
        glimt/glimt                           {:mvn/version "0.2.2"}
        day8.re-frame/http-fx                 {:mvn/version "0.2.4"}
        cljs-ajax/cljs-ajax                   {:mvn/version "0.8.4"}
        com.taoensso/timbre                   {:mvn/version "5.2.1"}
        venantius/accountant                  {:mvn/version "0.2.5"}
        org.clojure/core.match                {:mvn/version "1.0.0"}
        expound/expound                       {:mvn/version "0.9.0"}
        breaking-point/breaking-point         {:mvn/version "0.1.2"}
        pez/clerk                             {:mvn/version "1.0.0"}}

 :paths ["src" "test"]

 :aliases
 {:shadow
  {:extra-deps {thheller/shadow-cljs          {:mvn/version "2.20.5"}
                re-frisk/re-frisk             {:mvn/version "1.6.0"}}}}}
```

I just copied kee-frames dependencies from its `deps.edn`,
then I ran `neil upgrade` to upgrade them all.  I also
added `re-frisk`. Apparently there is also a `re-frame-10x` 
which is similiar, but more on that later.

The project can now be started in two ways:

```bash
yarn shadow-cljs watch app test
clojure -M:shadow -m shadow.cljs.devtools.cli watch app test
```

The second one is a bit of a mouthful, but including the
main function in the `:shadow` alias breaks the start via
yarn, and I prefer that one.  This could be solved
by using a second alias to provide the main namespace.

What I am actually doing is the third way:

```bash
yarn shadow-cljs start  # to start the server
```

Then I start the repl from [calva][calva].  I think the `deps.edn`
method works as well, but I choose `shadow-cljs` - this lets
me toggle automatic test execution on and off.  And by using the
server, re-connecting is less painful if the dependencies change
or you hose your repl in some way.


## A Little Bit Of Reagent

[Reagent][reagent] is a popular clojure React wrapper,
and it is used as the view layer in [re-frame][reframe].
I've never used reagent either, but I've used plenty
of react wrappers.

So the plan is to set up a basic reagent SPA, then
take my time and convert it to a full featured
re-frame application, taking kee-frame shortcuts as
much as possible.

I am stealing the page layout from the luminus
template.  It is a pretty standard [bulma][bulma]
layout, but I find the hiccup shortcuts 
for classes, ids and deeply nested html
very interesting.


```clojure
(defonce session (r/atom nil))

(defn nav-link [uri title page]
  [:a.navbar-item
   {:href   uri
    :class (when (= page (:page @session)) "is-active")}
   title])

(defn navbar []
  (r/with-let [expanded? (r/atom false)]
    [:nav.navbar.is-info>div.container
     [:div.navbar-brand
      [:a.navbar-item {:href "/" :style {:font-weight :bold}} "Learn You A Kee-Frame"]
      [:span.navbar-burger.burger
       {:data-target :nav-menu
        :on-click #(swap! expanded? not)
        :class (when @expanded? :is-active)}
       [:span] [:span] [:span]]]
     [:div#nav-menu.navbar-menu
      {:class (when @expanded? :is-active)}
      [:div.navbar-start
       [nav-link "#/" "Home" :home]
       [nav-link "#/about" "About" :about]]]]))
```

This does not seem like much, but as usual clojure code is very dense.

This is the markup for a bulma page header.  I've done this in
HTML, JSX, and all kinds of templating systems, usually you have to scroll
around to see the whole thing.  Me gusta hiccup.

And what if I told you that the snippet also contains the code
for handling the burger menu?  CSS media queries turn it off above
1024 pixels width, so make your window smaller. 
Clicking the burger menu then toggles the `:is-active` class - 
see the `(swap! expanded? false)`.

* `session` and `expande` are `ratoms` - what goes for state in reagent
  programs.  `session` is the global application state (currently unused),
  `expanded?` is component local state.  A fresh atom is created when
  the component is mounted.
* `nav-link` and `navbar` are reagent components.  Any arguments are props.
* Some very interesting shortcuts in there.  I invite to inspect
  the navbar in your developer console and see what `:nav.navbar.is-info>div.container`
  turns into.  Or `:div#nav-menu.navbar-menu`
* `hiccup` is the name for this style of clojure html templating.
  The original library is for server side rendering, but
  reagent adopts the syntax as a JSX replacement.

## Release Build on GH Pages

[https://grmble.github.io/learn-you-a-keeframe/part1/](https://grmble.github.io/learn-you-a-keeframe/part1/)

78 KB compressed.  Not a bad start.


[reagent]: https://reagent-project.github.io/
[reframe]: https://day8.github.io/re-frame/re-frame/
[keeframe]: https://github.com/ingesolvoll/kee-frame
[bulma]: https://bulma.io/
[calva]: https://calva.io/
