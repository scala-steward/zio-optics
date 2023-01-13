---
id: index
title: "Introduction to ZIO Optics"
sidebar_label: "ZIO Optics"
---

[ZIO Optics](https://github.com/zio/zio-optics) is a library that makes it easy to modify parts of larger data structures based on a single representation of an optic as a combination of a getter and setter.

@PROJECT_BADGES@

## Introduction

When we are working with immutable nested data structures, updating and reading operations could be tedious with lots of boilerplates. Optics is a functional programming construct that makes these operations more clear and readable.

Key features of ZIO Optics:

- **Unified Optic Data Type** — All the data types like `Lens`, `Prism`, `Optional`, and so forth are type aliases for the core `Optic` data type.
- **Composability** — We can compose optics to create more advanced ones.
- **Embracing the Tremendous Power of Concretion** — Using concretion instead of unnecessary abstractions, makes the API more ergonomic and easy to use.
- **Integration with ZIO Data Types** — It supports effectful and transactional optics that works with ZIO data structures like `Ref` and `TMap`.
- **Helpful Error Channel** — Like ZIO, the `Optics` data type has error channels to include failure details.
* **Zero dependencies** - No dependencies other than ZIO itself.
* **No unnecessary abstractions** - Concrete representation makes it easy to learn.

The optic also handles the possibility of failure for us, failing with an `OpticFailure` that is a subtype of `Throwable` and contains a helpful error message if the key cannot be found.

ZIO Optics makes it easy to compose more complex optics from simpler ones, to define optics for your own data types, and to work with optics that use `ZIO` or `STM` effects.

## Installation

In order to use this library, we need to add the following line in our `build.sbt` file:

```scala
libraryDependencies += "dev.zio" %% "zio-optics" % "@VERSION@"
```

## Example

ZIO Optics makes it easy to modify parts of larger data structures. For example, say we have a web application where users can vote on which of various topics they are interested in. We maintain our state of how many votes each topic has received as a `Ref[Map[String, Int]]`.

```scala mdoc
import zio._

lazy val voteRef: Ref[Map[String, Int]] =
  ???
```

If we want to increment the number of votes for one of the topics here is what it would look like:

```scala mdoc
def incrementVotes(topic: String): Task[Unit] =
  voteRef.modify { voteMap =>
    voteMap.get(topic) match {
      case Some(votes) =>
        (ZIO.unit, voteMap + (topic -> (votes + 1)))
      case None        =>
        val message = s"voteMap $voteMap did not contain topic $topic"
        (ZIO.fail(new NoSuchElementException(message)), voteMap)
    }
  }.flatten
```

This is alright, but there is a lot of code here for a relatively simple operation of incrementing one of the keys. We have to get the value from the `Ref`, then get the value from the `Map`, and finally set the new value in the `Map`.

We also have to explicitly handle the possibility that the value is not in the map. And this is all for a relatively simple data structure!

Here is what this would look like with ZIO Optics.

[//]: # (TODO: Make this snippet type-safe using mdoc modifer)

```scala
import zio.optics._

def incrementVotes(topic: String): Task[Unit] =
  voteRef.key(topic).update(_ + 1)
```

The `key` optic "zooms in" on part of a larger structure, in this case transforming the `Ref[Map[String, Int]]` into a `Ref` that accesses the value at the specified key. We can then simply call the `update` operator on `Ref` to increment the value.

Let's try another example. We are going to update a nested data structure using ZIO Optics:

```scala
import zio.optics._

case class Developer(name: String, manager: Manager)
case class Manager(name: String, rating: Rating)
case class Rating(upvotes: Int, downvotes: Int)

val developerLens = Lens[Developer, Manager](
  get = developer => Right(developer.manager),
  set = manager => developer => Right(developer.copy(manager = manager))
)

val managerLens = Lens[Manager, Rating](
  get = manager => Right(manager.rating),
  set = rating => manager => Right(manager.copy(rating = rating))
)

val ratingLens = Lens[Rating, Int](
  get = rating => Right(rating.upvotes),
  set = upvotes => rating => Right(rating.copy(upvotes = upvotes))
)

// Composing lenses
val optic = developerLens >>> managerLens >>> ratingLens

val jane    = Developer("Jane", Manager("Steve", Rating(0, 0)))
val updated = optic.update(jane)(_ + 1)

println(updated)
```

## Resources

- [Zymposium - Optics](https://www.youtube.com/watch?v=-km5ohYhJa4) by Adam Fraser and Kit Langton (June 2021) — Optics are great tools for working with parts of larger data structures and come up in disguise in many places such as ZIO Test assertions.