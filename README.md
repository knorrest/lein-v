# lein-v #

Drive leiningen project version from git instead of the other way around.

[![Clojars Project](https://img.shields.io/clojars/v/com.roomkey/lein-v.svg)](https://clojars.org/com.roomkey/lein-v)
[![Build Status](https://secure.travis-ci.org/com.roomkey/lein-v.png)](http://travis-ci.org/com.roomkey/lein-v)

## Motivation ##
The lein-v plugin was driven by several beliefs:

	1. Versioning should be painless in the simplest cases
	2. Unique (and reproducible/commited) source should produce unique versions
	3. Versioning information should live in the SCM repo -the source of source truth
	4. Version information is metadata and should not be stored within the data it describes

Lein-v uses git metadata to build a unique, reproducible and semantically meaningful version for every commit.  Along the way, it adds useful metadata to your project and artifacts (jar and war files) to tie them back to a specific commit.  Finally, it helps ensure that you never release an irreproduceable artifact.

## Task Usage ##

There are two lein sub-tasks within the v namespace intended for direct use:

### lein v
Show the effective version of the project and workspace state.

### lein v cache
Cache the effective version of the project to a file (default is `version.clj`) in the first source directory (typically `src`). It is possible to have the version cached to a file automatically by defining a prep task in your project like this:

    :prep-tasks [["v" "cache" "src"]]

The `lein v cache` command has a variadic argument that is the following: first is the directory to output to (default to `src`), then the rest is the list of the output suffixes you wish and hence format (default to `clj`). The available suffixes are: clj, cljs, cljx, cljc and edn.
You have two formats for file version output depending on the suffix:

- Clojure source code: available via the `clj`, `cljs`, `cljc` or `cljx` option
- EDN data structure: available via the `edn` option

#### Examples of lein v cache invocation

```sh
lein v cache src/cljs cljs
# => output only the src/cljs/version.cljs file
lein v cache src/clj clj edn
# => output the src/clj/version.clj and src/clj/version.edn file
lein v cache src edn
# => output the src/version.edn file
```

#### Example of Clojure source code version output

```clojure
;; This code was automatically generated by the 'lein-v' plugin
(ns version)
(def version "0.6.40")
(def raw-version "v0.6.40")
```

#### Example of EDN version output

```clojure
{:version "0.6.40", :raw-version "v0.6.40"}
```

#### Typical Use Cases of version.xxx file

- Displaying application's version in logs for backends or UI screen for frontend
- In Leiningen tasks you can use the project's version string in the EDN wherever you want, like in output's filename for uberjar or CLJS bundle:
```
~(str "target/releases/js/myapp-" (:version (clojure.edn/read-string (slurp "src/cljs/version.edn")))".min.js")
```

As of leiningen 2.4.1, a less intrusive means of extracting the verison (with caching to a dedicated file) is as follows:
```clojure
(let [pom-properties (with-open [pom-properties-reader (io/reader (io/resource "META-INF/maven/x/x/pom.properties"))]
                       (doto (java.util.Properties.)
                         (.load pom-properties-reader)))]
  (get pom-properties "version"))
```

## Hooks ##
Through the use of the Leiningen hooks functionality, lein-v ensures that
leiningen's own view of the current version is updated before tasks are run. Thus this

    (defproject my-group/my-project :lein-v
	  :plugins [[com.roomkey/lein-v "5.0.0"]]
      ...)

becomes this:

    (defproject my-group/my-project "1.0.1-2-0xabcd"
      :plugins [[com.roomkey/lein-v "5.0.0"]]
      ...)

Assuming that there is a git tag `v1.0.1` on the commit `HEAD~~`, and that the SHA of `HEAD` is uniquely identified by `abcd`.  This behavior is automatically enabled whenever lein-v finds the project version to be the keyword `:lein-v`.

## Dependencies

In case you're using a monorepository, you could also use lein-v to determine the current version of dependencies.

Add the `leiningen.v/dependency-version-from-scm` middleware to your project like this:

```
  :middleware [leiningen.v/version-from-scm
               leiningen.v/dependency-version-from-scm
               leiningen.v/add-workspace-data]
```

Now, if you set the version of a dependency from your monorepo to nil (just as you would using managed-dependencies), it will be replaced
with the current version from git (which is the same as the version of the project you're currently working on).

```
  :dependencies
  [[commons-io "2.5"]
   [example/lib-a nil]
   [example/lib-b nil]
   [org.clojure/clojure "1.8.0"]])
```

becomes

```
  :dependencies
  [[commons-io "2.5"]
   [example/lib-a "1.0.1-2-0xabcd"]
   [example/lib-b "1.0.1-2-0xabcd"]
   [org.clojure/clojure "1.8.0"]])
```


## Support for lein release ##
As of version 5.0, lein-v adds support for leiningen's `release` task.  Specifically, the `lein v update` task can anchor a release process that ensures that git tags are created and pushed, and that those tags conform to sane versioning expectations.  To use `lein release` with lein-v, first modify `project.clj` (or your leiningen user profile) to something such as this:

    :release-tasks [["vcs" "assert-committed"]
                    ["v" "update"] ;; compute new version & tag it
                    ["v" "push-tags"]
                    ["deploy"]]

To effect version changes, lein-v's `update-version` task sees the versioning parameter
provided to lein release and operates as follows:

|current       |directive         |result           |
|:-------------|:-----------------|:----------------|
|1.0.2         |`:major`          |2.0.0            |
|1.1.4         |`:minor`          |1.2.0            |
|2.5.6         |`:patch`          |2.5.7            |

In addition to incrementing the standard numeric version components, you can qualify
any of the above directives with typical qualifiers like `alpha`, `beta`, `rc`.  For example:

|current       |directive         |result           |
|--------------|------------------|-----------------|
|1.0.2         |`:minor-alpha`    |1.1.0-alpha      |
|4.2.8         |`:major-rc`       |5.0.0-RC         |

When the current version is a qualified version, you can increment the current qualifier,
advance to the next qualifier or simply release an unqualified version.  Here are some examples:

|current       |directive         |result           |
|--------------|------------------|-----------------|
|1.0.2-alpha   |`:alpha`          |1.0.2-alpha2     |
|1.0.2-alpha   |`:beta`           |1.0.2-beta       |
|1.0.2-beta    |`:rc`             |1.0.2-rc         |
|1.0.2-rc      |`:rc`             |1.0.2-rc2        |
|3.2.0-rc2     |`:release`        |3.2.0            |

Snapshot versions are similar, but the resulting version is never changed.

|current       |directive         |result           |
|--------------|------------------|-----------------|
|3.2.0         |`:minor-snapshot` |3.3.0-SNAPSHOT   |
|3.3.0-SNAPSHOT|`:snapshot`       |3.3.0-SNAPSHOT   |
|3.3.0-SNAPSHOT|`:release`        |3.3.0            |

Finally, lein-v enforces some common-sense rules:

* For commits without a version tag, the base version will be extended with a build number using the commit distance from
HEAD to the most recent version tag (looking towards the root of the tree) and the unique SHA prefix of the commit.
* You can never go backwards with versions.  This includes qualifiers, which are orderd as follows:
  1. `alpha`
  2. `beta`
  3. `rc`
  4. `snapshot`
* When a git repo is first used with lein-v and has no version tags, the default base version is 0.0.0, and it
  is reported with a distance from the root commit and the relevant SHA (0.0.0-23-0xabcd).
* When tags are created in the git repo, they are prefixed with the letter 'v'.

It is still possible to do a raw `lein deploy`, in which case the version will be that determined by
lein-v (most likely something like "1.0.1-2-0xabcd").

Note: you can provide your own implementation of many of these rules.  See the source code for details on defining data types adhering to the protocols in the `leiningen.v.protocols` namespace.  Currently there are implementations for maven (version 3) and Semantic Versioning (version 2) available.

### Migrating to lein-v

When migrating an existing project.clj, you may not want your project version to reset to v0.0.0 again. You may seed a git tag for lein-v to use. In the example below, the artifact's most recent release is 1.3.2.

```
git tag --annotate --message "Seeding lein-v version" v1.3.12
```

To verify the result had the desired effect, see output of `lein v version`.

```
> lein v version
Effective version: 1.3.12 ...
```

### References and Relevant Reading ###

* (<http://maven.apache.org/ref/3.2.5/maven-artifact/apidocs/org/apache/maven/artifact/versioning/ComparableVersion.html>)
* (<http://www.sonatype.com/books/mvnref-book/reference/pom-relationships-sect-pom-syntax.html>)
* (<http://semver.org/>)
* (<http://javamoods.blogspot.com/2010/10/world-of-versioning.html>)
* (<http://download.eclipse.org/aether/aether-core/1.0.1/apidocs/org/eclipse/aether/util/version/GenericVersionScheme.html>)
* (<http://git.eclipse.org/c/aether/aether-core.git/tree/aether-util/src/main/java/org/eclipse/aether/util/version/GenericVersion.java>)
* (<http://semver.org/>)
* (<http://books.sonatype.com/mvnref-book/reference/pom-relationships-sect-pom-syntax.html#pom-reationships-sect-versions>)
* (<http://mojo.codehaus.org/versions-maven-plugin/version-rules.html>)
* (<http://maven.40175.n5.nabble.com/How-to-use-alternative-version-numbering-scheme-td123806.html>)
* (<http://maven.apache.org/ref/3.2.5/maven-artifact/index.html>)
* (<https://cwiki.apache.org/confluence/display/MAVENOLD/Versioning>)
* (<http://docs.codehaus.org/display/MAVEN/Dependency+Mediation+and+Conflict+Resolution>)
* (<http://dev.clojure.org/display/doc/Maven+Settings+and+Repositories>)
* (<http://maven.40175.n5.nabble.com/How-to-use-SNAPSHOT-feature-together-with-BETA-qualifier-td73263.html>)

## License ##

Copyright (C) 2019 Room Key

Distributed under the Eclipse Public License, the same as Clojure.
