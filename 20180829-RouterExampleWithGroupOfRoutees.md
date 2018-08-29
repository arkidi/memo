### Cluster Aware Routers

All [routers](https://doc.akka.io/docs/akka/2.5/routing.html?language=java) can be made aware of member nodes in the cluster, i.e. deploying new routees or looking up routees on nodes in the cluster. When a node becomes unreachable or leaves the cluster routees of that node are automatically unregistered from the router. When new nodes join the cluster additional routees are added to the router, according to the configuration. Routees are also added when a node becomes reachable again, after having been unreachable. 

You can read more about cluster aware routers in the [documentation](https://doc.akka.io/docs/akka/2.5/cluster-usage.html?language=java#Cluster_Aware_Routers).

Let's take a look at a few samples that make use of cluster aware routers.


### Router Example with Group of Routees

Let's take a look at how to use a cluster aware router with a group of routees, i.e. a router which does not create its routees but instead forwards incoming messages to a given set of actors created elsewhere.

The example application provides a service to calculate statistics for a text. 

When some text is sent to the service it splits it into words, and delegates the task to count number of characters in each word to separate worker, a routee of a router.
> 일부 텍스트가 서비스로 전송되면 단어로 분리하고 각 단어별로 문자의 수를 계산하는 작업을 라우터의 라우티에 해당하는 개별 워커에게 위임합니다. 

The character count for each word is sent back to an aggregator that calculates the average number of characters per word when all results have been collected.
> 각 단어의 문자 수는 모든 결과를 수집될 때 단어당 평균 문자의 수를 계산하는 집계로 다시 전송됩니다. 

Open **StatsMessages.java**. it defines the messages that are sent betweeen the actors.

```java
package sample.cluster.stats;

import java.io.Serializable;

public interface StatsMessages {

  public static class StatsJob implements Serializable {
    private final String text;

    public StatsJob(String text) {
      this.text = text;
    }

    public String getText() {
      return text;
    }
  }

  public static class StatsResult implements Serializable {
    private final double meanWordLength;

    public StatsResult(double meanWordLength) {
      this.meanWordLength = meanWordLength;
    }

    public double getMeanWordLength() {
      return meanWordLength;
    }

    @Override
    public String toString() {
      return "meanWordLength: " + meanWordLength;
    }
  }

  public static class JobFailed implements Serializable {
    private final String reason;

    public JobFailed(String reason) {
      this.reason = reason;
    }

    public String getReason() {
      return reason;
    }

    @Override
    public String toString() {
      return "JobFailed(" + reason + ")";
    }
  }

}
```
The worker that counts number of characters in each word is defined in **StatsWorker.java**

```java
package sample.cluster.stats;

import java.util.HashMap;
import java.util.Map;
import akka.actor.AbstractActor;

public class StatsWorker extends AbstractActor {
  Map<String, Integer> cache = new HashMap<String, Integer>();

  @Override
  public Receive createReceive() {
    return receiveBuilder()
      .match(String.class, word -> {
        Integer length = cache.get(word);
        if (length == null) {
          length = word.length();
          cache.put(word, length);
        }
        sender().tell(length, self());
      })
      .build();
  }
}
```
The service that receives text from users and splits it up into words, delegates to workers and aggregates is defined in **StatsService.java** and **StatsAggregator.java**.

*StatsService.java*
```java
package sample.cluster.stats;

import sample.cluster.stats.StatsMessages.StatsJob;
import akka.actor.ActorRef;
import akka.actor.Props;
import akka.actor.AbstractActor;
import akka.routing.ConsistentHashingRouter.ConsistentHashableEnvelope;
import akka.routing.FromConfig;

public class StatsService extends AbstractActor {

  // This router is used both with lookup and deploy of routees. If you
  // have a router with only lookup of routees you can use Props.empty()
  // instead of Props.create(StatsWorker.class).
  ActorRef workerRouter = getContext().actorOf(
      FromConfig.getInstance().props(Props.create(StatsWorker.class)),
      "workerRouter");

  @Override
  public Receive createReceive() {
    return receiveBuilder()
      .match(StatsJob.class, job -> !job.getText().isEmpty(), job -> {
        String[] words = job.getText().split(" ");
        ActorRef replyTo = sender();

        // create actor that collects replies from workers
        ActorRef aggregator = getContext().actorOf(
          Props.create(StatsAggregator.class, words.length, replyTo));

        // send each word to a worker
        for (String word : words) {
          workerRouter.tell(new ConsistentHashableEnvelope(word, word),
            aggregator);
        }
      })
      .build();
  }
}
```

*StatsAggregator.java*
```java
package sample.cluster.stats;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

import sample.cluster.stats.StatsMessages.JobFailed;
import sample.cluster.stats.StatsMessages.StatsResult;
import scala.concurrent.duration.Duration;
import akka.actor.ActorRef;
import akka.actor.ReceiveTimeout;
import akka.actor.AbstractActor;

public class StatsAggregator extends AbstractActor {

  final int expectedResults;
  final ActorRef replyTo;
  final List<Integer> results = new ArrayList<>();

  public StatsAggregator(int expectedResults, ActorRef replyTo) {
    this.expectedResults = expectedResults;
    this.replyTo = replyTo;
  }

  @Override
  public void preStart() {
    getContext().setReceiveTimeout(Duration.create(3, TimeUnit.SECONDS));
  }

  @Override
  public Receive createReceive() {
    return receiveBuilder()
      .match(Integer.class, wordCount -> {
        results.add(wordCount);
        if (results.size() == expectedResults) {
          int sum = 0;
          for (int c : results)
            sum += c;
          double meanWordLength = ((double) sum) / results.size();
          replyTo.tell(new StatsResult(meanWordLength), self());
          getContext().stop(self());
        }
      })
      .match(ReceiveTimeout.class, x -> {
        replyTo.tell(new JobFailed("Service unavailable, try again later"),
          self());
        getContext().stop(self());
      })
      .build();
  }

}
```

Note, nothing cluster specific so far, just plain actors.

All nodes start ```StatsService``` and ```StatsWorker```actors. Remember, routees are the workers in this case.

Open **stat1.conf**. The router is configured with ```routees.paths```. This means that user requests can be sent to ```StatsService``` on any node and it will use ```StatsWorker``` on all nodes.

```bash
include "application"

akka.actor.deployment {
  /statsService/workerRouter {
    router = consistent-hashing-group
    routees.paths = ["/user/statsWorker"]
    cluster {
      enabled = on
      allow-local-routees = on
      use-role = compute
    }
  }
}
```

To run this sample, type ```sbt "runMain sample.cluster.stats.StatsSampleMain"``` if it is not already started.
StatsSampleMain starts 4 actor systems (cluster members) in the same JVM process. It can be more interesting to run them in separate processes. Stop the application and run the following commands in separate terminal windows.

```bash
sbt "runMain sample.cluster.stats.StatsSampleMain 2551"
sbt "runMain sample.cluster.stats.StatsSampleMain 2552"

sbt "runMain sample.cluster.stats.StatsSampleClientMain"

sbt "runMain sample.cluster.stats.StatsSampleMain 0"
```

![StatsSampleMain](https://github.com/arkidi/memo/blob/master/img/20180829-Akka-RouterExample-StatsSampleMain.png?raw=true)
