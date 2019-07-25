---
layout: post
title: Monad Transformers and Cats MTL
comments: true
tags: scala cats mtl monad-transformer state reader
license: true
revision: 1
summary: This post shows benefits of Cats MTL library with an example developed from monad transformers.
---

There are various ways to combine effects encoded in monads. One of them is monad transformers which help to stack up different monads into one. However, there're some drawbacks of monad transformers such as complexity. In this post, I used Cats MTL to reduce this complexity.

## Introduction

If you're not familiar with the monad transformers, you can find a nice article about the concept [here](https://blog.buildo.io/monad-transformers-for-the-working-programmer-aa7e981190e7). There's also a very comprehensive [Typelevel blog article](https://typelevel.org/blog/2018/10/06/intro-to-mtl.html) about Cats MTL.

To create an example, let's think about a use case. Suppose there is a requirement for querying users from a repository. In addition, the application should return how many times a particular user is queried. First let's try to find monads corresponding to required effects, and use monad transformers to combine them. For simplicity, assume that it's not necessary to think about IO interaction at this point. Rather than, this post will focus on how to _select_ the repository and _store_ the query counts. The problem will be stated after finishing the initial implementation.

**Versions**

Scala|2.13.0
cats-core|2.0.0-M4
cats-mtl-core|0.6.0
kind-projector|1.2.0

**Code**

To begin, let's define the `User` record.

```scala
type UserId = Long
final case class User(id: UserId, name: String)
```

Next is defining types for failure. Since users will be queried from some repository, it's possible that some users might not be found.

```scala
final case class Failure(message: String)
type EitherFailure[A] = Failure Either A
```

Using `EitherFailure`, now we can define an interface for querying.

```scala
trait Db {
  def user(id: UserId): EitherFailure[User]
}
```

It would be good to be able to _select_ the `Db` instance based on some _environment_. `Reader` monad perfectly suits for this job. `Reader` can be used for _fetching_ a read-only state (such as _configuration_) as well as _dependency injection_. And here it's utilised for the latter practice. Because `Either` is already in the playground, machinery is needed to combine both effects. What's required is exactly the `ReaderT`.

```scala
import cats.data.ReaderT
type ReaderTEither[A, B] = ReaderT[EitherFailure, A, B]
```

Not surprisingly, `ReaderT` is just an alias for `Kleisli` in Cats. Moreover, `Reader` is defined as simply setting `Id` as the inner type.

```scala
// cats.data
type ReaderT[F[_], A, B] = Kleisli[F, A, B]
type Reader[A, B] = ReaderT[Id, A, B]
```

Now let's define a `Service` interface which will _read_ `Db` instance from some _environment_ and query the user by id.

```scala
trait Service {
  def query(id: UserId): ReaderTEither[Db, User] = ReaderT(_.user(id))
}

import cats.implicits._
object Program1 extends Service {
  def result: ReaderTEither[Db, (User, User)] = for {
    user1 <- query(1L)
    user2 <- query(2L)
  } yield (u1, u2)
}
```

Above, `result` returns a function (a _Kleisli arrow_ to be precise) which takes `Db` as parameter and returns pair of `User` instances wrapped in `Either`. For given `Program1`, the paired instances will have id `1` and id `2` respectively.

What happens if there's no user with id `1`? Thanks to combined effect of `ReaderTEither`, whenever a `Failure` occurs, for comprehension will short-circuit immediately. To see it's working, let's create a dummy `Db` implementation and _run_ what's returned by `result`.

```scala
def inMemoryDb = new Db {

  val users = Map[UserId, User](
    1L -> User(1L, "Rick"),
    2L -> User(2L, "Morty"),
    3L -> User(3L, "Summer")
  )

  override def user(id: UserId): EitherFailure[User] =
    users.get(id).map(Right(_)).getOrElse(Left(Failure(s"No user found with id($id)")))
}

val p1 = Program1.result.run(inMemoryDb)
println(p1)
```

Note that, `run` gives back the composed _Kleisli arrow_, which needs to be _evaluated_ by concrete `inMemoryDb`. Here's the output:

<div class="console">Right((User(1,Rick),User(2,Morty)))
</div>

Since all the users are found, we see it `Right`.

This was the first part of the requirements. Question is, how to add the query counts on the top this implementation? There needs to be some _context_, where the program logic can _read_ and _write_ query counts as it processes through. Well, the answer is the `State` monad. But to add the stateful computation effect to the already populated `ReaderTEither` monad, we need `StateT`. First, let's define the data structure to hold the query counts.

```scala
type UserQueryCount = Map[UserId, Long]
```

Next is to define `StateTReaderTOption`, which encapsulates `ReaderTEither`. Here I've used `kind-projector` plugin to make types more readable. In the following type definition, we need to keep `ReaderTEither` as a container with a single type variable, which is achieved by type lambda (`?`).

```scala
import cats.data.StateT
type StateTReaderTOption[A, B] = StateT[ReaderTEither[A, ?], UserQueryCount, B]
```

Having this new type, now we can rewrite the `Service` class with utilising provided `UserQueryCount` state. Before let's define `QueryResult` type which is a product of user and how many times it's queried so far.

```scala
type QueryResult = (User, Long)

trait Service {
  def query(id: UserId): StateTReaderTOption[Db, QueryResult] = for {
    queryCounts <- StateT.get[ReaderTEither[Db, ?], UserQueryCount]
    count       =  queryCounts.getOrElse(id, 0L) + 1L
    _           <- StateT.set[ReaderTEither[Db, ?], UserQueryCount] {
                     queryCounts + (id -> count)
                   }
    reader      =  ReaderT[EitherFailure, Db, QueryResult] {
                     r => r.user(id).map(u => (u, count))
                   }
    result      <- StateT.apply[ReaderTEither[Db, ?], UserQueryCount, QueryResult] {
                     s => reader.map(r => (s, r))
                   }
  } yield result
}
```

Let's go step by step. The return type is `QueryResult` wrapped in `StateTReaderTOption`. Inside the for comprehension, first 3 line gets the _state_ (binding to `queryCounts`), increments the `count` by `1` and sets the _state_ with updated `Map`. Notice the **noise** around the types while calling the `get` and `set` functions on `StateT`. Next line queries the `Db`, constructs `QueryResult` by mapping over `EitherFailure[User]`, and wraps the value in `ReaderT`. Finally, `StateT` is constructed via `apply` taking `f: UserQueryCount => ReaderTEither[Db, (UserQueryCount, QueryResult)]`.

Again, what happens if the user with given id not found? For comprehension will short-circuit while performing the query on `Db`. So we have all the effects in one place. Here's `Program2` using the `Service` created above.

```scala
object Program2 extends Service {
  def result: StateTReaderTOption[Db, QueryResult] = for {
    _  <- query(1L)
    _  <- query(1L)
    u2 <- query(2L)
  } yield u2
}

val p2 = Program2.result.run(Map.empty).run(inMemoryDb)
println(p2)
```

Note that the first `run` _runs_ `State` monad with an initial state (here an empty `Map`). The second `run`, on the other hand, evaluates the function returned by `Reader` monad with `inMemoryDb`. Let's try to run it and see what happens.

<div class="console">Right((Map(1 -> 2, 2 -> 1),(User(2,Morty),1)))
</div>

Because `State` is a function returning a product of state and the result, we see both of them in the output. The _state_, namely `UserQueryCount` map, shows that `Rick` (id `1`) is queried `2` times and `Morty` (id `2`) is queried `1` time. The second part of the product is another product, and since the last query was made against id `2`, it gives the user `Morty` and its query count. Finally, as all the users are found in `inMemoryDb`, the entire value is `Right`.

To sum up, with the help of the monad transformers `StateT` and `ReaderT`, we managed to put all effects in once place and implement the requirements in functional way.

### Problem

It's easily noticed that we need to explicitly put type annotations in the previous implementation since Scala compiler cannot infer them completely.

This can be cumbersome when the transformer stack consists of multiple layers. More monad transformers, worse type inference. If the combined monad effects are widely used in the code base, having noisy type annotations will increase the development time.

Being said, Cats MTL library provides a cleaner way to deal with multiple effects.

## Cats MTL

Cats MTL substantiates the idea of encoding the effects in type classes rather than data structures. It helps to write the code against some type constructor `F[_]` fulfilling different type class constraints.

There are several type classes provided by the Cats MTL library to simulate most common  monadic effects. You can find the entire list [here](https://typelevel.org/cats-mtl/mtl-classes.html). One nice thing about the type classes is that they stick to the principle of the least power. For instance, `FunctorRaise` is enough to be `Functor` to provide functionality for raising errors. In addition, to avoid implicit ambiguities, Cats MTL library favors composition for such constraints.

Let's try to re-implement the requirements with using MTL library this time. To meet the requierements, we need 3 different effects: read-write state (`StateT`), read-only state (`ReaderT`) and error handling (`Either`). For the first two, we can pick `MonadState` and `ApplicativeAsk`. For the last effect, let's select `MonadError` to start with. New `Service` class will look like this:

```scala
import cats.MonadError
import cats.mtl.{ApplicativeAsk, MonadState}

trait Service {
  def query[F[_]](id: UserId)(
    implicit S: MonadState[F, UserQueryCount],
             A: ApplicativeAsk[F, Db],
             E: MonadError[F, Failure]
    ): F[QueryResult] = for {
    queryCounts <- S.get
    count       =  queryCounts.getOrElse(id, 0L) + 1L
    _           <- S.set(queryCounts + (id -> count))
    result      <- E.rethrow(A.reader(_.user(id).map(u => (u, count))))
  } yield result
}
```

Method declaration includes implicit parameters which _witnesses_ that selected `F[_]` _complies_ with required effects. Inside the for comprehension, state is fetched with simply calling `S.get`. Because the type `F[S]` can easily inferred by the Scala compiler, there's no need to explicitly set type annotations. This is very convenient compared to the previous implementation. Next, state is updated with the incremented query count. The last line is even more concise. With the help of `rethrow` function on `MonadError`, we could easily pass the `Either` value wrapped in `F[_]`.

Note that `query` is not ready to be run yet. So far we only specified the required effects but didn't provide the actual `F[_]` implementation meeting all the requirements.

Luckily, Cats MTL provides a way to materialize from a monad transformer stack. With using a bunch of _implicits_ defined under `cats.mtl.implicits` library, we can easily create a `materializedQuery` function from `query` function by giving the entire monad stack in the type signature.

```scala
import cats.mtl.implicits._
def materializedQuery(id: UserId) =
  query[StateT[ReaderT[EitherFailure, Db, ?], UserQueryCount, ?]](id)
```

Now let's run the `Program3` and see what happens.

```scala
object Program3 extends Service {
  def result: StateTReaderTOption[Db, QueryResult] = for {
    _  <- materializedQuery(1L)
    _  <- materializedQuery(1L)
    u2 <- materializedQuery(2L)
  } yield u2
}

val p3 = Program3.result.run(Map.empty).run(inMemoryDb))
```

<div class="console">Right((Map(1 -> 2, 2 -> 1),(User(2,Morty),1)))
</div>

We've got exactly the same result as in the previous implementation.

One more thing to consider is that the `MonadError` is directly coming from `cats` package and it's not a part of Cats MTL type classes. Here I've used for convenience, particularly for 2 reasons. Since `MonadError` extends `Monad`, `F[_]` could be used inside of the for comprehension, as `map` and `flatMap` functions are defined for it. Plus, `rethrow` is very handy when working with `Either`.

However, using a type class extending `Monad` might not fit well with the design philosophy of Cats MTL library. So let's try to replace `MonadError` with `FunctorRaise`. However, this time we need to be sure that `query` signature requires a `Monad` constraint on `F[_]`.

```scala
trait Service {
  def query[F[_] : Monad](id: UserId)(
    implicit S: MonadState[F, UserQueryCount],
             A: ApplicativeAsk[F, Db],
             E: FunctorRaise[F, Failure]
    ): F[(User, Long)] = for {
    queryCounts <- S.get
    count       =  queryCounts.getOrElse(id, 0L)
    _           <- S.set(queryCounts + (id -> (count + 1L)))
    reader      <- A.reader(_.user(id).map(u => (u, count)))
    result      <- reader match {
                      case Right(r) => Monad[F].pure(r)
                      case Left(e)  => E.raise[(User, Long)](e)
                    }
  } yield result

  def materializedQuery(id: UserId) =
    fetch[StateT[ReaderT[WithError, Db, ?], UserQueryCount, ?]](id)
}
```

### Performance

Monad transformers are infamous for their performance characteristics. I'm not going to deep dive into the reasons, but you can find a good article [here](http://degoes.net/articles/effects-without-transformers). Now let's compare the 2 implementations created so far. To do this, `jmh` can be used via `sbt-jmh` plugin. Let's create a benchmarking class which will call these 2 implementations `1000` times.

```scala
import cats.implicits._
import org.openjdk.jmh.annotations._

@BenchmarkMode(Array(Mode.AverageTime))
class BenchmarkTests {

  def transformers(count: Int): Unit =
    for (_ <- 0 until count) Program2.result.run(Map.empty).run(inMemoryDb)

  def mtl(count: Int): Unit =
    for (_ <- 0 until count) Program3.result.run(Map.empty).run(inMemoryDb)

  @Benchmark
  def transformersBenchmark: Unit = transformers(1000)

  @Benchmark
  def mtlBenchmark: Unit = mtl(1000)
}
```

Benchmark report will give scores as average operation times in seconds. Here's the result after running `10` warmup and `10` measurement iterations.

<div class="console">jmh:run -i 10 -wi 10 -f1 -t1
...
[info] Result "mtl.BenchmarkTests.mtlBenchmark":
[info]   0.004 ±(99.9%) 0.001 s/op [Average]
[info]   (min, avg, max) = (0.004, 0.004, 0.005), stdev = 0.001
[info]   CI (99.9%): [0.004, 0.005] (assumes normal distribution)

...
[info] Result "mtl.BenchmarkTests.transformersBenchmark":
[info]   0.003 ±(99.9%) 0.001 s/op [Average]
[info]   (min, avg, max) = (0.003, 0.003, 0.003), stdev = 0.001
[info]   CI (99.9%): [0.003, 0.003] (assumes normal distribution)
...
[info] Do not assume the numbers tell you what you want them to tell.
[info] Benchmark                             Mode  Cnt  Score    Error  Units
[info] BenchmarkTests.mtlBenchmark           avgt   10  0.004 ±  0.001   s/op
[info] BenchmarkTests.transformersBenchmark  avgt   10  0.003 ±  0.001   s/op
</div>

We observe similar performance characteristics. This is because we materialize `F[_]` over the monad transformer stack. However, this is not the only way while using the Cats MTL type classes. Another nice thing about Cats MTL is that there's a clear separation between 2 aspects it consists of: definition of type classes and interpretation. As you can see, program logic can be implemented without providing the interpreter. So that performance can be improved with different interpreters.

## Conclusion

Cats MTL significantly reduces the complexity introduced by monad transformers. We can summarize the benefits as:
* Fewer type annotations.
* Support for most common effects.
* Implicit materialization for monad transformers.

As a next step, I'll try type class instances having specialized data structures and see the benefits performance-wise.

Cheers!
