# Setting up Actors and scheduling asynchronous tasks

[Akka](http://akka.io) 使用 Actor 模型来提高抽象级别，它提供了一个更好的平台来构建正确的、并发的和可扩展的应用。在错误容忍度（fault-tolerance）上，它采用的是 'Let it crash' 模型，该模型被成功地应用于电信业，用于构建永不停歇的自愈系统。Actor 提供了对透明分布的抽象，以及构建真正可扩展和高错误容忍度应用的基础。

## 应用的 actor 系统

Akka 可以和一些叫做 `ActorSystem` 的容器一起工作。Actor 系统管理着它所配置的资源，这样它可以运行它所包含的 actor。

一个 Play 应用会定义一个特殊的 actor 系统来给该应用使用，这个 actor 系统贯穿了整个应用的生命周期，并且在应用重启时它会自动重启。

> 注意：在 Play 应用中，你也可以使用另外一个 actor 系统。默认提供的 actor 系统，只是在你想启动少量 actor 而又不想麻烦地去配置你自己的 actor 系统时，提供一些方便。

你可以通过 `play.api.libs.concurrent.Akka` 中的 helper 方法来使用默认的 actor 系统：

```scala
val myActor = Akka.system.actorOf(Props[MyActor], name = "myactor")
```

## 配置

默认 actor 系统的配置是从 Play 应用的配置文件中读取出来的。例如，想为应用的 actor 系统配置默认的调度器，需要在配置文件 `conf/application.conf` 中加入以下两行：

```scala
akka.default-dispatcher.fork-join-executor.pool-size-max =64
akka.actor.debug.receive = on
```

> 注意：你可以在相同的文件中配置其它任意的 actor 系统，记得提供一个顶层的配置键即可。

## 调度异步任务

你可以定时发送消息给 actor 并执行相应的任务（函数或 `Runnable`），然后你会收到一个 `Cancellable`，你可以调用 `cancel` 来取消相应操作的执行。

例如，每 300 毫秒发送一条消息给 `testActor`：

```scala
import play.api.libs.concurrent.Execution.Implicits._
Akka.system.scheduler.schedule(0.microsecond, 300.microsecond, testActor, "tick")
```

> 注意：上面的例子使用了定义在 `scala.concurrent.duration` 中的隐式转换，来将数字转换成不同时间单位的 `Duration` 对象。

相似地，下面的例子展示的是 1 秒后执行一个代码块：

```scala
import play.api.libs.concurrent.Execution.Implicits._
Akka.system.scheduler.scheduleOnce(1000.microsecond) {
  file.delete()
}
```
