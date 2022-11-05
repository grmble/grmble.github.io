{:title  "The Pouchening"
 :layout :post
 :draft? true
 :tags   ["clojure" "loglifter" "pouchdb"]}

Superficially, PouchDB looks similiar to local storage.  You store
json documents with a unique id.  For our use case,
[one database per user][one_db_per_user] is recommended as
well - so the keys would be the same as in a local storage
backend.

So how hard can it be, right?

## "The Pouchening"

Turns out that under the hood, it is very different.

* API is async - promises instead of sync results
* Local conflicts can be handled by the user - e.g. a popup
  informing about the conflict, and the possibility to re-submit
  or cancel
* Replication conflicts have to be queried and can
  either be merged automatically or with guidance from
  the user.  Otherwise a deterministic winner will
  be chosen, one set of changes is "hidden".  This may
  involve some kind of event sourcing - easier than
  trying to find the diffs
* CouchDB does not know about `spec.alpha`, so we have to validate
  before hitting the database.  A failed validation must
  cancel the write to the database.  Or alternatively,
  producing JSON spec e.g. from Malli?  I would like
  to avoid this.
* Failed validation must cancel the write to the database
* Incoming replicated database changes should go into app
  db as soon as possible
* We have to store the `_rev` fields somewhere,
  preferrably in the transient part of app db.  Otherwise
  we will always trigger 2 repaints, and because
  of the changed revision the values will not be equal.


### Current Program

`:current` contains the current progams `:slug` and the program data.
These two are tied together - changing the slug will invalidate
the data, so we will store them in a document named `current`.

Conflict resolution here is tricky.  Let's try to prefer
the entry with more data.  Or let the user decice.

### Exercises

The weights will be changed on program completion and
on resets.  Infrequently the exercises themselves might be
re-configured.  Store each exercise in a separate document
 `exercise/{{slug}}`.  This allows a simple `allDocs` query.

Conflict resolution: higher weight wins?  Or user decides.

### Programs

Same procedure as for exercises.

Conflict resolution? User?

### History

Historic data is stored in the user editable external format
(see line parser) under `history/{{date}}`.  All exercises
for the day are put in one JSON array.  On conflict: join
the arrays and remove duplicate lines?




[pouchdb]: https://pouchdb.com/
[one_db_per_user]: https://github.com/pouchdb-community/pouchdb-authentication/blob/master/docs/recipes.mdhttps://github.com/pouchdb-community/pouchdb-authentication/blob/master/docs/recipes.md#some-people-can-read-some-docs-some-people-can-write-those-same-docs
