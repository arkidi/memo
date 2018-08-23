원문: [https://doc.akka.io/docs/akka/2.5.14/cluster-singleton.html](https://doc.akka.io/docs/akka/2.5.14/cluster-singleton.html)

![Cluster-Singleton-Manager](https://developer.lightbend.com/guides/akka-distributed-workers-scala/images/singleton-manager.png)

### Dependency

To use Cluster Singleton, you must add the following dependency in your project:
> 클러스터 싱글톤을 사용하기 위해서 다음과 같이 프로젝트에 의존성을 추가해야합니다.

Maven
```Maven
<dependency>
  <groupId>com.typesafe.akka</groupId>
  <artifactId>akka-cluster-tools_2.12</artifactId>
  <version>2.5.14</version>
</dependency>
```

### Introduction

For some use cases it is convenient and sometimes also mandatory to ensure that you have exactly one actor of a certain type running somewhere in the cluster.
> 일부 유스 케이스에서 클러스터 어딘가에서 실행중인 특정 유형의 액터가 정확히 하나만 가지는지 확인하는 것은 편리하고 때로는 필수적입니다.

Some examples:
* single point of responsibility for certain cluster-wide consistent decision, or coordination of actions across the cluster system
* single entry point to an external system
* single master, many workers
* centralized naming service, or routing logic

> 1. 특정 클러스터 전체 일관된 결정 또는 클러스터 시스템 전반의 행동을 조정하기 위한 책임의 단일 지점 
> 2. 외부 시스템에게 단일 엔트리 지점
> 3. 하나의 마스터, 다수의 워커
> 4. 중앙 집중식 네이밍 서비스 또는 라우팅 로직

Using a singleton should not be the first design choice. It has several drawbacks, such as single-point of bottleneck. Single-point of failure is also a relevant concern, but for some cases this feature takes care of that by making sure that another singleton instance will eventually be started.
> 싱글톤을 사용하는 것이 첫번째 디자인 선택으로 되어선 안됩니다. 싱글톤은 병목의 단일 지점으로 몇 가지 단점을 가지고 있습니다. 단일장애지점(Single-point of failure)으로 관련된 우려한 사항이지만 일부 케이스에서 이 기능은 다른 싱글톤 인스턴스로 결국 시작되도록 함으로써 처리합니다.

The cluster singleton pattern is implemented by  ```akka.cluster.singleton.ClusterSingletonManager```.
It manages one singleton actor instance among all cluster nodes or a group of nodes tagged with a specific role.
> 클러스터 싱글톤 패턴은 ```akka.cluster.singleton.ClusterSinglnetonManager```에 의해 구현됩니다.
클러스터 싱글톤 패턴은 모든 클러스터 노드들 중에서 하나의 싱글톤 액터 인스턴스 또는 특정 역할을 가지는 태깅된 노드들의 그룹을 관리합니다.

```ClusterSingletonManager``` is an actor that is supposed to be started as early as possible on all nodes, or all nodes with specified role, in the cluster.
> ```ClusterSingletonManager```는 클러스터의 모든 노드 또는 지정된 역할을 가지는 노드에서 가능한 일찍 시작되는 액터입니다.

The actual singleton actor is started by the ```ClusterSingletonManager``` on the oldest node by creating a child actor from supplied ```Props```.
> 실제 싱글턴 액터는 제공되는 ```Props```로 부터 생성하는 자식 액터에 의해 가장 오래된 노드에서 ```ClusterSingletonManager```에 의해 시작됩니다.

```ClusterSingletonManager``` makes sure that at most one singleton instance is running at any point in time.
> ```ClusterSingletonManager```는 최대 한개의 싱글턴 인스턴스가 어느 시점에서 실행하는지 확실하게 합니다.

The singleton actor is always running on the oldest member with specified role.
> 싱글톤 액터는 지정된 역할을 가지는 가장 오래된 멤버에서 항상 실행됩니다.

The oldest member is determined by ```akka.cluster.Member#isOlderThan```.
> 가장 오래된 멤버는 ```akka.cluster.Member#isOlderThan```에 의해 결정됩니다.

This can change when removing the member from the cluster.
> 이는 클러스터에서 멤버가 제거될 때 변경될 수 있습니다.

Be aware that there is a short time period when there is no active singleton during the hand-over process.
> 핸드오버 프로세스 동안 활성 싱글톤이 없는 짧은 시간이 있음을 알아두십시오.

The cluster failure detector will notice when oldest node becomes unreachable due to things like JVM crash, hard shut down, on network failure.
> 클러스터 장애 감지는 가장 오래된 노드가 JVM 충돌, 하드 셧다운, 네트워크 장애등과 같은 이유로 도달할 수 없을 때 경고할 것입니다. 

Then a new oldest node will take over and a new singleton actor is created.
> 그런 다음 새로운 가장 오래된 노드가 인계받아 새로운 싱글톤 액터를 생성할 것입니다.

For these failure scenarios there will not be a graceful hand-over, but more than one active singletons is prevented by all reasonable means.
> 이러한 장애 시나리오는 우아하게 핸드오버를 수행하지 않지만 두 개 이상의 활성 싱글톤을 합리적인 수단으로 방지합니다.

Some corner cases are eventually resolved by configurable timeouts.
> 일부 모서리 케이스는 구성 가능한 타임아웃에 의해 결국 해결됩니다. 

You can access the singleton actor by using the provided ```akka.cluster.singleton.ClusterSingletonProxy```, which will route all messages to the current instance of the singleton.
> 제공된 ```akka.cluster.singleton.ClusterSingletonProxy```를 사용하여 싱글톤 액터에 접근할 수 있습니다. 그리고 그것은 싱글톤의 현재 인스턴스에게 모든 메시지를 라우팅 합니다. 

The proxy will keep track of the oldest node in the cluster and resolve the singleton's ```ActorRef``` by explicitly sending the singleton's ```actorSelection``` the ```akka.actor.Identify``` message and waiting for it to reply.
> 프록시는 클러스터에서 가장 오래된 노드를 추적하고 명시적으로 싱글톤의 ```actorSelection```에게 ```akka.actor.Identify``` 메시지를 전송하고 응답을 기다리는 것으로 싱글톤의 ```ActorRef```를 해결합니다. 

This is performed periodically if the singleton doesn't reply within a certain(configurable) time.
> 이것은(프록시) 싱글톤이 특정 시간 내 응답하지 않는 경우 주기적으로 수행됩니다.

Given the implementation, there might be periods of there during which the ```ActorRef``` is unavailable, e.g., when a node leaves the cluster.
> 구현이 주어지면 ```ActorRef```을 이용할 수 없는 시기가 있을 수 있습니다. 예: 노드가 클러스터에서 떠났을 때

In these cases, the proxy will buffer the messages sent to the singleton and then deliver them when the singleton is finally available.
> 이러한 케이스에서 프록시는 싱글톤에게 보낸 메시지를 버퍼되며 싱글톤이 최종 이용 가능할 때 버퍼된 메시지를 전달됩니다.

If the buffer is full the ```ClusterSingletonProxy``` will drop old messages when new messages are sent via the proxy.
> 만약 버퍼가 가득차면 신규 메시지가 프록시를 통해 전송될 때 ```ClusterSingletonProxy```가 기존 메시지를 삭제합니다. 

The size of the buffer is configurable and it can be disabled by using a buffer size of 0.
> 버퍼의 사이즈는 구성가능하며 버퍼 사이즈를 0으로 하여 비활성화 할 수 있습니다.

It's worth noting that messages can always be lost because of the distributed nature of these actors.
> 이 액터들의 분산 된 특성 때문에 항상 메시지를 잃어 버릴 가치는 없습니다.

As always, additional logic should be implemented in the singleton(acknowledgement) and in the client{retry} actors to ensure at-least-once mesage delivery.
> 싱글톤(acknowledgement)과 클라이언트{재시도} 액터 안에서 적어도 한번(at-least-once) 메시지를 전송하는 것을 보장하는 추가적인 로직을 구현해야합니다.

The singleton instance will not run on members with status [Weaklyup](https://doc.akka.io/docs/akka/2.5.14/cluster-usage.html#weakly-up).
> 싱글톤 인스턴스는 Weaklyup 상태의 멤버에서 실행되지 않습니다.

### Potential problems to be aware of

> 알아야하는 잠재적인 문제 

This pattern may seem to be very tempting to use at first, but it has several drawbacks, some of them are listed below:
> 이 패턴은 처음 사용하기에 매우 유혹적인 것처럼 보일 수 있습니다. 하지만 그것은 아래와 같이 몇가지 단점이 있습니다.

* the cluster singleton may quickly become a perfomance bottleneck,
  > 싱글턴 클러스터는 성능 병목이 빠르게 될 수 있습니다.

* you can not rely on the cluster singleton to be non-stop available - e.g. when the node on which the singleton has been running dies, it will take a few seconds for this to be noticed and the singleton be migrated to another node,
  > 논스톱으로 이용 가능한 싱글턴 클러스터를 기대할 수 없습니다. - 예 : 싱글턴이 실행 중인 노드가 죽었을 때, 싱글턴을 다른 노드로 이관하고, 통지하기 위해 몇 초 걸립니다.

* in the case of a network partition appearing in a Cluster that is using Aotomatic Downing(see docs for [Auto Downing](https://doc.akka.io/docs/akka/2.5.14/cluster-usage.html#automatic-vs-manual-downing)), it may happen that the isolated clusters each decide to spin up their own singleton, meaning that there might be multiple singletons running in the system, yet the Clusters have no way of finding out about them(because of the partition).
  > **Automatic Downing** 을 사용하는 클러스터에서 나타나는 네트워크 파티션의 경우, 격리된 클러스터에서 각각 자신의 싱글턴을 돌리기 위해 결정할 수 있습니다. 즉, 시스템에서 다수의 싱글턴이 실행될지도 모른다는 의미입니다. 그럼에도 불구하고 클러스터는 파티션의 이유에 대해 알아내기 위한 방법이 없습니다. 

### <span style="color:#ef5900;">Warning</span>
<span style="color:#ef5900;">****Don't use Cluster Singleton together with Automatic Downing****</span>, since it allows the cluster to split up into two separate cluster, which in turn will result in multiple singletons being started, one in each separate cluster!
> ****Automatic Downing****과 싱글턴 클러스터를 함께 사용하지 마십시오. 클러스터가 두개의 개별 클러스터로  분리되는 것을 허용하기 때문에 차례대로 개별 클러스터 각각 하나씩 시작되어 여러 개 싱글턴의 결과를 가져올 수 있습니다.


### An Example
Assume that we need one single entry point to external system. An actor that receives messages from a JMS queue with the strict requirement that only one JMS consumer must exist to make sure that the messages are processed in order.
> 외부 시스템에게 단일 엔트리 포인트가 필요하다고 가정합니다. 
메시지를 순서대로 처리하는 오직 하나의 JMS 컨슈머가 존재해야 하는 엄격한 요구사항을 가진 JMS 큐로부터 메시지를 수신 받는 액터가 있습니다. 

That is perhaps not how one would like to design things, but a typical real-world scenario when integrating with external systems.

Before explaining how to create a cluster singleton actor, let's define message classes which will be used by the singleton.

***scala***
```scala
object PointToPontChannel {
  case object UnregistrationOk
}

object Consumer {
  case object End
  case object GetCurrent
  case object Ping
  case object Pong
}
```

***java***
```java
public class TestSingletonMessages {
  public static class UnregistrationOk{}
  public static class End{}
  public static class Ping{}
  public static class Pong{}
  public static class GetCurrent{}

  public static UnregistrationOk unregistrationOk() { return new UnregistrationOk(); }
  public static End end() { return new End(); }
  public static Ping ping() { return new Ping(); }
  public static Pong pong() { return new Pong(); }
  public static GetCurrent getCurrent() { return new GetCurrent(); }
}
```
On each node in the cluster you need to start the ```ClusterSingletonManager``` and supply the ```Props``` of the singleton actor, in this case the JMS queue consumer.

***scala***
```scala
system.actorOf(
  ClusterSingletonManager.props(
    singletonProps = Props(classOf[Consumer], queue, testActor),
    terminationMessage = End,
    settings = ClusterSingletonManagerSettings(system).withRole("worker")),
  name = "consumer")
```

***java***
```java
final ClusterSingletonManagerSettings settings = 
  ClusterSingletonManagerSettings.create(system).withRole("worker");

system.actorOf(
  ClusterSingletonManager.props(
    Props.create(Consumer.class, () -> new Consumer(queue, testActor)),
    TestSingletonMessages.end(),
    settings),
  "consumer"
);
```

With the names given above, access to the singleton can be obtained from any cluster node using a properly configured proxy.
> 위에 주어진 이름으로 싱글톤에 접근하는 것은 적절히 구성된 프록시를 사용한 모든 클러스터의 노드로부터 얻을 수 있습니다. 

***scala***
```scala
val proxy = system.actorOf(
  ClusterSingletonProxy.props(
    singletonManagerPath = "user/consumer",
    setting = ClusterSingletonProxySetting(system).withRole("worker")),
  name = "consumerProxy"
)
```

***java***
```java
ClusterSingletonProxySettings proxySettings =
    ClusterSingletonProxySettings.create(system).withRole("worker");

ActorRef proxy =
  system.actorOf(ClusterSingletonProxy.props("/user/consumer", proxySettings),
    "consumerProxy"
  );
```
A more comprehensive sample is available in the tutorial named [Distributed workers with Akka and java!](https://github.com/typesafehub/activator-akka-distributed-workers-java)

### Configuration
The following configuration properties are read by the ```ClusterSingletonManagerSettings``` when created with a ```ActorSystem``` parameter.

It is also possible to amend the ```ClusterSingleManagerSettings``` or create it from another config selection with the same layout as below.

ClusterSingletonManagerSettings is a parameter to the ClusterSingletonManager.props factory method, i.e. each singleton can be configured with different settings if needed.

```bash
akka.cluster.singleton {
  # The actor name of the child singleton actor.
  singleton-name = "singleton"
  
  # Singleton among the nodes tagged with specified role.
  # If the role is not specified it's a singleton among all nodes in the cluster.
  role = ""
  
  # When a node is becoming oldest it send hand-over request to previous oldest, 
  # that might be leaving the cluster. This is retried with this interval untill
  # the previous oldest confirms that the hand over has started or the previous
  # oldest member is removed from the cluster (+ akka.cluster.down-removal-margin).
  hand-over-retry-interval = 1s
  
  # The number of retries are derived from hand-over-retry-interval and
  # akka.cluster.down-removal-margin (or ClusterSingletonManagerSettings.removalMargin),
  # but it will never be less than this property.
  min-number-of-hand-over-retries = 10
}
```

The following configuration properties are read by the ```ClusterSingletonProxySettings``` when created with a ``` ActorSystem``` parameter. It is also possible to amend the ```ClusterSingletonProxySettings``` or create it from another config section with the same layout as below. ```ClusterSingletonProxySettings``` is parameter to the ```ClusterSingletonProxy.props``` factory method. i.e. each singleton proxy can be configured with different setting if needed.

```bash
akka.cluster.singleton-proxy {
  # The actor name of the singleton actor that is started by the ClusterSingletonManager
  singleton-name = ${akka.cluster.singleton.singleton-name}
  
  # The role of the cluster nodes where the singleton can be deployed.
  # If the role is not specified then any node will do.
  role = ""
 
  # Interval at which the proxy will try to resolve the singleton instance.
  singleton-identification-interval = 1s
  
  # If the location of the singleton is unknown the proxy will buffer this
  # number of messages and deliver them when the singleton is identified. 
  # When the buffer is full old messages will be dropped when new messages are
  # sent via the proxy.
  # Use 0 to disable buffering, i.e. messages will be dropped immediately if
  # the location of the singleton is unknown.
  # Maximum allowed buffer size is 10000.
  buffer-size = 1000 
}
```

