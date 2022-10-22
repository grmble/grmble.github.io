{:title "Learn You A Kee-Frame, Part 2"
 :layout :post
 :tags  ["clojure" "lyakf"]}

Let's talk about [The Elm Architecture][TEA] for a bit.  You need 3 things:

* `Model`: the overall application state
* `View`: a function that takes that application state and produces Html
* `Update`: a function that takes the application state and an event and produces a new application state


Elm takes care of the rest.  Events emitted by the view are sent
to the update function, and any side effects will result in more calls
to the update function.  When the model changes, an async re-render is triggered.
All of this is done in a type safe manner (`View` really returns `Html Msg`, 
and `Update` returns `(Model, Cmd Msg)`).

This idea has spread far and wide with slightly different names for the update function. `fold` is common
in haskell circles, clojure programmers would probably call it a `reducer`.  And while plain
`Angular` or `React` only do the model/view bits, there are plenty of libraries that
fill in the gap (`Redux`, `NgRx`, ...)

I would argue that `reagent` is like `React`, and `re-frame` is TEA with a clojure twist.
Maybe not in the implementation details, but the concepts are similiar.
In particular, Elms `Cmd` is not a full blown monad - it's just a data structure,
so re-frames handling of side effects is not all that different.


## Re-Frame Concepts

* Views `subscribe` to a signal graph
  and they `dispatch` events (rarely: `dispatch-sync` for e.g. key presses)
* Event handlers are registered for events (`reg-event-db` and `reg-event-fx`)
* Subscription handlers transform application state into said signal graph (`reg-sub`)
* Side effects are just additional data that is interpreted by interceptors
  running *after* the event handler (`reg-fx` in a library somewhere - you just 
  use `reg-event-fx`).
* The concept of co-effects is new to me - this
  is effectful *input* for an event handler - it is provided
  by an interceptor that runs *before* the event handler (`reg-cofx` - probably
  in a library again, but you `inject-cofx` the interceptor)

`re-frame` is ... more modular?  More spread out?  At first it seemed to me
that subscriptions are littered all over the view code instead of nice
clean function arguments.  Event handlers too - the event handlers are
registered all around the code base.  There is no single top level
update function that calls other update functions that call other ...
hmmmm, maybe this is not so bad after all.

I think the main difference is really the lack of a type-safe compiler.
View functions with lots of arguments work much better with compiler
support.  Because typically you don't just pass an argument once,
oh no!  That argument gets passed around like a bad penny.
And without compiler support, every function call is an opportunity
for mistakes, especially with refactoring.

It also helps with `application state creep`.  As an application grows,
the sub-systems tend to need more and more state until all the subsystems
get all the state, just because one little function in one corner
needs some data from another corner.

## Kee-Frame additions and concepts

* Automatic `spec.alpha` validation for the application state
* Client-side router via `reitit`
* Computing URLs from route data (`k/path-for`)
* Browser history/navigation side effects (`:navigate-to`)
* Scrolling on navigation (`(:require [kee-frame.scroll])`)
* Helper function to start the application (`k/start!`)
* Helpers functions for event handler chaining (`k/reg-chain`)
* HTTP FSM via `glimt` as an even nicer option ajax calls
* Subscriptions for screen size breaking-points (`:breaking-point.core/*`)
* Error boundaries for components that break the rendering tree (`kee-frame.error/boundary`)
* Controllers (I think this concept was borrowed from `Keechma` and gave `Kee-frame` its name)

These do not need much explanation, except for controllers.

Controllers produce events from route data.  They have
a params function that is called with the route data
when the URL changes. If this function
goes from nil to non-nil, the `:start` function is called.
If it goes from non-nil to nil, the `:stop` function is called.
And if it goes from one non-nil value to another,
the `:stop` function is called with the old value,
then the `:start` function is called with the new value.


## Hold my Kee-Frame I am going in

Converting a reagent app to kee-frame is surprisingly easy:
the `start!` function only requires a root component,
everything else is optional.  That is how I started,
and then I added config loading, a spec and routing.

### Initialization

This replaces the intialization code in `main.cljs`
(formerly `core.cljs`):

```clojure
;; init! is called initially by shadlow-cljs (init-fn)
;; after-load! is called after every load
(defn ^:dev/after-load after-load! []
  (k/start! {;; renders into dom element #app - hard coded
             :root-component [loader [page/current-page]]
             :initial-db initial-db
             :app-db-spec ::grmble.lyakf.frontend.spec/db-spec
             :routes routes
             ;; route-hashing does not work with gh pages deployment
             ;; via compile time BASE-PATH
             :hash-routing? false}))
(defn init! []
  (>evt [:load-config])
  (after-load!)
  (println "init! complete"))
```

`>evt` and `<sub` are alternative syntax for `dispatch` and `subscribe`.
This is straight from the re-frame documentation: [Lambda Island Naming][LIN]

I don't want hard coded configuration - I want to be able to change things at deployment
time.  That is what `(>evt [:load-config])` and `[loader [page/current-page]]`
is about.  Again from the re-frame documentation: 
[Loading Initial Data][loading_initial_data]


```clojure
(rf/reg-event-fx :load-config
                 (fn [_ _]
                   {:http-xhrio {:uri             "config.json"
                                 :method          :get
                                 :response-format (ajax/json-response-format {:keywords? true})
                                 :on-success [:config-loaded]
                                 :on-error [:config-not-found]}}))
(rf/reg-event-db :config-loaded
                 (fn [db [_ config]]
                   (-> db
                       (assoc :config (merge (:config db) config))
                       (assoc-in [:ui :initialized?] true))))
(rf/reg-event-db :config-not-found
                 (fn [db _]
                   (assoc-in db [:ui :initialized?] true)))

(defn loader [body]
  (error/boundary
   (if (and true (<sub [:initialized?]))
     body
     [page/loading-page])))
```

I did play around with effect chaining and glimt, but for this case
I prefer plain re-frame.  Chaining does not buy much, especially with
named events.  I think glimt would be nice for a a view that has to 
change as the request progresses.

`error/boundary` on the other hand is *very*, *very* useful.  I have a habit
of writing `[:input "xxx"]` instead of `[:input {:value "xxx"}]` - without
`error/boundary` this will produce a white screen.  With `error/boundary` I get a helpful message telling 
me what went wrong.

The `(and true)` bit is for cheap "pre-rendered" html:  I toggle it to false
and copy the the app node from the browsers development tools into `index.html`.

And this is the subscription for `:initialized?`:

```clojure
(rf/reg-sub :initialized?
            (fn [db] (-> db :ui :initialized?)))
```

### Spec

But ... what config are we loading exactly?  I am glad you asked!  
Let us look at the spec:

```clojure
(s/def ::db-spec (s/keys :req-un [::ui
                                  ::config
                                  ;; there is more, but let's ignore that for now
                                  ]))

(s/def ::ui (s/keys :req-un [::initialized?]))
(s/def ::initialized? boolean?)

(s/def ::config (s/keys :req-un [::show-dev-tab?]))
(s/def ::show-dev-tab? boolean?)
```

The `:app-db-spec` option to `k/start!` starts validating
the app db everytime it changes (on initialization too!).
The error messages come from `expound` and are very helpful.


### Routing

```clojure
;; another compile time constant - base-path for router
(goog-define ^String BASE-PATH "")

(def routes
  [BASE-PATH
   ["/" :home]
   ["/data" :data]
   ["/config" :config]
   ["/dev" :dev]])
```

There are 3 options:

* a controller that puts the current route into app db
* using `k/case-route` to subscribe to the route data
* `(rf/subscribe [:kee-frame/route])` is used by `k/case-route`,
  this could be used in views that need the current route.
  It is undocument so there be dragons.

```clojure
(defn current-page []
  [:<>
   (k/case-route (comp :name :data)
                 :home [show-tab :home [home-tab]]
                 :data [show-tab :data [data-tab]]
                 :config [show-tab :config [config-tab]]
                 :dev [show-tab :dev [dev/dev-tab]]
                 [loading-tab])])
```

I chose option B because I was afraid of event ping-pong,
but I assume as long as you don't use `:navigate-to`
you should be safe.  `k/case-route` does mean that I propagate
the current route through `[show-tab]` into `[navbar]`.

`[navbar]` also needs to be changed to use `k/path-for`
for linking - otherwise the deployment to github pages
does not work.  Locally, routes are nested with a
compile time `BASE-PATH` of `""`, for github pages
the deployment script sets it to `"/learn-you-a-keeframe/partX"`.

The release build for github pages is done by [bb.edn][lyakf_bb_edn].

```clojure
(defn nav-link [current-tab title tab]
  [:a.navbar-item
   {:href  (k/path-for [tab])
    :class (when (= tab current-tab) "is-active")}
   title])

(defn navbar [tab]
  (r/with-let [expanded? (r/atom false)]
    [:nav.navbar.is-info>div.container
     [:div.navbar-brand
      [:a.navbar-item {:href  (k/path-for [:home])
                       :style {:font-weight :bold}} "Learn You A Kee-Frame"]
      [:span.navbar-burger.burger
       {:data-target :nav-menu
        :on-click #(swap! expanded? not)
        :class (when @expanded? :is-active)}
       [:span] [:span] [:span]]]
     [:div#nav-menu.navbar-menu
      {:class (when @expanded? :is-active)}
      [:div.navbar-start
       [nav-link tab "Home" :home]
       [nav-link tab "Data" :data]
       [nav-link tab "Config" :config]
       (when (<sub [:show-dev-tab?])
         [nav-link tab "Dev" :dev])]]]))
```

### The Dev Tab

For now, it just contains documenation links.  I tend
to have lots of browser tabs open.  With this example
project they were so many that I had trouble finding the
project tab again.

```clojure
(defn dev-tab []
  [:section.section
   [:h1.title "Development Tools"]
   [:h2.subtitle "Useful Links"]
   [:ul
    [link-entry "Bulma Docs" "https://bulma.io/documentation/"]
    [link-entry "Clojure Spec Guide" "https://clojure.org/guides/spec"]
    ;; there is more, if you want the full list check the source code
    ]])
```

Now I can close all those browser tabs because I can easily
open the documentation page again.

Be aware that when the dev tab is configured away,
it is still present in the code.  A random user will not find
it.  But anybody who knows about it can just append `#/dev` to the browser
location and will see the content.

That is precisely why I like the method.  As a real world example,
I am in a team for a company wide search engine.  We use
a routing parameter without UI to control the display of
the search engine score.  We can toggle it on to diagnose problems,
but the users do not see it and we not have to answer questions
about it.

But a determined attacker will read the javascript, and 
he will find your little easter egg.  Search engine scores
or documentation links are fine.  Admin passwords would be
very, very bad.


## Odds and ends


### Error printing
This prints errors to the
browser console instead of the terminal.  I find
this much nicer when playing around in the browser,
and I did not know about this until now.

```clojure
(enable-console-print!)
```

### Components vs function calls

`[my-component]` is using a component,
`(my-component)` is a function call.  If `my-component` subscribes
to events, this leads to a re-frame error pointing you to
[Use a Subscription in an Event Handler][subscription_in_event_handler].
This is a bit misleading: it explains a more advanced method of
producing that error.

### Subscriptions vs Local State vs Props

As of right now, the example program uses all three.

* The initialization code uses app-db.  An argument could be
  made that the `initialized?` flag does not need to be
  in app db.  But `:config` is stored in the app db, so
  I kept it as simple as possible and put everything there.
* The navigation bar still uses a reagent atom for its
  burger menu toggle (`:expanded?`).  If some other component
  would need that information I would put it in app db,
  but it doesn't so I didn't.
* I chose to use `k/case-route` to render the component
  for the current tab, and the current tab is passed as a prop
  to `[navbar]` and `[nav-link]`.  This is still okay but
  borderline: if other views need that information 
  I will revisit that decision.

## Release Build on GH Pages

[Demo: Learn you a Kee-Frame, Part 2][lyakf_part2]

211 KB compressed.  Part 1 was a glorified `Hello world`.  Part 2
does not look much different in the browser, but it does perform
some work and pulls in more libraries.

* Spec Validation: `kee-frame`, `spec.alpha`
* Configuration loading: `re-frame`, `re-frame-http-fx`, `cljs-ajax`
* Routing: `kee-frame`, `reitit`

[TEA]: https://guide.elm-lang.org/architecture/
[LIN]: https://day8.github.io/re-frame/correcting-a-wrong/#lambdaisland-naming-lin
[loading_initial_data]: https://day8.github.io/re-frame/Loading-Initial-Data/
[subscription_in_event_handler]: https://day8.github.io/re-frame/FAQs/UseASubscriptionInAnEventHandler/
[lyakf_bb_edn]: https://github.com/grmble/learn-you-a-keeframe/blob/master/bb.edn
[lyakf_part2]: https://grmble.github.io/learn-you-a-keeframe/part2/
