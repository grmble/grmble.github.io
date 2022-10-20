{:title "Learn You A Kee-Frame, Part 2"
 :layout :post
 :tags  ["clojure" "lyakf"]}

Let's talk about [The Elm Architecture][TEA] for a bit.  You need 3 things:

* `Model`: the overall application state
* `View`: a function that takes that application state and produces Html
* `Update`: a function that takes the application state and an event and produces a new application state


Elm takes care of the rest.  Events emitted by the view are sent
to the update function, and any side effects will result in more calls
to the update function.  After model changes, an async re-render is triggered.
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
And if goes from one non-nil value to another one,
the `:stop` function is called with the old value,
then the `:start` function is called with the new value.




[TEA]: https://guide.elm-lang.org/architecture/
