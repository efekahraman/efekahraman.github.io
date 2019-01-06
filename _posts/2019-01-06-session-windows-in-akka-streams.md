---
layout: post
title: Session Windows in Akka Streams
comments: true
tags: akka scala window stream time flow
license: true
revision: 1
---

<div class="message">
  One liner summary
</div>

### TL;DR

_Windowing_ is an important concept in _streaming_. [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) is a great source for detailed information. There is also [an excellent blog post](https://softwaremill.com/windowing-data-in-akka-streams/) summarizing the idea supported by examples. In this post, I implemented a session based window as a custom operator in Akka Streams. You can get the source code from my [GitHub repo](https://github.com/efekahraman/akka-streams-session-window) as well.

### Concept

Before deep diving into the implementation, it would be good to talk about concepts. However, it's recommended for readers to go through [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) who are not familiar with the streaming at all.

**Windowing**

Because of the unbounded nature of data, it's hard to choose how much to process at one shot. To cope with this problem, there needs to be a mechanism to chop down the data using temporal boundaries. This is where the idea of _windowing_ kicks in. It's basically the process of slicing up continuous data. Generally, there are 3 types of _windows_:

* _Tumbling window_ divides data into non-overlapping chunks where each element of data belongs to a single window. Each window has the same length and does not overlap.
* _Sliding window_ puts elements into several overlapping windows which advance by slices over time. Each window has the same length.
* _Session window_ is different from _tumbling window_ and _sliding window_ in a way that each window might have a different length - it does not have a fixed start and end time. Rather it's defined by idleness between two element sets.

**Session Windows**

Since it's the main subject of this blog post, let's look at it deeply. _Session windows_ represent a period of activity. Concretely, _windows_ are separated by a predefined inactivity/gap period. This leads some consequences:

* Any element that falls within the inactivity gap will be merged into the existing session.
* When used on _keyed streams_ each window will have different start and end times for different keys.
* Each window might have various sizes. Window size totally depends on the data.
* And they do not overlap.

Here's the visual representation of a _session window_ on multiple _keyed stream_:

<center><img src="{{ site.baseurl }}/public/media/3-Session-Window.png" height="320" width="640"></center>

_Session windows_ are useful when grouping elements which "belong" together. It captures bursts of activity based on the times they occur. For example, it can help to analyze user behaviours or grouping alerts (or any temporally-related events). In that vein, generally, it's useful to apply session windows together with _event time_ processing.

One more thing, the idle time/gap of a session window can be defined either statically or dynamically. The former specifies a static value for the idle period, whereas the latter uses a function to be applied to each element to get the updated value. That being said, I've implemented the session window only with predefined values for now.

### Design

Akka Streams do not provide a mechanism for _session windowing_ out of the box, however, there are a couple of ways to implement custom components. Akka Streams already provides a number of operators which can be combined and form a "box". Alternatively, a custom `GraphStage` can be implemented for the desired behaviour. All information related to operators and custom stream processing is given in the excellent [documentation](https://doc.akka.io/docs/akka/current/stream/index.html?language=scala) of the Akka Streams.

Before going further, let's check similar operators present in the Akka Stream toolbox.
* `groupedWithin`: _Chunk up the stream into groups of elements received within a time window, or limited by the number of the elements_ [(from the official doc)](https://doc.akka.io/docs/akka/current/stream/operators/Source-or-Flow/groupedWithin.html). This can be considered as a _tumbling window_.
* `sliding`: _Provide a sliding window over the incoming stream and pass the windows as groups of elements downstream_ [(from the official doc)](https://doc.akka.io/docs/akka/current/stream/operators/Source-or-Flow/sliding.html). Not surprisingly, this should be the sliding window.
* `idleTimeout`: _If the time between two processed elements exceeds the provided timeout, the stream is failed with a TimeoutException_ [(from the official doc)](https://doc.akka.io/docs/akka/current/stream/operators/Source-or-Flow/idleTimeout.html). This is of course not related to _session windowing_ but provides a timeout control on the incoming events.

In the _Code_ section, I've implemented the _session window_ mechanism by creating a new transformation operator, getting inspired by `idleTimeout`. Since all events need to be buffered until the _session window_ is closed, there needs to be a data structure to efficiently store events. For this purpose, I've selected `immutable.Queue` as it has `O(C)` and amortized `O(C)` time complexities for `append` (`enqueue`) and `tail` functions respectively. The need for the latter will be discussed in the later explanations of the _Code_ section.

It's good to mention that the idea of _session window_ is natively supported in other streaming platforms, such as Apache Flink. But unlike these distributed platforms, Akka Streams target a single JVM (data node).

### Code

**Versions**

Scala|2.12.8
Akka|2.5.16

**Implementation**

The first thing is to create a class extending `GraphStage[FlowShape[T, T]]` since the _session window_ will always be a _Flow Shape_ processing the same type of elements. In the concrete class, 2 members need to be implemented: `shape` and `createLogic`. The former defines the shape of a graph by `Inlet` and `Outlet` instances. The latter includes the inner logic which is the part doing the heavy work. It's important to emphasize that all the methods inside of the `GraphStageLogic` instance are never called concurrently. Which, in turn, gives flexibility to modify any _state_ without any further synchronization from the callback methods. Just like Actors!

Another useful class to use in this case is `TimerGraphStageLogic` which provides methods for scheduling timers. So the idea behind the implementation below is firing up a timer on intervals of `inactivity` time and check when the last event was pushed in. If the last event was pushed older than `inactivity` time then emit all the events in the buffer. Meanwhile, if an event arrives, append it to the buffer and update the last seen timestamp (named `nextDeadline` below).

Logically, the _session window_ needs a configured inactivity/gap duration which can be placed as a constructor argument. Besides, since there's a memory limit of the JVM process, there needs to be another constructor argument giving the upper bound of elements in the window. Having these in mind, let's check the code.

(Yes, there are `var`s. Sad but true.)

```scala
final class SessionWindow[T](val inactivity: FiniteDuration, val maxSize: Int)
  extends GraphStage[FlowShape[T, T]] {

  val in  = Inlet[T]("SessionWindow.in")
  val out = Outlet[T]("SessionWindow.out")

  override val shape: FlowShape[T, T] = FlowShape(in, out)

  override def createLogic(inheritedAttributes: Attributes): GraphStageLogic =
    new TimerGraphStageLogic(shape) with InHandler with OutHandler {

      private var queue: Queue[T]      = Queue.empty[T]
      private var nextDeadline: Long   = System.nanoTime + inactivity.toNanos

      setHandlers(in, out, this)

      override def preStart(): Unit = schedulePeriodically(shape, inactivity)
      override def postStop(): Unit = queue = Queue.empty[T]

      override def onPush(): Unit = {
        val element: T = grab(in)

        queue = if (queue.size < maxSize) queue.enqueue(element) else queue
        nextDeadline = System.nanoTime + inactivity.toNanos

        if (!hasBeenPulled(in)) pull(in)
      }

      override def onPull(): Unit = if (!hasBeenPulled(in)) pull(in)

      override def onUpstreamFinish(): Unit = {
        if (queue.nonEmpty) {
          emitMultiple(out, queue)
        }
        super.onUpstreamFinish()
      }

      final override protected def onTimer(key: Any): Unit =
        if (nextDeadline - System.nanoTime < 0 && queue.nonEmpty) {        
            emitMultiple(out, queue)
            queue = Queue.empty[T]
        }
    }
}
```

Let's go step by step.

`setHandlers` method can directly refer to `this` since the class instance has the types of `InHandler` and `OutHandler`.

`preStart` hook is invoked at the startup of the operator which makes it a suitable place to schedule the timer. Compared to scheduling a timer every time when an event is pushed, running a periodical timer and checking the timestamp is much more cost effective. Another lifecycle hook, `postStop`, is invoked when the operator is about to stop or fail. In this method, assigning the `queue` to the empty `Queue` object will help garbage collector to immediately detect and collect no-longer-necessary entries.

`onPush` callback is invoked when there is a new element from the upstream. In this method, once the element is `grab`bed, `queue` is updated with the reference of the new structure if it's not exceeding the `maxSize`. Also, the `nextDeadline` is updated as the current time plus the inactivity time, which will be checked from `onTimer` method when the timer triggered. Lastly, `pull(in)` is called if there's no pending `pull` request. This needs to be done since cannot rely on pulling frequency solely from downstream. On the other hand, `onPull` callback is invoked when the output port has received a pull. Because `pull(in)` is also called in the `onPush` method, there needs to be a check for pending `pull` request using `hasBeenPulled` function.

`onUpstreamFinish` callback is invoked once the upstream has completed. So what needs to be done is to emit all the elements in the buffer.

`onTimer` is the callback when the scheduled timer is triggered. Here's the logic to determine whether the window should be closed or not. If the `nextDeadline` is older than the current system time, then it means the last element showed up before the desired gap time and therefore window can be closed. Therefore all elements should be emitted and buffer needs to be cleared up. Notice that `emitMultiple` is used to emit all elements. This is very important. Because there's no guarantee that outport is available in that particular moment calling `push(out)` might fail. However, `emitMultiple` emits elements only when there is demand, which is a guaranteed way to do in this callback. One side note, it can be dangerous to mix `pull`/`push` along with the `emit`/`read` APIs, but this example is fairly small and it's all under control.

**Tests**

Let's create a very basic test and see what happens.

```scala
class SessionWindowSpec extends WordSpec {

  implicit val system: ActorSystem = ActorSystem("SessionWindowSpec")
  implicit val materializer: ActorMaterializer = ActorMaterializer()

  "SessionWindow" must {
    "accumulate window respecting the order of elements until a gap occurs" in {
      def sessionWindow[T]: SessionWindow[T] = SessionWindow(1 second, 10)
      val future: Future[List[Int]] =
        Source(1 to 10)
          .via(sessionWindow)
          .runFold(Nil: List[Int])(_ :+ _)
      val result: List[Int] = Await.result(future, 2 seconds)
      assert(result == (1 to 10).toList)
    }
  }
}
```

Here gap is configured to `1 second`, assuming that all `10` elements will be pushed less than a second. Note that the order of the elements is preserved.

**Adding Custom Overflow Strategy**

The implementation below discards the elements once the queue reaches the `maxSize`. But wouldn't it be better if the user could decide what's going to happen when buffer overflows? For this purpose, let's introduce the following _ADT_:

```scala
sealed trait SessionOverflowStrategy
case object DropOldest extends SessionOverflowStrategy
case object DropNewest extends SessionOverflowStrategy
case object FailStage  extends SessionOverflowStrategy
```

Since the user will specify the strategy, `SessionWindow` class will have another constructor argument of type `SessionOverflowStrategy`. `DropOldest` indicates oldest element should be discarded when a new element is pushed (FIFO approach). Contrarily, `DropNewest` indicates that new elements will be dropped silently. `FailStage`, on the other hand, will fail the entire pipeline. Furthermore, when `FailStage` is used, it's better to have a specific `Exception` type as follows:

```scala
final case class SessionOverflowException(msg: String) extends RuntimeException(msg)
```

The overflow control logic is be implemented in the `onPush` method, since it's the only place new elements are pushed.

```scala
final class SessionWindow[T](val inactivity: FiniteDuration, val maxSize: Int, val overflowStrategy: SessionOverflowStrategy)
  extends GraphStage[FlowShape[T, T]] {

  // ...

  override def onPush(): Unit = {
    val element: T = grab(in)
    queue =
      if (queue.size < maxSize) queue.enqueue(element)
      else overflowStrategy match {
          case DropOldest =>
            if (queue.isEmpty) queue.enqueue(element)
            else queue.tail.enqueue(element)
          case DropNewest => queue
          case FailStage  =>
            failStage(SessionOverflowException(s"Received messages are more than $maxSize"))
            Queue.empty[T]
      }
    nextDeadline = System.nanoTime + inactivity.toNanos

    if (!hasBeenPulled(in)) pull(in)
  }

  // ...
}
```

In the case of `DropOldest` the head element of the queue is discarded. Note that `tail` takes amortized constant time (`O(aC)`) on `Queue`. If `FailStage` strategy is selected, then `failStage` method is called with a `SessionOverflowException` instance and `queue` is immediately assigned to the empty `Queue` object to help garbage collector.

**More Tests**

Let's check some test cases for the improved functionality.

```scala
class SessionWindowSpec extends WordSpec {

  implicit val system: ActorSystem = ActorSystem("SessionWindowSpec")
  implicit val materializer: ActorMaterializer = ActorMaterializer()

  "SessionWindow" must {
    def run(range: Range,
            sessionWindow: SessionWindow[Int],
            atMost: FiniteDuration): Try[List[Int]] = {
      val future: Future[List[Int]] =
        Source(range)
          .via(sessionWindow)
          .runFold(Nil: List[Int])(_ :+ _)
      Try(Await.result(future, atMost))
    }

    "drop oldest entries when maxSize is reached" in {
      val window: SessionWindow[Int] = SessionWindow(1 second, 5, DropOldest)
      val result: Try[List[Int]] = run(1 to 10, window, 2 seconds)
      assert(result.isSuccess)
      assert(result.get == (6 to 10).toList)
    }

    "drop newest entries when maxSize is reached" in {
      val window: SessionWindow[Int] = SessionWindow(1 second, 5, DropNewest)
      val result: Try[List[Int]] = run(1 to 10, window, 2 seconds)
      assert(result.isSuccess)
      assert(result.get == (1 to 5).toList)
    }

    "fail the graph stage when maxSize is reached" in {
      val window: SessionWindow[Int] = SessionWindow(1 second, 5, FailStage)
      val result: Try[List[Int]] = run(1 to 10, window, 2 seconds)
      assert(result.isFailure)
      assert(result.failed.get.isInstanceOf[SessionOverflowException])
    }
  }
}
```

Pretty self-explanatory!

### Conclusion

The presented code provides _session windows_ extension for the Akka Streams. When compared to other streaming platforms such as Apache Flink or Kafka Streams. it's not supported natively, however, it can be implemented very easily with the help of the custom stream processing infrastructure.

Later on, I'll convert the [project code](https://github.com/efekahraman/akka-streams-session-window) to a library.

Cheers!
