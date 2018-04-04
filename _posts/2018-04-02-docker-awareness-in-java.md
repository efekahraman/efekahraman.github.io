---
layout: post
title: Docker Awareness in Java
comments: true
tags: java jvm docker memory cpu container
license: true
revision: 1
---

<div class="message">
  This post summarizes how the effects of the Docker containerization are handled in latest Java versions.
</div>

### TL;DR

Starting from Java 8 update 131, a number of features are introduced to Java to improve getting the correct resource limits when running in a Docker containers. In this blog post, I experimented these features for each Java version (8, 9 and 10) under different container configurations.

### <a name="Docker"></a>Docker

Before stating the problem, it would be good to refresh the basics behind Docker containerization. Docker uses two main Linux kernel components: [namespaces(7)](http://man7.org/linux/man-pages/man7/namespaces.7.html) and [cgoups(7)](http://man7.org/linux/man-pages/man7/cgroups.7.html).

In general, `namespaces` wraps a _global_ resource (such as network, mount point, PID) and makes it visible to a particular _namespace_ where it's associated with a _local_ resource.  In Docker, the `namespaces` make the containerized process isolated from other processes running on the same Docker machine. For example, the result of `ps -ef` shows only one process - the running container itself.

On the other hand, the `cgroups` -which stands for _control groups_- provides a facility to limit the resource consumption of processes in a hierarchical way. For example, resources suchs as CPU and memory can be confined to each container process using `cgroups`. The CPU usage can be limited by two parameters:
* `cpu` (_CPU Shares_): Specifies a relative share of CPU time for a container. In Docker, it's configured via `--cpu-shares` parameter. Default value is `1024` [[ArchLinux Wiki]](https://wiki.archlinux.org/index.php/Cgroups)[[Docker Docs]](https://docs.docker.com/config/containers/resource_constraints/#configure-the-default-cfs-scheduler).
    * It's worth to mention that when there's no competitor for the container, it can use CPU cycles as much as possible.
    * For multi core systems, the `share` is distributed to all cores.
* `cpusets` (_CPU Sets_): Binds the container to a specified set of CPUs. In Docker, it's configured via `--cpuset-cpus` parameter. [[Docker Docs]](https://docs.docker.com/config/containers/resource_constraints/#configure-the-default-cfs-scheduler).

### Problem

The problem arises when JVM gets the configuration for CPUs and memory directly from the underlying host instead of Docker container. In the other words, regardless of how many containers are running in parallel or any given limits on CPU and/or memory for particular containers, JVM always favours the configuration of Docker machine itself.

This problem is alleviated to some extent in Java 8 update 131 and Java 9, however, it's completely solved in Java 10. Before jumping into the
[Improvements](#Improvements) section, let's talk a bit about the possible outcomes of this situation.

CPU limits can affect Java application in various ways. From JVM perspective, the number of GC threads and JIT compiler threads are set according to available processors (unless they're specified explicitly via JVM parameters). Furthermore, the application itself might build a thread pool according to this limit. For example, some executors in Akka [Dispatcher](https://doc.akka.io/docs/akka/current/dispatchers.html#looking-up-a-dispatcher) are using this limit to calculate their thread pool size.

CPU limits can drain the performance but memory limits can be fatal as leading to `OutOfMemoryError`s. Unlike specified explicitly via `-Xmx`, JVM allocates one-fourth of the system's memory for heap space (this can be checked from _HotSpotVM GC Ergonomics_ of version [8](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html#sthref5), [9](https://docs.oracle.com/javase/9/gctuning/ergonomics.htm#JSGCT-GUID-DA88B6A6-AF89-4423-95A6-BBCBD9FAE781) and [10](https://docs.oracle.com/javase/10/gctuning/ergonomics.htm#JSGCT-GUID-DA88B6A6-AF89-4423-95A6-BBCBD9FAE781)). Together with the native memory usage of JVM, it's clear that this calculation will bring a high risk when the container is limited to use a lesser amount of memory.

### <a name="Improvements">Improvements

This section summarizes the improvements per Java versions.

**Java 8 update 131 and Java 9**

Issue Id| [JDK-8170888](https://bugs.openjdk.java.net/browse/JDK-8170888)
Issue Id|[JDK-6515172](https://bugs.openjdk.java.net/browse/JDK-6515172)

A new JVM option `-XX:+UseCGroupMemoryLimitForHeap` is introduced for getting the correct memory limit. When this flag is used along with `-XX:+UnlockExperimentalVMOptions`, JVM becomes capable of reading the memory limit from `cgroups`.

Another improvement is that JVM became capable of utilizing _CPU Sets_ automatically (See [Docker](#Docker) section).

**Java 10**

Issue Id| [JDK-8146115](https://bugs.openjdk.java.net/browse/JDK-8146115)
Issue Id| [JDK-8179498](https://bugs.openjdk.java.net/browse/JDK-8179498)

The main improvement makes JVM configure memory and CPU solely from `cgroups`. As a result, not only _CPU Sets_ but also _CPU Shares_ are now examined by JVM. Furthermore, this becomes the **default behaviour**, and can only be disabled via `-XX:-UseContainerSupport` option.

Another improvement makes diagnostic commands to be capable of attaching JVM process inside a container. (See the `namespaces` paragraph of [Docker](#Docker)).

### Setup

Docker Version|18.03.0

First, let's create the Docker machine with 2 CPUs and 1GB of memory.

```
docker-machine create \
    --driver virtualbox \
    --virtualbox-cpu-count "2" \
    --virtualbox-memory "1024" \
    default
```

A separate `Dockerfile` is created per Java version. Files are based on the latest `Ubuntu` image and using the **OpenJDK** distributions. The source of the `DockerTest.class` is listed in the next section.

**Java 8**

```
FROM ubuntu:latest
RUN  apt-get update
RUN  apt-get --assume-yes install openjdk-8-jre
COPY DockerTest.class /
CMD  java ${JAVA_OPT} DockerTest
```
* Note: the `openjdk-8-jre` package contains OpenJDK version 1.8.0 update 151.

**Java 9**
```
FROM ubuntu:latest
RUN  apt-get update
RUN  apt-get --assume-yes install wget
RUN  wget https://download.java.net/java/GA/jdk9/9.0.4/binaries/openjdk-9.0.4_linux-x64_bin.tar.gz
RUN  tar -zxvf openjdk-9.0.4_linux-x64_bin.tar.gz
COPY DockerTest.class /
CMD  /jdk-9.0.4/bin/java ${JAVA_OPT} DockerTest
```

* Note: Here the OpenJDK 9.0.4 is manually installed instead of `apt-get install openjdk-9-jre` because the latter is shipped with a release which hasn't ported the `XX:+UseCGroupMemoryLimitForHeap` yet.

**Java 10**
```
FROM ubuntu:latest
RUN  apt-get update
RUN  apt-get --assume-yes install wget
RUN  wget https://download.java.net/java/GA/jdk10/10/binaries/openjdk-10_linux-x64_bin.tar.gz
RUN  tar -zxvf openjdk-10_linux-x64_bin.tar.gz
COPY DockerTest.class /
CMD  /jdk-10/bin/java ${JAVA_OPT} DockerTest
```

#### Code

Below code simply prints the number of available processors and the maximum memory used by the JVM process. The last statement is added to keep JVM process alive.

```java
public class DockerTest {
  public static void main(String[] args) throws InterruptedException {
    Runtime runtime = Runtime.getRuntime();
    int  cpus = runtime.availableProcessors();
    long mmax = runtime.maxMemory() / 1024 / 1024;
    System.out.println("System properties");
    System.out.println("Cores       : " + cpus);
    System.out.println("Memory (Max): " + mmax);
    while (true) Thread.sleep(1000);
  }
}
```

### Results

#### Java 8 update 151

<div class="console">$ docker build -f Dockerfile-JDK8 .
[TRUNCATED]
Successfully built d9b244c265c5

$ docker run d9b244c265c5
System properties
Cores       : 2
Memory (Max): 241
</div>

Initially, when the Java 8 container starts, it sees `2` cores and allocates `241MB` of memory (`1024/4=256`). Now let's try to limit the resources and see the results.

<div class="console">$ docker run -c 512 -m 512MB d9b244c265c5
System properties
Cores       : 2
Memory (Max): 241
</div>

Here the `-c 512` sets the _CPU Shares_ to `512`, which advises using half of the available CPU time. And the `-m 512MB` limits the memory to given number. As expected, these arguments are not working in this Java version.

However, Java 8 update 151 has the _CPU Sets_ improvement. This time let's try with setting the  `--cpuset-cpus` to a single core.

<div class="console">$ docker run --cpuset-cpus 0 -m 512MB d9b244c265c5
System properties
Cores       : 1
Memory (Max): 241
</div>

And it's working. This version also allows us to use the `-XX:+UseCGroupMemoryLimitForHeap` option to get the correct memory limit.

<div class="console">$ docker run --cpuset-cpus 0 -m 512MB -e JAVA_OPT="-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap" d9b244c265c5
System properties
Cores       : 1
Memory (Max): 123
</div>

With the help of this option, finally, `123MB` of heap space is allocated, which perfectly makes sense for the upper limit of `512MB`.

#### Java 9

It would be enough to repeat the last step from the previous section since the functionality is same.

<div class="console">$ docker build -f Dockerfile-JDK9 .
[TRUNCATED]
Successfully built b11e577c5e3b

docker run --cpuset-cpus 0 -m 512MB -e JAVA_OPT="-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap" b11e577c5e3b
System properties
Cores       : 1
Memory (Max): 123
</div>

As expected, Java 9 recognized the _CPU Sets_ and the memory limits when `-XX:+UseCGroupMemoryLimitForHeap` is used.

#### Java 10

Since Java 10 is the Docker-aware version, resource limits should have taken effect without any explicit configuration.

<div class="console">$ docker run --cpuset-cpus 0 -m 512MB 0636036af04d
System properties
Cores       : 1
Memory (Max): 123
</div>

The previous snippet shows that _CPU Sets_ are handled correctly. Now let's try with setting _CPU Shares_:

<div class="console">$ docker run -c 512 -m 512MB 0636036af04d
System properties
Cores       : 1
Memory (Max): 123
</div>

It's working as expected. Also, it's worth to see this feature can be disabled via the `-XX:-UseContainerSupport` option (note that it starts with `-` after the `-XX:` prefix):

<div class="console">$ docker run -c 512 -m 512MB -e JAVA_OPT=-XX:-UseContainerSupport 0636036af04d
System properties
Cores       : 2
Memory (Max): 241
</div>

This time JVM reads the configuration from the Docker machine. So these outputs show how the resource limits are correctly handled in Java 10. As mentioned in the [Improvements](#Improvements) section, this version also includes changes in Attach API. To demonstrate this, first, let's install the JDK 10 in the Docker machine.

<div class="console">$ docker-machine ssh default
[TRUNCATED]

docker@default:~$ wget https://download.java.net/java/GA/jdk10/10/binaries/openjdk-10_linux-x64_bin.tar.gz
[TRUNCATED]

docker@default:~$ tar -zxvf openjdk-10_linux-x64_bin.tar.gz
[TRUNCATED]
</div>

As the JDK 10 is now ready, `jstack` command can be tested using the PID which is visible on the host machine.

<div class="console">docker@default:~$ ps -ef | grep DockerTest
root     10294 10279  0 00:47 ?        00:00:00 /bin/sh -c /jdk-10/bin/java ${JAVA_OPT} DockerTest
root     10319 10294  0 00:47 ?        00:00:01 /jdk-10/bin/java DockerTest
docker   11148  9567  0 01:09 pts/0    00:00:00 grep DockerTest

docker@default:~$ sudo ./jdk-10/bin/jstack 10319
2018-04-04 01:09:54
Full thread dump OpenJDK 64-Bit Server VM (10+46 mixed mode):

Threads class SMR info:
_java_thread_list=0x00007ff5c4002680, length=10, elements={
0x00007ff5f4010000, 0x00007ff5f4089000, 0x00007ff5f408b000, 0x00007ff5f40a2000,
0x00007ff5f40a4000, 0x00007ff5f40a6000, 0x00007ff5f40a8000, 0x00007ff5f4131800,
0x00007ff5f413f800, 0x00007ff5c4001000
}

"main" #1 prio=5 os_prio=0 tid=0x00007ff5f4010000 nid=0x6 waiting on condition  [0x00007ff5fa860000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(java.base@10/Native Method)
	at DockerTest.main(DockerTest.java:9)
[TRUNCATED]
</div>

It's important to mention that `10319` is the PID visible on the host machine. For example, below output shows the actual PID inside of the container, which is different as expected (`5`).

<div class="console">$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
556079ca8668        0636036af04d        "/bin/sh -c '/jdk-10…"   37 minutes ago      Up 37 minutes                           confident_euclid

$ docker exec 556079ca8668 /jdk-10/bin/jps
5 DockerTest
26 Jps
</div>

### Conclusion

Even though there're a couple of features added prior to Java 10, the newest Java release is the most container ready version experienced so far. This blog post solely focused on single Docker containers. It would be good to experiment how Java 10 plays under orchestration frameworks as well.

Cheers!
