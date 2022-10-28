{:title "Learn You A Kee-Frame, Part 3"
 :layout :post
 :tags  ["clojure" "lyakf"]}

>  A wizard's staff has a knob on the end.
>
>  -- Traditional

It is time to write a more substantial example.
We are going to write a lifter's log, with records (haha!) of
weights lifted and fancy charts.  By pure chance I already
have such a thing and I love it, except for 2 deficiencies:
data entry sucks and it does not run on my phone.

## Requirements

* keeps track of the current weights
* knows about training programs and will suggest the next workout
* stores data from completed workouts
* produces the charts I crave, with lines going ever up
* runs on my phone so I have it with me when I work out
* runs offline because reception in my basement is poor

Long term I want configurable training programs,
but to get started we will bake in a few examples.

```clojure
(def glp-upper-split
  (map->Program
   {:slug "glp-upper-split"
    :name "Greyskull LP Upper Body Split"
    :min-days 2
    :workouts
    [[(->ExerciseRef "squat" :linear repout-suggestion)
      [(->ExerciseRef "bench" :linear repout-suggestion)
       (->ExerciseRef "overhead" :linear repout-suggestion)]
      (->ExerciseRef "deadlift" :linear deadlift-suggestion)]]}))
```

Our first example.  1 workout with 3 exercises, then the program
repeats.  But do you see that extra pair of brackets around `bench` and
`overhead`?  That is the "upper body split" a.k.a. alternating exercises:
one workout is squat-bench-deadlift, the next one squat-overhead-deadlift.

Linear progressions are otherwise simple: you do the reps, increase the weight,
repeat.  Different rep schemes tend to be used, that's why the `:linear` progression
takes a suggestion template as argument.


```clojure
(def five-three-one
  (map->Program
   {:slug "five-three-one"
    :name "Five Three One"
    :min-days 2
    :workouts
    [[{:slug "squat" :progression :five-three-one :opts :five}
      {:slug "bench" :progression :five-three-one :opts :five}]
     [{:slug "deadlift" :progression :five-three-one :opts :five}
      {:slug "overhead" :progression :five-three-one :opts :five}]
     [{:slug "squat" :progression :five-three-one :opts :three}
      {:slug "bench" :progression :five-three-one :opts :three}]
      ;; and so on, cutting this short for brevity
    ]}))
```

And this is the second case we want to support: 5-3-1 has 4 phases
with multiple workouts each, with different percentages and repetitions.
There are multiple sessions for a given exercise, so you have to keep
track of that. And because the percentages and reps are different
in each phase, the `:five-three-one` progression takes the phase
as argument: `:five`, `:three`, `:one` and `:rec` (= a recovery phase).

There are also body weight exercises where the progression
is in the number of repetions and sets.  Kettlebells tend
to have big jumps between the weights, usually there is
a progression in the rep/set scheme *and* a weight progression.
We will ignore both of these for now, but in theory
we could make them work as well.


## The Wizard

I want to get started with the trickiest bit because
it has the most influence on the overall design.
The needs of the wizard drive what has to go into
the program definition, and since the program
definitions will go into re-frame's `app-db`
there would be a lot of breakage if this is done
as an afterthought.

You may notice that in our program definition,
there is no place to store the current state.
I found the program definitions hard
to work with, so I decided to do as much work
as possible with a simpler data structure.
I also needed something that I could put in subscriptions
and events.

```clojure
;; a selector is vector of indices that points to an exercise-ref
;; it is either [w x] as in Workout and eXercise
;; or  `{:path [w x a] :number-of-alternatives na} 
;; as in Workout, eXercise, Alternative, Number of Alternatives
(defrecord Selector [path number-alternatives])
```

Boo, `defrecord`, why you no take docstring?  Yeah I know,
a Java class is generated, but I sure wish `defrecord` would take a docstring.

Anyway, the neat thing is that a selector can do 2 tricks:
It can read from a program definition and give you the `ExerciseRef`
(that is the bit with `{:slug "squat" :progression :linear :opt "xxx"}`).

And it can update the programs state that lives in another part of `app-db`.
It is a map indexed by the numbers from the selector, but it only nests 2
levels deep meaning alternating exercises are not quite as irregular.

```clojure
(def data nil)
(def sels (current-selectors glp-upper-split data))
(pprint sels)
;; ({:path [0 0], :number-alternatives 0}
;;  {:path [0 1 0], :number-alternatives 2}
;;  {:path [0 2], :number-alternatives 0})
(def data (complete "@pr 600x3" glp-upper-split data (first sels)))
(pprint data)
;; {0 {0 {:completed "@pr 600x3"}}}
(def data (complete "440x2" glp-upper-split data (second sels)))
(pprint data)
;; {0 {0 {:completed "@pr 600x3"}, 1 {:completed "440x2"}}}
(complete-with-slugs "800x1" glp-upper-split data (nth sels 2)))
;; => [#{"squat" "bench" "deadlift"} {0 {1 {:alternative 1}}}]
```

I am using `complete` instead of `complete-with-slug` because
it is easier to thread around.  But when the program completes
we also get a set of completed exercises (note that "overhead" is
not in that set) - the weights for these exercises will be
increased for the next iteration.

### Events

There are no side effects yet, so all the additional event handlers
are `reg-effect-db`.  I am not showing them all,
but there is a button on the dev tab now that will set
distinct weights for all defined exercises.

```clojure
(defn complete-handler [db [_ selector repsets]]
  (let [slug (get-in db [:current :slug])
        program (-> db :programs (get slug))
        [completed-slugs data] (program/complete-with-slugs repsets program
                                                            (-> db :current :data)
                                                            selector)]
    (-> db
        (update :exercises
                #(reduce exercise/increment-exercise % completed-slugs))
        (update :current #(assoc % :data data)))))

(rf/reg-event-db :complete complete-handler)
```

When the last exercise of the program is completed we reduce
over the set of exercise slugs.  When the program does not complete,
we get `nil` and reduce does not change anything.

### Subscriptions

Now it gets interesting.  Remember part 2 where I proclaimed
that `re-frame` is `Elm` with a clojure twist?  As it turns out
re-frames signal graph is the greatest thing since sliced bread,
and for once I am not joking.

```clojure
(rf/reg-sub :workout-selectors
            (fn [_qv]
              [(rf/subscribe [:current-program])
               (rf/subscribe [:current])])
            (fn [[program {:keys [data]}] _]
              (program/current-selectors program data)))
```

I had a really hard time with the current workout selectors.
The plan was to put this in the transient part of `app-db`,
but it was janky. The update logic was horrible and
I was not even loading from storage yet.  But then it 
clicked - with re-frame, things do not need to be in `app-db`
so you can pass it to a view.  The current workout selectors
are derived data, you just make a materialized view and
let `re-frame` handle the rest.

And just like that, the tricky part was not all that tricky
anymore.  Have you ever heard the phrase "Erlang makes hard
things easy and easy things hard."?

> Re-frame makes hard things easy.
>
> -- Me (2022)

After that realization I abused subscriptions wherever I could.
If you are going to drink the cool-aid, why not DRINK ALL THE COOL-AID?

```clojure
(rf/reg-sub :current-workout-info
            (fn [_qv]
              [(rf/subscribe [:current-program])
               (rf/subscribe [:exercises])
               (rf/subscribe [:workout-selectors])
               (rf/subscribe [:current])])
            (fn [[program exercises selectors {:keys [data]}] _]
              (let [completed?     (program/mk-completed? data)
                    uncompleted    (remove completed? selectors)]
                (mapv (fn [sel]
                        (let [completed (completed? sel)
                              xref      (program/exercise-ref program sel)
                              exercise  (->> xref
                                             :slug
                                             exercises)]
                          {:exercise exercise
                           :selector sel
                           :repsets completed
                           ;; focus seems to go to the LAST element with auto-focus
                           :focus (first uncompleted)
                           :suggestion (program/wizard-suggestion xref exercises)}))
                      selectors))))
```

### The View

The user enters `repsets` - lines of the form "100x3" (= 100kg, 3 reps) or
"80x5x3" (= 80 kg, 3 sets of 5 reps).  I really do not like having to
tab through multiple text fields to e.g. enter an address.  Just give me
a textarea, kk thx bye.

So I made a parser using `instaparse` and that was the other pleasant surprise.
I really can not talk highly enough about `instaparse`.  In my experience
with parser generator/combinator libraries, you can have speed, ease of use and useful error
messages.  Choose any two.  Instaparse gives you all three.

Anyway, the parser is used for validating these repsets. Lots of suggestions
will be initially invalid because they say something like "80x3x2+".
This is bad UX but I don't want to spend time on UI  -
we are aiming for PWA, but React Native remains a backup plan.

I did spend time on making the controlled input field behave correctly.
I did not want to put these into `app-db` because a) apparently this can
lead to strange behaviour with fast typing and b) `app-db` is complicated
enough already.

I tried using local state, and I am now convinced that it is not possible to make
this work with only one atom.  With one atom either your on-change
wins which will break updates from subscriptions, or the subscriptions
win which means your text field will not change.

With two atoms, there are multiple ways to make it work.
I compare the incoming prop with the value changed
by `on-change`, and I discard the changed value if they match.
This works as long as your subscription will cause an event
with an equal value.

The other way is to let an incoming prop change override
the changed value.  But this caused flicker - I would delete
the trailing +, then get one event with a trailing +,
and another one with the correct new value.

Re-frame documentation points you
at [re-com][re-com_text_input] for this, but I found it hard
to understand because of all the abstraction - it is a workhorse,
not an example.

So here is my simple version - again, the subscription
has to echo your change, or this will not work.

```clojure
(defn- exercise-wizard [_ _ _ suggestion repsets]
  (let [prop-value (r/atom (or repsets suggestion))
        changed-value (r/atom nil)]
    (fn exercise-wizard-fn [{name :name :as _exercise}
                            selector focus suggestion repsets]
      (reset! prop-value (or repsets suggestion))
      (when (= @prop-value @changed-value)
        (reset! changed-value nil))

      (let [value    (or @changed-value @prop-value)
            invalid? (parser/field-invalid? value)
            swap-controlled-value #(reset! changed-value (-> % .-target .-value))]

        [:form {:on-submit (fn [evt]
                             (>evt [:complete selector value])
                             (.preventDefault evt))}
         ;; this is already getting too long so i only show the interesting bits
         ;; just image that there is a [:input {:on-change swap-controlled-value}]
         ]))))
```

## Odds and Ends

There is a full spec for app db.  It is given as an argument to `keeframe.core/start!`,
and it boggles my mind how much this helps.
I have not written much clojure since I changed jobs in 2014, and
I vividly remember hunting down typos in deeply nested maps.

But kids these days, they are not happy with `spec.alpha`!  Now there is `malli` too!
I was wondering if I had to choose a camp and so I evaluated both.

Now I think it does not matter.  They are both great libraries.  I like being
able to sprinkle little bits around the codebase with `spec.alpha`, and `malli` is probably nicer
if you want to do fancy transformations.

## Release Build on GH Pages

* [Demo: Learn you a Kee-Frame, Part 3][lyakf_part3]
* [Source code][lyakf_part3_source]

250 KB compressed.  The home screen may only show a form
with 3 input boxes, but there is some substance to the program
now.  Also additional libraries are pulled in

* Parsing: `instaparse`
* String formatting: `cuerdas`
* Utility: `medley`
* Time: `tick`

[re-com_text_input]: https://github.com/day8/re-com/blob/master/src/re_com/input_text.cljs
[lyakf_part3]: https://grmble.github.io/learn-you-a-keeframe/part3/
[lyakf_part3_source]: https://github.com/grmble/learn-you-a-keeframe/tree/part3
