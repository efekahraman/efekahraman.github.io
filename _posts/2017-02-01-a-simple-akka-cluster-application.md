---
layout: post
title: A Simple Akka Cluster Application
comments: true
tags: akka scala cluster routing load-balancing
license: true
revision: 2
summary: This post explains a basic Akka Cluster application consisting of Producer and Consumer roles where each role runs on a separate node and it's own JVM.
---

Akka Cluster is not new. So is load balancing. You can find some cool articles [here](http://freecontent.manning.com/akka-in-action-why-use-clustering/) and [here](http://blog.kamkor.me/Akka-Cluster-Load-Balancing/). Also, there’s a really nice sample at Akka’s own [GitHub repo](https://github.com/akka/akka-samples/tree/master/akka-sample-cluster-scala). In this post, I made a similar work and created a simple cluster application as it’s going to be the foundation of upcoming posts. You can get the project code from my [GitHub repo](https://github.com/efekahraman/akka-cluster-demo) as well.

### <a name="Design"></a>Design

The goal of the design is to have a cluster of _P_ x _C_ nodes where _P_ and _C_ denote the number of _Producer_ and _Consumer_ nodes respectively. As the number of nodes is not known in the beginning and can vary over time, it's useful to have a _Seed_ node to form the cluster. _Seed_ node is an actor system without any actor, and it's used as an entry point to the cluster for newly created nodes. To make _Seed_ node reachable, its address is fixed and predefined in all other nodes.

When a _Consumer_ node is up, each _Producer_ is notified and start sending messages to the new node. To resolve the Consumer actor properly, I decided to go further with an assumption that every _Consumer_ node will have a single Consumer actor. However, _Consumer_ actor itself can behave as a "notifier" in its own actor system and pass messages to other local actors as well.

At the first stage, _Producer_ role will use Round Robin routing. However, routing mechanism will be kept as abstract.

For example, 2 _Producer_ x 3 _Consumer_ cluster will look like as follows:

<center><img src="{{ site.baseurl }}/public/media/1-Diagram.png" height="320" width="640"></center>

It's worth mentioning that Akka has built-in [Cluster Aware Router](http://doc.akka.io/docs/akka/2.4/scala/cluster-usage.html#Cluster_Aware_Routers) which provides a Router mechanism aware of member nodes in the cluster. When used with *Group of Routees*, it covers the goal of this design either. That being said, I've implemented the whole mechanism in this project.

### Code

**Versions**

Scala|2.12.1
Akka|2.4.17
sbt|0.13.13

**Structure**

All of the roles are placed in a single project. SBT [assembly plugin](https://github.com/sbt/sbt-assembly) is used to generate standalone JAR file.

**Configuration**

The only configuration file is `application.conf`. To make the application **container friendly** from day zero, I defined several placeholders under the `args`. (Containerization of cluster nodes will be demonstrated in upcoming posts).

```
args {
  host = "127.0.0.1"
  host = ${?host}
  port = 0
  port = ${?port}
  app-name = "AkkaClusterDemo"
  app-name = ${?app-name}
  seed-host = "127.0.0.1"
  seed-host = ${?seed.host}
  seed-port = 2551
  seed-port = ${?seed.port}
}
```

In the next `akka` section of the same file, `provider` is set to `cluster` which enables Cluster extension to be used. Also, `seed-nodes` is configured to point single node address, which is composed of variables shown above.

```
akka {
  actor {
    provider = cluster
  }

  cluster {
    seed-nodes = ["akka.tcp://"${args.app-name}"@"${args.seed-host}":"${args.seed-port}]
    # roles = ["role"]
  }

  remote {
    log-remote-lifecycle-events = off
    netty.tcp {
      hostname = ${args.host}
      port = ${args.port}
    }
  }
}
```

Note that `roles` definition is disabled, as this property is overwritten in [main](#Main) method.

**ConsumerActor**

ConsumerActor is kept minimal as it prints incoming message's content only.

```scala
override def receive(): Receive = {
  case msg => log.info(s"""${sender.path.address} : $msg""")
}
```

**ProducerActor**

This is the part doing the heavy work. As discussed in the [Design](#Design) section, _Producer_ role should be capable of receiving events for _Consumer_ nodes. To achieve this, `Actor` subscribes itself to `MemberUp` notification, which occurs when a new node is accepted to the cluster.

```scala
val cluster = Cluster(context.system)
override def preStart(): Unit = cluster.subscribe(self, classOf[MemberUp])
```

The `preStart` function is called just after the actor instance is created (see Akka [docs](http://doc.akka.io/docs/akka/current/scala/actors.html#Actor_Lifecycle)), so it's definitely the only place to subscribe to `Cluster`. Once Producer receives the `MemberUp` notification for particular Consumer, it needs to subscribe itself to the lifecycle events of that node to catch the termination. Besides, routings should be updated as new destination becomes available. To achieve this, Consumer node is resolved in `receive` function, which looks like as follows:

```scala
override def receive(): Receive = {
  case MemberUp(member) if (member.roles.contains(Consumer)) =>
    log.info(s"""Received member up event for ${member.address}""")
    val consumerRootPath = RootActorPath(member.address)
    val consumerSelection = context.actorSelection(consumerRootPath / "user" / Consumer)

    import scala.concurrent.duration.DurationInt
    import context.dispatcher
    consumerSelection.resolveOne(5.seconds).onComplete(registerConsumer)
  case Terminated(actor) => strategy.removeRoutee(actor)
  case SimpleMessage =>
    strategy.sendMessage(s"#$counter")
    counter += 1
  case _ =>
}
```

First, let's focus on the first `case` statement, which is matched when `MemberUp` message is received for any node having the _Consumer_ role. As `Address` is accessible from `Member` object, `ActorPath` can be easily built for targeted Consumer actor which is used for resolving the target `ActorRef` with the help of `ActorSelection`. This point actually brings the restriction of having single Consumer actor at a remote node, since it'd be unclear for Producer if there were multiple `Actor`s sharing the same path. Now let's look at the `registerConsumer` function, which is run when resolving completes:

```scala
def registerConsumer(refTry: Try[ActorRef]): Unit = refTry match {
  case Success(ref) =>
    context watch ref
    strategy.addRoutee(ref)
  case Failure(_) => log.error("Couldn't find consumer on path!")
}
```

In `Success` case, `Actor` registers itself for the lifecycle events of the targeted Consumer actor which is referenced by `ref`. So that `Terminated` case could have been implemented in `receive` function. Next, `ref` is used to updating routings. This is done via `strategy`, which has the type of `RouterStrategy` defined as follows:

```scala
trait RouterStrategy {
    def addRoutee(ref: ActorRef): Unit;
    def removeRoutee(ref: ActorRef): Unit;
    def sendMessage[M >: Message](msg: M): Unit;
}
```

The current implementor of this trait is `RoundRobinStrategy`, which utilizes `RoundRobinGroup` underneath. Here I added  `RouterStrategy` as an abstraction to easily extend the project with different routing capabilities in future.

<a name="Main"></a>**Main**

The `main` method determines the role from the first argument and sets the configuration as follows:

```scala
val role = args.headOption.getOrElse(Seed)
val config = ConfigFactory.parseString(s"""akka.cluster.roles = ["$role"]""")
  .withFallback(ConfigFactory.load())
```

Once the role is determined, an `ActorSystem` along with a corresponding `Actor` starts. If the role is defined as `producer`, a message generator also starts for demonstration purpose. After an initial delay of 10 seconds, a new message is generated in every 2 seconds.

```scala
def startMessageGenerator(producer: ActorRef): Unit = {
  import scala.concurrent.duration.DurationInt
  import system.dispatcher
  system.scheduler.schedule(10 seconds, 2 seconds, producer, SimpleMessage)
}
```
### Results

<div class="message">
  In this section, I truncated big portions of output logs for the sake of simplicity.
</div>

Let's start a seed node at first. Here `port` is set to `2551` as it's the default port for _Seed_. Note that this is done via exporting the environment variable.

<div class="console">$ export port=2551
$ java -jar AkkaClusterDemo-assembly-1.0.jar
</div>

Next, let's start a _Consumer_ node.

<div class="console">$ java -jar AkkaClusterDemo-assembly-1.0.jar consumer
[main] [akka.remote.Remoting] Starting remoting
[main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://AkkaClusterDemo@127.0.0.1:55643]
...
</div>

As soon as _Consumer_ starts and joins the cluster, _Seed_ node transfers the leadership to that node:

<div class="console">[akka.cluster.Cluster(akka://AkkaClusterDemo)] Cluster Node [akka.tcp://AkkaClusterDemo@127.0.0.1:2551] - Leader is moving node [akka.tcp://AkkaClusterDemo@127.0.0.1:55643] to [Up]
</div>

However, we still need to keep _Seed_ node to welcome next nodes. Also, note that Consumer node took port `55643` as there was no particular port defined. Now let's start a _Producer_ node and see what happens.

<div class="console">$ java -jar AkkaClusterDemo-assembly-1.0.jar producer
[main] [akka.remote.Remoting] Starting remoting
[main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://AkkaClusterDemo@127.0.0.1:55645]
...
</div>

And once the _Producer_ node joined to the cluster, it receives `MemberUp` event for the _Customer_ node:
<div class="console">[akka.tcp://AkkaClusterDemo@127.0.0.1:55645/user/producer] Received member up event for akka.tcp://AkkaClusterDemo@127.0.0.1:55643
</div>

So that shows whenever a node joins the cluster, it receives the member events for running nodes as well. Now if we check the output of _Consumer_ node, we can see the messages started arriving from _Producer_, which is started on port `55645`.

<div class="console">[akka.tcp://AkkaClusterDemo@127.0.0.1:55643/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #0
[akka.tcp://AkkaClusterDemo@127.0.0.1:55643/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #1
[akka.tcp://AkkaClusterDemo@127.0.0.1:55643/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #2
[akka.tcp://AkkaClusterDemo@127.0.0.1:55643/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #3
...
</div>

That's cool. Right now, we have a 1 _Producer_ x 1 _Consumer_ cluster. What happens if we add two more _Consumer_ nodes to the cluster ? Let's start and see.

<div class="console">$ java -jar AkkaClusterDemo-assembly-1.0.jar consumer
[main] [akka.remote.Remoting] Starting remoting
[main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://AkkaClusterDemo@127.0.0.1:55648]
...
[akka.tcp://AkkaClusterDemo@127.0.0.1:55648/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #12
[akka.tcp://AkkaClusterDemo@127.0.0.1:55648/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #14
[akka.tcp://AkkaClusterDemo@127.0.0.1:55648/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #16
[akka.tcp://AkkaClusterDemo@127.0.0.1:55648/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #18
...
</div>

This _Customer_ node started on port `55648` and started to receive messages immediately after joining the cluster. Note that message numbers are started to increase by 2. Why ? This is because _Producer_ is started to distribute messages among 2 _Consumer_ nodes in Round Robin way.

Next, let's start another _Consumer_ node. This time expected behaviour is that every _Consumer_ node will receive messages with numbers increasing by 3.

<div class="console">$ java -jar AkkaClusterDemo-assembly-1.0.jar consumer
[main] [akka.remote.Remoting] Starting remoting
[main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://AkkaClusterDemo@127.0.0.1:55652]
...
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #26
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #29
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #32
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #35
...
</div>

And new logs are written at _Producer_ node showing the events for new _Consumer_ nodes as expected:

<div class="console">...
[akka.tcp://AkkaClusterDemo@127.0.0.1:55645/user/producer] Received member up event for akka.tcp://AkkaClusterDemo@127.0.0.1:55648
[akka.tcp://AkkaClusterDemo@127.0.0.1:55645/user/producer] Received member up event for akka.tcp://AkkaClusterDemo@127.0.0.1:55652
...
</div>

So far good. Right now there is one _Producer_ node dispatching messages to 3 _Consumer_ nodes. Let's assume we need one more _Producer_ node in this cluster. So let's start a new _Producer_ and see what happens.

<div class="console">$ java -jar AkkaClusterDemo-assembly-1.0.jar producer
[main] [akka.remote.Remoting] Starting remoting
[main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://AkkaClusterDemo@127.0.0.1:55657]
...
[akka.tcp://AkkaClusterDemo@127.0.0.1:55657/user/producer] Received member up event for akka.tcp://AkkaClusterDemo@127.0.0.1:55652
[akka.tcp://AkkaClusterDemo@127.0.0.1:55657/user/producer] Received member up event for akka.tcp://AkkaClusterDemo@127.0.0.1:55643
[akka.tcp://AkkaClusterDemo@127.0.0.1:55657/user/producer] Received member up event for akka.tcp://AkkaClusterDemo@127.0.0.1:55648
...
</div>

New _Producer_ node started on port `55657` and received `MemberUp` events immediately after joining the cluster. Now there are 3 _Consumer_ nodes, which are receiving Round Robin distributed messages from 2 different _Producer_ nodes. To be concrete, the output of latest _Consumer_ node looks like follows:

<div class="console">...
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #41
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55657 : #0
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #43
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55657 : #3
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #46
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55657 : #6
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55645 : #49
[akka.tcp://AkkaClusterDemo@127.0.0.1:55652/user/consumer] akka.tcp://AkkaClusterDemo@127.0.0.1:55657 : #9
...
</div>

### Conclusion

The presented application provides core functionality for forming an Akka cluster which can easily scale over time.

In this application, `ProducerActor` is the **only place** which is responsible for _Consumer_ node management, in contrast to [the Akka's own sample application](https://github.com/akka/akka-samples/tree/master/akka-sample-cluster-scala/src/main/scala/sample/cluster/transformation), where _Frontend_ receives state messages and informs _Backend_ with a special message type.

Depending on the design, one can say that _Seed_ node can be a single point of failure. However, since _Seed_ node is only needed when accepting a new node into the cluster, it can be even stopped or restarted rest of the time, without any impact on messaging between _Producer_ and _Consumer_ nodes.

Next post will be built on this application and demonstrate how to collect metrics on Akka clusters.

Cheers!
