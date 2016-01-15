# case-app

*Type-level & seamless command-line argument parsing for Scala*

[![Build Status](https://travis-ci.org/alexarchambault/case-app.svg)](https://travis-ci.org/alexarchambault/case-app)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/alexarchambault/case-app?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

This README now documents the current development version (`1.0.0-SNAPSHOT`)
of case-app. See [the `0.3.x` branch](https://github.com/alexarchambault/case-app/tree/0.3.x) for the stable version.

### Imports

The code snippets below assume that the content of `caseapp` is imported,

```scala
import caseapp._
```

### Parse a simple set of options

```scala
case class Options(
  user: Option[String],
  enableFoo: Boolean,
  file: List[String]
)

CaseApp.parse[Options](
  Seq("--user", "alice", "--file", "a", "--file", "b")
).right.get._1 == Options(Some("alice"), false, List("a", "b"))
```

### Default values

Most primitive types have default values, like `0` for `Int`, or `false`
for `Double`. Options default to `None`, and sequences to `Nil`.
Full list in the [source](https://github.com/alexarchambault/case-app/blob/master/src/main/scala/caseapp/core/Default.scala).

```scala
case class Options(
  user: Option[String],
  enableFoo: Boolean,
  file: List[String]
)

CaseApp.parse[Options](Seq()).right.get._1 == Options(None, false, Nil)
```

Alternatively, default values can be manually specified, like
```scala
case class Options(
  user: String = "default",
  enableFoo: Boolean = true
)

CaseApp.parse[Options](Seq()).right.get._1 == Options("default", true)
```

### Lists

Some arguments can be specified several times on the command-line. These
should be typed as lists, e.g. `file` in

```scala
case class Options(
  user: Option[String],
  enableFoo: Boolean,
  file: List[String]
)

CaseApp.parse[Options](
  Seq("--file", "a", "--file", "b")
).right.get._1 == Options(None, false, List("a", "b"))
```

If an argument is specified several times, but is not typed as a `List` (or an accumulating type,
see below), the final value of its corresponding field is the last provided in the arguments.

### Whole application with argument parsing

*case-app* can take care of the creation of the `main` method parsing
command-line arguments.

```scala
import caseapp._

case class Example(
  foo: String,
  bar: Int
) extends App {

  // Core of the app
  // ...

}

object ExampleApp extends AppOf[Example]
```

This uses the `DelayedInit` mechanism, as `scala.App` does, so that the body
of `Example` above isn't called immediately when an `Example` is instantiated.

`ExampleApp` in the above example will then have a `main` method, parsing
the arguments it is given to an `Example`, then calling the body of the latter.

Note that the singleton extending `AppOf[...]` should have a different name
than the type argument it is given, e.g. above it should *not* be called
`Example`. This originates in it needing some type classes about its type
argument, themselves automatically generated by macros that need the
singleton before-hand.

### Automatic help and usage options

Running the above example with the `--help` (or `-h`) option will print an help message
of the form
```
Example
Usage: example [options]
  --foo <value>
  --bar <value>
```

Calling it with the `--usage` option will print
```
Usage: example [options]
```

### Customizing items of the help / usage message

Several parts of the above help message can be customized by annotating
`Example` or its fields:

```scala
@AppName("MyApp")
@AppVersion("0.1.0")
@ProgName("my-app-cli")
case class Example(
  @HelpMessage("the foo")
  @ValueDescription("foo")
    foo: String,
  @HelpMessage("the bar")
  @ValueDescription("bar")
    bar: Int
) extends App {

  // Core of the app
  // ...

}
```




Called with the `--help` or `-h` option, would print
```
MyApp 0.1.0
Usage: my-app-cli [options]
  --foo <foo>: the foo
  --bar <bar>: the bar
```

Note the application name that changed, on the first line. Note also the version
number appended next to it. The program name, after `Usage: `, was changed too.

Lastly, the options value descriptions (`<foo>` and `<bar>`) and help messages
(`the foo` and `the bar`), were customized.

### Extra option names

Alternative option names can be specified, like
```scala
case class Example(
  @ExtraName("f")
    foo: String,
  @ExtraName("b")
    bar: Int
) extends App {

  // ...

}
```




`--foo` and `-f`, and `--bar` and `-b` would then be equivalent.

### Long / short options

Field names, or extra names as above, longer than one letter are considered
long options, prefixed with `--`. One letter long names are short options,
prefixed with a single `-`.

```scala
case class Example(
  a: Int,
  foo: String
) extends App {

  // ...

}
```




would accept `--foo bar` and `-a 2` as arguments to set `foo` or `a`.

### Pascal case conversion

Field names or extra names as above, written in pascal case, are split
and hyphenized.

```scala
case class Options(
  fooBar: Double
)
```




would accept arguments like `--foo-bar 2.2`.


### Reusing options

Sets of options can be shared between applications:

```scala
case class CommonOptions(
  foo: String,
  bar: Int
)

case class First(
  baz: Double,
  @Recurse
    common: CommonOptions
) extends App {

  // ...

}

case class Second(
  bas: Long,
  @Recurse
    common: CommonOptions
) {

  // ...

}
```




### Commands

*case-app* has a support for commands.

```scala
sealed trait DemoCommand extends Command

case class First(
  foo: Int,
  bar: String
) extends DemoCommand {

  // ...

}

case class Second(
  baz: Double
) extends DemoCommand {

  // ...

}

object MyApp extends CommandAppOf[DemoCommand]
```

`MyApp` can then be called with arguments like
```
my-app first --foo 2 --bar a
my-app second --baz 2.4
```

- help messages
- customization
- base command
- ...

### Counters

*Needs to be updated*

Some more complex options can be specified multiple times on the command-line and should be
"accumulated". For example, one would want to define a verbose option like
```scala
case class Options(
  @ExtraName("v") verbose: Int
)
```




Verbosity would then have be specified on the command-line like `--verbose 3`.
But the usual preferred way of increasing verbosity is to repeat the verbosity
option, like in `-v -v -v`. To accept the latter,
tag `verbose` type with `Counter`:
```scala
case class Options(
  @ExtraName("v") verbose: Int @@ Counter
)
```

`verbose` (and `v`) option will then be viewed as a flag, and the
`verbose` variable will contain
the number of times this flag is specified on the command-line.

It can optionally be given a default value other than 0. This
value will be increased by the number of times `-v` or `--verbose`
was specified in the arguments.


### User-defined option types

*Needs to be updated*

Use your own option types by defining implicit `ArgParser`s for them, like in
```scala
import caseapp.core.ArgParser

trait Custom

implicit val customArgParser: ArgParser[Custom] =
  ArgParser.instance[Custom] { s =>
    // parse s
    // return
    // - Left("error message") in case of error
    // - Right(custom) in case of success
    ???
  }
```

Then use them like
```scala
case class Options(
  custom: Custom,
  foo: String
)
```




### Migration from the previous version

Shared options used to be automatic, and now require the `@Recurse`
annotation on the field corresponding to the shared options. This prevents
ambiguities with custom types as above.

## Usage

The above documentation applies to the to-be-released 1.0 version. It can
be used via snapshot artefacts, by adding to your `build.sbt`,
```scala
resolvers ++= Seq(
  Resolver.sonatypeRepo("releases"),
  Resolver.sonatypeRepo("snapshots")
)

libraryDependencies += "com.github.alexarchambault" %% "case-app" % "1.0.0-SNAPSHOT"
```

Note that it depends on shapeless 2.3.0-SNAPSHOT.

If you are using scala 2.10.x, also add the macro paradise plugin to your build,
```scala
libraryDependencies +=
  compilerPlugin("org.scalamacros" % "paradise" % "2.0.1" cross CrossVersion.full)
```

## See also

Eugene Yokota, the current maintainer of scopt, and others, compiled
an (eeextremeeeely long) list of command-line argument parsing
libraries for Scala, in [this StackOverflow question](http://stackoverflow.com/questions/2315912/scala-best-way-to-parse-command-line-parameters-cli).

Unlike [scopt](https://github.com/scopt/scopt), case-app is less monadic / abstract data types based, and more
straight-to-the-point and descriptive / algebric data types oriented.
 
## Notice
 
Copyright (c) 2014-2015 Alexandre Archambault.
See LICENSE file for more details.

Released under Apache 2.0 license.

