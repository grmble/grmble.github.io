{:title "Learn You A Kee-Frame, Part 3"
 :layout :post
 :tags  ["clojure" "lyakf"]}

>>  A wizard's staff has a knob on the end.
>>
>>  -- Traditional

It is time to write a more substantial example.
We are going to write a lifter's log, with records (haha!) of
weights lifted and fancy charts.  By pure chance I already
have such a thing and I love it, except for 2 deficencies:
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
I found it hard to work with these deeply nested
data structures, so I decided to put the mutable
state in a separate location.  I also needed a
something that I could put in a subscription so
the wizard can display the current workout
and dispatch an event when an exercise is completed.

```clojure
;; a selector is vector of indices that points to an exercise-ref
;; it is either [w x] as in Workout and eXercise
;; or  `{:path [w x a] :number-of-alternatives na} 
;; as in Workout, eXercise, Alternative, Number of Alternatives
(defrecord Selector [path number-alternatives])
```

Boo, `defrecord`, why you no take docstring?  Yeah I know,
java class, but I sure wish `defrecord` would take a docstring.

Anyway, the neat thing is that a selector can do 2 tricks:
It can read from a program definition and give you the `ExerciseRef`
(that is the bit with `{:slug "squat" :progression :linear :opt "xxx"}`).

And it can update the programs state that lives in another part of `app-db`.

```clojure
;;; repl session
```
