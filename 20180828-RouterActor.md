A router can also be created as a self contained actor that manages the routees itself and loads routing logic and other settings from configuration.

This type of router actor comes in two distinct flavors:

* Pool - The router creates routees as child acotrs and removes them from the router if they terminate.
* Group - The routee actors are created externally to the router and the router sends messages to the specified path using actor selection, without watching for termination.

The setting for a router actor can be defined in configuration or programmatically.
In other to make an actor to make use of an externally configurable router the ```FromConfig``` props wrapper must be used to denote that te actor accepts routing settings from configuration.

This is in constrast with Remote Deployment where such maker props is not necessary.

If the props of an actor is NOT wrapped IN ```FromConfig``` it will ignore the router section of the deployment configuration.

*Example*
```java
public class StatsService extends AbstractActor {

  // This router is used both with lookup and deploy of routees. If you
  // have a router with only lookup of routees you can use Props.empty()
  // instead of Props.create(StatsWorker.class).
  ActorRef workerRouter = getContext().actorOf(
      FromConfig.getInstance().props(Props.create(StatsWorker.class)),
      "workerRouter");

```
You send messages to the routees via the router in the same way as for ordinary actors, i.e. via its ```ActorRef```. The router actor forwards messages onto its routees without changing the original sender. When a routee replies to a routed message, the reply will be sent to the original sender, not to the router actor.

In general, any message sent to a router will be sent onwards to its routees, but there are a few exceptions. These are documented in the [Specially Handled Messages](https://doc.akka.io/docs/akka/current/routing.html#router-special-messages) section below.


### Pool
---
The following code and configuration snippets show how to create a [round-robin](https://doc.akka.io/docs/akka/current/routing.html#round-robin-router) router that forwards messages to five ```Worker``` routees. The routees will be created as the router's children.

```bash
  akka.actor.deployment {
    /parent/router1 {
      router = round-robin-pool
      nr-of-instances = 5
    }
  }
```

```java
ActorRef router1 = 
  getContext().actor(FromConfig.getInstance().props(Props.create(Worker.class)),
   "router1");
```

Here is the sample example, but with the router configuration provided programmatically instead of from configuration.

```java
ActorRef router2 = 
  getContext().actorOf(new RoundRobinPool(5).props(Props.create(Worker.class)),
   "router2");
```

### Remote Deployed Routees
---
In addition to being able to create local actors as routees, you can instruct the router to deploy its created children on a set of remote hosts. Routees will be deployed in round-robin fashion. In order to deploy routees remotely, wrap the router configuration in a ```RemoteRouterConfig```, attaching the remote addresses of the nodes to deploy to. Remote deployment requires the ```akka-remote``` module to be included in the classpath.

> 로컬 액터를 라우티로 생성할 수 있을 뿐만 아니라, 라우터에게 생성 된 하위를 원격 호스트 집합에 배포하도록 지시 할 수 있습니다. 라우티는 라운드-로빈 방식으로 배포될 것입니다. 원격으로 라우티를 배포하기 위해 ```RemoteRouterConfig```에 라우터 설정을 감싸고, 배포하기 위해 노드들의 원격 주소를 첨부하십시오. 원격 배포에서 ```akka-remote``` 모듈을 클래스 패스에 포함시켜야 합니다. 


```java
Address[] addresses = {
  new Address("akka.tcp", "remotesys", "otherhost", 1234),
  AddressFromURIString.parse("akka.tcp://othersys@anotherhost:1234")
};

ActorRef routerRemote = system.actorOf(
  new RemoteRouterConfig(new RoundRobinPool(5), address).props(
    Props.create(Echo.class)));
```

### Senders
---
By default, when a routee sends a message, it will [implicitly set itself as the sender](https://doc.akka.io/docs/akka/current/actors.html#actors-tell-sender).
```java
getSender().tell("reply", getSelf());
```

However, it is often useful for routees to set the *router* as a sender. For example, you might want to set the router as the sender if you want to hide te details of the routees behind the router. The following code snippet shows how to set the parent router as sender.

```java
getSender().tell("reply", getContext().getParent());
```

### Supervision
---
Routees that are created by a pool router will be created as the router's childern. The router it therefore also the children's supervisor.
> 라우터 풀에 의해 성성되는 라우티는 라우터의 자식들처럼 생성됩니다. 라우터는 자식들의 슈퍼바이져입니다. 

The supervision strategy of the router actor can be configurated with the ```supervisiorStrategy``` property of the Pool. If no configuration is provided, routers default to a strategy of "always escalate". This means that errors are passed up to the router's supervisor for handling. The router's supervisor will decide what to do about any errors.

> 라우터 액터의 슈퍼바이져 전략은 ```supervisorStrategy``` 프로퍼티로 설정됩니다. 만약 설정하지 않으면 "always escalate" 라우터 기본 전략으로 제공됩니다. 이것은 라우터의 슈퍼바이저 핸들링으로  에러를 전달하는 의미입니다. 라우터의 슈퍼바이저는 어떤 에러에 대해 무엇을 수행할지 결정합니다.

Note the router's supervisor will treat the error as an error with the router itself.
> 라우터의 슈퍼바이져는 라우터 자체 에러 처럼 취급합니다.

Therefore a directive to stop or restart will cause the router *itself* to stop or restart. 
The router, in turn, will cause its children to stop and restart.
> 따라서 중지 및 재시작 지시자는 라우터 자체를 중지하거나 재시작할 것입니다. 라우터는 차례대로 자식들을 중지하고 재시작할 것입니다.

It should be mentioned that the router's restart behavior has been overriden so that a restart, while still re-creating the children, will still preseve the same number of actors in the pool.
> 라우터의 재시작 행동은 재정의 되어 있습니다. 그래서 재시작은 자식을 재생성하는 동안 풀에서 동일한 액터의 수를 유지합니다.

This means that if you have not specified ```superviorStrategy``` of the router or its parent a failure in a routee will escalate to the parent of the router, which will by default restart the router, which will restart all routees(it uses Escalate and does not stop routees during restart).
> 라우터에 ```supervisorStrategy```가 지정되지 않거나 라우티에서 부모가 실패할 경우 라우터의 부모에게 확대될 것입니다. 그리고 기본적으로 라우터가 재시작되고 모든 라우티(Esclate를 사용하고 재시작 동안 라우티는 중지되지 않습니다.)가 재시작 될것입니다.

The reason is to make the default behave such that adding ```.withRouter``` to a child's definition does not change the supervision strategy applied to the child.
> 그 이유는 자식 정의에 ```.withRouter```를 추가해도 자식에게 적용된 슈퍼바이저 전략이 변경되지 않도록 기본 동작을 만들기 때문입니다. 

> 

This might be an inefficiency that you can avoid by specifying the strategy when defining the router.
> 라우터를 정의할 때 전략을 지정함으로써 회피할 수 있기에 비효율적일수 있습니다.


Setting the strategy is done like this:

```java
final SupervisorStrategy strategy = 
  new OneForOnStrategy(5, Duration.ofMinutes(1),
    Collections.<Class<? extends Throwable>>singletonList(Exception.class));
    
final ActorRef router = system.actorOf(new RoundRobinPool(5).withSupervisorStrategy(strategy).props(Props.create(Echo.class)));
```

Note
If the child of a pool router terminates, the pool router will not automatically spawn a new child.
In the event that all children of a pool router have terminated the router will terminate itself unless it is a dynamic router. e.g. using a resizer.