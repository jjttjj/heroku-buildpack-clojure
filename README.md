# Heroku buildpack: Clojure

This is a Heroku buildpack for Clojure apps. It uses
[Leiningen](http://leiningen.org).

Note that you don't have to do anything special to use this buildpack
with Clojure apps on Heroku; it will be used by default for all
projects containing a project.clj file, though it may be an older
revision than current master. 

## Usage

Example usage for an app already stored in git:

    $ tree
    |-- Procfile
    |-- project.clj
    |-- README
    `-- src
        `-- sample
            `-- core.clj

    $ heroku create

    $ git push heroku master
    ...
    -----> Heroku receiving push
    -----> Fetching custom buildpack
    -----> Clojure app detected
    -----> Installing Leiningen
           Downloading: leiningen-2.0.0-preview10-standalone.jar
           Writing: lein script
    -----> Building with Leiningen
           Running: with-profile production compile :all
           Downloading: org/clojure/clojure/1.2.1/clojure-1.2.1.pom from central
           Downloading: org/clojure/clojure/1.2.1/clojure-1.2.1.jar from central
           Copying 1 file to /tmp/build_2e5yol0778bcw/lib
    -----> Discovering process types
           Procfile declares types -> core
    -----> Compiled slug size is 10.0MB
    -----> Launching... done, v4
           http://gentle-water-8841.herokuapp.com deployed to Heroku

The buildpack will detect your app as Clojure if it has a
`project.clj` file in the root. If you use the
[clojure-maven-plugin](https://github.com/talios/clojure-maven-plugin),
[the standard Java buildpack](http://github.com/heroku/heroku-buildpack-java)
should work instead. Leiningen 1.7.1 will be used by default, but if
you have `:min-lein-version "2.0.0"` in project.clj then Leiningen 2.x
will be used instead.

## Configuration

If your project uses Leiningen 2 (highly recommended) you should
include `:min-lein-version "2.0.0"` (or higher) in your
`project.clj`.

Your `Procfile` should declare what process types which make up your
app. Typically in development Leiningen projects are launched using
`lein run -m my.project.namespace`, but this is not recommended in
production because it leaves Leiningen running in addition to your
project's process.

### Leiningen at Runtime

One way to avoid this is with the `trampoline` task. This will cause
Leiningen to calculate the classpath and code to run for your project,
then exit and execute your project's JVM. If you do this it's
recommended you use the `:production` profile to avoid having
development or test dependencies or configuration visible:

    web: lein with-profile offline,production trampoline run -m myapp.web

If you don't need to add anything to the `:production` profile then
you can leave it out and the one from `opt/profiles.clj` in the
buildpack will be used. If you do need to add something, it's
recommended you include `:mirrors` for faster dependency resolution
from S3 for Central:

```clj
:production {:app-specific "config" ; put your own config here if needed
             :mirrors {"central" "http://s3pository.herokuapp.com/maven-central"}}
```

Since Clojars currently mixes snapshots and releases it's currently
not appropriate to mirror to S3 unless you know for sure you're not
using any snapshots even transitively.

### Uberjars

Another simpler way is to create an uberjar during build and not
involve Leiningen at all at runtime. If your `Procfile` does not
mention `lein` at all, then the buildpack will run `lein
uberjar`. Then your `Procfile` entries should just consist of `java`
invocations.

If you have `:main` in `project.clj` and `:gen-class` in your main
namespace you can just use `java $JVM_OPTS -jar
target/myproject-standalone.jar`. If you're not using `:main` then you
can use `clojure.main/main` as your entry and point it to your app
using the `-m` argument: `java $JVM_OPTS -cp
target/myproject-standalone.jar clojure.main -m myproject.web`.

This will reduce the size of your slug since Leiningen will not be
included. If you need Leiningen in a `heroku run` session, it will be
downloaded automatically.

Adding `:uberjar-name` to `project.clj` will prevent the uberjar
filename from changing when your version number changes, which will
make your Procfile easier to maintain.

Note that uberjars require a `:main` in `project.clj`. Usually you
would also include a `:gen-class` directive in the `ns` form of the
main namespace, though you can skip this by using `clojure.main` as
the `:main` namespace and using the `-m` argument to specify your
application's namespace.

## JDK Version

By default you will get OpenJDK 1.6. To use a different version, you
can commit a `system.properties` file to your app.

```
$ echo "java.runtime.version=1.7" > system.properties
$ git add system.properties
$ git commit -m "JDK 7"
```

## Hacking

To change this buildpack, fork it on GitHub. Push up changes to your
fork, then create a test app with `--buildpack YOUR_GITHUB_URL` and
push to it. If you already have an existing app you may use
`heroku config:add BUILDPACK_URL=YOUR_GITHUB_URL` instead.

For example, you could adapt it to generate an uberjar at build time.

Open `bin/compile` in your editor, and replace the block labeled
"fetch deps with lein" with something like this:

    echo "-----> Generating uberjar with Leiningen:"
    echo "       Running: lein uberjar"
    cd $BUILD_DIR
    PATH=.lein/bin:/usr/local/bin:/usr/bin:/bin JAVA_OPTS="-Xmx500m -Duser.home=$BUILD_DIR" lein uberjar 2>&1 | sed -u 's/^/       /'
    if [ "${PIPESTATUS[*]}" != "0 0" ]; then
      echo " !     Failed to create uberjar with Leiningen"
      exit 1
    fi

Commit and push the changes to your buildpack to your GitHub fork,
then push your sample app to Heroku to test. The output should include:

    -----> Generating uberjar with Leiningen:

If it's something other users would find useful, pull requests are welcome.

## Troubleshooting

To see what the buildpack has produced, do `heroku run bash` and you
will be logged into an environment with your compiled app available.
From there you can explore the filesystem and run `lein` commands.
