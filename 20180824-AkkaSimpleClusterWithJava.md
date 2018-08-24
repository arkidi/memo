원문: [https://developer.lightbend.com/guides/akka-sample-cluster-java/](https://developer.lightbend.com/guides/akka-sample-cluster-java/)

This tutorial contains 4 samples illustrating different [Akka cluster](https://doc.akka.io/docs/akka/2.5/cluster-usage.html?language=java) features.

* Subscribe to cluster membership events
* sending messages to actors running on nodes in the cluster
* Cluster aware routers
* Cluster metrics

### A simple Cluster Example

Open application.conf. 
``` bash
akka {
  actor {
    provider = "cluster"
  }
  remote {
    netty.tcp {
      hostname = "127.0.0.1"
      port = 0
    }
    artery {
      enabled = on
      canonical.hostname = "127.0.0.1"
      canonical.port = 0
    }
  }

  cluster {
    seed-nodes = [
      "akka://ClusterSystem@127.0.0.1:2551",
      "akka://ClusterSystem@127.0.0.1:2552"]

    # auto downing is NOT safe for production deployments.
    # you may want to use it during development, read more about it in the docs.
    auto-down-unreachable-after = 10s
  }
}

# Enable metrics extension in akka-cluster-metrics.
akka.extensions=["akka.cluster.metrics.ClusterMetricsExtension"]

# Sigar native library extract location during tests.
# Note: use per-jvm-instance folder when running multiple jvm on one host. 
akka.cluster.metrics.native-library-extract-folder=${user.dir}/target/native
```

To enable cluster capabilities in your Akka project you should, at a minimum, add the remote settings, and use ```cluster``` for ```akka.actor.provider```. The ```akka.cluster.seed-nodes``` should normally also be added to your application.conf file.

The seed nodes are configured contact points whiche newly started nodes will try to connect with in order to join the cluster.

Note that if you are going to start the nodes on different machines you need to specify the ip-address or host names of the machines in ```application.conf``` instead of ```127.0.0.1```.

open SimpleClusterApp.java.
``` java
package sample.cluster.simple;

import akka.actor.ActorSystem;
import akka.actor.Props;

import com.typesafe.config.Config;
import com.typesafe.config.ConfigFactory;

public class SimpleClusterApp {

  public static void main(String[] args) {
    if (args.length == 0)
      startup(new String[] { "2551", "2552", "0" });
    else
      startup(args);
  }

  public static void startup(String[] ports) {
    for (String port : ports) {
      // Override the configuration of the port
      Config config = ConfigFactory.parseString(
          "akka.remote.netty.tcp.port=" + port + "\n" +
          "akka.remote.artery.canonical.port=" + port)
          .withFallback(ConfigFactory.load());

      // Create an Akka system
      ActorSystem system = ActorSystem.create("ClusterSystem", config);

      // Create an actor that handles cluster domain events
      system.actorOf(Props.create(SimpleClusterListener.class),
          "clusterListener");

    }
  }
}
```
the small program together with its configuration starts an ActorSystem with the Cluster enabled.
> 작은 프로그램은 구성과 함께 클러스터가 활성화된 액터 시스템을 시작합니다. 

it joins the cluster and starts an actor that logs some membership events. Take a look at the SimpleClusterListener actor.

> 클러스터에 조인하고 일부 멤버쉽 이벤트를 기록하는 액터를 시작합니다. SimpleClusterListener 액터를 살펴보세요.


```java
package sample.cluster.simple;

import akka.actor.AbstractActor;
import akka.cluster.Cluster;
import akka.cluster.ClusterEvent;
import akka.cluster.ClusterEvent.MemberEvent;
import akka.cluster.ClusterEvent.MemberUp;
import akka.cluster.ClusterEvent.MemberRemoved;
import akka.cluster.ClusterEvent.UnreachableMember;
import akka.event.Logging;
import akka.event.LoggingAdapter;

public class SimpleClusterListener extends AbstractActor {
  LoggingAdapter log = Logging.getLogger(getContext().system(), this);
  Cluster cluster = Cluster.get(getContext().system());

  //subscribe to cluster changes
  @Override
  public void preStart() {
    cluster.subscribe(self(), ClusterEvent.initialStateAsEvents(),
        MemberEvent.class, UnreachableMember.class);
  }

  //re-subscribe when restart
  @Override
  public void postStop() {
    cluster.unsubscribe(self());
  }

  @Override
  public Receive createReceive() {
    return receiveBuilder()
      .match(MemberUp.class, mUp -> {
        log.info("Member is Up: {}", mUp.member());
      })
      .match(UnreachableMember.class, mUnreachable -> {
        log.info("Member detected as unreachable: {}", mUnreachable.member());
      })
      .match(MemberRemoved.class, mRemoved -> {
        log.info("Member is Removed: {}", mRemoved.member());
      })
      .match(MemberEvent.class, message -> {
        // ignore
      })
      .build();
  }
}
```

You can read more about the cluster concepts in the [documentation](https://doc.akka.io/docs/akka/2.5/cluster-usage.html?language=java).

To run this sample, type ```sbt "runMain sample.cluster.simple.SimpleClusterApp"``` if it is not already started. If you are using maven, use ```mvn comple exec:java -Dexec.mainClass="sample.cluster.simple.SimpleClusterApp"``` instead. To pass parameters to maven add ```Dexec.args="..."```.

```SimpleClusterApp``` starts three actor systems (cluster members) in the same JVM process. it can be more interesting to run them in separate processes. Stop the application and then open three timinal windows

in the first terminal window, start the first seed node with the following command:

***sbt***
``` bash
sbt "runMain sample.cluster.simple.SimpleClusterApp 2551"
```

***maven***
``` bash
mvn compile exec:java -Dexec.mainClass="sample.cluster.simple.SimpleClusterApp" -Dexec.args="2251"
```

2551 corresponds to the port of the first seed nodes element in the configuration. In the log output you see that the cluster node has been started and changed status to 'Up'.

In the second terminal window, start the second seed node with the following command:
```bash
sbt "runMain sample.cluster.simple.SimpleClusterApp 2252"
```

2252 corresponds to the port of the second seed-nodes element in the configuration. In the log output you see that the cluster node has been started and joins the other seed node and becomes a member of the cluster. Its status changed to 'Up'.

Switch over to the first terminal window and see in the log output that the member joined.

Start another node in the third terminal window with the following command:
```bash
sbt "runMain sample.cluster.simple.SimpleClusterApp 0"
```

Now you don't need to specify the port number, 0 means that it will use a random available port. It joins one of the configured seed nodes. Look at the log output in the different terminal windows.

Start even more nodes in the same way, if you like.

Shut down one of the nodes by pressing 'ctrl-c' in one of the terminal windows.

The other nodes will detect the failure after a while, which you can see in the log output in the other terminals.

Look at the source code of the actor again. It registers itself as subscriber of certain cluster events.

It gets notified with an snapshot event, ```CurrentClusterState``` that holds full state information of the cluster. After that it receives events for changes that happen in the cluster