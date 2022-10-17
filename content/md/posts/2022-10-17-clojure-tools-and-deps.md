{:title "Clojure Tools and Deps"
 :layout :post
 :tags  ["clojure"]
 :toc true}

I used to write a lot of clojure, but I have not for some years.  In particular, I am
pretty used to leiningen, but `deps.edn` is all new to me.  The information here is
mostly from [deps_and_cli_guide] and [deps_and_cli_ref]

## Command line switches

* `-X` to activate aliases and call a function from the command line with a map argument.
  The map is given as key-paths and values.
* `-Sdescribe`, `-Spath`, `-Stree`: env and cmd parsing info, classpath, dependency tree.
* `-A` activates aliases. The aliases are defined under `:aliases` in `deps.edn`. 
  Typically they get `:extra-paths` and `:extra-deps`.
* `-T` for tools - these do not use the project deps or paths, only from the alias plus . in paths.
* `-M` is outdated - but it takes a namespace, separated by a colon.  NS must have public `-main` defn,
  no keywordification of arguments.


### Builtin deps alias

The builtin deps alias does include the project classpath, it provides additional programs.

* `-X:deps tree` - print dependency tree
* `-X:deps list` - print depencency list and license information
* `-X:deps aliases` - print all aliases
* `-X:deps mvn-pom` - generate or update a pom.xml
* `-X:deps git-resolve-tags` - resolve git coordinates tags to shas and update deps.edn
* `-X:deps find-versions` - find versions for `:lib` or `:tool`

It also has `help/doc` and `help/dir` to introspect how a tool can be used.

```bash
clojure -A:deps -Ttools help/dir  # listing functions for the tools tool
```

### Builtin tools tool

* `clj -Ttools list` - list installed tools
* `clj -Ttools install-latest :lib some.dep/coords` - install latest version of tool.  If already installed works also with the tool name `:tool toolname`
* `clj -Ttools show tooname` - show usage information

## Dependencies

In `deps.edn`, under `:deps`.  The library name is the key, followed by the version.

* Maven: `{:mvn/version "x.y.z"}`
* Local: `{:local/root "../relative-dir"}`
* Local Jar: `{:local/root "/path/to/blubb.jar"}`
* Git: `{:git/tag "some-tag" :git/sha "shorthash"}`


### Git Dependencies

For Git, the libary name is `io.github.grmble/repo`.

Use `git rev-parse --short some-tag` to get the hash.

For remote repos: `clojure -X:deps git-resolve-tags`

### Cognitect test runner

Finds and run `clojure.test` tests in your program.

Run the tests using `clj -X:test`

```clojure
{:aliases
 {:test {:extra-paths ["test"]
         :extra-deps {io.github.cognitect-labs/test-runner
                      {:git/url "https://github.com/cognitect-labs/test-runner.git"
                       :sha "9e35c979860c75555adaff7600070c60004a0f44"}}
         :main-opts ["-m" "cognitect.test-runner"]
         :exec-fn cognitect.test-runner.api/test}}}
```

Or `neil add test`


## Tools

### deps-new

```bash
clojure -Ttools install-latest :lib io.github.seancorfield/deps-new :as new
```

For help

```bash
clojure -A:deps -Tnew help/doc
```

Create an application (also try `lib` or `template` instead of `app`)

```bash
clojure -Tnew app :name user/appname
```

Create a new project from a template

```bash
clojure -Tnew :template io.github.whatever/blubb :name user/project
```

### antq

```
clojure -Ttools install-latest :lib com.github.liquidz/antq :as antq
```

To list outdated dependencies

```bash
clojure -Tantq outdated  # :upgrade true

clojure -A:deps -Tantq help/doc
```

## Neil

Neil is a `babashka` helper, installation instructions at https://github.com/babashka/neil

* add common aliases and deps (cognitect test runner, build tools build.clj, ...).
* create new projects using deps.new
* `search` clojars for a artifacts
* `upgrade` all libs in deps.edn
* `test` to run tests


[deps_and_cli_guide]: https://clojure.org/guides/deps_and_cli
[deps_and_cli_ref]: https://clojure.org/reference/deps_and_cli
