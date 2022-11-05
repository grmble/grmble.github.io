{:title "Learn You A Kee-Frame, Part 5"
 :layout :post
 :tags  ["clojure" "lyakf"]}

> It was my pork chop. But that's ok. I ate his dog food.
>
> -- Bam Bam Bigelow 

The program is not finished, but this part is when it becomes useful.
I just used it in a training session for the first time.  Haha!  Hahaha!

## Storage

I had a hard time deciding what to use for storage.  I was pretty far
along with a PouchDB version.  I very much want some kind of sync
between my phone and other devices, PouchDB would be one option.
But this program might be useful for non-technical people,
so I want to make it rock solid.  And so far I have not come
up with a convincing strategy for dealing with eventual conflicts.

But the storage layer is built with an eye to PouchDB.
In particular, we are producing bog standard JSON that
could be used in a JSON aware database.

On initialization, there is code in place that reads
`:current`, `:exercises` and `:programs` from local storage.
Right now I injecting them as coeffects, this is only
possible with local storage because it is synchronous.
With PouchDB or IndexedDB, we would have to dispatch
additional events, like we do after reading `config.json`
over http.

```clojure
(rf/reg-event-fx :config-loaded
                 [(rf/inject-cofx :grmble.lyakf.frontend.storage.local/load 
                   [:current :exercises :programs])]
                 (fn [{:keys [db current exercises programs]} [_ config]]
                   {:db (cond-> (assoc db :config (merge (:config db) config))
                          true      (assoc-in [:transient :initialized?] true)
                          current   (assoc :current (foreign/js->current current))
                          exercises (assoc :exercises (foreign/js->exercises exercises))
                          programs  (assoc :programs (foreign/js->programs programs)))}))
```

This is the event handler for the `:complete` event that is
dispatched by the program wizard.

2 new side effects are used: `:store` writes the `:current` part
of app db to local storage. `:append-history` writes history entries.
This will be used later on to produce our fancy charts (with lines
going ever up!!!).

The current weights for all exercises are now stored under `[:current :weights]`.
This is for PouchDB: if there is a conflict with writing this, there might
be changes that are hidden to the user, but at least it would be consistent.

Now `:exercises` only changes for configuration changes, same thing with
`:programs`.

```clojure
(defn complete-handler [{:keys [db current-date]} [_ selector repsets]]
  (let [slug                   (get-in db [:current :slug])
        program                (-> db :programs (get slug))
        xref                   (program/exercise-ref program selector)
        [completed-slugs data] (program/complete-with-slugs repsets program
                                                            (-> db :current :data)
                                                            selector)
        db (-> db
               (update :current
                       #(reduce (incrementer (:exercises db)) % completed-slugs))
               (update :current #(assoc % :data data)))]
    {:db db

     :grmble.lyakf.frontend.storage.local/store
     {:kvs {:current (foreign/current->js (:current db))}
      :db db}

     :grmble.lyakf.frontend.storage.local/append-history
     {:current-date current-date
      :slug (:slug xref)
      :repsets repsets}}))
```

Note the use of namespaced keywords.  The idea here is that additional
storage backends would provide the same effects and co-effects,
just in different namespaces.  This would allow to switch between
them.  We would have to stop using coeffects for reading though.

## History

The data tab now shows our history entries.

Codemirror is very powerful, we could make a
`lezer` version of our entry parser and we would
have syntax hightlighting.


```clojure
;; the ^js hint fixes the "can  not infer" warning
(defn- codemirror-content [^js view]
  (. (. view -state) sliceDoc))


;; the inner / outer pattern comes straigt from the docs
;; https://day8.github.io/re-frame/Using-Stateful-JS-Components/
(defn codemirror-inner []
  (let [view     (atom nil)
        init!    (fn [comp]
                   (let [history     (:history (r/props comp))
                         state       (.-state @view)
                         length      (or (some-> state .-doc .-length)
                                         0)
                         transaction (.update state #js {:changes #js {:from 0
                                                                       :to length
                                                                       :insert history}})]
                     (.dispatch @view transaction)))]

    (r/create-class
     {:reagent-render         (fn []
                                [:<>
                                 [:div#codemirror]
                                 [:div.control
                                  [:button.button.is-primary
                                   {:on-click
                                    #(>evt [:save-history (codemirror-content @view)])}
                                   "Save"]]])

      :component-did-mount    (fn [comp]
                                (let [elem  (js/document.getElementById "codemirror")
                                      cm    (EditorView.
                                             #js {:extensions #js [basicSetup]
                                                  :parent elem})]
                                  (reset! view cm)
                                  (init! comp)))
      :component-did-update    init!
      :display-name            "codemirror-inner"})))


(defn codemirror-outer []
  (let [transient (<sub [:transient])]
    (fn []
      ;; it is a map so it can be accessed as (r/props cmop)
      [codemirror-inner transient])))
```

For now we just detect errors using our insta-parse parser
(see `parser/parse-history`) and display the affected
line numbers via flash.

The history is not usually loaded. For charting, we will only
use a limited subset of the data, so we do not keep it in memory.
I actually worked for quite a while on this because I forgot
about kee-frame's controllers.  Duh.  This is just what
they are for, and the code is so much nicer than my
pure re-frame attempt using `reg-sub-raw`:

```clojure
(k/reg-controller :data
                  {:params (fn [match]
                             (when (= (get-in match [:data :name]) :data)
                               true))
                   :start  [:load-history]
                   :stop   [:dispose-history]})
```

## Resetting Exercises and Snapshots

In the wizard, there is now a reset button next to each exercise.
This toggles a modal form for changing the exercise's current weight:

```clojure
(defn reset-exercise [exercise show? toggle]
  (r/with-let [value (r/atom (:weight exercise))]
    (let [reset-exercise! (fn [evt]
                            (>evt [:reset-exercise (:slug exercise) @value])
                            (toggle evt))]
      [:form.modal {:class (when @show? :is-active)
                    :on-submit reset-exercise!}
       [:div.modal-background]
       [:div.modal-card
        [:header.modal-card-head
         [:p.modal-card-title (str "Reset " (:name exercise))] ; without the str a very strange error ...
         [:button.delete {:aria-label "close" :type "button" :on-click toggle}]]
        [:section.modal-card-body
         [:input.input {:on-change #(reset! value (-> % .-target .-value))
                        :value @value}]]
        [:footer.modal-card-foot
         [:button.button.is-success
          {:type "submit"
           :on-click reset-exercise!}
          "Save"]
         [:button.button {:on-click toggle} "Cancel"]]]])))
```

There is also a snapshotting feature:  since I am already using the program,
trying things out on my phone now messes up my data.  The history
is easily cleaned up on the data tab, but the current wizard state
is complicated.  So on the dev tab, there are 2 new buttons:
"Snapshot" and "Restore".  I make a snapshot before loading a new build.
This stores the entire `:current` part of the model into local storage
under a different name.  When I am done playing around I press `:restore`,
this replaces regular `:current` storage with the snapshot and also
loads it back in.

```clojure
(rf/reg-event-fx :snapshot-current
                 (fn [{:keys [db]} [_]]
                   {:db db

                    :grmble.lyakf.frontend.storage.local/store
                    {:kvs {:snapshot (foreign/current->js (:current db))}
                     :db db}}))

(rf/reg-event-fx :restore-snapshot
                 [(rf/inject-cofx :grmble.lyakf.frontend.storage.local/load [:snapshot])]
                 (fn [{:keys [db snapshot]} [_]]
                   {:db (cond-> db
                          snapshot  (assoc :current (foreign/js->current snapshot)))

                    :grmble.lyakf.frontend.storage.local/store
                    {:kvs {:current snapshot}
                     :db db}}))
```

## Release Build on GH Pages

* [Demo: Learn you a Kee-Frame, Part 5][lyakf_part5]
* [Source code][lyakf_part5_source]

340 KB compressed.  Codemirror is a big dependency, but so far I am happy
with the load times.  My phone is a couple of years old now and it runs
like a charm.

[lyakf_part5]: https://grmble.github.io/learn-you-a-keeframe/part5/
[lyakf_part5_source]: https://github.com/grmble/learn-you-a-keeframe/tree/part5
[codemirror]: https://codemirror.net/
