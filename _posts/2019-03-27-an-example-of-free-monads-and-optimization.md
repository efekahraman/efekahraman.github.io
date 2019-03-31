---
layout: post
title: An Example of Free Monads and Optimization
comments: true
tags: scala cats free-monad optimization io state
license: true
revision: 2
summary: This post gives an example usage of Free Monad and how to make optimizations using Cats.
---

_Free Monads_ is a well-known subject in the functional programming community. You can find some cool articles [here](https://softwaremill.com/free-monads/) and [here](https://perevillega.com/understanding-free-monads). In this post, I used _Free Monad_ to optimize a network routing mechanism.

## Introduction

Suppose a routing logic needs to be _defined_ which selects a destination host from a pool for each payload. Instead of sticking into one type of implementation, when _Free Monad_ is used to define the logic, any optimized implementations can be added to the system with minimum effort.

This can be seen a bit different use of _Free Monads_. Most of the time, it's used for abstracting over _effects_ like `IO`. However, this post is focused on the advantage of optimization coming from the nature of _Free Monad_.

In this section, an algebra with an initial interpreter will be created. The problem will be stated after the result of the first interpreter and a possible solution will be discussed.

**Versions**

Scala|2.12.8
Cats Core|1.1.0
Cats Free|1.1.0
Cats Effect|1.2.0
Cats Collections|0.7.0

**Code**

Let's define the payload _type_ assuming the size and the content (in bytes) are known:

```scala
final case class Payload(size: Long, bytes: Vector[Byte])
```

Next is to define an algebra for the routing logic. Say it's required to have 2 functions; adding a new host and selecting a destination with respect to the payload to be sent.

```scala
sealed trait Routing[T]
final case class AddHost(host: String) extends Routing[Unit]
final case class GetHost(payload: Payload) extends Routing[Option[String]]
```

The result of selection is `Option[String]`. This is because there might not be any host defined before the query is made. Once the _ADT_ is in place, the only thing remaining is to lift it to `Free`.

```scala
type RoutingF[T] = Free[Routing, T]

def addHost(host: String): RoutingF[Unit] =
  Free.liftF[Routing, Unit](AddHost(host))

def getHost(payload: Payload): RoutingF[Option[String]] =
  Free.liftF[Routing, Option[String]](GetHost(payload))
```

So far clear. Now let's create a program which is going to use `RoutingF`.

```scala
val DummyPayload: Vector[Byte] = Vector()

val largePayload: Payload = Payload(2048, DummyPayload)
val smallPayload: Payload = Payload(1024, DummyPayload)

val program1: Free[Routing, List[Option[String]]] = for {
  _     <- addHost("host1")
  _     <- addHost("host2")
  _     <- addHost("host3")
  host1 <- getHost(largePayload)
  host2 <- getHost(smallPayload)
  host3 <- getHost(smallPayload)
  host4 <- getHost(largePayload)
  host5 <- getHost(smallPayload)
  host6 <- getHost(smallPayload)
} yield List(host1, host2, host3, host4, host5, host6)
```

Say we have 2 payloads, having 1KB and 2KB of size. Note that content is totally trivial in this example (which is left as empty). As it's seen in the code, the result type the `program1` is `RoutingF[List[Option[String]]]`. Therefore an interpreter needs to be created for the _actual_ implementation.

First, let's start with implementing _Round Robin_ routing. But remember, this implementation requires to **store** hosts and possibly some state for routing. This actually gives a clue that it can be _effectful_. Question is, how to do that in a  _functional_ way?

```scala
type RoundRobinState[T] = State[(Map[Int, String], Int), T]

def roundRobinRouting: Routing ~> RoundRobinState = new (Routing ~> RoundRobinState) {
  override def apply[A](fa: Routing[A]): RoundRobinState[A] = fa match {
    case AddHost(host) => State.modify { state =>
      val oldMap = state._1
      val newMap = if (oldMap.isEmpty) Map(0 -> host) else oldMap + (oldMap.size -> host)
      state.copy(_1 = newMap)
    }
    case GetHost(_) => State { state =>
      val hostMap = state._1
      if (hostMap.isEmpty) state -> None
      else {
        val counter = state._2
        val host = hostMap(counter % hostMap.size)
        state.copy(_2 = counter + 1) -> Some(host)
      }
    }
  }
}
```

Here the basic idea is to use `State` monad to accumulate changes. So the interpreter becomes a _natural transformation_ from `Routing` to `RoundRobinState`.

Let's go by step by step. The _state_ is a tuple of `Map` and `Int`. `Map` is going to store hosts with some index. Here I preferred to use `Map` over `List` because accessing element is `O(N)` for `List` whereas it's `O(1)` for `Map`. The second member of the tuple will be used as a counter. In case of `GetHost`, interpreter is going to select the host by the modulo of the counter (number of payloads so far) over the size of the `Map` (number of hosts added so far). And of course, _state_ will be updated with the incremented counter. Pretty simple logic. Now let's see when the `program1` is executed with this interpreter.

```scala
val result = program1.foldMap(roundRobinRouting).run((Map.empty, 0)).value._2
println(result)
```

Notice that after `foldMap` function, `State` is being `run` with initial values, which is an empty `Map` for hosts and `0` for the counter.

<div class="console">List(Some(host1), Some(host2), Some(host3), Some(host1), Some(host2), Some(host3))
</div>

Works as expected! For each payload, interpreter produced the result as selecting host in Round Robin fashion. However, this result can lead to some problems.

## Problem

Below is the table showing payload sizes and destination hosts.

*Payload #*|1|2|3|4|5|6
*Payload Size*|2Kb|1Kb|1Kb|2Kb|1Kb|1Kb
*Destination*|host1|host2|host3|host1|host2|host3

So in total, 4Kb of data is routed to `host1` whereas remaining hosts received only 2Kb. This shows that the bandwidth usage towards each host is totally dependent on the **order** of processing in the application! Obviously, this can lead to  unpredictable network traffic to each of the hosts.

How to overcome this? How we can ensure that payloads will be distributed among hosts _fairly_?

## Priority Queue

A data structure is required to select most available (in this case having the least traffic) host when new payload will be routed. Luckily, `cats-collection` library has `BinaryHeap` which is an implementation of a _Priority Queue_ data structure. So here the idea is to store hosts in `BinaryHeap` along with the sent bytes so far to each of them. To do this, we need a new record as follows:

```scala
final case class TargetHost(host: String, tx: Long)
```

The `tx` value will store the total bytes sent to the host. So to use in the `BinaryHeap`, the `Order` implementation will use `tx` for comparing the entries:

```scala
implicit val targetHostOrdering: Order[TargetHost] = new Order[TargetHost] {
  override def compare(x: TargetHost, y: TargetHost): Int = (x.tx - y.tx).toInt
}
```

When a payload is going to be routed, the new (optimized) interpreter will remove the minimum element from the `BinaryHeap`, return the host as result and then add a new `TargetHost` with updating the `tx` with payload size. In this new interpreter, _state_ will only be the `BinaryHeap` itself. Let's look at it:

```scala
type FairRoutingState[T] = State[Heap[TargetHost], T]

def fairRouting: Routing ~> FairRoutingState = new (Routing ~> FairRoutingState) {
  override def apply[A](fa: Routing[A]): FairRoutingState[A] = fa match {
    case AddHost(host) => State.modify { pq => pq.add(TargetHost(host, 0L)) }
    case GetHost(payload) => State { pq =>
      pq.getMin.fold[(Heap[TargetHost], Option[String])]
        { pq -> None }
        { next => pq.remove.add(next.copy(tx = next.tx + payload.size)) -> Some(next.host)}
    }
  }
}
```

The optimized interpreter is even more simple than the initial one! Neat. Note that new hosts are added to `BinaryHeap` with `tx = 0`. So they immediately become available. It's good to mention that `BinaryHeap` has `O(logN)` for `remove` and `add` operations.

Let's re-run `program1` with the new optimized interpreter.

```scala
val result2: Seq[Option[String]] = program1.foldMap(fairRouting).run(Heap.empty).value._2
println(result2)
```

The result is:

<div class="console">List(Some(host1), Some(host3), Some(host2), Some(host3), Some(host2), Some(host1))
</div>

If it's tabulized:

*Payload #*|1|2|3|4|5|6
*Payload Size*|2Kb|1Kb|1Kb|2Kb|1Kb|1Kb
*Destination*|host1|host3|host2|host3|host2|host1

Which says that both `host1` and `host3` will receive total 3Kb of data and `host2` will receive 2Kb. Et voilà! Fair distribution is in place!

## Wiring with IO

The goal of creating an optimized interpreter is achieved. However, for a real-world example, let's try to introduce another algebra for network communication. Since there's logic for selecting a destination host, a network layer will be needed for transferring the payload!

```scala
sealed trait Network[T]
final case class Send(payload: Payload, host: String) extends Network[Unit]

type NetworkF[T] = Free[Network, T]

class NetworkI[F[_]](implicit I: InjectK[Network, F]) {
  def sendI(payload: Payload, host: String): Free[F, Unit] =
    Free.inject[Network, F](Send(payload, host))
}
```

To keep it simple, I only added a single function for sending the payload. Let's define an interpreter for `IO`.

```scala
def networkIO: Network ~> IO = ???
```

But before continuing to actual implementation, a problem needs to be solved. Imagine, both `RouteF` and `NetworkF` are used in the same _coproduct_:

```scala
type BrokenApp[T] = EitherK[Routing, Network, T]
```

With the following combined interpreter:

```scala
def brokenInterpreter: App ~> IO = fairRouting or networkIO
```

That will work? **No!** The reason is `fairRouting` is a _natural transformation_ from `Routing` to `FairRoutingState` whereas `networkIO` is a _natural transformation_ from `Network` to `IO`. Furthermore, this doesn't make any sense, since `Routing` was meant to create **optimized** routes for payloads, which has to be done **before** sending packets to the network. So how do we compose these two then?

A solution is to create a new algebra for processing the payloads in advance. Let's try:

```scala
trait RoutingTable[T]
case class Calculate(payloads: List[Payload])
  extends RoutingTable[List[(Payload, Option[String])]]

class RoutingTableI[F[_]](implicit I: InjectK[RoutingTable, F]) {
  def calculate(payloads: List[Payload]): Free[F, List[(Payload, Option[String])]] =
    Free.inject[RoutingTable, F](Calculate(payloads))
}
```

Say there's a function `calculate` which takes a list of payloads as a parameter and returns a list of the product of payload and host. The interpreter would look like:

```scala
val hosts = Seq("host1", "host2", "host3")

val routingTable: RoutingTable ~> IO = new (RoutingTable ~> IO) {
  override def apply[A](fa: RoutingTable[A]): IO[A] = fa match {
    case Calculate(payloads: List[Payload]) => {
      val routingF =
        hosts.toList
          .traverse(addHost)
          .flatMap(_ => payloads.traverse[RoutingF, Option[String]](getHost))
      routingF.foldMap(fairRoutingIO).run(Heap.empty).map(_._2).map(r => payloads.zip(r))
    }
  }
}
```

Which actually adds hosts to `RoutingF` and then selects destinations for all payloads received from the `Calculate`. As a careful reader, you can see that `routingF` itself is interpreted with `fairRoutingIO`. Actually, `routingTable` is an interpreter using one another interpreter beneath!

On the other hand, since `routingTable` transforms into `IO`, there needs to be a clever way to convert `Routing` to `IO`! But previously stated, `fairRouting` has a different signature. Well, here the solution lies in the definition of `State` monad in _Cats_ library.

```scala
// From cats-core
type State[S, A] = StateT[Eval, S, A]
```

So the idea is why not using `IO` instead of `Eval`? Now let's try to define `fairRoutingIO`:

```scala
type FairRoutingIOState[T] = StateT[IO, Heap[TargetHost], T]

val fairRoutingIO: Routing ~> FairRoutingIOState = new (Routing ~> FairRoutingIOState) {
  override def apply[A](fa: Routing[A]): FairRoutingIOState[A] = fa match {
    case AddHost(host) =>
      StateT { state => IO { state.add(TargetHost(host, 0L)) -> () } }
    case GetHost(payload) => StateT { state =>
      IO {
        state.getMin.fold[(Heap[TargetHost], Option[String])]
          { state -> None }
          { next =>
            state.remove.add(next.copy(tx = next.tx + payload.size)) -> Some(next.host)
          }
      }
    }
  }
}
```

Which is very similar to `fairRouting`, except now results are wrapped in `IO`.

The only thing left is `networkIO` implementation. Let's mock it for this example.

```scala
def networkIO: Network ~> IO = new (Network ~> IO) {
  override def apply[A](fa: Network[A]): IO[A] = fa match {
    case Send(payload, host) =>
      IO { println(s"Sending ${payload.size} bytes to $host") }
  }
}
```

Finally, we're ready to define `App` _coproduct_ along with the combined interpreter:

```scala
type App[T] = EitherK[RoutingTable, Network, T]
def interpreter: App ~> IO = routingTable or networkIO
```

As everything is in place, let's define `program2` and try to run it:

```scala
implicit def routingTableI[F[_]](implicit I: InjectK[RoutingTable, F]): RoutingTableI[F] =
  new RoutingTableI[F]()

implicit def networkI[F[_]](implicit I: InjectK[Network, F]): NetworkI[F] =
  new NetworkI[F]()

def program2(implicit routingTable: RoutingTableI[App], network: NetworkI[App])
  : Free[App, Unit] = {
  import routingTable._
  import network._

  val payloads: List[Payload] = List(
      largePayload,
      smallPayload,
      smallPayload,
      largePayload,
      smallPayload,
      smallPayload
  )

  type FreeApp[T] = Free[App, T]
  for {
    entries <-  calculate(payloads)
    _       <-  entries
                  .traverse[FreeApp, Unit](pair => sendI(pair._1, pair._2.get))
                  .map(_ => ())
  } yield ()

}

program2.foldMap(interpreter).unsafeRunSync()
```

And the result is:

<div class="console">Sending 2048 bytes to host1
Sending 1024 bytes to host3
Sending 1024 bytes to host2
Sending 2048 bytes to host3
Sending 1024 bytes to host2
Sending 1024 bytes to host1
</div>

Which shows, all payloads are fairly routed in an `IO` application.

## Conclusion

This work shows the possibility of using _Free Monads_ whenever an optimization is required. In addition, it's shown how to combine different interpreters targeting different _effects_.

As a next step, I'll try to do similar optimization with _tagless final_ approach. So 2 approaches can be compared in these blog posts. Later on, I will push the entire code to a Github repository as well.

Cheers!
