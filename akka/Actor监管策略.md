###Supervisor

Akka中每个Actor都是在其他的Actor中创建，换句话说：每个Actor都存在于它父亲的context中。每个父Actor负责监管它下边的所有子Actor。当一个Actor产生异常行为时，它的父Actor就出马对该如何处置这个Actor做出决策。决策包括以下几种：

- Restart：重新启动出现异常的Actor;

- Stop：停掉出现异常的Actor；

- Resume：忽略异常，保留其当前状态，继续执行；

- Escalate：不知道如何处理，将异常向上抛出，交给自己的父亲执行。



当我们实现一个新Actor，在Actor基类中是有默认的Supervision实现的。默认的实现是：OneForOneStrategy。

```scala

/**

 * When supervisorStrategy is not specified for an actor this

 * is used by default. OneForOneStrategy with decider defined in

 * [[#defaultDecider]].

 */

final val defaultStrategy: SupervisorStrategy = {

  OneForOneStrategy()(defaultDecider)

}





/**

 * When supervisorStrategy is not specified for an actor this

 * `Decider` is used by default in the supervisor strategy.

 * The child will be stopped when [[akka.actor.ActorInitializationException]],

 * [[akka.actor.ActorKilledException]], or [[akka.actor.DeathPactException]] is

 * thrown. It will be restarted for other `Exception` types.

 * The error is escalated if it's a `Throwable`, i.e. `Error`.

 */

final val defaultDecider: Decider = {

  case _: ActorInitializationException ⇒ Stop

  case _: ActorKilledException         ⇒ Stop

  case _: DeathPactException           ⇒ Stop

  case _: Exception                    ⇒ Restart

}

```



如果有额外的需求，也可以自己重新实现。Akka提供了两种策略实现：OneForOneStrategy和AllForOneStrategy。

- OneForOneStrategy：只有当前抛出异常的Actor响应该策略；

- AllForOneStrategy：所有的Actor都响应该策略。

如果自己不满足这两个预设的策略，自己也还可以实现一种定制策略，除非自己确定真的需要才去定制。如何重新实现一种策略也非常的简单，



```scala

override val supervisorStrategy = AllForOneStrategy(maxNrOfRetries = 2, withinTimeRange = 3 second) {

    case _ =>

      SupervisorStrategy.Stop

  }

```



###Death Watch

一个Actor需要监控另外一个Actor的声明周期时，我们可以利用DeathWatch来实现。在Actor Context中我们可以通过ActorContext.watch来监控其他的Actor。与上边的父子关系不同的是，当你监控的Actor停掉之后你会收到一个Terminated消息，来告知被监控的那个消息挂掉了。Actor也可以通过ActorContext.unwatch解除对某个Actor的监管。如果watch一个dead Actor仍会收到Terminated消息。



###Actor的层级

![Akka Top-Lavel Supervison](http://7xjjxu.com1.z0.glb.clouddn.com/guardians.png)

//待补充