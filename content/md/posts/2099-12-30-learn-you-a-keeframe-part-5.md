{:title "Learn You A Kee-Frame, Part 5"
 :layout :post
 :tags  ["clojure" "lyakf"]}

> It was my pork chop. But that's ok. I ate his dog food.
>
> -- Bam Bam Bigelow 

The program is not finished, but this part is when it becomes useful.
As I am typing this I just finished my first training.  Haha!  Hahaha!

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
                 [(rf/inject-cofx :grmble.lyakf.frontend.storage.local/load [:current :exercises :programs])]
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

The data tab now shows our history entries.  For this I am using
[codemirror][codemirror].



[codemirror]: https://codemirror.net/
