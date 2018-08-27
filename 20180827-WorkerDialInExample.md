In the previous sample we saw how to subscribe to cluster membership events.
You can read more about it in the [documentation](https://doc.akka.io/docs/akka/2.5/cluster-usage.html?language=java#Subscribe_to_Cluster_Events). How can cluster membership events be used?
> 이전 샘플 예제에서 클러스터 멤버쉽 이벤트를 어떻게 구독하는지에 대해 살펴보았습니다. 보다 자세한 사항은 문서를 참조하세요. 클러스터 멤버쉽 이벤트를 어떤 방법으로 사용할 수 있을까요?

Let's take a look at an example that illustrates how workers, here named *backend*, can detect and register to new master nodes, here named *frontend*.
> 백엔드라고 하는 워커가 프론트 엔드라고 하는 신규 마스터 노드를 어떻게 감지하고 등록하는 방법에 대해 설명하는 예제를 살펴보겠습니다.

The example application provides a service to transform text. When some text is sent to one of the frontend services, It will be delegated to one of the backend workers, which performs the transformation job and sends the result back to the original client.
> 예제 애플리케이션은 텍스트 변환(소문자를 대문자로 변환)을 서비스로 제공합니다. 일부 텍스트가 프론트엔드 서비스 중 하나로 전송할 때 백엔드 워커들 중 하나가 위임되어 트랜스포메이션 작업과 결과를 원래 클라이언트로 전송하는 것을 수행합니다. 

New backend nodes, as well as new frontend nodes, can be added or removed to the cluster dynamically.
> 신규 프론트엔드 노드 뿐만 아니라 신규 백엔드 노드들을 동적으로 클러스터에 추가 또는 제거할 수 있습니다. 

Open **TransformationMessages.java**. It defines the messages that are sent between the actors.
> TransformationMessages.java 파일을 열어보세요. 액터들간 메시지 전송에 대해 정의하고 있습니다.

```java
package sample.cluster.transformation;

import java.io.Serializable;

public interface TransformationMessages {

  public static class TransformationJob implements Serializable {
    private final String text;

    public TransformationJob(String text) {
      this.text = text;
    }

    public String getText() {
      return text;
    }
  }

  public static class TransformationResult implements Serializable {
    private final String text;

    public TransformationResult(String text) {
      this.text = text;
    }

    public String getText() {
      return text;
    }

    @Override
    public String toString() {
      return "TransformationResult(" + text + ")";
    }
  }

  public static class JobFailed implements Serializable {
    private final String reason;
    private final TransformationJob job;

    public JobFailed(String reason, TransformationJob job) {
      this.reason = reason;
      this.job = job;
    }

    public String getReason() {
      return reason;
    }

    public TransformationJob getJob() {
      return job;
    }

    @Override
    public String toString() {
      return "JobFailed(" + reason + ")";
    }
  }

  public static final String BACKEND_REGISTRATION = "BackendRegistration";

}
```

The backend worker that preforms the transformation job is defined in **TransformationBackend.java**
> TransformationBackend.java에 정의된 트랜스포메이션을 수행하는 백엔드 워커 소스 코드입니다.

```java
package sample.cluster.transformation;

import static sample.cluster.transformation.TransformationMessages.BACKEND_REGISTRATION;
import sample.cluster.transformation.TransformationMessages.TransformationJob;
import sample.cluster.transformation.TransformationMessages.TransformationResult;
import akka.actor.AbstractActor;
import akka.cluster.Cluster;
import akka.cluster.ClusterEvent.CurrentClusterState;
import akka.cluster.ClusterEvent.MemberUp;
import akka.cluster.Member;
import akka.cluster.MemberStatus;

public class TransformationBackend extends AbstractActor {

  Cluster cluster = Cluster.get(getContext().system());

  //subscribe to cluster changes, MemberUp
  @Override
  public void preStart() {
    cluster.subscribe(self(), MemberUp.class);
  }

  //re-subscribe when restart
  @Override
  public void postStop() {
    cluster.unsubscribe(self());
  }

  @Override
  public Receive createReceive() {
    return receiveBuilder()
      .match(TransformationJob.class, job -> {
        sender().tell(new TransformationResult(job.getText().toUpperCase()),
          self());
      })
      .match(CurrentClusterState.class, state -> {
        for (Member member : state.getMembers()) {
          if (member.status().equals(MemberStatus.up())) {
            register(member);
          }
        }
      })
      .match(MemberUp.class, mUp -> {
        register(mUp.member());
      })
      .build();
  }

  void register(Member member) {
    if (member.hasRole("frontend"))
      getContext().actorSelection(member.address() + "/user/frontend").tell(
          BACKEND_REGISTRATION, self());
  }
}
```

Note that the ```TransformationBackend``` actor subscribes to cluster events to detect new, potential, frontend nodes, and send them a registration message so that they know that they can use the backend worker.

> TransformationBackend 액터는 새로운, 잠재적인 프론트 엔드 노드들의 클러스터 이벤트를 구독하며, 백엔드 워커를 사용할 수 있음을 알수 있도록 등록 메시지를 프론트 엔드 노드에게 전송합니다.


The frontend that receives user jobs and delegates to one of the registered backend workers is defined in **TransformationFrontend.java**

> 등록된 백엔드 워커들 중 하나에게 사용자의 작업과 위임을 받는 프론트엔드는 TransformationFrontend.java에 정의됩니다. 

``` java
package sample.cluster.transformation;

import static sample.cluster.transformation.TransformationMessages.BACKEND_REGISTRATION;

import java.util.ArrayList;
import java.util.List;

import sample.cluster.transformation.TransformationMessages.JobFailed;
import sample.cluster.transformation.TransformationMessages.TransformationJob;
import akka.actor.ActorRef;
import akka.actor.Terminated;
import akka.actor.AbstractActor;

public class TransformationFrontend extends AbstractActor {

  List<ActorRef> backends = new ArrayList<>();
  int jobCounter = 0;

  @Override
  public Receive createReceive() {
    return receiveBuilder()
      .match(TransformationJob.class, job -> backends.isEmpty(), job -> {
        sender().tell(new JobFailed("Service unavailable, try again later", job),
          sender());
      })
      .match(TransformationJob.class, job -> {
        jobCounter++;
        backends.get(jobCounter % backends.size())
          .forward(job, getContext());
      })
      .matchEquals(BACKEND_REGISTRATION, message -> {
        getContext().watch(sender());
        backends.add(sender());
      })
      .match(Terminated.class, terminated -> {
        backends.remove(terminated.getActor());
      })
      .build();
  }
}
```

Note that the **TransformationFrontend** actor watch the registered backend to be able to remove it from its list of available backend workers. Death watch uses the cluster failure detector for nodes in the cluster, i.e. it detects network failures and JVM crashes, in addition to graceful termination of watched actor.

> **TransformationFrontend**  액터는 등록 된 백앤드가 이용 가능한 백앤드 워커들의 리스트로부터 제거할 수 있는지 감시합니다. Death Watch는 클러스터 내 노드들을 위해 클러스터 장애 감지기를 사용합니다. 예를 들어 네트워크 장애와 JVM 크래쉬, 추가적으로 감시하고 있는 액터의 우아한 종료를 감지합니다.

To run this sample, type ```sbt "runMain sample.cluster.transformation.TransformationApp"``` if it is not aleady started.
> 아직 시작하지 않은 경우 예제를 실행하기 위해 입력하세요. ```sbt "runMain sample.cluster.transformation.TransformationApp"```

TransformationApp starts 5 actor systems(cluster members) in the same JVM process. It can be more interesting to run in separate processes. Stop the application and run following commands in separate terminal windows.

> TransformationApp은 같은 JVM 프로세스에서 5개 액터 시스템(클러스터 멤버)을 시작합니다. 별도의 프로세스에서 실행하는 것이 보다 흥미로울 수 있습니다. 애플리케이션을 중지하고 별도의 터미널창에서 다음 명령으로 실행하십시오. 

```bash
sbt "runMain sample.cluster.transformation.TransformationFrontendMain 2551"

sbt "runMain sample.cluster.transformation.TransformationBackendMain 2552"
sbt "runMain sample.cluster.transformation.TransformationBackendMain 0"
sbt "runMain sample.cluster.transformation.TransformationBackendMain 0"

sbt "runMain sample.cluster.transformation.TransformationFrontendMain 0"
```

**TransformationFrontendMain.java**

```java  
package sample.cluster.transformation;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import com.typesafe.config.Config;
import com.typesafe.config.ConfigFactory;

import sample.cluster.transformation.TransformationMessages.TransformationJob;
import scala.concurrent.ExecutionContext;
import scala.concurrent.duration.Duration;
import scala.concurrent.duration.FiniteDuration;
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;
import akka.dispatch.OnSuccess;
import akka.util.Timeout;
import static akka.pattern.Patterns.ask;

public class TransformationFrontendMain {

  public static void main(String[] args) {
    // Override the configuration of the port when specified as program argument
    final String port = args.length > 0 ? args[0] : "0";
    final Config config = 
      ConfigFactory.parseString(
          "akka.remote.netty.tcp.port=" + port + "\n" +
          "akka.remote.artery.canonical.port=" + port)
      .withFallback(ConfigFactory.parseString("akka.cluster.roles = [frontend]"))
      .withFallback(ConfigFactory.load());

    ActorSystem system = ActorSystem.create("ClusterSystem", config);

    final ActorRef frontend = system.actorOf(
        Props.create(TransformationFrontend.class), "frontend");

    final FiniteDuration interval = Duration.create(2, TimeUnit.SECONDS);

    final Timeout timeout = new Timeout(Duration.create(5, TimeUnit.SECONDS));

    final ExecutionContext ec = system.dispatcher();

    final AtomicInteger counter = new AtomicInteger();

    system.scheduler().schedule(interval, interval, new Runnable() {
      public void run() {
        ask(frontend,
            new TransformationJob("hello-" + counter.incrementAndGet()),
            timeout).onSuccess(new OnSuccess<Object>() {
          public void onSuccess(Object result) {
            System.out.println(result);
          }
        }, ec);
      }

    }, ec);

  }
}

```